# 前端编辑锁方案设计文档

## 一、业务背景

项目中有多个页面支持"编辑模式"，用户点击 Edit 按钮进入编辑状态时需对页面资源上锁，防止多人同时编辑产生冲突。退出编辑模式时释放锁。为防止用户异常退出（关闭浏览器、断网等）导致死锁，后端设计了心跳续锁机制，前端需要每 20 秒调用一次续锁接口。

## 二、核心问题：浏览器后台标签页定时器节流

### 2.1 问题描述

直接使用 `setInterval` 做 20 秒轮询**在后台标签页中不可靠**。现代浏览器会对非活跃标签页的定时器进行节流（throttle）：

| 浏览器 | 节流策略 |
|---|---|
| Chrome 57+ | 后台 tab 的 `setInterval` / `setTimeout` 最多每 **1 分钟**执行一次 |
| Chrome 88+ | "Intensive Throttling"：后台超过 5 分钟未交互后，定时器可能延迟到每 **5 分钟**才执行 |
| Firefox | 后台 tab 定时器至少 **1 秒**间隔，长时间后台后进一步节流 |
| Safari | 更激进，后台 tab 可能直接**挂起**定时器 |

结果：20 秒一次的续锁轮询在后台可能变成 1~5 分钟才执行一次，锁会因为续期不及时而过期，其他人就能抢到锁。

### 2.2 电脑休眠 / 合盖场景

当电脑休眠时，**所有 JS 执行都会暂停**，包括 Web Worker。唤醒后需要立即检测并补续锁。

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

let timer: ReturnType<typeof setInterval> | null = null;

self.onmessage = (e: MessageEvent<{ type: 'start' | 'stop'; interval?: number }>) => {
  const { type, interval } = e.data;

  if (type === 'start') {
    if (timer) clearInterval(timer);
    timer = setInterval(() => {
      self.postMessage({ type: 'tick' });
    }, interval ?? 20_000);
  }

  if (type === 'stop') {
    if (timer) {
      clearInterval(timer);
      timer = null;
    }
  }
};
```

**引入方式（Vite）**：

```ts
const worker = new Worker(
  new URL('../workers/heartbeat.worker.ts', import.meta.url),
  { type: 'module' },
);
```

> 如果是 CRA / Webpack 项目，需要使用 `worker-loader` 或者将文件放在 `public/` 目录下通过路径引入。

### 4.2 API 层

```ts
// src/api/lock.ts
import axios from 'axios'; // 替换为项目中已封装的 axios 实例

export interface LockInfo {
  lockId: string;
  lockedBy: string;
  lockedAt: string;
  expiresAt: string;
}

/** 上锁 */
export function acquireLock(resourceId: string) {
  return axios.post<LockInfo>(`/api/resources/${resourceId}/lock`);
}

/** 解锁 */
export function releaseLock(resourceId: string) {
  return axios.delete(`/api/resources/${resourceId}/lock`);
}

/** 续锁 */
export function renewLock(resourceId: string) {
  return axios.put<LockInfo>(`/api/resources/${resourceId}/lock/renew`);
}

/** 查询锁状态（SWR fetcher） */
export async function fetchLockStatus(resourceId: string): Promise<LockInfo | null> {
  const { data } = await axios.get<LockInfo | null>(`/api/resources/${resourceId}/lock`);
  return data;
}

/**
 * sendBeacon 解锁 —— 用于页面关闭时的 best-effort 解锁
 * sendBeacon 在页面卸载时比 XMLHttpRequest / fetch 更可靠
 */
export function releaseLockBeacon(resourceId: string) {
  const url = `/api/resources/${resourceId}/lock`;
  navigator.sendBeacon(url + '/release', JSON.stringify({ _method: 'DELETE' }));
}
```

### 4.3 核心 Hook — `useEditLock`

这是整套方案的核心，封装了上锁、解锁、续锁、异常处理的全部逻辑，业务页面只需调用即可。

```ts
// src/hooks/useEditLock.ts
import { useCallback, useEffect, useRef, useState } from 'react';
import useSWR from 'swr';
import {
  acquireLock,
  releaseLock,
  renewLock,
  fetchLockStatus,
  releaseLockBeacon,
  type LockInfo,
} from '@/api/lock';

// ======================== 类型定义 ========================

interface UseEditLockOptions {
  /** 资源 ID */
  resourceId: string;
  /** 心跳间隔（ms），默认 20000 */
  heartbeatInterval?: number;
  /** 连续续锁失败多少次后判定锁丢失，默认 2 */
  maxFailCount?: number;
  /** 续锁失败回调 */
  onLockLost?: () => void;
  /** 锁被他人持有时的回调 */
  onLockConflict?: (holder: string) => void;
}

interface UseEditLockReturn {
  /** 当前是否处于编辑模式 */
  isEditing: boolean;
  /** 是否正在获取/释放锁 */
  isLoading: boolean;
  /** 当前锁信息（SWR 查询） */
  lockInfo: LockInfo | null | undefined;
  /** 进入编辑模式 */
  enterEdit: () => Promise<boolean>;
  /** 退出编辑模式 */
  exitEdit: () => Promise<void>;
  /** 续锁失败次数 */
  failCount: number;
}

// ======================== Hook 实现 ========================

export function useEditLock(options: UseEditLockOptions): UseEditLockReturn {
  const {
    resourceId,
    heartbeatInterval = 20_000,
    maxFailCount = 2,
    onLockLost,
    onLockConflict,
  } = options;

  const [isEditing, setIsEditing] = useState(false);
  const [isLoading, setIsLoading] = useState(false);
  const [failCount, setFailCount] = useState(0);

  const workerRef = useRef<Worker | null>(null);
  const lastRenewTimeRef = useRef<number>(0);
  const isEditingRef = useRef(false); // 用于事件回调中读取最新值
  const resourceIdRef = useRef(resourceId);
  resourceIdRef.current = resourceId;

  // ==================== SWR：锁状态查询 ====================
  // 非编辑模式下用于展示 "当前正在被 xxx 编辑"
  // 编辑模式下由 Worker 接管心跳，SWR 停止轮询
  const { data: lockInfo, mutate: mutateLockInfo } = useSWR(
    resourceId ? `lock-status-${resourceId}` : null,
    () => fetchLockStatus(resourceId),
    {
      refreshInterval: isEditing ? 0 : 30_000,
      revalidateOnFocus: true,
    },
  );

  // ==================== 续锁逻辑 ====================
  const doRenew = useCallback(async () => {
    if (!isEditingRef.current) return;

    try {
      await renewLock(resourceIdRef.current);
      lastRenewTimeRef.current = Date.now();
      setFailCount(0);
    } catch (err) {
      setFailCount((prev) => {
        const next = prev + 1;
        if (next >= maxFailCount) {
          onLockLost?.();
          cleanupWorker();
          setIsEditing(false);
          isEditingRef.current = false;
        }
        return next;
      });
    }
  }, [maxFailCount, onLockLost]);

  // ==================== Worker 管理 ====================
  const startWorker = useCallback(() => {
    const worker = new Worker(
      new URL('../workers/heartbeat.worker.ts', import.meta.url),
      { type: 'module' },
    );

    worker.onmessage = (e: MessageEvent<{ type: string }>) => {
      if (e.data.type === 'tick') {
        doRenew();
      }
    };

    worker.postMessage({ type: 'start', interval: heartbeatInterval });
    workerRef.current = worker;
  }, [heartbeatInterval, doRenew]);

  const cleanupWorker = useCallback(() => {
    if (workerRef.current) {
      workerRef.current.postMessage({ type: 'stop' });
      workerRef.current.terminate();
      workerRef.current = null;
    }
  }, []);

  // ==================== 进入编辑模式 ====================
  const enterEdit = useCallback(async (): Promise<boolean> => {
    if (isEditingRef.current) return true;
    setIsLoading(true);

    try {
      await acquireLock(resourceId);
      lastRenewTimeRef.current = Date.now();
      setIsEditing(true);
      isEditingRef.current = true;
      setFailCount(0);
      startWorker();
      mutateLockInfo();
      return true;
    } catch (err: any) {
      if (err?.response?.status === 409) {
        const holder = err.response.data?.lockedBy ?? 'unknown';
        onLockConflict?.(holder);
      }
      return false;
    } finally {
      setIsLoading(false);
    }
  }, [resourceId, startWorker, mutateLockInfo, onLockConflict]);

  // ==================== 退出编辑模式 ====================
  const exitEdit = useCallback(async () => {
    cleanupWorker();
    setIsEditing(false);
    isEditingRef.current = false;
    setFailCount(0);

    try {
      await releaseLock(resourceId);
    } catch {
      // 解锁失败允许静默，服务端会自动过期
    } finally {
      mutateLockInfo();
    }
  }, [resourceId, cleanupWorker, mutateLockInfo]);

  // ================== visibilitychange ==================
  // 电脑休眠唤醒 / 切回 tab 后立即补续锁
  useEffect(() => {
    const handleVisibility = () => {
      if (document.visibilityState === 'visible' && isEditingRef.current) {
        const elapsed = Date.now() - lastRenewTimeRef.current;
        if (elapsed >= heartbeatInterval) {
          doRenew();
        }
      }
    };

    document.addEventListener('visibilitychange', handleVisibility);
    return () => document.removeEventListener('visibilitychange', handleVisibility);
  }, [heartbeatInterval, doRenew]);

  // ==================== beforeunload ====================
  // 页面关闭时尽力解锁
  useEffect(() => {
    const handleUnload = () => {
      if (isEditingRef.current) {
        releaseLockBeacon(resourceIdRef.current);
      }
    };

    window.addEventListener('beforeunload', handleUnload);
    return () => window.removeEventListener('beforeunload', handleUnload);
  }, []);

  // ==================== 组件卸载清理 ====================
  // 路由跳转等场景
  useEffect(() => {
    return () => {
      if (isEditingRef.current) {
        cleanupWorker();
        releaseLock(resourceIdRef.current).catch(() => {});
        isEditingRef.current = false;
      }
    };
  }, [cleanupWorker]);

  return {
    isEditing,
    isLoading,
    lockInfo,
    enterEdit,
    exitEdit,
    failCount,
  };
}
```

### 4.4 业务页面使用示例

```tsx
// src/pages/EditPage.tsx
import { useEditLock } from '@/hooks/useEditLock';
import { message, Modal } from 'antd';

function EditPage({ resourceId }: { resourceId: string }) {
  const {
    isEditing,
    isLoading,
    lockInfo,
    enterEdit,
    exitEdit,
    failCount,
  } = useEditLock({
    resourceId,
    heartbeatInterval: 20_000,
    maxFailCount: 2,
    onLockLost: () => {
      Modal.error({
        title: '编辑锁已丢失',
        content: '由于网络问题或超时，您的编辑锁已失效。请保存好当前内容后重新进入编辑模式。',
      });
    },
    onLockConflict: (holder) => {
      message.warning(`当前页面正在被 ${holder} 编辑，请稍后再试`);
    },
  });

  const handleEdit = async () => {
    const success = await enterEdit();
    if (success) {
      message.success('已进入编辑模式');
    }
  };

  const handleSave = async () => {
    // await saveData(); // 先保存业务数据
    await exitEdit();
    message.success('保存成功');
  };

  const handleCancel = async () => {
    await exitEdit();
  };

  return (
    <div>
      {/* 非编辑模式下展示锁状态 */}
      {!isEditing && lockInfo && (
        <div className="lock-banner">
          当前正在被 {lockInfo.lockedBy} 编辑中
        </div>
      )}

      {/* 续锁失败警告 */}
      {isEditing && failCount > 0 && (
        <div className="warning-banner">
          ⚠️ 网络异常，续锁失败 ({failCount} 次)，请检查网络连接
        </div>
      )}

      {!isEditing ? (
        <button onClick={handleEdit} disabled={isLoading || !!lockInfo}>
          {isLoading ? '获取锁中...' : '编辑'}
        </button>
      ) : (
        <>
          <div className="edit-area">{/* 编辑区域 */}</div>
          <button onClick={handleSave}>保存</button>
          <button onClick={handleCancel}>取消</button>
        </>
      )}
    </div>
  );
}
```

两个页面复用只需传入不同的 `resourceId`：

```tsx
<PageA resourceId="page-a-123" />
<PageB resourceId="page-b-456" />
```

## 五、边界场景处理

| 场景 | 风险 | 处理方式 |
|---|---|---|
| **用户切到其他 tab / 最小化窗口** | `setInterval` 被浏览器节流至 1~5 分钟，锁过期 | Web Worker 定时器不受节流影响 |
| **电脑休眠 / 合盖** | 所有 JS（含 Worker）暂停 | `visibilitychange` 唤醒后立即补续锁 |
| **用户直接关闭浏览器 / tab** | `beforeunload` 中异步请求不可靠 | `sendBeacon` 发送解锁（best effort）+ 服务端锁超时自动释放 |
| **网络断开** | 续锁请求失败 | 连续失败计数，超过阈值触发 `onLockLost` 通知用户 |
| **路由跳转（组件卸载）** | 锁未释放 | `useEffect` cleanup 中主动解锁 + 清理 Worker |
| **同一用户两个 tab 打开同一页面** | 可能重复上锁 | 服务端返回 409 Conflict，前端走 `onLockConflict` 回调提示 |
| **续锁失败但网络恢复** | 锁可能已被他人获取 | 失败计数未达阈值时继续重试；达阈值后通知用户，避免静默覆盖 |

## 六、对后端的配合要求

1. **锁超时时间建议设为 60 秒**（3 倍于心跳间隔 20 秒），给网络抖动留出容错空间
2. **必须实现锁自动过期释放**，不能完全依赖前端主动解锁
3. **上锁接口在锁被占用时返回 409 Conflict**，响应体中包含 `lockedBy` 字段
4. **续锁接口应为幂等操作**，重复调用不产生副作用
5. **提供 POST 方式的解锁端点**（或支持 `_method=DELETE`），以兼容 `sendBeacon`（`sendBeacon` 只能发 POST）

## 七、方案优势总结

- **可靠性**：Web Worker 心跳不受浏览器后台节流影响，是保证续锁时效的关键
- **健壮性**：覆盖了 tab 切换、休眠唤醒、页面关闭、路由跳转、网络异常等全部边界场景
- **复用性**：封装为 `useEditLock` Hook，不同页面传入 `resourceId` 即可复用
- **用户体验**：续锁失败有实时提示，锁丢失有弹窗告知，锁被占用有明确反馈
- **渐进降级**：即使前端全部手段失效，服务端锁超时机制仍能保证不会产生死锁
