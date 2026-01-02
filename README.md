# TradeSync-Pro-Architecture
Architecture &amp; system design docs for TradeSync Pro — a .NET (WinForms) MT4/MT5 trade copier with licensing, symbol mapping, runtime controls, and resilient reconnection (codebase private).


## Overview
TradeSync Pro is a desktop trade-copying application for MetaTrader 4/5 client accounts.  
It enables **real-time trade replication** between a selected **Master** and **Slave** account without requiring MT4/MT5 terminals to remain open, by using third-party MT4/MT5 .NET APIs to monitor positions and place orders directly on broker servers.

This repository contains **system design and architecture documentation only** (no proprietary source code).

---

## Key Features
- **Master → Slave trade copying (MT4/MT5)** using broker-server APIs (not terminal/EAs).
- **Account management**
  - Add/edit/delete multiple accounts
  - Connect/disconnect individual or all accounts
  - Persistent local storage (JSON) for one-time setup
- **Symbol Mapping**
  - Master-to-Slave symbol translation
  - Supports **multiple mapping fallbacks** per symbol (retry until a valid symbol works)
  - Persistent local storage (JSON)
- **Runtime trade controls (while copying)**
  - Lot sizing: multiplier or fixed lot
  - Optional delay before copying
  - Reverse trade direction
  - Optional TP/SL copy mode
- **Unique session feature**
  - **Close Master trades when Slave trades are closed** (designed for 1 Master ↔ 1 Slave sessions)
- **Production reliability**
  - Robust logging for all trade actions and failure cases
  - Automatic retry for failed trade placements (symbol mapping mistakes, temporary errors)
  - Internet/broker reconnection logic with backoff

---

## Architecture
### Main UI / Modules
- **Activation / Licensing**
  - License activation + heartbeat validation
  - Local encrypted license storage
  - Force-stop trading and disconnect accounts if license becomes invalid
- **Accounts View**
  - Manages account CRUD + connection lifecycle
  - Persists accounts to JSON and restores on startup
- **Symbol Mapping View**
  - Maintains Master↔Slave symbol mappings
  - Persists mappings to JSON
- **Manage Accounts View**
  - Select Master and Slave accounts
  - Starts/stops copying session
  - Live trade monitoring grids (balance/equity/open trades)
  - Live settings updates during active copying

### Core Services (shared across views)
- **AccountCoordinator**
  - Shared in-memory state of accounts and connected MT4/MT5 API clients
  - Synchronizes access and prevents conflicting updates
  - Tracks accounts participating in an active copy session (prevents editing/deleting in-use accounts)
- **TradeCopyingService (copy engine)**
  - Polls master account orders at short intervals
  - Performs symbol mapping lookup and order placement on slave
  - Tracks Master↔Slave ticket mappings for lifecycle management
  - Supports “close master when slave closes” workflow using tracked mapping state
- **InternetConnectivityMonitor + ReconnectionService**
  - Detects internet loss (ICMP ping)
  - Reconnects previously connected accounts on restore
  - Handles broker-side disconnects during active sessions

### Data persistence
- Accounts stored under AppData as JSON (restored on restart)
- Symbol mapping stored under AppData as JSON (restored on restart)
- Trade mapping state stored to survive restarts during sessions (prevents duplicate copy / supports close operations)

See `docs/` for detailed diagrams and workflows.

---

## Error Handling & Reliability
TradeSync Pro is designed to minimize missed trades and reduce operational failure cases:

- **Trade placement retry strategy**
  - When a trade cannot be placed (e.g., mapping issue or invalid stops), the system logs the reason, retries with backoff, and can retry immediately after symbol mappings change.
- **Symbol mapping guardrails**
  - Multiple slave symbol candidates can be attempted for a single master symbol to handle broker symbol differences.
- **Connectivity resilience**
  - Internet status detection triggers reconnection attempts for accounts that were previously connected.
  - Active session accounts are monitored for broker-side disconnects and reconnection is attempted with backoff.
- **Safety controls**
  - During an active copy session, master/slave accounts are locked from editing/deletion.
  - License invalidation triggers immediate stop of copy operations and disconnects accounts.


---

## Why the code is private
This repository documents the architecture and system design only. The full implementation is not published because TradeSync Pro is a commercial, licensed product.

**What is public here**
- High-level architecture and workflow
- Trade copying lifecycle and state-machine description
- Reliability and reconnection strategy
- Redacted screenshots and UX walkthrough

**What is not public**
- Proprietary source code and trade-copy logic
- Broker/API integration details and production endpoints
- Licensing server implementation details
- Any secrets, keys, or private configuration

If you are a recruiter/hiring manager and would like a deeper technical walkthrough, I can provide a private demo and/or a redacted code walkthrough under NDA.

