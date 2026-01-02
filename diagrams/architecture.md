```mermaid block

flowchart TB

  App["TradeSync Pro (WinForms App)"] --> Shell["MainForm (Navigation)"]

  %% --- Licensing ---
  Shell --> License["LicenseService"]
  Shell --> Activation["ActivationForm"]
  License --> LicenseStore["Local License Store (DPAPI)\n%LocalAppData%/TradeSync Pro/license.dat"]
  License --> LicenseServer["License Server (HTTPS)\n(activation + heartbeat)"]

  %% --- Connectivity + Reconnection ---
  Shell --> NetMon["InternetConnectivityMonitor\n(ICMP ping)"]
  NetMon --> ReconnectSvc["ReconnectionService\n(throttled reconnect + backoff)"]
  ReconnectSvc --> Coordinator["AccountCoordinator (Singleton)\nAccounts + MT4/MT5 client lists + Gate"]

  %% --- UI Views ---
  Shell --> AccountsView["AccountsView\n(Add/Edit/Delete + Connect/Disconnect)"]
  Shell --> SymbolMapView["SymbolMapView\n(Symbol mapping grid)"]
  Shell --> ManageView["ManageAccountsView\n(Session control + live trade grids)"]

  %% --- Services used by views ---
  AccountsView --> Coordinator
  ManageView --> Coordinator

  SymbolMapView --> SymbolSvc["SymbolMappingService (Singleton)\nIn-memory mapping + persistence"]
  ManageView --> TradeEngine["TradeCopyingService\n(Polling engine + tracking + retries)"]
  TradeEngine --> SymbolSvc
  TradeEngine --> Coordinator

  %% --- Broker APIs (abstracted) ---
  Coordinator --> MT4["MT4 API Clients\n(QuoteClient / OrderClient)"]
  Coordinator --> MT5["MT5 API Clients\n(MT5API)"]

  %% --- Persistence ---
  SymbolSvc --> SymbolMapJson["Symbol Mapping JSON\n%AppData%/TradeSync Pro/symbol-map.json"]
  TradeEngine --> TradeStateJson["Trade Session State JSON\n%AppData%/TradeSync Pro/trade-mappings.json"]

  AccountsView --> AccountsStore["Accounts JSON (local)\n%AppData%/TradeSync Pro/.../accounts.json"]

  %% --- Runtime safety ---
  Coordinator --> Gate["Coordinator Gate (SemaphoreSlim)\nSerializes client mutations"]

  %% --- Notes ---
  classDef core fill:#eef7ff,stroke:#2b6cb0,stroke-width:1px,color:#111;
  classDef ui fill:#f7fafc,stroke:#4a5568,stroke-width:1px,color:#111;
  classDef svc fill:#f0fff4,stroke:#2f855a,stroke-width:1px,color:#111;
  classDef store fill:#fffaf0,stroke:#b7791f,stroke-width:1px,color:#111;

  class Shell,Coordinator core;
  class AccountsView,SymbolMapView,ManageView,Activation ui;
  class TradeEngine,SymbolSvc,ReconnectSvc,NetMon,License svc;
  class AccountsStore,SymbolMapJson,TradeStateJson,LicenseStore store;
