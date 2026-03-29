# 编辑锁心跳续锁方案设计

## 一、问题背景

### 业务场景

- 页面点击编辑按钮后调用接口**加锁**，防止多人同时编辑
- 退出编辑时调用接口**解锁**
- 编辑期间前端每 **20 秒**发送一次心跳续锁请求
- 后端设定超过 **60 秒**未收到心跳自动解锁

### 原方案

轮询 + Web Worker（Web Worker 仅作为定时器，续锁请求仍在**主线程**发送）

### 遇到的问题

编辑状态下将页面最小化或切到其他 Tab/应用，十几分钟后回来发现页面已自动解锁（tester 电脑未休眠/锁屏）。

---

## 二、问题根因分析

### 浏览器后台标签页节流策略（Background Tab Throttling）

现代浏览器（尤其 Chrome）对后台标签页有**多层级节流**：

| 后台时长      | 限制                                                           |
| ------------- | -------------------------------------------------------------- |
| 立即          | `setTimeout` / `setInterval` 最低间隔提升至 **1 秒**           |
| **~5 分钟后** | **Intensive Throttling** — 定时器唤醒限制为**每分钟最多 1 次** |
| **更长时间**  | 标签页可能被**完全冻结**（Tab Freezing），所有 JS 停止执行     |

### 原方案的两个关键缺陷

1. **Web Worker 定时器并非完全不受节流**：Chrome 87+ 的 Intensive Throttling 也会影响后台标签页中 Web Worker 的定时器，5 分钟后 Worker 的 `setInterval` 也可能被降频到每分钟一次。

2. **主线程通信瓶颈**：即使 Worker 定时器按时触发，它通过 `postMessage` 通知主线程去发请求。但主线程在后台被节流/冻结时，**消息会排队但不被处理**，导致实际请求严重延迟甚至不发送。

### 失败时间线

```
0 ~ 5 分钟:  心跳基本正常（20 秒间隔）
5 ~ 10 分钟: 心跳变为约 60 秒间隔（Intensive Throttling）
10+ 分钟:    心跳可能完全停止（Tab Freezing）
→ 超过 60 秒无心跳 → 后端自动解锁
```

---

## 三、Web Worker 发送心跳请求是否仍会被节流/冻结？

### 结论：仍有可能，但概率大幅降低

将 `fetch` 移入 Web Worker 内部后，**网络请求本身不会被节流**——一旦 `fetch` 被调用，浏览器会正常完成网络请求。但 Worker 中的**定时器（`setInterval`）仍可能被节流**，只是浏览器对 Worker 的限制比主线程宽松。

### 各时间段风险评估

| 后台时长     | 风险等级 | 说明                                                                                                                           |
| ------------ | -------- | ------------------------------------------------------------------------------------------------------------------------------ |
| 0 ~ 5 分钟   | ✅ 极低  | Worker 定时器基本正常，心跳可靠                                                                                                |
| 5 ~ 15 分钟  | ⚠️ 低~中 | Intensive Throttling 可能影响 Worker 定时器间隔（从 20s 降为约 60s），但 fetch 被调用后仍能正常发出。配合 60s 超时**勉强可用** |
| 15 ~ 30 分钟 | ⚠️ 中    | 定时器降频更明显，Tab Freezing 风险增加（取决于系统内存压力和浏览器策略）                                                      |
| 30+ 分钟     | 🔴 较高  | Tab Freezing 概率显著上升，整个 Worker 可能被冻结，所有 JS 停止执行                                                            |

### 关键区分

- **定时器节流**：Worker 的 `setInterval` 可能被降频 → 心跳间隔变长 → 但只要间隔 < 后端超时时间，锁就不会丢
- **Tab Freezing**：整个页面（包括 Worker）被冻结 → JS 完全停止 → **没有任何前端方案能解决这个问题**

### Tab Freezing 的触发条件（Chrome）

- 页面处于后台超过 5 分钟
- 没有活跃的连接（WebSocket、WebRTC、Web Transport 等）
- 没有使用某些 API（如正在播放音频、持有 Web Lock 等）
- 系统内存压力较大时更容易触发
- 不同 Chrome 版本策略不同，且在持续收紧

### 概率总结

改用 Worker 内 fetch 后，**能解决 90%+ 的常规场景**（用户切换 Tab 几分钟到十几分钟的情况）。对于极端长时间后台（30 分钟以上），仍需 `visibilitychange` 兜底。

---

## 四、推荐方案：Worker 内 fetch + visibilitychange 兜底

### 整体架构

```
┌─────────────────────────────────────────────┐
│                  主线程                       │
│                                             │
│  编辑页面组件                                 │
│    ├─ 点击编辑 → 加锁接口 → 启动 Worker       │
│    ├─ 监听 Worker 消息（心跳结果/token请求）   │
│    ├─ 监听 visibilitychange（兜底补发）        │
│    └─ 退出编辑 → 解锁接口 → 终止 Worker       │
│                                             │
│  Token 管理（主线程独占）                      │
│    ├─ 获取当前 token                          │
│    └─ 刷新 token（refresh token 逻辑）        │
└──────────────┬──────────────────────────────┘
               │ postMessage
               ▼
┌─────────────────────────────────────────────┐
│              Web Worker                      │
│                                             │
│  ├─ 持有 token（主线程传入）                   │
│  ├─ setInterval 定时发 fetch 心跳             │
│  ├─ 遇到 401 → 向主线程请求新 token            │
│  └─ 收到新 token → 更新本地 token，重试请求     │
└─────────────────────────────────────────────┘
```

### Token 传递机制

**核心原则**：token 的获取和刷新始终由主线程负责，Worker 只持有一份 token 副本用于发请求。

**原因**：refresh token 逻辑通常涉及全局状态（localStorage / cookie、拦截器队列、并发刷新去重），放主线程统一管理更安全，避免主线程和 Worker 同时刷新导致竞态。

### Token 传递流程

```
主线程:  acquireLock()
          ├─ getToken() → token_A
          ├─ POST /lock → lockId
          ├─ new Worker()
          └─ postMessage({ type:'start', token: token_A, lockId })

Worker:   收到 start → 保存 token_A → 开始定时心跳
          ├─ fetch(Authorization: token_A) → 200 OK ✓
          ├─ fetch(Authorization: token_A) → 200 OK ✓
          ├─ fetch(Authorization: token_A) → 401 ✗ Token 过期!
          │   └─ postMessage({ type:'token-expired' })

主线程:   收到 token-expired
          ├─ refreshToken() → token_B
          └─ postMessage({ type:'update-token', token: token_B })

Worker:   收到 update-token → currentToken = token_B
          └─ 立即重试 fetch(Authorization: token_B) → 200 OK ✓
```

---

## 五、代码实现

### 5.1 Web Worker — `heartbeat.worker.js`

```js
// heartbeat.worker.js
let config = null
let timer = null
let currentToken = null
let pendingRetry = null

/**
 * 发送心跳请求
 */
async function sendHeartbeat() {
  if (!config || !currentToken) return

  try {
    const res = await fetch(config.url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${currentToken}`,
      },
      body: JSON.stringify({ lockId: config.lockId }),
    })

    if (res.status === 401) {
      // Token 过期，向主线程请求新 token
      pendingRetry = true
      self.postMessage({ type: 'token-expired' })
      return
    }

    if (!res.ok) {
      self.postMessage({
        type: 'heartbeat-fail',
        status: res.status,
        message: `HTTP ${res.status}`,
      })
      return
    }

    const data = await res.json()
    self.postMessage({ type: 'heartbeat-ok', data, timestamp: Date.now() })
  } catch (err) {
    self.postMessage({
      type: 'heartbeat-fail',
      message: err.message,
      timestamp: Date.now(),
    })
  }
}

function startHeartbeat() {
  stopHeartbeat()
  sendHeartbeat() // 立即发一次
  timer = setInterval(sendHeartbeat, config.interval)
}

function stopHeartbeat() {
  if (timer) {
    clearInterval(timer)
    timer = null
  }
  pendingRetry = null
}

self.onmessage = function (e) {
  const { type } = e.data

  switch (type) {
    case 'start':
      config = {
        url: e.data.url,
        lockId: e.data.lockId,
        interval: e.data.interval || 20000,
      }
      currentToken = e.data.token
      startHeartbeat()
      break

    case 'stop':
      stopHeartbeat()
      self.close()
      break

    case 'update-token':
      currentToken = e.data.token
      if (pendingRetry) {
        pendingRetry = null
        sendHeartbeat()
      }
      break
  }
}
```

### 5.2 React Hook — `useEditLock.js`

```js
// useEditLock.js
import { useRef, useCallback, useEffect } from 'react'
import { getToken, refreshToken } from '@/utils/auth'

const HEARTBEAT_INTERVAL = 20_000 // 20 秒
const LOCK_EXPIRE_THRESHOLD = 55_000 // 55 秒视为可能过期

export function useEditLock({ lockApiUrl, heartbeatApiUrl, unlockApiUrl }) {
  const workerRef = useRef(null)
  const lastHeartbeatRef = useRef(0)
  const lockIdRef = useRef(null)
  const isLockedRef = useRef(false)

  /**
   * 处理 Worker 消息
   */
  const handleWorkerMessage = useCallback(async e => {
    const { type } = e.data

    switch (type) {
      case 'heartbeat-ok':
        lastHeartbeatRef.current = e.data.timestamp
        break

      case 'heartbeat-fail':
        console.warn('[Heartbeat] 心跳失败:', e.data.message)
        break

      case 'token-expired':
        try {
          const newToken = await refreshToken()
          workerRef.current?.postMessage({
            type: 'update-token',
            token: newToken,
          })
        } catch (err) {
          console.error('[Heartbeat] Token 刷新失败:', err)
          // 可能需要提示用户重新登录
        }
        break
    }
  }, [])

  /**
   * 页面可见性变化 — 兜底逻辑
   */
  const handleVisibilityChange = useCallback(async () => {
    if (document.visibilityState !== 'visible') return
    if (!isLockedRef.current || !lockIdRef.current) return

    const elapsed = Date.now() - lastHeartbeatRef.current

    if (elapsed > LOCK_EXPIRE_THRESHOLD) {
      // 锁可能已过期，需要业务层处理
      console.warn(`[Heartbeat] 距上次心跳已过 ${Math.round(elapsed / 1000)}s，锁可能已失效`)
      // 根据业务需要：尝试重新加锁 / 提示用户
    }

    // 推送最新 token 给 Worker（后台期间 token 可能已过期）
    try {
      const token = await getToken()
      workerRef.current?.postMessage({ type: 'update-token', token })
    } catch {
      // ignore
    }
  }, [])

  /**
   * 加锁 + 启动心跳
   */
  const acquireLock = useCallback(
    async resourceId => {
      const token = await getToken()

      const res = await fetch(lockApiUrl, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${token}`,
        },
        body: JSON.stringify({ resourceId }),
      })

      if (!res.ok) throw new Error('加锁失败')

      const { lockId } = await res.json()
      lockIdRef.current = lockId
      isLockedRef.current = true
      lastHeartbeatRef.current = Date.now()

      // 启动 Worker
      const worker = new Worker(new URL('./heartbeat.worker.js', import.meta.url), {
        type: 'module',
      })
      worker.onmessage = handleWorkerMessage
      worker.postMessage({
        type: 'start',
        url: heartbeatApiUrl,
        lockId,
        token,
        interval: HEARTBEAT_INTERVAL,
      })
      workerRef.current = worker

      document.addEventListener('visibilitychange', handleVisibilityChange)
      return lockId
    },
    [lockApiUrl, heartbeatApiUrl, handleWorkerMessage, handleVisibilityChange]
  )

  /**
   * 解锁 + 停止心跳
   */
  const releaseLock = useCallback(async () => {
    if (workerRef.current) {
      workerRef.current.postMessage({ type: 'stop' })
      workerRef.current = null
    }

    document.removeEventListener('visibilitychange', handleVisibilityChange)

    if (lockIdRef.current) {
      try {
        const token = await getToken()
        await fetch(unlockApiUrl, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${token}`,
          },
          body: JSON.stringify({ lockId: lockIdRef.current }),
        })
      } catch (err) {
        console.error('[Lock] 解锁请求失败:', err)
      }
    }

    lockIdRef.current = null
    isLockedRef.current = false
    lastHeartbeatRef.current = 0
  }, [unlockApiUrl, handleVisibilityChange])

  // 组件卸载时自动清理
  useEffect(() => {
    return () => {
      if (isLockedRef.current) releaseLock()
    }
  }, [releaseLock])

  return { acquireLock, releaseLock }
}
```

### 5.3 业务组件使用示例

```jsx
function EditPage({ resourceId }) {
  const { acquireLock, releaseLock } = useEditLock({
    lockApiUrl: '/api/lock',
    heartbeatApiUrl: '/api/lock/heartbeat',
    unlockApiUrl: '/api/lock/unlock',
  })

  const handleStartEdit = async () => {
    try {
      await acquireLock(resourceId)
      // 进入编辑模式
    } catch (err) {
      message.error('加锁失败，可能有其他人正在编辑')
    }
  }

  const handleStopEdit = async () => {
    await releaseLock()
    // 退出编辑模式
  }

  return (
    <div>
      <button onClick={handleStartEdit}>编辑</button>
      <button onClick={handleStopEdit}>退出编辑</button>
    </div>
  )
}
```

---

## 六、方案要点总结

| 问题                           | 解决方案                                                                        |
| ------------------------------ | ------------------------------------------------------------------------------- |
| 心跳请求被主线程节流           | fetch 移入 Web Worker 内部执行                                                  |
| Token 怎么传给 Worker          | `postMessage` 在 `start` 时传入初始 token                                       |
| Token 过期怎么办               | Worker 检测 401 → 通知主线程 → 主线程 `refreshToken()` → 推送新 token 回 Worker |
| 回到前台锁可能已过期           | `visibilitychange` 兜底检查 + 立即推送最新 token                                |
| Token 刷新为什么不放 Worker 里 | 避免与主线程同时刷新的竞态；refresh 逻辑依赖全局状态                            |
| 极端长时间后台 Tab 冻结        | 前端无法完全避免，通过 `visibilitychange` 回到前台时检测并提示用户              |

## 七、补充建议

1. **调整后端超时时间**：建议从 60 秒调到 90~120 秒，增加容错余量
2. **心跳间隔可适当缩短**：如改为 10~15 秒，在定时器被降频时仍有更大概率在超时前成功发送
3. **验证方法**：在 Worker 中加入时间戳日志，切到后台等待不同时长后回来观察实际心跳间隔
