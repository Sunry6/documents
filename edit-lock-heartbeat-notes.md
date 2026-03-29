# Edit Lock Heartbeat Design Notes

## 1. Problem Background

Current design:

- User clicks Edit, frontend calls lock API.
- While editing, frontend sends a renew-lock heartbeat every 20 seconds.
- When leaving edit mode, frontend calls unlock API.
- Backend automatically releases the lock if no heartbeat is received for more than 60 seconds.
- Current implementation uses polling plus Web Worker as the timer, but the renew request itself is still executed from the main thread.

Observed issue from testing:

- User enters edit mode.
- User minimizes the page, switches to another browser tab, or switches to another app.
- After more than 10 minutes, when the user comes back, the page is already unlocked.

This is very likely not a tester operation issue. It is much more likely a design robustness issue around browser background lifecycle behavior.

## 2. Core Conclusion

The current heartbeat design is fragile in background scenarios.

Main reasons:

1. Background pages can be throttled even if the machine is not sleeping.
2. Background pages can sometimes be frozen after staying inactive for some time.
3. A regular Web Worker is not a reliable always-on background process.
4. A 20-second heartbeat with a 60-second timeout leaves too little margin for delay, throttling, retry, and network jitter.

So the issue should be framed as:

Frontend lock renewal currently depends too heavily on stable browser-side timing, but browser background execution is not a reliable assumption.

## 3. Throttling vs Freezing

These two concepts must be separated.

### 3.1 Throttling

Throttling means JavaScript can still run, but less frequently.

Typical effects:

- `setTimeout` and `setInterval` are delayed.
- Task execution interval becomes longer than expected.
- A 20-second heartbeat may no longer execute every 20 seconds.

Typical triggers:

- Switching to another tab.
- Minimizing the browser window.
- Switching to another application for a long time.

### 3.2 Freezing

Freezing means the page stops getting execution time for normal JavaScript work.

Typical effects:

- Timers stop firing.
- Heartbeat scheduling stops.
- Main-thread logic pauses.
- Regular Web Worker logic may also pause with the page lifecycle.

Typical triggers:

- A page stays in the background for a long time.
- Browser memory-saving or tab-sleep policies kick in.
- Resource pressure becomes high.
- The machine sleeps, locks, or the browser process is deprioritized aggressively.

Important:

Freezing is not limited to system sleep. A page can be backgrounded, throttled, and eventually frozen without the machine sleeping.

## 4. Will This Happen Only During Sleep?

No.

The following can all affect heartbeat stability:

1. Switching to another browser tab.
2. Minimizing the browser window.
3. Switching to another desktop application.
4. Browser memory-saver or tab-discard policies.
5. System sleep or lock screen.

System sleep is only the most obvious and most extreme case.

## 5. Main Thread vs Web Worker

### 5.1 Main Thread

The main thread can be throttled in the background, and under some conditions it can effectively stop executing while the page is frozen.

### 5.2 Regular Web Worker

A regular Web Worker can reduce the impact of main-thread blocking, but it is still part of the page context and still subject to browser lifecycle policies.

That means:

- Moving heartbeat scheduling to a Worker may improve stability.
- Moving the renew request itself to a Worker may also help avoid some main-thread delay.
- But it still does not guarantee reliable execution for long background periods.

### 5.3 Service Worker

Service Worker is event-driven, not a persistent background timer mechanism.

It should not be treated as a reliable way to send a renew request every 20 seconds.

## 6. Why the Current Design Is Risky

The current design uses:

- Heartbeat interval: 20 seconds
- Lock timeout: 60 seconds

This means the system only has a small safety window.

Practical risks:

1. One or two delayed heartbeats can consume most of the lock TTL.
2. A single network failure without fast retry can push the lock close to expiration.
3. Background throttling can stretch timing enough to cross the 60-second threshold.
4. If the page is frozen, heartbeat may stop completely until the user returns.

So even if the implementation is technically correct, the design is still not robust enough.

## 7. Should Renew Requests Be Moved into a Web Worker?

Yes, but only as an enhancement.

Benefits:

1. Reduced impact from main-thread rendering or UI blocking.
2. Better timing stability while the page is still actively executing.
3. Cleaner separation of heartbeat scheduling logic.

Limitations:

1. Worker execution is still not guaranteed in long-running background scenarios.
2. Worker does not eliminate background throttling or freezing risk.
3. Worker alone cannot guarantee that renew requests will keep running when the page is deprioritized.

Decision guidance:

- Moving heartbeat work into a Worker is reasonable.
- It should not be treated as the only fix.
- The real fix is architectural robustness against background lifecycle behavior.

## 8. Recommended Frontend Strategy

The key idea is this:

Do not assume the page will keep sending heartbeat reliably while hidden. Instead, design for recovery and confirmation when the page becomes active again.

Recommended principles:

1. Normal heartbeat while the page is active.
2. Do not trust hidden-page timing to be precise.
3. Recover immediately when the page becomes visible again.
4. Detect long timing drift and treat it as a risk signal.
5. Block editing as soon as lock loss is confirmed.

## 9. Suggested State Machine

Recommended states:

### 9.1 `idle`

Not editing, no lock held.

### 9.2 `acquiring`

User clicked Edit, frontend is requesting the lock.

### 9.3 `active`

Lock is held and heartbeat is healthy.

### 9.4 `suspect`

Lock may be at risk.

Typical reasons:

- Heartbeat failed.
- Timing drift is too large.
- Page returned from background after a long time.

### 9.5 `lost`

Lock is confirmed lost.

Editing must be blocked until the user reacquires the lock.

## 10. Required Frontend Signals

The frontend should at least watch these signals:

1. Scheduled heartbeat timer.
2. `visibilitychange`.
3. `focus`.
4. Time drift between expected execution time and actual execution time.

Recommended behavior:

- When `document.visibilityState` changes back to `visible`, immediately trigger renew or lock-status verification.
- When the window regains `focus`, do the same.
- If the actual time gap is much larger than the intended interval, treat the lock as at-risk and renew immediately.

## 11. Recommended Timing Parameters

Current values are too aggressive.

Suggested adjustments:

1. Keep heartbeat interval around 20 seconds if desired.
2. Increase lock TTL from 60 seconds to 120 to 180 seconds.
3. Add fast retry after heartbeat failure, for example after 2 seconds and 5 seconds.
4. When returning to the foreground, do not wait for the next scheduled 20-second heartbeat. Renew immediately.

Why this helps:

- A larger TTL creates tolerance for background delays.
- Fast retry reduces the chance that one failed renew leads directly to lock loss.
- Immediate foreground recovery reduces user confusion and narrows the inconsistency window.

## 12. Recommended Failure Handling

When a renew request fails:

1. Do not immediately assume the lock is gone.
2. Retry quickly one or two times.
3. If time since last successful renew is close to or beyond TTL, treat the lock as lost.
4. Disable editing UI when lock loss is confirmed.
5. Show a clear message telling the user to reacquire the lock.

## 13. Recommended User Experience

### 13.1 Normal Editing

No interruption, no noisy messaging.

### 13.2 Lock Risk Detected

Optional lightweight message:

"Checking whether your edit lock is still valid..."

### 13.3 Lock Lost

Use a blocking modal or clear error state.

Suggested message:

"Your edit lock has expired because the page stayed inactive too long or the heartbeat was interrupted. Please reacquire the lock before continuing."

## 14. Recommended Backend Cooperation

If backend changes are possible, return more lease information with the lock and renew APIs.

Recommended fields:

1. `lockId`
2. `owner`
3. `expiresAt`
4. `serverNow`
5. `leaseVersion` or another fencing token

This helps the frontend make more reliable decisions about:

- Whether it still owns the lock.
- How close the lock is to expiration.
- Whether a concurrent takeover happened.

## 15. Recommended Delivery Order

Most practical implementation order:

1. Increase backend lock TTL from 60 seconds to 120 to 180 seconds.
2. Add immediate renew on `visibilitychange` and `focus` recovery.
3. Add fast retry after renew failure.
4. Add drift detection and suspect-state handling.
5. Optionally move timer and renew logic into a Worker.
6. If the business needs stronger guarantees, evolve toward a more explicit lease-based design.

## 16. Final Recommendation

The right conclusion for review is:

1. This issue is consistent with normal browser background lifecycle behavior.
2. It is not safe to assume that either the main thread or a regular Web Worker will always keep running in background scenarios.
3. Moving renew requests into a Worker is useful, but it is not sufficient by itself.
4. The most important improvements are:
   - larger lock TTL,
   - immediate recovery renew when the page becomes active again,
   - fast retry on failure,
   - explicit lock-loss handling in the UI,
   - and, if possible, stronger backend lease semantics.

In short:

The system should be designed to survive browser background behavior, not to assume it away.
