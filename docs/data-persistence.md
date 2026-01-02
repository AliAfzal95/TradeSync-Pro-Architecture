# TradeSync Pro — Data Persistence (Local JSON Files)

This document describes the **local persistence model** used by TradeSync Pro: which files exist, where they live, what they contain (high-level), and when they are written.

> This repo is documentation-only. File names and behaviors below are based on the observed architecture and code paths, while intentionally omitting any sensitive data (credentials, tokens, endpoints).

---

## 1. Storage locations (Windows)

TradeSync Pro uses standard Windows per-user application folders:

- `%AppData%`  
  `Environment.SpecialFolder.ApplicationData`  
  Used for operational/state files (mappings, trade session state).

- `%LocalAppData%`  
  `Environment.SpecialFolder.LocalApplicationData`  
  Used for local license storage.

---

## 2. JSON state files in `%AppData%`

### 2.1 Symbol mapping store
**Path**
- `%AppData%\TradeSync Pro\symbol-map.json`

**Purpose**
- Persists user-configured **Master → Slave symbol mappings**.

**Logical schema**

```

{ "Rows": [ { "MasterSymbol": "XAUUSD", "SlaveSymbol": "GOLD" }
{ "MasterSymbol": "XAUUSD", "SlaveSymbol": "XAUUSDm" } ] }

```


**When it is written**
- When the user saves/edits mappings in the symbol mapping UI.
- Writes are typically:
  - JSON `WriteIndented = true`
  - atomic replace using a `.tmp` file then move/replace.

**When it is read**
- On application startup and/or when the mapping view/service initializes.
- Loaded into `SymbolMappingService` to support runtime lookups.

---

### 2.2 Trade copy session mapping state
**Path**
- `%AppData%\TradeSync Pro\trade-mappings.json`

**Purpose**
- Persists **session state** needed to prevent duplicate copying and to support lifecycle actions across restarts (e.g., master↔slave ticket mapping).

**High-level contents**
- Master and Slave account identity (titles)
- Ticket mapping lists:
  - MasterTicket → SlaveTicket mappings
  - SlaveTicket → MasterTicket mappings
- Pending close state (master tickets marked for close)
- Known master tickets (guardrail against duplicate work)
- Timestamp of last update

**When it is written**
- During an active copy session, after reconciliation steps complete.
- Also written after certain actions like clearing tracked orders.

**Write behavior**
- JSON is written with indentation.
- Uses atomic replace:
  - write to `trade-mappings.json.tmp`
  - then replace/move to `trade-mappings.json`.

**When it is read**
- When `TradeCopyingService.StartCopying(...)` is called:
  - load the file if present
  - restore only if the stored `MasterAccountTitle` and `SlaveAccountTitle` match the selected session accounts
  - otherwise, discard (delete) to avoid cross-session contamination.

**When it is deleted**
- When copying stops (`StopCopying()`), the persisted state is cleared to prevent stale session restoration.

---

## 3. Accounts persistence (local configuration)

TradeSync Pro persists account configuration to JSON (accounts list, display titles, connection target info).

**Known behavior (high-level)**
- Accounts are saved and restored between application runs.
- The accounts list must remain aligned with MT4/MT5 client lists via `accIndex` reindexing rules.

> The exact file path for the accounts JSON store is intentionally omitted from this public repository. Recruiters can assume it is stored under `%AppData%\TradeSync Pro\...` as an application-managed configuration file.

**When it is written**
- When a user adds/edits/deletes an account.
- Optionally on app shutdown (ensuring the latest state is stored).

**When it is read**
- On application startup to restore accounts for the UI.

**Security note**
- The production application stores account connection information locally. Any sensitive fields (credentials) should be treated as secrets and are not documented here beyond “stored locally”.

---

## 4. License persistence (not JSON)

Although the licensing workflow may exchange JSON with the license server, local license persistence is stored as an encrypted binary file, not JSON.

### 4.1 Local license file
**Path**
- `%LocalAppData%\TradeSync Pro\license.dat`

**Characteristics**
- Binary file encrypted with Windows DPAPI:
  - `ProtectedData.Protect(..., DataProtectionScope.CurrentUser)`
- Stores local session token + expiration timestamps.
- Deleted when license becomes invalidated or expires.

**When it is written**
- After successful activation.

**When it is read**
- On app startup to restore a valid session without requiring re-activation.

---

## 5. Atomic write strategy (why it matters)

For JSON state that must not corrupt easily (mappings/session state), the app uses:
1. Serialize to JSON
2. Write to `*.tmp`
3. Replace/move over the real file

This reduces:
- partial files on crash/power loss
- corrupt JSON mid-write
- “half old / half new” content issues

---

## 6. What is intentionally not persisted

To reduce risk and avoid irrecoverable inconsistency, some data is not meant to survive outside a controlled file:
- live transient retry timers (recomputed during runtime)
- process timers and UI-specific flags
- volatile in-memory caches (e.g., symbol subscription cache)

---

