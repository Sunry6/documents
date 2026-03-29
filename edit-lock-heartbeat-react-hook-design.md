# 编辑锁续期 React Hook 设计稿

## 1. 目标

这份设计稿的目标不是给出完整可运行代码，而是给出一套适合 React 项目落地的 Hook 设计思路，解决下面几个问题：

1. 编辑态进入后自动加锁。
2. 编辑期间自动续锁。
3. 页面切到后台后，不假设定时器仍然可靠。
4. 页面恢复可见时，立即补做续锁或锁状态校验。
5. 一旦确认失锁，立即阻断编辑并提示用户。

适用前提：

- 前端通过接口加锁、续锁、解锁。
- 后端有锁超时机制。
- 当前业务是页面级或资源级独占编辑。

## 2. 设计原则

### 2.1 不把浏览器后台执行当成可靠前提

不要假设页面在后台时还能稳定每 20 秒发送一次心跳。

### 2.2 Hook 管状态，API 层管请求

Hook 只负责：

- 状态机管理
- 调度心跳
- 监听页面生命周期事件
- 判定是否失锁

接口实现由外部注入，避免 Hook 和具体接口模块强耦合。

### 2.3 不使用固定 `setInterval`

更推荐“本次结束后安排下一次”的 `setTimeout` 方案。

原因：

1. 便于处理失败重试。
2. 便于处理时间漂移。
3. 不会因为请求未完成而堆积多个并发心跳。

### 2.4 把“恢复”当成一等场景

真正关键的不是后台能不能一直稳定续锁，而是：

1. 从后台恢复后能否立即发现风险。
2. 能否第一时间补续锁或确认已失锁。

## 3. 推荐状态机

建议使用下面这些状态：

```ts
type LockPhase = 'idle' | 'acquiring' | 'active' | 'suspect' | 'releasing' | 'lost' | 'error'
```

### 3.1 各状态含义

`idle`

- 未进入编辑态。
- 当前没有持有锁。

`acquiring`

- 正在调用加锁接口。

`active`

- 已持有锁。
- 心跳工作正常。

`suspect`

- 怀疑锁可能失效。
- 例如心跳失败、页面恢复后发现时间漂移过大。

`releasing`

- 正在调用解锁接口。

`lost`

- 已确认失锁。
- 不允许继续编辑或提交。

`error`

- 发生非预期异常。
- 可由业务决定是否展示错误页或降级处理。

## 4. Hook 对外 API 设计

### 4.1 入参

```ts
export type LockInfo = {
  lockId: string
  owner?: string
  expiresAt?: number
  leaseVersion?: string
}

export type RenewResult = {
  success: boolean
  lockInfo?: LockInfo
  reason?: string
}

export type UseEditLockOptions = {
  enabled: boolean
  resourceId: string
  heartbeatIntervalMs?: number
  lockTtlMs?: number
  driftThresholdMs?: number
  quickRetryDelaysMs?: number[]
  acquireLock: (resourceId: string) => Promise<LockInfo>
  renewLock: (lockInfo: LockInfo) => Promise<RenewResult>
  releaseLock: (lockInfo: LockInfo) => Promise<void>
  getNow?: () => number
  onLockLost?: (reason: string) => void
  onPhaseChange?: (phase: LockPhase) => void
  onHeartbeatLog?: (payload: Record<string, unknown>) => void
}
```

### 4.2 出参

```ts
export type UseEditLockReturn = {
  phase: LockPhase
  lockInfo: LockInfo | null
  lastSuccessAt: number | null
  enterEdit: () => Promise<boolean>
  exitEdit: () => Promise<void>
  renewNow: (reason?: string) => Promise<void>
  isEditingAllowed: boolean
  isHeartbeatHealthy: boolean
}
```

### 4.3 设计意图

`enterEdit`

- 用户点击编辑按钮时调用。

`exitEdit`

- 用户退出编辑时调用。

`renewNow`

- 用于页面恢复可见、手工兜底、调试或异常恢复。

`isEditingAllowed`

- 用于页面表单禁用态判断。

`isHeartbeatHealthy`

- 用于页面轻提示或状态展示。

## 5. Hook 内部核心状态

建议使用 `useState + useRef` 组合，而不是把所有运行态都放进 `useState`。

### 5.1 React 状态

```ts
const [phase, setPhase] = useState<LockPhase>('idle')
const [lockInfo, setLockInfo] = useState<LockInfo | null>(null)
const [lastSuccessAt, setLastSuccessAt] = useState<number | null>(null)
```

### 5.2 运行时引用

```ts
const timerRef = useRef<number | null>(null)
const inFlightRef = useRef(false)
const lastAttemptAtRef = useRef<number | null>(null)
const nextDueAtRef = useRef<number | null>(null)
const lockInfoRef = useRef<LockInfo | null>(null)
const phaseRef = useRef<LockPhase>('idle')
```

原因：

1. 避免调度逻辑因为闭包拿到旧值。
2. 避免每次调度细节都触发重渲染。
3. 保证事件监听器里拿到的是最新状态。

## 6. 推荐默认参数

```ts
const heartbeatIntervalMs = options.heartbeatIntervalMs ?? 20_000
const lockTtlMs = options.lockTtlMs ?? 120_000
const driftThresholdMs = options.driftThresholdMs ?? 40_000
const quickRetryDelaysMs = options.quickRetryDelaysMs ?? [2_000, 5_000]
```

说明：

1. 20 秒续锁可以保留。
2. 锁超时建议至少 120 秒，不建议继续维持 60 秒。
3. 时间漂移阈值建议设置为 2 倍心跳间隔左右。

## 7. 推荐 Hook 结构

### 7.1 辅助函数

```ts
function clearScheduledHeartbeat() {
  if (timerRef.current !== null) {
    window.clearTimeout(timerRef.current)
    timerRef.current = null
  }
}

function syncRefs(nextPhase: LockPhase, nextLockInfo: LockInfo | null) {
  phaseRef.current = nextPhase
  lockInfoRef.current = nextLockInfo
}

function updatePhase(nextPhase: LockPhase) {
  phaseRef.current = nextPhase
  setPhase(nextPhase)
  options.onPhaseChange?.(nextPhase)
}
```

### 7.2 安排下一次心跳

```ts
function scheduleNextHeartbeat(now: number) {
  clearScheduledHeartbeat()

  const nextDueAt = now + heartbeatIntervalMs
  nextDueAtRef.current = nextDueAt

  timerRef.current = window.setTimeout(() => {
    void renewNow('timer')
  }, heartbeatIntervalMs)
}
```

如果团队已经明确要把“调度器”放到 Worker，这个函数可以替换成 Worker 通讯版本，但 Hook 本身的状态机逻辑不需要变。

## 8. enterEdit 设计

```ts
const enterEdit = useCallback(async () => {
  if (!options.enabled) return false
  if (phaseRef.current !== 'idle' && phaseRef.current !== 'lost') return false

  updatePhase('acquiring')

  try {
    const acquired = await options.acquireLock(options.resourceId)
    const now = (options.getNow ?? Date.now)()

    setLockInfo(acquired)
    setLastSuccessAt(now)
    syncRefs('active', acquired)
    setPhase('active')

    lastAttemptAtRef.current = now
    scheduleNextHeartbeat(now)
    return true
  } catch (error) {
    syncRefs('error', null)
    setPhase('error')
    options.onHeartbeatLog?.({
      type: 'acquire-error',
      error: String(error),
    })
    return false
  }
}, [options])
```

建议点：

1. 加锁成功后，把“本次加锁时间”也视为首次心跳成功时间。
2. 不要等 20 秒后再初始化状态。

## 9. renewNow 设计

这是整个 Hook 的核心。

```ts
const renewNow = useCallback(
  async (reason = 'manual') => {
    if (!options.enabled) return
    if (!lockInfoRef.current) return
    if (inFlightRef.current) return

    const now = (options.getNow ?? Date.now)()
    const lastAttemptAt = lastAttemptAtRef.current
    const drift = lastAttemptAt ? now - lastAttemptAt : 0

    if (drift > driftThresholdMs) {
      updatePhase('suspect')
      options.onHeartbeatLog?.({
        type: 'drift-detected',
        reason,
        now,
        drift,
        visibility: document.visibilityState,
      })
    }

    inFlightRef.current = true
    lastAttemptAtRef.current = now

    try {
      const result = await options.renewLock(lockInfoRef.current)

      if (!result.success) {
        throw new Error(result.reason || 'renew failed')
      }

      const successAt = (options.getNow ?? Date.now)()
      const nextLockInfo = result.lockInfo ?? lockInfoRef.current

      setLockInfo(nextLockInfo)
      setLastSuccessAt(successAt)
      syncRefs('active', nextLockInfo)
      setPhase('active')

      options.onHeartbeatLog?.({
        type: 'renew-success',
        reason,
        at: successAt,
        visibility: document.visibilityState,
      })

      scheduleNextHeartbeat(successAt)
    } catch (error) {
      options.onHeartbeatLog?.({
        type: 'renew-failed',
        reason,
        at: (options.getNow ?? Date.now)(),
        error: String(error),
        visibility: document.visibilityState,
      })

      const recovered = await retryRenewQuickly(reason)
      if (!recovered) {
        const nowAfterRetry = (options.getNow ?? Date.now)()
        const latestSuccessAt = lastSuccessAt ?? 0
        const elapsed = nowAfterRetry - latestSuccessAt

        if (elapsed >= lockTtlMs) {
          handleLockLost('续锁超时，编辑锁已失效')
        } else {
          updatePhase('suspect')
          scheduleNextHeartbeat(nowAfterRetry)
        }
      }
    } finally {
      inFlightRef.current = false
    }
  },
  [driftThresholdMs, lastSuccessAt, lockTtlMs, options]
)
```

## 10. 快速重试逻辑

```ts
async function retryRenewQuickly(reason: string) {
  for (const delay of quickRetryDelaysMs) {
    await new Promise(resolve => window.setTimeout(resolve, delay))

    try {
      const currentLock = lockInfoRef.current
      if (!currentLock) return false

      const result = await options.renewLock(currentLock)
      if (result.success) {
        const successAt = (options.getNow ?? Date.now)()
        const nextLockInfo = result.lockInfo ?? currentLock

        setLockInfo(nextLockInfo)
        setLastSuccessAt(successAt)
        syncRefs('active', nextLockInfo)
        setPhase('active')
        scheduleNextHeartbeat(successAt)

        options.onHeartbeatLog?.({
          type: 'renew-retry-success',
          reason,
          at: successAt,
        })
        return true
      }
    } catch (error) {
      options.onHeartbeatLog?.({
        type: 'renew-retry-failed',
        reason,
        delay,
        error: String(error),
      })
    }
  }

  return false
}
```

建议点：

1. 不要失败一次就直接等下一轮 20 秒。
2. 2 秒、5 秒两次重试是一个比较稳妥的起点。

## 11. 失锁处理

```ts
function handleLockLost(reason: string) {
  clearScheduledHeartbeat()
  syncRefs('lost', lockInfoRef.current)
  setPhase('lost')
  options.onLockLost?.(reason)
}
```

触发后建议业务立刻做这些事：

1. 禁用表单编辑。
2. 阻止保存。
3. 提示用户重新进入编辑态。

## 12. exitEdit 设计

```ts
const exitEdit = useCallback(async () => {
  clearScheduledHeartbeat()

  const currentLock = lockInfoRef.current
  if (!currentLock) {
    syncRefs('idle', null)
    setLockInfo(null)
    setLastSuccessAt(null)
    setPhase('idle')
    return
  }

  updatePhase('releasing')

  try {
    await options.releaseLock(currentLock)
  } catch (error) {
    options.onHeartbeatLog?.({
      type: 'release-failed',
      error: String(error),
    })
  } finally {
    syncRefs('idle', null)
    setLockInfo(null)
    setLastSuccessAt(null)
    setPhase('idle')
  }
}, [options])
```

说明：

1. 解锁失败可以记录日志，但一般不阻止本地退出编辑。
2. 如果业务要求更严格，可以在解锁失败时做额外提示。

## 13. 页面生命周期事件接入

这部分是方案成败关键。

```ts
useEffect(() => {
  if (!options.enabled) return

  function handleResume(source: 'visible' | 'focus') {
    const currentPhase = phaseRef.current
    if (currentPhase !== 'active' && currentPhase !== 'suspect') return
    if (!lockInfoRef.current) return

    void renewNow(source)
  }

  function onVisibilityChange() {
    if (document.visibilityState === 'visible') {
      handleResume('visible')
    }
  }

  function onFocus() {
    handleResume('focus')
  }

  document.addEventListener('visibilitychange', onVisibilityChange)
  window.addEventListener('focus', onFocus)

  return () => {
    document.removeEventListener('visibilitychange', onVisibilityChange)
    window.removeEventListener('focus', onFocus)
  }
}, [options.enabled, renewNow])
```

为什么必须做：

1. 页面从后台回来时，不能等下一轮 20 秒心跳。
2. 需要立即确认锁是否仍然有效。

## 14. 页面卸载与路由切换

如果是单页应用，还要考虑路由切换和组件卸载。

```ts
useEffect(() => {
  return () => {
    clearScheduledHeartbeat()
  }
}, [])
```

如果业务要求“离开编辑页自动解锁”，建议在页面路由离开时显式调用 `exitEdit`，而不是只靠 `beforeunload`。

原因：

1. `beforeunload` 可靠性有限。
2. 某些异步请求无法在卸载阶段稳定完成。

## 15. 页面组件接入方式示例

```tsx
function EditPage({ recordId }: { recordId: string }) {
  const [editing, setEditing] = useState(false)

  const lock = useEditLock({
    enabled: editing,
    resourceId: recordId,
    acquireLock: api.acquireLock,
    renewLock: api.renewLock,
    releaseLock: api.releaseLock,
    onLockLost: reason => {
      setEditing(false)
      openLockLostModal(reason)
    },
    onHeartbeatLog: payload => {
      reportLockMetric(payload)
    },
  })

  async function handleClickEdit() {
    const ok = await lock.enterEdit()
    if (ok) {
      setEditing(true)
    }
  }

  async function handleExitEdit() {
    await lock.exitEdit()
    setEditing(false)
  }

  return (
    <Page>
      <Toolbar>
        {!editing ? (
          <button onClick={handleClickEdit}>编辑</button>
        ) : (
          <button onClick={handleExitEdit}>退出编辑</button>
        )}
      </Toolbar>

      <EditorForm disabled={!lock.isEditingAllowed} />
    </Page>
  )
}
```

## 16. 是否要把心跳调度放进 Worker

这套 Hook 设计兼容两种调度方式：

### 16.1 主线程调度

适合先快速落地。

优点：

1. 实现简单。
2. 调试方便。

缺点：

1. 容易受到主线程卡顿影响。

### 16.2 Worker 调度

适合作为增强项。

做法：

1. Hook 保留状态机。
2. 定时器由 Worker 发消息驱动。
3. Hook 收到 Worker 消息后调用 `renewNow('worker-timer')`。

优点：

1. 减少主线程卡顿导致的误差。

缺点：

1. 后台节流和冻结问题仍然存在。
2. 不能把 Worker 当成永久可靠的后台心跳进程。

结论：

Worker 值得做，但不是根治手段。

## 17. 监控与埋点建议

建议为下面这些事件打点：

1. 加锁成功/失败。
2. 续锁成功/失败。
3. 快速重试成功/失败。
4. 时间漂移超阈值。
5. 页面从 hidden 恢复到 visible。
6. 锁丢失。

推荐字段：

1. `resourceId`
2. `lockId`
3. `phase`
4. `reason`
5. `visibilityState`
6. `lastSuccessAt`
7. `driftMs`
8. `elapsedSinceLastSuccessMs`

这会帮助你们判断：

1. 是不是后台生命周期导致的问题。
2. 是不是网络抖动导致的。
3. 快速重试是否已经足够。

## 18. 最终建议

如果你们要在 React 项目里落地，优先顺序建议是：

1. 先上 Hook 状态机。
2. 加 `visibilitychange` 和 `focus` 恢复补续锁。
3. 加快速重试。
4. 把锁 TTL 提高到 120 到 180 秒。
5. 再评估是否把调度迁移到 Worker。

一句话总结：

React Hook 真正要解决的，不是“如何每 20 秒精确发一次请求”，而是“当浏览器不再可靠执行时，如何快速发现、恢复或中止编辑”。
