<script>
  import { io } from "socket.io-client";
  import { onMount } from "svelte";

  const backendOrigin =
    import.meta.env.VITE_BACKEND_ORIGIN ||
    `${window.location.protocol}//${window.location.hostname}:5000`;

  const socket = io(backendOrigin, {
    transports: ["websocket", "polling"],
    reconnection: true,
  });

  let status = $state("Offline");
  let feedNonce = $state(Date.now());
  let feedError = $state("");
  let logs = $state([]);
  let terminalInput = $state("");
  let terminalHistory = $state([]);
  let activeDirection = $state(null);

  const feedUrl = $derived(`${backendOrigin}/video_feed?ts=${feedNonce}`);

  const addLog = (message, type = "info") => {
    const timestamp = new Date().toLocaleTimeString("en-US", { hour12: false });
    logs = [...logs.slice(-49), { timestamp, message, type }];
  };

  onMount(() => {
    socket.on("connect", () => {
      status = "Online";
      addLog("Connected to backend", "success");
    });

    socket.on("disconnect", () => {
      status = "Offline";
      addLog("Disconnected from backend", "error");
    });

    socket.on("connect_error", () => {
      status = "Offline";
      addLog("Connection error", "error");
    });

    socket.on("log", (data) => {
      addLog(data.message, data.type || "info");
    });

    socket.on("terminal_output", (data) => {
      terminalHistory = [...terminalHistory, { type: "output", content: data.output }];
    });

    addLog("Initializing controller...", "info");

    return () => {
      socket.disconnect();
    };
  });

  const sendMove = (dir) => {
    socket.emit("move", { direction: dir });
    if (dir !== "stop") {
      activeDirection = dir;
      addLog(`Move: ${dir.toUpperCase()}`, "info");
    } else {
      activeDirection = null;
    }
  };

  const reloadFeed = () => {
    feedError = "";
    feedNonce = Date.now();
    addLog("Reloading camera feed...", "info");
  };

  const executeCommand = () => {
    if (!terminalInput.trim()) return;
    
    const cmd = terminalInput.trim();
    terminalHistory = [...terminalHistory, { type: "command", content: cmd }];
    socket.emit("ssh_command", { command: cmd });
    addLog(`Executed: ${cmd}`, "info");
    terminalInput = "";
  };

  const handleKeydown = (e) => {
    if (e.key === "Enter") {
      executeCommand();
    }
  };

  // Keyboard Listeners for movement
  const handleGlobalKeydown = (e) => {
    if (e.target.tagName === "INPUT") return;
    const key = e.key.toLowerCase();
    if (["w", "a", "s", "d"].includes(key)) {
      e.preventDefault();
      sendMove(key);
    }
  };

  const handleGlobalKeyup = (e) => {
    if (e.target.tagName === "INPUT") return;
    const key = e.key.toLowerCase();
    if (["w", "a", "s", "d"].includes(key)) {
      sendMove("stop");
    }
  };
</script>

<svelte:window onkeydown={handleGlobalKeydown} onkeyup={handleGlobalKeyup} />

<main class="controller">
  <!-- Header -->
  <header class="header">
    <div class="header-left">
      <h1 class="title">SEWBOT</h1>
      <span class="version">v1.0</span>
    </div>
    <div class="header-right">
      <div class="status-group">
        <span class="status-label">STATUS</span>
        <span class="status-value" class:online={status === "Online"}>
          <span class="status-dot"></span>
          {status}
        </span>
      </div>
      <div class="status-group">
        <span class="status-label">BACKEND</span>
        <span class="status-value backend">{backendOrigin}</span>
      </div>
    </div>
  </header>

  <!-- Main Grid -->
  <div class="main-grid">
    <!-- Camera Feed -->
    <section class="panel camera-panel">
      <div class="panel-header">
        <span class="panel-title">CAMERA FEED</span>
        <button class="btn-icon" onclick={reloadFeed} title="Refresh feed">
          <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M23 4v6h-6M1 20v-6h6"/>
            <path d="M3.51 9a9 9 0 0114.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0020.49 15"/>
          </svg>
        </button>
      </div>
      <div class="feed-container">
        {#if feedError}
          <div class="feed-error">
            <svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <circle cx="12" cy="12" r="10"/>
              <line x1="12" y1="8" x2="12" y2="12"/>
              <line x1="12" y1="16" x2="12.01" y2="16"/>
            </svg>
            <span>{feedError}</span>
            <button class="btn-retry" onclick={reloadFeed}>Retry</button>
          </div>
        {:else}
          <img
            src={feedUrl}
            class="feed"
            alt="Robot camera feed"
            onerror={() => (feedError = "Feed unavailable")}
            onload={() => (feedError = "")}
          />
        {/if}
        <div class="feed-overlay">
          <span class="recording-indicator">
            <span class="rec-dot"></span>
            REC
          </span>
        </div>
      </div>
    </section>

    <!-- Controls Panel -->
    <section class="panel controls-panel">
      <div class="panel-header">
        <span class="panel-title">CONTROLS</span>
        <span class="panel-hint">WASD or click</span>
      </div>
      <div class="controls-grid">
        <div class="control-row">
          <button 
            class="control-btn" 
            class:active={activeDirection === "w"}
            onmousedown={() => sendMove("w")}
            onmouseup={() => sendMove("stop")}
            onmouseleave={() => activeDirection === "w" && sendMove("stop")}
          >
            <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M12 19V5M5 12l7-7 7 7"/>
            </svg>
            <span class="key-hint">W</span>
          </button>
        </div>
        <div class="control-row">
          <button 
            class="control-btn" 
            class:active={activeDirection === "a"}
            onmousedown={() => sendMove("a")}
            onmouseup={() => sendMove("stop")}
            onmouseleave={() => activeDirection === "a" && sendMove("stop")}
          >
            <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M19 12H5M12 19l-7-7 7-7"/>
            </svg>
            <span class="key-hint">A</span>
          </button>
          <button 
            class="control-btn" 
            class:active={activeDirection === "s"}
            onmousedown={() => sendMove("s")}
            onmouseup={() => sendMove("stop")}
            onmouseleave={() => activeDirection === "s" && sendMove("stop")}
          >
            <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M12 5v14M5 12l7 7 7-7"/>
            </svg>
            <span class="key-hint">S</span>
          </button>
          <button 
            class="control-btn" 
            class:active={activeDirection === "d"}
            onmousedown={() => sendMove("d")}
            onmouseup={() => sendMove("stop")}
            onmouseleave={() => activeDirection === "d" && sendMove("stop")}
          >
            <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M5 12h14M12 5l7 7-7 7"/>
            </svg>
            <span class="key-hint">D</span>
          </button>
        </div>
      </div>
      <div class="emergency-stop">
        <button class="btn-emergency" onclick={() => sendMove("stop")}>
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <rect x="6" y="6" width="12" height="12"/>
          </svg>
          EMERGENCY STOP
        </button>
      </div>
    </section>

    <!-- Logs Panel -->
    <section class="panel logs-panel">
      <div class="panel-header">
        <span class="panel-title">SYSTEM LOGS</span>
        <span class="log-count">{logs.length}</span>
      </div>
      <div class="logs-container">
        {#each logs as log}
          <div class="log-entry" class:log-success={log.type === "success"} class:log-error={log.type === "error"}>
            <span class="log-time">{log.timestamp}</span>
            <span class="log-message">{log.message}</span>
          </div>
        {/each}
        {#if logs.length === 0}
          <div class="logs-empty">No logs yet</div>
        {/if}
      </div>
    </section>

    <!-- Terminal Panel -->
    <section class="panel terminal-panel">
      <div class="panel-header">
        <span class="panel-title">SSH TERMINAL</span>
        <span class="terminal-status">
          <span class="status-dot" class:online={status === "Online"}></span>
          {status === "Online" ? "Connected" : "Disconnected"}
        </span>
      </div>
      <div class="terminal-container">
        <div class="terminal-output">
          {#each terminalHistory as entry}
            {#if entry.type === "command"}
              <div class="terminal-line command">
                <span class="prompt">$</span>
                <span>{entry.content}</span>
              </div>
            {:else}
              <div class="terminal-line output">{entry.content}</div>
            {/if}
          {/each}
          {#if terminalHistory.length === 0}
            <div class="terminal-welcome">
              <span>Sewbot SSH Terminal</span>
              <span class="terminal-hint">Enter commands to execute on the robot</span>
            </div>
          {/if}
        </div>
        <div class="terminal-input-row">
          <span class="prompt">$</span>
          <input 
            type="text" 
            class="terminal-input" 
            placeholder="Enter command..."
            bind:value={terminalInput}
            onkeydown={handleKeydown}
            disabled={status !== "Online"}
          />
        </div>
      </div>
    </section>
  </div>
</main>

<style>
  .controller {
    min-height: 100vh;
    padding: 16px;
    display: flex;
    flex-direction: column;
    gap: 16px;
    background: var(--bg-primary);
  }

  /* Header */
  .header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 12px 16px;
    background: var(--bg-secondary);
    border: 1px solid var(--border-default);
    border-radius: 8px;
  }

  .header-left {
    display: flex;
    align-items: baseline;
    gap: 8px;
  }

  .title {
    margin: 0;
    font-size: 18px;
    font-weight: 600;
    letter-spacing: 0.1em;
    color: var(--text-primary);
  }

  .version {
    font-size: 11px;
    color: var(--text-muted);
  }

  .header-right {
    display: flex;
    gap: 24px;
  }

  .status-group {
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  .status-label {
    font-size: 10px;
    color: var(--text-muted);
    letter-spacing: 0.05em;
  }

  .status-value {
    font-size: 12px;
    color: var(--text-secondary);
    display: flex;
    align-items: center;
    gap: 6px;
  }

  .status-value.online {
    color: var(--accent-green);
  }

  .status-value.backend {
    font-size: 11px;
    max-width: 200px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .status-dot {
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: var(--accent-red);
  }

  .status-value.online .status-dot,
  .status-dot.online {
    background: var(--accent-green);
    box-shadow: 0 0 8px var(--accent-green);
  }

  /* Main Grid */
  .main-grid {
    flex: 1;
    display: grid;
    grid-template-columns: 1fr 280px;
    grid-template-rows: 1fr 1fr;
    gap: 16px;
  }

  /* Panel Base */
  .panel {
    background: var(--bg-secondary);
    border: 1px solid var(--border-default);
    border-radius: 8px;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  .panel-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 10px 14px;
    border-bottom: 1px solid var(--border-default);
    background: var(--bg-tertiary);
  }

  .panel-title {
    font-size: 11px;
    font-weight: 500;
    letter-spacing: 0.08em;
    color: var(--text-secondary);
  }

  .panel-hint {
    font-size: 10px;
    color: var(--text-muted);
  }

  /* Camera Panel */
  .camera-panel {
    grid-row: span 2;
  }

  .feed-container {
    flex: 1;
    position: relative;
    background: var(--bg-primary);
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .feed {
    width: 100%;
    height: 100%;
    object-fit: contain;
  }

  .feed-overlay {
    position: absolute;
    top: 12px;
    right: 12px;
  }

  .recording-indicator {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 10px;
    color: var(--accent-red);
    background: rgba(0, 0, 0, 0.6);
    padding: 4px 8px;
    border-radius: 4px;
  }

  .rec-dot {
    width: 6px;
    height: 6px;
    background: var(--accent-red);
    border-radius: 50%;
    animation: pulse 1.5s ease-in-out infinite;
  }

  @keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.3; }
  }

  .feed-error {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 12px;
    color: var(--text-muted);
  }

  .btn-retry {
    padding: 6px 12px;
    font-size: 12px;
    background: var(--bg-tertiary);
    border: 1px solid var(--border-default);
    color: var(--text-secondary);
    border-radius: 4px;
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .btn-retry:hover {
    background: var(--bg-elevated);
    border-color: var(--text-muted);
  }

  .btn-icon {
    padding: 6px;
    background: transparent;
    border: none;
    color: var(--text-muted);
    border-radius: 4px;
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .btn-icon:hover {
    color: var(--text-primary);
    background: var(--bg-elevated);
  }

  /* Controls Panel */
  .controls-panel {
    display: flex;
    flex-direction: column;
  }

  .controls-grid {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 8px;
    padding: 16px;
  }

  .control-row {
    display: flex;
    gap: 8px;
  }

  .control-btn {
    width: 60px;
    height: 60px;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 4px;
    background: var(--bg-tertiary);
    border: 1px solid var(--border-default);
    color: var(--text-secondary);
    border-radius: 8px;
    cursor: pointer;
    transition: all 0.1s ease;
  }

  .control-btn:hover {
    background: var(--bg-elevated);
    border-color: var(--text-muted);
    color: var(--text-primary);
  }

  .control-btn.active {
    background: var(--accent-green-muted);
    border-color: var(--accent-green);
    color: var(--accent-green);
  }

  .key-hint {
    font-size: 10px;
    opacity: 0.6;
  }

  .emergency-stop {
    padding: 12px;
    border-top: 1px solid var(--border-default);
  }

  .btn-emergency {
    width: 100%;
    padding: 10px;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    font-size: 11px;
    font-weight: 500;
    letter-spacing: 0.05em;
    background: var(--accent-red-muted);
    border: 1px solid var(--accent-red);
    color: var(--accent-red);
    border-radius: 6px;
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .btn-emergency:hover {
    background: var(--accent-red);
    color: var(--text-primary);
  }

  /* Logs Panel */
  .logs-panel {
    grid-column: 2;
  }

  .log-count {
    font-size: 10px;
    padding: 2px 6px;
    background: var(--bg-elevated);
    border-radius: 4px;
    color: var(--text-muted);
  }

  .logs-container {
    flex: 1;
    overflow-y: auto;
    padding: 8px;
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  .log-entry {
    display: flex;
    gap: 10px;
    padding: 4px 8px;
    font-size: 11px;
    border-radius: 4px;
  }

  .log-entry:hover {
    background: var(--bg-tertiary);
  }

  .log-time {
    color: var(--text-muted);
    flex-shrink: 0;
  }

  .log-message {
    color: var(--text-secondary);
    word-break: break-word;
  }

  .log-success .log-message {
    color: var(--accent-green);
  }

  .log-error .log-message {
    color: var(--accent-red);
  }

  .logs-empty {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    color: var(--text-muted);
    font-size: 12px;
  }

  /* Terminal Panel */
  .terminal-panel {
    grid-column: 2;
  }

  .terminal-status {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 10px;
    color: var(--text-muted);
  }

  .terminal-container {
    flex: 1;
    display: flex;
    flex-direction: column;
    background: var(--bg-primary);
  }

  .terminal-output {
    flex: 1;
    overflow-y: auto;
    padding: 12px;
    font-size: 12px;
    line-height: 1.6;
  }

  .terminal-welcome {
    display: flex;
    flex-direction: column;
    gap: 4px;
    color: var(--text-muted);
  }

  .terminal-hint {
    font-size: 11px;
    opacity: 0.6;
  }

  .terminal-line {
    font-family: var(--font-mono);
  }

  .terminal-line.command {
    color: var(--accent-green);
    display: flex;
    gap: 8px;
  }

  .terminal-line.output {
    color: var(--text-secondary);
    white-space: pre-wrap;
    padding-left: 16px;
  }

  .prompt {
    color: var(--accent-green);
    font-weight: 600;
  }

  .terminal-input-row {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 12px;
    border-top: 1px solid var(--border-default);
    background: var(--bg-secondary);
  }

  .terminal-input {
    flex: 1;
    background: transparent;
    border: none;
    color: var(--text-primary);
    font-family: var(--font-mono);
    font-size: 12px;
    outline: none;
  }

  .terminal-input::placeholder {
    color: var(--text-muted);
  }

  .terminal-input:disabled {
    opacity: 0.5;
  }

  /* Responsive */
  @media (max-width: 900px) {
    .main-grid {
      grid-template-columns: 1fr;
      grid-template-rows: auto auto auto auto;
    }

    .camera-panel {
      grid-row: span 1;
      min-height: 300px;
    }

    .logs-panel,
    .terminal-panel {
      grid-column: 1;
    }

    .header {
      flex-direction: column;
      gap: 12px;
      align-items: flex-start;
    }

    .header-right {
      width: 100%;
      justify-content: space-between;
    }
  }
</style>
