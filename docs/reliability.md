# TradeSync Pro — Reliability, Connectivity & Recovery

This document explains how TradeSync Pro remains stable during:
- internet outages,
- broker/API disconnects,
- transient execution failures (trade placement / closing),
- concurrency hazards (multiple services touching shared clients).

> Design goal: maximize continuity of trade copying while making failures visible and actionable via logs.

---

## 1. Reliability strategy (overview)

TradeSync Pro uses **layered resilience**:

1. **Internet-level monitoring** (`InternetConnectivityMonitor`)
   - Detects if the machine is truly offline.
   - Triggers reconnection orchestration when connectivity returns.

2. **Broker-level connection monitoring**
   - Both `ReconnectionService` and the `TradeCopyingService` validate **actual client connection state** (`client.Connected`), not UI flags alone.

3. **Backoff + retry**
   - Disconnect recovery uses exponential backoff.
   - Trade placement uses bounded retries with increasing delays (and fast retry after symbol map updates).

4. **Concurrency controls**
   - A global async gate (`AccountCoordinator.Gate: SemaphoreSlim`) serializes operations that replace/modify MT4/MT5 client instances.
   - Both reconnect paths track “accounts being reconnected” to prevent duplicate reconnect attempts.

---

## 2. InternetConnectivityMonitor (global connectivity detection)

### 2.1 How it works
- Runs on a dedicated background thread (`TaskCreationOptions.LongRunning`).
- Every ~3 seconds:
  - Pings `8.8.8.8` (Google DNS)
  - If that fails, pings `1.1.1.1` (Cloudflare DNS)
- Emits an event when status changes:
  - `StatusChanged(isOnline: bool)`

### 2.2 Why this is effective
- ICMP ping is lightweight and independent of DNS resolution issues.
- Using a fallback endpoint reduces false negatives.

### 2.3 Consumer behavior
When internet is lost/restored, the app triggers reconnection workflows and updates UI logs (e.g., in `ManageAccountsView`).

---

## 3. ReconnectionService (internet outage recovery)

`ReconnectionService` is responsible for reconnecting accounts **after a global internet outage**.

### 3.1 Capture on internet loss
When an outage is detected:
1. The service captures the list of currently connected accounts.
2. Each captured account is marked:
   - `connectionStatus = "Reconnecting..."`
   - `isConnected = false`
3. `AccountsChanged` is raised so UI updates.

This ensures users clearly see that the app is in recovery mode.

### 3.2 Reconnect orchestration on internet restored
On restore, the service attempts reconnection:
- **Selected first** (Master/Slave if needed): `ReconnectSelectedAccountsAsync(master, slave)`
- **Remaining accounts**: `ReconnectRemainingAccountsAsync(excludeAccounts)`
- Or a full reconnect: `ReconnectAllAccountsAsync()`

### 3.3 Throttled parallelism
`ReconnectRemainingAccountsAsync` uses:
- `SemaphoreSlim(initialCount: 4, maxCount: 4)`

So at most **4 accounts** are actively reconnecting at a time to avoid:
- CPU spikes
- broker connection throttling
- cascading timeouts

### 3.4 Reconnect retry loop (exponential backoff)
For each account, `ReconnectAccountWithRetryAsync` attempts reconnection with:
- `MaxReconnectAttempts = 10`
- `InitialReconnectDelayMs = 2000`
- `ReconnectBackoffMultiplier = 1.5`
- `MaxReconnectDelayMs = 60000`

Conceptually:
- 2s, 3s, 4.5s, 6.75s, … capped to 60s

### 3.5 Timeouts per attempt
Each attempt is protected with:
- `CancellationTokenSource(TimeSpan.FromSeconds(15))`

If the reconnect attempt blocks, it becomes a controlled failure (“Timeout”) and retries can proceed.

### 3.6 Conflict prevention
ReconnectionService maintains:
- `_accountsBeingReconnected: HashSet<string>`

This prevents multiple reconnect requests from racing to reconnect the same account.

### 3.7 Safe client replacement
When reconnecting:
- MT4: replaces the client at the account’s `accIndex` with a new `QuoteClient`.
- MT5: replaces the client at the account’s `accIndex` with a new `MT5API`.

Crucially, this happens under:
- `AccountCoordinator.Gate` to prevent concurrent modification.

---

## 4. TradeCopyingService (broker-side disconnect recovery)

The trade copying engine includes its own broker-level disconnect recovery for the **active session accounts** (Master/Slave).

### 4.1 Why it exists
Internet may be *online* while:
- the broker endpoint is unavailable,
- the API session is dropped,
- the trading server disconnects.

The engine handles this case without waiting for the global internet monitor.

### 4.2 Detecting actual connectivity
On each poll tick, the engine evaluates:
- `IsClientConnected(masterAccount)`
- `IsClientConnected(slaveAccount)`

If a client disconnects, it logs:
- `⚠ Master ... disconnected. Trade copying paused.`
- `⚠ Slave ... disconnected. Trade copying paused.`

Trade processing is skipped until both are connected again.

### 4.3 Reconnect policy (engine-level)
The engine implements exponential backoff scheduling:
- `MaxReconnectAttempts = 10`
- `InitialReconnectDelayMs = 2000`
- `ReconnectBackoffMultiplier = 1.5`
- `MaxReconnectDelayMs = 60000`

It tracks:
- `_reconnectAttempts[loginId]`
- `_nextReconnectTime[loginId]`

So reconnection retries are **scheduled** and only retried when due, rather than hammering the broker.

### 4.4 “Keep trying” behavior
If attempts exceed `MaxReconnectAttempts`, the engine logs a warning but still continues attempting with the max delay. This is an intentional “eventual recovery” strategy.

### 4.5 Concurrency safety with ReconnectionService
ReconnectionService and TradeCopyingService can both reconnect accounts. To avoid stepping on each other:

- TradeCopyingService:
  - uses `_accountsBeingReconnected` to avoid duplicate reconnect threads
  - uses `AccountCoordinator.Gate` during client replacement (serialized with ReconnectionService)
  - double-checks “already connected” inside the gate before replacing the client

This prevents:
- double client creation
- inconsistent indices
- partial reconnected client state

---

## 5. Backoff behavior (summary table)

| Concern | Component | Backoff / Retry |
|---|---|---|
| Internet offline | `InternetConnectivityMonitor` | Detection only (poll every ~3s) |
| Reconnect after internet restore | `ReconnectionService` | up to 10 attempts; 2s → 60s exponential backoff; per-attempt timeout (15s) |
| Broker disconnect during active copy | `TradeCopyingService` | scheduled retries with backoff; avoids hammering |
| Trade placement failure | `TradeCopyingService` | bounded retries (3 attempts) with delays: 30s / 120s / 300s |
| Mapping corrected by user | `SymbolMappingService` → engine | resets retry timers so failures retry immediately |

---

## 6. Trade operation reliability (non-connectivity failures)

### 6.1 Symbol mapping fallback
For each master symbol, the engine can try multiple candidate slave symbols:
- Example: `XAUUSD` → [`GOLD`, `XAUUSDm`, `XAUUSD#`, ...]
This reduces failures due to broker instrument naming differences.

### 6.2 TP/SL safety fallback
If TP/SL is enabled and the broker rejects stops:
- engine retries opening the trade without TP/SL
- logs the fallback

### 6.3 Closing reliability
Closing operations use retry-like safeguards:
- “Already closed / not found” responses are treated as success where appropriate (idempotency).
- Master-close (triggered by slave close) uses retry with short increasing delays.

---

## 7. Key design choices that improve stability

- **Actual connectivity check** uses API client state, not UI state.
- **Atomic persistence** reduces corruption risk on abrupt shutdown.
- **Serialized client mutation** via `AccountCoordinator.Gate` avoids concurrency bugs.
- **Throttled reconnect** prevents bursts (especially important with many accounts).
- **Explicit logs** provide diagnosable, user-actionable feedback.

---
