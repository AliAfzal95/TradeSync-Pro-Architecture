# TradeSync Pro — Architecture

This repository documents the architecture of **TradeSync Pro**, a Windows desktop trade-copying application designed to copy trades between **one Master account** and **one Slave account** (MT4/MT5), with robust licensing enforcement, persistence, and connectivity recovery.

> Note: This repo intentionally contains **no proprietary source code**. It focuses on components, responsibilities, data flows, and reliability strategy.

---

## 1. High-level component map

TradeSync Pro is a WinForms application organized into:
- **UI Layer** (views/forms)
- **Shared State / Coordination**
- **Domain Services**
- **Connectivity & Recovery**
- **Persistence** (local JSON state)

### Core building blocks
- **`Form1`**: main shell, navigation, lifecycle, internet monitoring, license gating.
- **`ActivationForm`** + **`LicenseService`**: activation workflow + heartbeat verification.
- **`AccountsView`** + **`AddAccount`** + **`Account`**: account CRUD, connect/disconnect, persistence.
- **`SymbolMapView`** + **`SymbolMappingService`**: symbol mapping CRUD + centralized mapping lookup.
- **`ManageAccountsView`** + **`TradeCopyingService`**: active copy session UI + trade copying engine.
- **`InternetConnectivityMonitor`** + **`ReconnectionService`**: network status detection + reconnect orchestration.
- **`AccountCoordinator`**: shared account/client storage, serialized mutations, and active-session protection.

---

## 2. Data model and shared state

### `Account`
Represents one configured trading account:
- Type: `MT4` or `MT5`
- Title (user-friendly label)
- Login/password + server host/port
- Connection flags: `isConnected`, `connectionStatus`
- `accIndex` used to map to the correct MT4/MT5 API client instance.

### `AccountCoordinator` (singleton)
Central shared state container for the entire app:
- `Accounts`: `List<Account>`
- `MT4Clients`: `List<QuoteClient>`
- `MT5Clients`: `List<MT5API>`
- `Gate`: `SemaphoreSlim` to serialize sensitive client mutations across threads/services.
- `AccountsChanged` event to notify views.
- Active copy session tracking:
  - `SetActiveCopySession(master, slave)`
  - `IsAccountInActiveCopySession(loginId)`
  - `GetActiveCopySessionAccounts()`

**Key invariant**  
`Account.accIndex` must always correctly match the index of the corresponding client instance in:
- `MT4Clients` for MT4 accounts
- `MT5Clients` for MT5 accounts

This is why delete/edit logic reindexes accounts when an account is removed or changes type.

---

## 3. UI layer (views/forms)

### 3.1 `Form1` (application shell)
Responsibilities:
- Hosts the navigation and content panel for the views.
- Starts `InternetConnectivityMonitor`.
- Initializes `LicenseService` and enforces license validity.
- Handles `OnLicenseInvalidated`:
  - Force-stops copy session
  - Disconnects accounts
  - Prompts for activation again or exits

### 3.2 `ActivationForm`
Responsibilities:
- Captures an activation key from the user.
- Calls `LicenseService.CheckBeforeActivateAsync` to warn about device switching.
- Calls `LicenseService.ActivateAsync` to activate and persist a valid local license.

### 3.3 `AccountsView` + `AddAccount`
Responsibilities:
- Add/edit/delete accounts (stored locally as JSON).
- Connect/disconnect accounts via MT4 or MT5 API clients.
- Load accounts on startup (restores the app state).
- Prevent editing/deleting accounts that are in an **active copy session**.

### 3.4 `SymbolMapView`
Responsibilities:
- Provides a grid UI to map `MasterSymbol -> SlaveSymbol`.
- Saves mappings to JSON with debounce and atomic file replace.
- Pushes mapping updates into `SymbolMappingService` so the engine can use them immediately.

### 3.5 `ManageAccountsView`
Responsibilities:
- Select Master and Slave accounts from currently connected accounts.
- Start/stop copy session.
- Live-update copy settings during a session:
  - Fixed lot vs multiplier
  - Delay
  - Reverse direction
  - TP/SL copy option
  - Close master when slave closes
- Displays live open trades for selected accounts (independent timer).

---

## 4. Domain services

### 4.1 `LicenseService`
Responsibilities:
- Local license storage (encrypted) + session token management.
- Heartbeat polling to validate license periodically.
- Enforces invalidation actions (device switch / expired / prolonged offline).

Important behavior:
- Offline grace (time-boxed) for temporary network failures.
- `OnLicenseInvalidated` event escalated to UI for force-stop/disconnect.

### 4.2 `SymbolMappingService` (singleton)
Responsibilities:
- Central in-memory mapping store.
- Case-insensitive lookup:
  - `GetSlaveSymbol(masterSymbol)` for single mapping
  - `GetAllSlaveSymbols(masterSymbol)` to support fallback attempts
- Persists/loads mappings from `%AppData%/TradeSync Pro/symbol-map.json`.
- Raises `MappingsChanged` so copy engine can react immediately.

### 4.3 `TradeCopyingService` (copy engine)
Responsibilities:
- Runs a polling loop (short interval) to detect master trade changes.
- Copies new master trades to slave using:
  - Current `CopySettings`
  - Symbol mapping candidates (`GetAllSlaveSymbols`)
- Tracks master/slave ticket mappings (`TrackedOrder`) to manage lifecycle.
- Implements the **unique session feature**:
  - *Close master trade when slave trade closes* (supported for 1-to-1 sessions).
- Provides events to UI:
  - logs, errors, trade copied, trade closed, master closed.

Engine characteristics:
- Uses internal state to prevent double-copy or loop behavior:
  - known ticket tracking
  - pending lists for operations
  - recently-closed window
- Supports “retry with delay” when trade placement fails, and resets retry timing when mappings change.

---

## 5. Connectivity and recovery

### 5.1 `InternetConnectivityMonitor`
Responsibilities:
- Poll-based internet check (ICMP ping).
- Raises `StatusChanged` events when online/offline changes.

### 5.2 `ReconnectionService` (singleton)
Responsibilities:
- Captures accounts that were connected when internet is lost.
- Marks them as reconnecting and attempts reconnection on restore.
- Throttles parallel reconnect attempts (prevents overload).
- Uses retry with exponential backoff.

### 5.3 Broker-side disconnect handling (active session)
In addition to internet detection, the copy engine checks the **actual client connection** for the active master/slave accounts and can trigger reconnection attempts even if the device still has internet. This addresses the case where public endpoints are reachable but broker connectivity is disrupted.

### 5.4 Concurrency safety
Both reconnection paths can touch MT4/MT5 client instances. To avoid conflicts:
- `AccountCoordinator.Gate` is used to serialize sensitive operations that mutate shared client instances.
- Reconnection services track “accounts being reconnected” to avoid duplicate work.

---

## 6. Persistence model

TradeSync Pro persists one-time configuration and session state into `%AppData%`:

- Accounts:
  - `%AppData%/TradeSync Pro/data/accounts.json`
- Symbol mappings:
  - `%AppData%/TradeSync Pro/symbol-map.json`
- Trade-copy mapping state (to avoid duplicate trade copying after restart):
  - `%AppData%/TradeSync Pro/trade-mappings.json`

Persistence is implemented with:
- Debounced writes (where appropriate)
- Atomic replace patterns (`.tmp` then replace/move)
- Safe load behavior (corrupt or missing files fall back to empty state)

---

## 7. Key runtime flows (summary)

### 7.1 Startup flow
1. `Form1` initializes.
2. License is validated; activation prompt shown if invalid.
3. Accounts and mappings are loaded from JSON.
4. Internet monitor begins tracking connectivity.

### 7.2 “Start Copying” flow
1. User selects Master and Slave accounts (connected only).
2. UI validates settings and connection state.
3. Coordinator marks active copy session.
4. `TradeCopyingService.StartCopying(master, slave, settings)` starts polling loop.
5. Engine maps and copies new trades; tracks ticket mapping.

### 7.3 “Close Master when Slave closes”
1. Engine tracks MasterTicket ↔ SlaveTicket for each copied trade.
2. When a slave position disappears, engine:
   - marks the master ticket pending close
   - attempts to close the master trade with retry
3. “pending close” state prevents re-copying during the close lifecycle.

### 7.4 License invalidation flow
1. Heartbeat detects invalidation/expiry.
2. App force-stops copy session.
3. Disconnects accounts and cleans in-memory session flags.
4. Prompts user to activate again or exits.

---

## 8. What is intentionally omitted from this repo
- Proprietary source code and broker API integration details
- Production endpoints, keys, tokens, and sensitive configuration
- Any trade strategy-specific parameters or internal heuristics beyond architecture-level description
