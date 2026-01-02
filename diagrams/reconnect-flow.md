```mermaid block

flowchart TB
  Mon["InternetConnectivityMonitor\npoll ~3s + ICMP ping"] --> Status{"StatusChanged?"}

  Status -- "Offline" --> Capture["ReconnectionService.CaptureConnectedAccounts()"]
  Capture --> Mark["Mark accounts\nconnectionStatus=Reconnecting...\nisConnected=false"]
  Mark --> Notify1["AccountCoordinator.NotifyAccountsChanged()"]
  Notify1 --> UiLost["InternetLost event\nManageAccountsView logs:\nConnection lost... / Trying to reconnect..."]

  Status -- "Online" --> Restored["ReconnectionService.OnInternetRestoredAsync()"]
  Restored --> UiRestored["InternetRestored event\nManageAccountsView logs:\nConnection recovered..."]
  Restored --> ReAll["ReconnectAllAccountsAsync()"]

  ReAll --> StartEvt["ReconnectionStarted event"]
  StartEvt --> FanOut["For each previously connected account\nstart ReconnectAccountWithRetryAsync\nparallel; throttle=4 for remaining"]

  FanOut --> Dedup{"Already being reconnected?"}
  Dedup -- "Yes" --> Skip["Skip duplicate reconnect attempt"]
  Dedup -- "No" --> Loop["Retry loop\nmax 10 attempts\ninitial 2s, x1.5 backoff, cap 60s"]

  Loop --> Already{"Client already connected?"}
  Already -- "Yes" --> Ok1["Set Connected\nNotify + StatusChanged(Already connected)"]
  Already -- "No" --> Attempt["AttemptReconnectAsync\nper attempt timeout 15s"]

  Attempt --> Gate{"Acquire AccountCoordinator.Gate\nwait up to 5s"}
  Gate -- "No" --> GateFail["Fail attempt\nwill retry with backoff"]
  Gate -- "Yes" --> Recheck{"Recheck connected?"}

  Recheck -- "Yes" --> GateOk["Set Connected\nrelease gate"]
  Recheck -- "No" --> Create{"Create new client by account type"}
  Create -- "MT4" --> Mt4["Disconnect old client\nnew QuoteClient(login,pw,ip,port).Connect()"]
  Create -- "MT5" --> Mt5["Disconnect old client\nnew MT5API(login,pw,ip,port).Connect()"]

  Mt4 --> SetOk["Set isConnected=true\nconnectionStatus=Connected"]
  Mt5 --> SetOk
  SetOk --> Notify2["AccountCoordinator.NotifyAccountsChanged()"]
  Notify2 --> StatusOk["ReconnectionStatusChanged\n✓ Reconnected (attempt N)"]
  StatusOk --> Done["Release gate"]

  Attempt --> Timeout["Timeout\nconnectionStatus=Timeout\nNotify + StatusChanged(Timeout)"]
  GateFail --> Wait["Delay backoff then retry"]
  Timeout --> Wait
  Wait --> Loop

  Loop --> Failed{"Attempts exhausted?"}
  Failed -- "Yes" --> FinalFail["Set Disconnected\nNotify + StatusChanged(Failed)"]
  Failed -- "No" --> Loop

  Ok1 --> Complete
  GateOk --> Complete
  FinalFail --> Complete

  Complete["ReconnectionCompleted event\nClearPreviouslyConnectedAccounts()"]
