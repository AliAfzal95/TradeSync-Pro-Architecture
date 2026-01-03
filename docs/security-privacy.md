# TradeSync Pro — Security & Privacy (Public Summary)

This document summarizes what TradeSync Pro stores locally, what it does **not** store, how secrets are handled, and how the public architecture repo avoids exposing sensitive information.

> This is a **high-level security and privacy summary** intended for a public repository. It does not include proprietary endpoints, secrets, or implementation details.

---

## 1. Security goals

TradeSync Pro is built around these security goals:

- **No secrets in source control** (this repo contains docs only).
- **Minimal local persistence** needed for usability and reliability.
- **User-scoped encryption** for licensing state using Windows DPAPI.
- **Fail-safe license enforcement**: if license cannot be verified beyond a time window, trading actions are stopped.

---

## 2. What is stored locally (and why)

### 2.1 Account configuration (local, user-provided)
Stored to allow one-time setup and faster daily operation.

Typical fields (conceptual):
- account type (MT4/MT5)
- account title / label
- broker connection target (server host/IP and port)
- login identifier

### 2.2 Symbol mappings (local JSON)
Stored so symbol translations persist across app restarts.

- Master symbol → one or more slave symbol candidates
- Supports fallback mapping to reduce broker symbol mismatch failures

### 2.3 Trade session state (local JSON)
Stored to support reliability and prevent duplication during restarts.

High-level state includes:
- master↔slave ticket mappings
- pending master close intent (for “close master on slave close” feature)
- known tickets list (guards against repeated copy loops)

### 2.4 License state (local, encrypted)
Stored to allow the app to remember activation state without prompting every startup.

- Stored under the user profile as an encrypted file.
- Protected using **Windows DPAPI** with `DataProtectionScope.CurrentUser`.

This means:
- the license file is bound to the current Windows user profile
- it is not portable across machines/users

---

## 3. What is NOT stored locally

TradeSync Pro is designed not to persist the following:

- **No private keys** (no signing keys, no certificate private keys).
- **No hard-coded secrets in the public repo** (endpoints, tokens, keys are excluded).
- **No raw license keys** committed anywhere in this repository.
- **No brokerage trade history databases** (only transient session state needed for the copy engine).
- **No personally identifying information (PII)** by design (beyond what a user enters as local account labels).

---

## 4. Network security and data flow (high level)

### 4.1 Broker connectivity (MT4/MT5 APIs)
- TradeSync Pro connects directly to broker servers via MT4/MT5 API libraries (not via MT terminals/EAs).
- Data handled at runtime:
  - open orders/positions
  - order placement and closing responses
  - account balance/equity for display

This repo does not publish:
- broker endpoints
- protocol details
- implementation specifics for each API library

### 4.2 Licensing server connectivity
- License activation and heartbeat validation occur over HTTPS.
- The app implements:
  - periodic heartbeat checks
  - offline grace period to tolerate temporary outages
  - invalidation logic if subscription expires or session is moved to another device

This repo does not publish:
- production URLs
- request/response payloads beyond conceptual descriptions

---

## 5. Handling failures safely (security + operational safety)

### 5.1 License invalidation behavior
If a license is invalidated or cannot be verified beyond the allowed grace period:
- active copy sessions are force-stopped
- accounts can be disconnected
- the user is returned to activation flow

This prevents prolonged operation in an unlicensed state.

### 5.2 Reconnection logic safety
Two systems can attempt recovery:
- an internet monitor + reconnection service (global outage recovery)
- engine-level broker reconnect (active session recovery)

To prevent unsafe concurrency:
- shared client modifications are serialized via a coordinator gate
- reconnect attempts are deduplicated per account


