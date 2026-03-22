# 前端编辑锁方案设计文档

## 一、业务背景

项目中有多个页面支持"编辑模式"，用户点击 Edit 按钮进入编辑状态时需对页面资源上锁，防止多人同时编辑产生冲突。退出编辑模式时释放锁。为防止用户异常退出（关闭浏览器、断网等）导致死锁，后端设计了心跳续锁机制，前端需要每 20 秒调用一次续锁接口。

## 二、核心问题：浏览器后台标签页定时器节流

### 2.1 问题描述

直接使用 `setInterval` 做 20 秒轮询**在后台标签页中不可靠**。现代浏览器会对非活跃标签页的定时器进行节流（throttle）：

| 浏览器     | 节流策略                                                                             |
| ---------- | ------------------------------------------------------------------------------------ |
| Chrome 57+ | 后台 tab 的 `setInterval` / `setTimeout` 最多每 **1 分钟**执行一次                   |
| Chrome 88+ | "Intensive Throttling"：后台超过 5 分钟未交互后，定时器可能延迟到每 **5 分钟**才执行 |
| Firefox    | 后台 tab 定时器至少 **1 秒**间隔，长时间后台后进一步节流                             |
| Safari     | 更激进，后台 tab 可能直接**挂起**定时器                                              |

结果：20 秒一次的续锁轮询在后台可能变成 1~5 分钟才执行一次，锁会因为续期不及时而过期，其他人就能抢到锁。

### 2.2 电脑休眠 / 合盖场景

当电脑休眠时，**所有 JS 执行都会暂停**，包括 Web Worker。唤醒后需要立即检测并补续锁。

### 2.3 Web Worker 在各场景下的真实表现

Web Worker **并非万能**，不同场景下有本质差异：

| 场景                           | Worker 定时器是否正常 | 说明                                                                                                                    |
| ------------------------------ | --------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **切到其他 Tab**               | ✅ 正常               | Worker 线程不受 tab 级别节流影响                                                                                        |
| **窗口最小化**                 | ✅ 正常               | 与切 tab 同理，Worker 仍在独立线程运行                                                                                  |
| **长时间停留在编辑模式不操作** | ✅ 正常               | 浏览器节流针对的是「非活跃 tab」而不是「无用户交互」，即使 30 分钟没碰键盘鼠标，Worker 照常运行                         |
| **操作系统锁屏（未休眠）**     | ⚠️ 通常正常           | 锁屏 ≠ 休眠，大多数情况下进程仍在运行，取决于 OS 电源策略                                                               |
| **电脑休眠 / 合盖 / 睡眠**     | ❌ 完全暂停           | **所有 JS 执行都会冻结**，包括 Worker，这是操作系统级别的进程挂起                                                       |
| **macOS App Nap**              | ⚠️ 基本不影响         | macOS 会对"不可见且无音频"的应用降低资源分配，但 Chrome 已针对此做了处理，有活跃 Worker 或网络请求时不会被 App Nap 影响 |
| **移动端浏览器切后台**         | ❌ 大概率暂停         | iOS Safari 和 Android Chrome 会冻结后台页面的所有执行                                                                   |

> **⚠️ 重要澄清："长时间不操作"指的是不操作浏览器页面，而不是不操作电脑。**
>
> | 具体情况                                                     | Worker 是否正常 | 原因                                                  |
> | ------------------------------------------------------------ | --------------- | ----------------------------------------------------- |
> | 不操作页面，但在用其他软件（写文档、聊天等）                 | ✅ 正常         | 浏览器进程还在跑，Worker 不受影响                     |
> | 不操作页面，页面挂在前台，人在看屏幕不动                     | ✅ 正常         | 浏览器没有"用户不活跃就冻结 Worker"的机制             |
> | 不操作电脑，人离开了，但电脑没休眠（屏幕亮着或设置了不休眠） | ✅ 正常         | CPU 还在跑，一切正常                                  |
> | 不操作电脑，人离开了，电脑自动进入休眠/睡眠                  | ❌ 暂停         | OS 冻结了进程，回到休眠场景由 `visibilitychange` 兜底 |
>
> 关键点：**浏览器的定时器节流机制是基于"tab 是否在前台"来决定的，不是基于"用户是否在交互"**。所以哪怕用户 2 小时没碰页面，只要电脑没休眠，Worker 就正常 tick。唯一需要担心的是：用户长时间离开 → 电脑因电源策略自动休眠 → 回到休眠场景。

关键区分：

- **锁屏但没休眠**（如 macOS 设置了「锁屏后不睡眠」或接着电源）→ Worker **正常运行**
- **锁屏且进入休眠/睡眠** → Worker **完全冻结**，OS 级别挂起进程，唤醒后才恢复
- 判断标准：看 CPU 是否还在跑。锁屏只是 UI 层面的遮挡，休眠是 OS 冻结进程

**结论**：Web Worker 能稳定覆盖日常 90%+ 的场景（切 tab、最小化、不操作、锁屏未休眠），唯一无法解决的是 OS 级别的进程冻结（休眠/合盖/移动端切后台）。

### 2.4 visibilitychange 补偿机制的真实价值

在「20 秒心跳 / 60 秒锁超时」的配置下，`visibilitychange` 的价值需要客观看待：

| 休眠时长                        | 锁状态                     | visibilitychange 的作用                         |
| ------------------------------- | -------------------------- | ----------------------------------------------- |
| **< 60 秒**（如短暂合盖又打开） | 锁还没过期                 | ✅ 补续锁成功，真正救回了锁                     |
| **≥ 60 秒**（绝大多数实际场景） | 锁已过期，可能已被他人获取 | ❌ 续锁失败，**但能立即检测到锁丢失并通知用户** |

实际中用户合盖/休眠大概率远超 60 秒，所以 `visibilitychange` 的**核心价值不是"续上锁"，而是"唤醒后第一时间告诉用户锁已经丢了"**，避免用户以为自己还在编辑，结果保存时才发现冲突。

工作流程：

```
休眠前              休眠期间                    唤醒后
  │                   │                          │
Worker 正常 tick   全部冻结（0 执行）         visibilitychange 触发
续锁正常发送       续锁停止                        ↓
  │                锁可能已过期              立即尝试补续锁
  │                   │                     ↓           ↓
  │                   │                  成功         失败(锁已过期/被抢)
  │                   │                  继续编辑     触发 onLockLost
  │                   │                               通知用户锁已丢失
```

因此 `visibilitychange` 在本方案中承担的是**「锁丢失检测」**的角色，而非「续锁救命」的角色。这一点在实现中已经体现 —— 续锁失败会触发 `onLockLost` 回调通知用户。

## 三、技术方案

### 3.1 技术栈

- React + TypeScript
- SWR（锁状态查询）
- Axios（HTTP 请求）
- Web Worker（后台心跳定时器）

### 3.2 架构总览

```
src/
├── workers/
│   └── heartbeat.worker.ts       # Web Worker 心跳定时器（不受浏览器节流）
├── api/
│   └── lock.ts                   # 锁相关 API（上锁/解锁/续锁/查询）
├── hooks/
│   └── useEditLock.ts            # 核心 Hook（封装全部锁逻辑）
└── pages/
    └── XxxPage.tsx               # 业务页面（调用 Hook 即可）
```

### 3.3 时序图

```
用户点击编辑 ──→ acquireLock() ──→ 成功 ──→ 启动 Worker
                                              │
                              ┌────────────── ↓ ──────────────┐
                              │  Worker 每 20s 发出 tick       │
                              │       ↓                        │
                              │  renewLock() 续锁              │
                              │  失败次数 >= 阈值 → onLockLost │
                              └────────────────────────────────┘
                                              │
用户点击保存/取消 ──→ releaseLock() ──→ 停止 Worker
用户关闭页面     ──→ sendBeacon 解锁（best effort）
用户切回页面     ──→ visibilitychange → 补续锁
组件卸载(路由跳转) ──→ releaseLock() + 清理 Worker
```

## 四、实现细节

### 4.1 Web Worker — 心跳定时器

**核心思路**：Web Worker 中的定时器**不受浏览器后台标签页节流影响**，能保证在用户切走 tab 或最小化窗口后，续锁请求仍按预期间隔发出。

```ts
// src/workers/heartbeat.worker.ts

let timer: ReturnType<typeof setInterval> | null = null

self.onmessage = (e: MessageEvent<{ type: 'start' | 'stop'; interval?: number }>) => {
  const { type, interval } = e.data

  if (type === 'start') {
    if (timer) clearInterval(timer)
    timer = setInterval(() => {
      self.postMessage({ type: 'tick' })
    }, interval ?? 20_000)
  }

  if (type === 'stop') {
    if (timer) {
      clearInterval(timer)
      timer = null
    }
  }
}
```

**引入方式（Vite）**：

```ts
const worker = new Worker(new URL('../workers/heartbeat.worker.ts', import.meta.url), {
  type: 'module',
})
```

> 如果是 CRA / Webpack 项目，需要使用 `worker-loader` 或者将文件放在 `public/` 目录下通过路径引入。

### 4.2 API 层

```ts
// src/api/lock.ts
import axios from 'axios' // 替换为项目中已封装的 axios 实例

export interface LockInfo {
  lockId: string
  lockedBy: string
  lockedAt: string
  expiresAt: string
}

/** 上锁 */
export function acquireLock(resourceId: string) {
  return axios.post<LockInfo>(`/api/resources/${resourceId}/lock`)
}

/** 解锁 */
export function releaseLock(resourceId: string) {
  return axios.delete(`/api/resources/${resourceId}/lock`)
}

/** 续锁 */
export function renewLock(resourceId: string) {
  return axios.put<LockInfo>(`/api/resources/${resourceId}/lock/renew`)
}

/** 查询锁状态（SWR fetcher） */
export async function fetchLockStatus(resourceId: string): Promise<LockInfo | null> {
  const { data } = await axios.get<LockInfo | null>(`/api/resources/${resourceId}/lock`)
  return data
}

/**
 * sendBeacon 解锁 —— 用于页面关闭时的 best-effort 解锁
 * sendBeacon 在页面卸载时比 XMLHttpRequest / fetch 更可靠
 */
export function releaseLockBeacon(resourceId: string) {
  const url = `/api/resources/${resourceId}/lock`
  navigator.sendBeacon(url + '/release', JSON.stringify({ _method: 'DELETE' }))
}
```

### 4.3 核心 Hook — `useEditLock`

这是整套方案的核心，封装了上锁、解锁、续锁、异常处理的全部逻辑，业务页面只需调用即可。

```ts
// src/hooks/useEditLock.ts
import { useCallback, useEffect, useRef, useState } from 'react'
import useSWR from 'swr'
import {
  acquireLock,
  releaseLock,
  renewLock,
  fetchLockStatus,
  releaseLockBeacon,
  type LockInfo,
} from '@/api/lock'

// ======================== 类型定义 ========================

interface UseEditLockOptions {
  /** 资源 ID */
  resourceId: string
  /** 心跳间隔（ms），默认 20000 */
  heartbeatInterval?: number
  /** 连续续锁失败多少次后判定锁丢失，默认 2 */
  maxFailCount?: number
  /** 续锁失败回调 */
  onLockLost?: () => void
  /** 锁被他人持有时的回调 */
  onLockConflict?: (holder: string) => void
}

interface UseEditLockReturn {
  /** 当前是否处于编辑模式 */
  isEditing: boolean
  /** 是否正在获取/释放锁 */
  isLoading: boolean
  /** 当前锁信息（SWR 查询） */
  lockInfo: LockInfo | null | undefined
  /** 进入编辑模式 */
  enterEdit: () => Promise<boolean>
  /** 退出编辑模式 */
  exitEdit: () => Promise<void>
  /** 续锁失败次数 */
  failCount: number
}

// ======================== Hook 实现 ========================

export function useEditLock(options: UseEditLockOptions): UseEditLockReturn {
  const {
    resourceId,
    heartbeatInterval = 20_000,
    maxFailCount = 2,
    onLockLost,
    onLockConflict,
  } = options

  const [isEditing, setIsEditing] = useState(false)
  const [isLoading, setIsLoading] = useState(false)
  const [failCount, setFailCount] = useState(0)

  const workerRef = useRef<Worker | null>(null)
  const lastRenewTimeRef = useRef<number>(0)
  const isEditingRef = useRef(false) // 用于事件回调中读取最新值
  const resourceIdRef = useRef(resourceId)
  resourceIdRef.current = resourceId

  // ==================== SWR：锁状态查询 ====================
  // 非编辑模式下用于展示 "当前正在被 xxx 编辑"
  // 编辑模式下由 Worker 接管心跳，SWR 停止轮询
  const { data: lockInfo, mutate: mutateLockInfo } = useSWR(
    resourceId ? `lock-status-${resourceId}` : null,
    () => fetchLockStatus(resourceId),
    {
      refreshInterval: isEditing ? 0 : 30_000,
      revalidateOnFocus: true,
    }
  )

  // ==================== 续锁逻辑 ====================
  const doRenew = useCallback(async () => {
    if (!isEditingRef.current) return

    try {
      await renewLock(resourceIdRef.current)
      lastRenewTimeRef.current = Date.now()
      setFailCount(0)
    } catch (err) {
      setFailCount(prev => {
        const next = prev + 1
        if (next >= maxFailCount) {
          onLockLost?.()
          cleanupWorker()
          setIsEditing(false)
          isEditingRef.current = false
        }
        return next
      })
    }
  }, [maxFailCount, onLockLost])

  // ==================== Worker 管理 ====================
  const startWorker = useCallback(() => {
    const worker = new Worker(new URL('../workers/heartbeat.worker.ts', import.meta.url), {
      type: 'module',
    })

    worker.onmessage = (e: MessageEvent<{ type: string }>) => {
      if (e.data.type === 'tick') {
        doRenew()
      }
    }

    worker.postMessage({ type: 'start', interval: heartbeatInterval })
    workerRef.current = worker
  }, [heartbeatInterval, doRenew])

  const cleanupWorker = useCallback(() => {
    if (workerRef.current) {
      workerRef.current.postMessage({ type: 'stop' })
      workerRef.current.terminate()
      workerRef.current = null
    }
  }, [])

  // ==================== 进入编辑模式 ====================
  const enterEdit = useCallback(async (): Promise<boolean> => {
    if (isEditingRef.current) return true
    setIsLoading(true)

    try {
      await acquireLock(resourceId)
      lastRenewTimeRef.current = Date.now()
      setIsEditing(true)
      isEditingRef.current = true
      setFailCount(0)
      startWorker()
      mutateLockInfo()
      return true
    } catch (err: any) {
      if (err?.response?.status === 409) {
        const holder = err.response.data?.lockedBy ?? 'unknown'
        onLockConflict?.(holder)
      }
      return false
    } finally {
      setIsLoading(false)
    }
  }, [resourceId, startWorker, mutateLockInfo, onLockConflict])

  // ==================== 退出编辑模式 ====================
  const exitEdit = useCallback(async () => {
    cleanupWorker()
    setIsEditing(false)
    isEditingRef.current = false
    setFailCount(0)

    try {
      await releaseLock(resourceId)
    } catch {
      // 解锁失败允许静默，服务端会自动过期
    } finally {
      mutateLockInfo()
    }
  }, [resourceId, cleanupWorker, mutateLockInfo])

  // ================== visibilitychange ==================
  // 电脑休眠唤醒 / 切回 tab 后立即补续锁
  useEffect(() => {
    const handleVisibility = () => {
      if (document.visibilityState === 'visible' && isEditingRef.current) {
        const elapsed = Date.now() - lastRenewTimeRef.current
        if (elapsed >= heartbeatInterval) {
          doRenew()
        }
      }
    }

    document.addEventListener('visibilitychange', handleVisibility)
    return () => document.removeEventListener('visibilitychange', handleVisibility)
  }, [heartbeatInterval, doRenew])

  // ==================== beforeunload ====================
  // 页面关闭时尽力解锁
  useEffect(() => {
    const handleUnload = () => {
      if (isEditingRef.current) {
        releaseLockBeacon(resourceIdRef.current)
      }
    }

    window.addEventListener('beforeunload', handleUnload)
    return () => window.removeEventListener('beforeunload', handleUnload)
  }, [])

  // ==================== 组件卸载清理 ====================
  // 路由跳转等场景
  useEffect(() => {
    return () => {
      if (isEditingRef.current) {
        cleanupWorker()
        releaseLock(resourceIdRef.current).catch(() => {})
        isEditingRef.current = false
      }
    }
  }, [cleanupWorker])

  return {
    isEditing,
    isLoading,
    lockInfo,
    enterEdit,
    exitEdit,
    failCount,
  }
}
```

### 4.4 业务页面使用示例

```tsx
// src/pages/EditPage.tsx
import { useEditLock } from '@/hooks/useEditLock'
import { message, Modal } from 'antd'

function EditPage({ resourceId }: { resourceId: string }) {
  const { isEditing, isLoading, lockInfo, enterEdit, exitEdit, failCount } = useEditLock({
    resourceId,
    heartbeatInterval: 20_000,
    maxFailCount: 2,
    onLockLost: () => {
      Modal.error({
        title: '编辑锁已丢失',
        content: '由于网络问题或超时，您的编辑锁已失效。请保存好当前内容后重新进入编辑模式。',
      })
    },
    onLockConflict: holder => {
      message.warning(`当前页面正在被 ${holder} 编辑，请稍后再试`)
    },
  })

  const handleEdit = async () => {
    const success = await enterEdit()
    if (success) {
      message.success('已进入编辑模式')
    }
  }

  const handleSave = async () => {
    // await saveData(); // 先保存业务数据
    await exitEdit()
    message.success('保存成功')
  }

  const handleCancel = async () => {
    await exitEdit()
  }

  return (
    <div>
      {/* 非编辑模式下展示锁状态 */}
      {!isEditing && lockInfo && (
        <div className='lock-banner'>当前正在被 {lockInfo.lockedBy} 编辑中</div>
      )}

      {/* 续锁失败警告 */}
      {isEditing && failCount > 0 && (
        <div className='warning-banner'>⚠️ 网络异常，续锁失败 ({failCount} 次)，请检查网络连接</div>
      )}

      {!isEditing ? (
        <button onClick={handleEdit} disabled={isLoading || !!lockInfo}>
          {isLoading ? '获取锁中...' : '编辑'}
        </button>
      ) : (
        <>
          <div className='edit-area'>{/* 编辑区域 */}</div>
          <button onClick={handleSave}>保存</button>
          <button onClick={handleCancel}>取消</button>
        </>
      )}
    </div>
  )
}
```

两个页面复用只需传入不同的 `resourceId`：

```tsx
<PageA resourceId="page-a-123" />
<PageB resourceId="page-b-456" />
```

## 五、边界场景处理

| 场景                              | 风险                                          | 处理方式                                                                                | 处理层              |
| --------------------------------- | --------------------------------------------- | --------------------------------------------------------------------------------------- | ------------------- |
| **用户切到其他 tab**              | `setInterval` 被浏览器节流至 1~5 分钟，锁过期 | Web Worker 定时器不受节流影响                                                           | Worker              |
| **窗口最小化**                    | 同上                                          | Web Worker 正常运行，无影响                                                             | Worker              |
| **长时间不操作（挂机）**          | 无风险                                        | 浏览器节流针对的是非活跃 tab，不是无交互，Worker 正常运行                               | Worker              |
| **操作系统锁屏（未休眠）**        | 低风险                                        | Worker 通常正常运行，取决于 OS 电源策略                                                 | Worker              |
| **电脑休眠 / 合盖**               | 所有 JS（含 Worker）暂停，锁可能过期          | 唤醒后 `visibilitychange` 立即尝试续锁；若超过 60s 锁已过期则触发 `onLockLost` 通知用户 | visibilitychange    |
| **macOS App Nap**                 | 低风险                                        | Chrome 已做处理，有活跃 Worker/网络请求时不受影响                                       | Worker              |
| **移动端切后台**                  | 页面可能被完全冻结                            | 同休眠处理：回到前台时 `visibilitychange` 检测锁状态                                    | visibilitychange    |
| **用户直接关闭浏览器 / tab**      | `beforeunload` 中异步请求不可靠               | `sendBeacon` 发送解锁（best effort）+ 服务端锁超时自动释放                              | sendBeacon + 服务端 |
| **网络断开**                      | 续锁请求失败                                  | 连续失败计数，超过阈值触发 `onLockLost` 通知用户                                        | 失败计数            |
| **路由跳转（组件卸载）**          | 锁未释放                                      | `useEffect` cleanup 中主动解锁 + 清理 Worker                                            | useEffect cleanup   |
| **同一用户两个 tab 打开同一页面** | 可能重复上锁                                  | 服务端返回 409 Conflict，前端走 `onLockConflict` 回调提示                               | 服务端 409          |
| **续锁失败但网络恢复**            | 锁可能已被他人获取                            | 失败计数未达阈值时继续重试；达阈值后通知用户，避免静默覆盖                              | 失败计数            |

## 六、对后端的配合要求

1. **锁超时时间建议设为 60 秒**（3 倍于心跳间隔 20 秒），给网络抖动留出容错空间
2. **必须实现锁自动过期释放**，不能完全依赖前端主动解锁
3. **上锁接口在锁被占用时返回 409 Conflict**，响应体中包含 `lockedBy` 字段
4. **续锁接口应为幂等操作**，重复调用不产生副作用
5. **提供 POST 方式的解锁端点**（或支持 `_method=DELETE`），以兼容 `sendBeacon`（`sendBeacon` 只能发 POST）

## 七、三层防御体系总结

本方案采用三层防御，各层分工明确：

| 层级       | 机制                 | 解决的问题                  | 覆盖场景                                                             |
| ---------- | -------------------- | --------------------------- | -------------------------------------------------------------------- |
| **第一层** | Web Worker 心跳      | 浏览器后台标签页定时器节流  | 切 tab、最小化、不操作、锁屏未休眠（90%+ 场景）                      |
| **第二层** | visibilitychange     | OS 级进程冻结后的锁状态检测 | 休眠唤醒、移动端切后台（核心价值是检测锁丢失并通知用户，而非续上锁） |
| **第三层** | 服务端锁超时自动释放 | 前端所有手段均失效的兜底    | 关闭浏览器、断网、崩溃等一切极端场景                                 |

## 八、方案优势总结

- **可靠性**：Web Worker 心跳不受浏览器后台节流影响，是保证续锁时效的关键
- **健壮性**：覆盖了 tab 切换、窗口最小化、长时间不操作、锁屏、休眠唤醒、页面关闭、路由跳转、网络异常等全部边界场景
- **复用性**：封装为 `useEditLock` Hook，不同页面传入 `resourceId` 即可复用
- **用户体验**：续锁失败有实时提示，锁丢失有弹窗告知，锁被占用有明确反馈
- **务实的设计认知**：明确 `visibilitychange` 的核心价值是「锁丢失检测」而非「续锁救命」，不过度承诺方案能力
- **渐进降级**：即使前端全部手段失效，服务端锁超时机制仍能保证不会产生死锁
