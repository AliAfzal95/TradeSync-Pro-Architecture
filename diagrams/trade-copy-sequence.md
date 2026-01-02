```mermaid block

flowchart TB
  Start["StartCopying(master, slave, settings)"] --> Load["LoadPersistedState()\nread trade-mappings.json if present"]
  Load --> Timer["Start polling timer\n200ms tick"]

  Timer --> Tick["Poll tick"]
  Tick --> Conn{"Master and Slave connected?"}

  Conn -- No --> Pause["Skip copying\nLog + schedule reconnect (backoff)"]
  Pause --> Tick

  Conn -- Yes --> Fetch["Fetch open orders\nMaster + Slave"]
  Fetch --> Diff["Diff + reconcile\nProcessTrades(...)"]
  Diff --> New{"New master trade detected?"}

  New -- No --> Persist["Save state (optional)\ntrade-mappings.json"]
  Persist --> Tick

  New -- Yes --> Map["Resolve symbol mapping\nSymbolMappingService:\nmasterSymbol -> candidates"]
  Map --> Sub["Ensure slave symbol subscribed\nif required"]
  Sub --> Delay{"Delay enabled?"}

  Delay -- Yes --> Wait["Wait delayMs"]
  Delay -- No --> Open["Open slave order\nTP/SL optionally copied"]
  Wait --> Open

  Open --> Ok{"Order accepted?"}

  Ok -- Yes --> Track["Track mapping\nmasterTicket -> slaveTicket"]
  Track --> RaiseOk["Raise events\nOnTradeCopied + OnLogMessage"]
  RaiseOk --> Persist2["Persist state\ntrade-mappings.json atomic write"]
  Persist2 --> Tick

  Ok -- No --> Fail["Record failure\nMarkTradeAsFailed(...)"]
  Fail --> RaiseErr["Raise events\nOnError + OnLogMessage"]
  RaiseErr --> Tick

  Stop["StopCopying()"] --> StopTimer["Stop timer"]
  StopTimer --> Clear["ClearPersistedState()\ndelete trade-mappings.json"]
  Clear --> End["Idle"]
