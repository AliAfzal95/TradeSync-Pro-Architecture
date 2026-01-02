# TradeSync Pro — Trade Copying Engine (Design, State Machine & Pseudocode)

This document describes the trade copying engine at a **design level** using **state machines, data structures, and pseudocode**. It intentionally omits proprietary implementation details and any broker-specific API specifics.

---

## 1. Goals

The engine is designed to:
- Copy trades from **one Master** account to **one Slave** account (MT4/MT5).
- Support **symbol mapping** (Master symbol → one or more Slave symbol candidates).
- Apply runtime-adjustable settings:
  - lot multiplier or fixed lot size
  - optional delay
  - optional reverse direction
  - optional TP/SL copying (with safety fallback)
  - **close master when slave closes** (unique 1-to-1 feature)
- Be **fault tolerant**:
  - avoid duplicate copies
  - retry failed trade placement with backoff
  - recover from broker/internet disconnects

---

## 2. Inputs / Outputs (conceptual)

### Inputs
- `masterAccount`, `slaveAccount`
- `CopySettings`:
  - `LotMultiplier`
  - `FixedLotSize?`
  - `ReverseTradeDirection`
  - `EnableDelay` + `DelayMs`
  - `CopySameTpSl`
  - `CloseMasterOnSlaveClosed`
- `SymbolMappingService`:
  - `GetAllSlaveSymbols(masterSymbol)` returns candidates in preferred order

### Outputs
- Events for UI logging and telemetry:
  - Log messages
  - Error messages
  - Trade copied notifications
  - Trade closed notifications
  - Master-closed (because slave closed) notifications

---

## 3. Core state (in-memory)

The engine is built around these internal structures:

### 3.1 Ticket mapping dictionaries
- `masterToSlaveOrders: Map<MasterTicket, TrackedOrder>`
- `slaveToMasterOrders: Map<SlaveTicket, TrackedOrder>`

Purpose: enable lifecycle actions such as:
- close orphaned slave trades when master trade disappears
- close master trades when slave trade disappears (unique feature)

### 3.2 Safety sets
- `pendingMasterTickets: Set<MasterTicket>`
  - prevents concurrent duplicate copy attempts for the same master ticket
- `knownMasterTickets: Set<MasterTicket>`
  - prevents re-copy loops after failures / state churn
- `pendingSlaveCloses: Set<SlaveTicket>`
  - prevents multiple close attempts for the same slave ticket
- `pendingMasterCloses: Set<MasterTicket>`
  - indicates master trade must be closed due to a slave close decision
- `recentlyClosedMasters: Map<MasterTicket, DateTime>`
  - short time window to avoid re-copying a master ticket that was just handled

### 3.3 Failure tracking / retry scheduling
- `failedTrades: Map<MasterTicket, FailedTradeInfo>`
  - `RetryCount`, `NextRetryTime`, `PermanentlyFailed`
- `permanentlyFailedSymbols: Set<Symbol>`
  - if a symbol repeatedly fails, pause attempts until mappings/settings change

### 3.4 Reconnection tracking
- `lastMasterConnected`, `lastSlaveConnected` (actual client connectivity)
- `accountsBeingReconnected: Set<LoginId>`
- `reconnectAttempts: Map<LoginId, int>`
- `nextReconnectTime: Map<LoginId, DateTime>`

These protect against:
- connection flapping
- duplicate reconnection threads
- excessive repeated reconnect load

---

## 4. State machine (high level)

The engine’s top-level lifecycle:

    [*] --> Idle
    
    Idle --> Running: StartCopying(master, slave, settings)

    Running --> Paused: PauseCopying()
    returns IDisposable handle
    Paused --> Running: Dispose(pauseHandle)
    (resume)

    Running --> Idle: StopCopying()
    Paused --> Idle: StopCopying()

    note right of Running
      Poll loop active (timer)
      Processes trades only if:
      - master connected
      - slave connected
    end note


### Running sub-state: connectivity gate
Inside `Running`, the engine only processes trades when:
- `masterClientConnected == true`
- `slaveClientConnected == true`

If either is disconnected, processing is skipped until reconnection succeeds.

---

## 5. Polling loop (pseudocode)

The engine uses a short-interval polling loop:

```

function pollTick():
    if not isCopying or isPaused: return
    if masterAccount == null or slaveAccount == null: return

    masterConnected = isClientConnected(masterAccount)
    slaveConnected  = isClientConnected(slaveAccount)

    // detect connection state transitions (for logs + state reset)
    handleConnectionTransition(masterAccount, masterConnected, ref lastMasterConnected)
    handleConnectionTransition(slaveAccount,  slaveConnected,  ref lastSlaveConnected)

    // schedule retries for still-disconnected accounts
    checkAndRetryReconnection(masterAccount)
    checkAndRetryReconnection(slaveAccount)

    if not masterConnected or not slaveConnected: return

    masterTrades = getOpenTrades(masterAccount)
    slaveTrades  = getOpenTrades(slaveAccount)

    processTrades(masterTrades, slaveTrades)

```

---

## 6. Trade reconciliation pipeline (order lifecycle)

The `processTrades(...)` pipeline runs a deterministic set of steps:

```

function processTrades(masterTrades, slaveTrades): settings = getSettingsSnapshot()

masterTickets = set(masterTrades.ticket)
slaveTickets  = set(slaveTrades.ticket)

cleanupRecentlyClosedMasters()
cleanupFailedTradesForClosedMasters(masterTickets)
cleanupKnownTicketsForClosedMasters(masterTickets)

confirmPendingMasterCloses(masterTickets)

handleSlaveClosedTickets(slaveTickets, settings)

removeStaleMappings(slaveTickets)

copyNewMasterTrades(masterTrades, settings)

closeOrphanedSlaveTrades(masterTickets)

persistState()

```


### 6.1 Copy eligibility rules (important)
A master trade is eligible to copy only if it is **not**:
- already mapped (`masterToSlaveOrders contains ticket`)
- pending copy (`pendingMasterTickets contains ticket`)
- pending close (`pendingMasterCloses contains ticket`)
- recently closed (`recentlyClosedMasters contains ticket`)
- known and intentionally blocked (`knownMasterTickets contains ticket`)
- blocked by symbol-level permanent failure (`permanentlyFailedSymbols contains symbol`)
- failed recently and not yet due (`failedTrades[ticket].NextRetryTime > now`)

---

## 7. Copying a new Master trade (mapping + settings)

### 7.1 Candidate symbol selection
Symbol mapping returns one or more candidates:

```

candidates = mappingService.getAllSlaveSymbols(masterTrade.symbol) // if no mapping exists, candidates defaults to [masterTrade.symbol]

```


### 7.2 Pre-subscribe optimization
Before placing orders, subscribe to candidates to remove subscription latency:

```

for each symbol in candidates: ensureSymbolSubscribed(slaveAccount, symbol)

```


### 7.3 Copy operation pseudocode

```

async function copyTradeToSlave(masterTrade, settings): if pendingMasterCloses contains masterTrade.ticket: return
candidates = mappingService.getAllSlaveSymbols(masterTrade.symbol) preSubscribe(slaveAccount, candidates)
if settings.enableDelay: await delay(settings.delayMs)

// compute lot size
masterLots = parse(masterTrade.size)
slaveLots = settings.fixedLotSize ?? round(masterLots * settings.lotMultiplier, 2)
slaveLots = max(slaveLots, 0.01)

// direction selection
slaveType = masterTrade.type if settings.reverseTradeDirection: slaveType = invert(masterTrade.type)

// attempt each candidate until success
for each slaveSymbol in candidates: ticket = tryOpenOrder(slaveAccount, slaveSymbol, slaveLots, slaveType, settings, masterTrade)
if ticket not empty: recordTrackedOrder(masterTrade, slaveSymbol, ticket, slaveType, slaveLots)
clearFailureState(masterTrade.ticket, masterTrade.symbol)
emitTradeCopiedEvent(...) return

// all candidates failed => mark failed + schedule retry recordFailure(masterTrade.ticket, masterTrade.symbol, lastOrderError)

```


---

## 8. TP/SL copy logic (safe mode)

When `CopySameTpSl` is enabled, the engine attempts to apply master SL/TP on the slave order **only when direction is not reversed**.

Safety fallback:
- If the broker rejects SL/TP (invalid stops), the engine retries opening the order **without** SL/TP and logs the fallback.

Pseudocode:

```

function tryOpenOrder(account, symbol, lots, type, settings, masterTrade): applyStops = settings.copySameTpSl AND (NOT settings.reverseTradeDirection)
sl = applyStops ? masterTrade.stopLoss  : 0
tp = applyStops ? masterTrade.takeProfit : 0

try: return open(symbol, lots, type, sl, tp)
catch err where applyStops AND isInvalidStopsError(err):
log("Invalid stops; retrying without SL/TP") return
open(symbol, lots, type, 0, 0) catch err: lastOrderError = err return empty

```


---

## 9. Retry logic (failed trade placement)

The engine uses bounded retries for copy placement attempts.

### 9.1 Retry scheduling
- Maximum attempts: `MaxRetryAttempts = 3`
- Delay schedule (example): `30s`, `120s`, `300s`
- After max attempts:
  - mark as permanently failed for that symbol
  - prevent further attempts until mappings/settings change

Pseudocode:

```

function recordFailure(ticket, symbol, error): info = failedTrades.getOrAdd(ticket)
info.retryCount++ info.lastAttempt = now
if info.retryCount >= MaxRetryAttempts: info.permanentlyFailed = true
permanentlyFailedSymbols.add(symbol)
knownMasterTickets.add(ticket)
logError("Permanently failed;

fix mapping/settings then update a setting to retry.") else: delay = retryDelaysSeconds[info.retryCount - 1]
info.nextRetryTime = now + delay logError("Will retry in delay seconds")

```


### 9.2 Why retries recover quickly after mapping fixes
When symbol mappings are updated, the mapping service raises `MappingsChanged`. The engine responds by forcing non-permanent failures to retry immediately:


```

onMappingsChanged(): for each failedTrade where not permanentlyFailed: failedTrade.nextRetryTime = now

```


This aligns with the UX behavior:
- mapping is fixed
- engine retries without waiting for the original backoff window

---

## 10. Unique feature: close Master when Slave closes

This feature depends on 1-to-1 mapping between master and slave orders.

### 10.1 Detect slave close
If a tracked slave ticket is missing from current slave trades:

```

function handleSlaveClosedTickets(currentSlaveTickets, settings): closedTrackedOrders = tracked orders where tracked.slaveTicket not in currentSlaveTickets
for each tracked in closedTrackedOrders: remove tracking (slaveToMaster + masterToSlave)

if settings.closeMasterOnSlaveClosed:
  pendingMasterCloses.add(tracked.masterTicket)
  async closeMasterWithRetry(tracked)
else:
  recentlyClosedMasters[tracked.masterTicket] = now

```


### 10.2 Close master with retry

```

async function closeMasterWithRetry(tracked): for attempt in 1..N:
if tryCloseMaster(tracked.masterTicket): pendingMasterCloses.remove(tracked.masterTicket)
recentlyClosedMasters[tracked.masterTicket] = now
emitMasterClosedEvent(...)
persistState() return
await delay(500ms * attempt)
pendingMasterCloses.remove(tracked.masterTicket)
logError("Failed to close master after retries")

```



### 10.3 Why “recently closed” exists
The engine maintains a short “recently closed” window so that a master ticket that was just handled does not get re-copied due to timing jitter.

---

## 11. Orphan cleanup behavior

### 11.1 Orphaned slave trades
If a master ticket disappears but the mapping still exists, close the slave trade:


```

function closeOrphanedSlaveTrades(currentMasterTickets): toClose = tracked orders where tracked.masterTicket not in currentMasterTickets and tracked.slaveTicket not in pendingSlaveCloses
mark pendingSlaveCloses for each async closeSlave(tracked)

```



### 11.2 Stale mappings
If the slave ticket is missing but the engine didn’t observe a clean slave-close event (crash/restart/race), stale mappings are removed and master ticket placed into “recently closed” unless it is pending master close.

---

## 12. Reconnection strategy (broker-side)

Two important principles:
1. **Use actual client connection state**, not only UI flags.
2. **Avoid concurrent reconnect attempts** for the same account.

Pseudocode:


```

async function tryReconnectWithBackoff(account): if accountsBeingReconnected contains account.loginId: return
accountsBeingReconnected.add(account.loginId) attempts = reconnectAttempts[loginId] + 1
success = reconnectAccount(account) // serialized via a coordinator gate
if success: resetReconnectState(loginId) log("Reconnected")
else: delay = exponentialBackoff(attempts)
nextReconnectTime[loginId] = now + delay
logError("Reconnect failed; retry scheduled")
accountsBeingReconnected.remove(loginId)

```



---

## 13. Persisted session state (conceptual)

To reduce duplication after restarts and to support close logic, the engine can persist:
- master↔slave tracked order mappings
- pending master closes
- known master tickets

On new session start, persisted state is only reused if it matches the same master/slave account titles.

---

## 14. Notes on scalability

The engine is designed as a reusable service with:
- stable input contracts (accounts + settings + mapping service)
- event-driven outputs (logs + lifecycle events)

A future **“1 master → N slaves”** view can reuse most of the engine logic by:
- abstracting the Slave side into a list of targets
- adjusting mapping/tracked order data structures to include slave identity
- rethinking the semantics of “close master when slave closes” (not generally practical for N slaves)

---
