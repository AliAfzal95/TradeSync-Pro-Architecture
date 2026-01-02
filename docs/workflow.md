# TradeSync Pro — End-to-End Workflow

This document describes the complete runtime workflow of TradeSync Pro from app startup through active trade copying, error handling, reconnection, and shutdown.

> This is a documentation-only repository. All steps below are derived from the observed architecture and runtime responsibilities, not from publicly published source.

---

## 1. Startup and license gate

### 1.1 App start
1. Application launches and the main shell (`Form1`) is created.
2. `Form1` initializes:
   - Navigation views
   - `InternetConnectivityMonitor` (continuous connectivity checks)
   - `LicenseService` (loads local license and starts heartbeat when valid)

### 1.2 Local license validation
1. On initial load, the app checks the local license state:
   - If license is valid: app continues normally.
   - If invalid or expired: the main window is hidden and activation is required.

### 1.3 Activation workflow
1. User opens the activation dialog (`ActivationForm`).
2. User enters activation key.
3. App performs a pre-check (`CheckBeforeActivateAsync`) to detect if the license is active on another device.
4. If active on another device, the user must explicitly confirm proceeding.
5. App activates (`ActivateAsync`), stores encrypted license locally, and starts heartbeat polling.
6. If activation succeeds, main UI becomes usable.

---

## 2. One-time setup: Accounts

### 2.1 Add accounts (MT4/MT5)
1. In **Accounts** view, user clicks add account.
2. User inputs:
   - Account type (MT4/MT5)
   - Title (label)
   - Login credentials
   - Server host and port
3. User may run “Get IP” helper scripts (optional) for broker host discovery.
4. User can test connectivity and validate the entered credentials before saving.
5. Account is saved into local storage and appears in the accounts grid.

### 2.2 Account persistence
- On app close, accounts are persisted to JSON under `%AppData%`.
- On next startup, accounts are loaded automatically and the account list is reconstructed.

### 2.3 Connect accounts
1. User selects an account and clicks connect (or uses “Connect All”).
2. The app creates a dedicated API client instance per account:
   - MT4: `QuoteClient`
   - MT5: `MT5API`
3. Upon success:
   - `isConnected` becomes true
   - `connectionStatus` becomes `Connected`
   - balance/equity are displayed in the grid

---

## 3. One-time setup: Symbol Mapping

### 3.1 Configure symbol mapping
1. User opens **Symbol Mapping** view.
2. User maps:
   - Master symbol (e.g., `XAUUSD`)
   - Slave symbol (e.g., `GOLD`, `XAUUSDm`, `XAUUSD#`)
3. Multiple slave symbols can be mapped to the same master symbol to support broker differences.

### 3.2 Symbol mapping persistence and propagation
1. Mapping grid changes are saved to JSON under `%AppData%/TradeSync Pro/symbol-map.json`.
2. Mappings are also pushed into `SymbolMappingService`, the centralized in-memory store used by the copy engine.
3. When mappings change, the service raises `MappingsChanged`, which can trigger immediate retries for previously failed trades.

---

## 4. Start an active copy session (Master ↔ Slave)

### 4.1 Select accounts in Manage Accounts
1. User opens **Manage Accounts** view.
2. Master and Slave combo boxes only show **connected accounts**.
3. User selects:
   - One Master account
   - One Slave account

> The application is intentionally designed for **one Master and one Slave** to support “close master on slave close” reliably.

### 4.2 Configure copy settings (before start)
User configures the session parameters:
- Lot sizing:
  - Fixed lot mode, or
  - Multiplier mode
- Reverse direction (optional)
- Delay (optional)
- Copy TP/SL (optional)
- Close master when slave closes (optional)

### 4.3 Start Copying
1. User clicks **Start Copying**.
2. UI validates:
   - Master and Slave selected
   - Both accounts exist
   - Both accounts are connected
   - Settings are valid (lot size, delay numeric, etc.)
3. Coordinator marks the session as active:
   - This prevents editing/deleting the active master/slave accounts while copying.
4. `TradeCopyingService.StartCopying(master, slave, settings)` starts the engine polling loop.

---

## 5. Trade copying lifecycle (core engine behavior)

### 5.1 Polling loop
The copy engine runs a periodic cycle:
1. Confirms the session is running and not paused.
2. Checks actual client connectivity for Master and Slave.
3. When connected:
   - Fetches current open trades from the Master account.
   - Fetches current open trades from the Slave account.
4. Runs reconciliation logic:
   - Copy new master trades
   - Close orphaned slave trades (if master trade disappears)
   - Handle slave-closed trades (optionally close master)

### 5.2 New trade detected on Master
When a new trade appears on Master:
1. Engine checks internal state to avoid duplicates:
   - not already mapped
   - not pending close
   - not in a "recently closed" window
   - not permanently failed due to repeated mapping issues
2. Engine determines slave symbol candidates:
   - Calls `GetAllSlaveSymbols(masterSymbol)`
   - Returned list may include multiple slave symbols (fallback candidates)
3. Engine pre-subscribes slave account to candidate symbols to reduce order placement latency.
4. Engine applies settings:
   - delay (optional)
   - lot sizing (fixed or multiplier)
   - reverse direction (optional)
   - TP/SL copy logic (optional and conditional)
5. Engine attempts to open the trade on Slave:
   - Tries each candidate slave symbol until one succeeds.
6. On success:
   - Stores mapping: `MasterTicket ↔ SlaveTicket` in tracked state.
   - Logs “Copied trade”.
7. On failure:
   - Logs the failure reason.
   - Adds the trade to a retry schedule (backoff-based).
   - If failures exceed max attempts, the symbol can be flagged as “permanently failed” until mappings/settings change.

### 5.3 TP/SL handling
If TP/SL copy is enabled:
- TP and SL are applied when the trade direction is not reversed.
- If broker rejects stops (invalid stops), the engine can open the trade without TP/SL and logs the fallback.

---

## 6. Unique feature: Close Master when Slave closes

This feature depends on the 1 Master ↔ 1 Slave session design.

### 6.1 Slave trade disappears
1. The engine compares tracked SlaveTicket values against the slave’s currently opened trades.
2. If a tracked slave trade is no longer open:
   - The mapping is removed from active tracking.
   - Master ticket is placed into a “pending close” state (prevents re-copy loops).
3. If “Close master on slave closed” is enabled:
   - Engine attempts to close the corresponding master trade.
   - Retries close attempts if necessary.
4. When the master trade is confirmed closed (no longer present in master open trades):
   - The ticket is marked as completed and temporarily tracked as “recently closed” to avoid re-copy.

---

## 7. Live session controls and operational UX

### 7.1 Live settings updates
While copying is active:
- UI changes (lot mode, multiplier, delay, reverse, TP/SL, close-master behavior) are validated and pushed to the running engine using `UpdateSettings`.

### 7.2 Live trade display
Manage Accounts view also runs a separate timer to display:
- current open trades for selected Master and Slave
- balance / equity
- total PnL

This display is independent of whether the copying session is running.

### 7.3 Close all trades (manual action)
1. User can trigger “Close all trades” for selected Master and/or Slave.
2. If copying is active, the engine is paused temporarily to avoid race conditions.
3. Trades are closed (often in parallel).
4. If trades were closed during active copying:
   - tracked order mappings are cleared to prevent stale mappings.

---

## 8. Reliability and reconnection behavior

### 8.1 Internet connectivity loss
1. `InternetConnectivityMonitor` detects loss (ping-based).
2. `ReconnectionService` captures currently connected accounts and marks them as reconnecting.
3. UI can reflect reconnection state (e.g., `Reconnecting...` status).

### 8.2 Internet restored
1. `InternetConnectivityMonitor` detects restoration.
2. `ReconnectionService` attempts to reconnect previously connected accounts using:
   - throttled parallelism
   - backoff/retry logic

### 8.3 Broker-side disconnect while internet is up
If internet is available but broker connectivity drops:
- The active copy engine detects actual connection state per account and triggers reconnection attempts for master/slave so the session can recover.

---

## 9. License invalidation during runtime

If the license heartbeat fails due to expiration or device transfer:
1. `LicenseService` raises invalidation event.
2. UI force-stops any active copy session.
3. Accounts are disconnected.
4. App returns to activation prompt.
5. If user does not reactivate, the application exits.

---

## 10. Shutdown behavior

1. On close, views persist any pending state:
   - Accounts are saved to JSON.
   - Symbol mapping grid is flushed to JSON.
2. Copy engine stops and clears temporary session state.
3. Services and timers are disposed.
