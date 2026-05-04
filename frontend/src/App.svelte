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
  let logsContainer = $state(null);
  let terminalOutput = $state(null);

  const feedUrl = $derived(`${backendOrigin}/video_feed?ts=${feedNonce}`);

  const addLog = (message, type = "info") => {
    const timestamp = new Date().toLocaleTimeString("en-US", { 
      hour12: false,
      hour: "2-digit",
      minute: "2-digit",
      second: "2-digit"
    });
    logs = [...logs.slice(-99), { timestamp, message, type, id: Date.now() }];
    
    // Auto-scroll logs
    setTimeout(() => {
      if (logsContainer) {
        logsContainer.scrollTop = logsContainer.scrollHeight;
      }
    }, 10);
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
      setTimeout(() => {
        if (terminalOutput) {
          terminalOutput.scrollTop = terminalOutput.scrollHeight;
        }
      }, 10);
    });

    addLog("Controller initialized", "info");
    addLog("Waiting for backend connection...", "info");

    return () => {
      socket.disconnect();
    };
  });

  const sendMove = (dir) => {
    socket.emit("move", { direction: dir });
    if (dir !== "stop") {
      activeDirection = dir;
      addLog(`Movement: ${dir.toUpperCase()}`, "info");
    } else {
      activeDirection = null;
    }
  };

  const reloadFeed = () => {
    feedError = "";
    feedNonce = Date.now();
    addLog("Camera feed refreshed", "info");
  };

  const executeCommand = () => {
    if (!terminalInput.trim()) return;
    
    const cmd = terminalInput.trim();
    terminalHistory = [...terminalHistory, { type: "command", content: cmd }];
    socket.emit("ssh_command", { command: cmd });
    addLog(`SSH: ${cmd}`, "info");
    terminalInput = "";
    
    setTimeout(() => {
      if (terminalOutput) {
        terminalOutput.scrollTop = terminalOutput.scrollHeight;
      }
    }, 10);
  };

  const handleKeydown = (e) => {
    if (e.key === "Enter") {
      executeCommand();
    }
  };

  const clearTerminal = () => {
    terminalHistory = [];
  };

  const clearLogs = () => {
    logs = [];
    addLog("Logs cleared", "info");
  };

  // Keyboard Listeners for movement
  const handleGlobalKeydown = (e) => {
    if (e.target.tagName === "INPUT" || e.target.tagName === "TEXTAREA") return;
    const key = e.key.toLowerCase();
    if (["w", "a", "s", "d"].includes(key)) {
      e.preventDefault();
      sendMove(key);
    }
  };

  const handleGlobalKeyup = (e) => {
    if (e.target.tagName === "INPUT" || e.target.tagName === "TEXTAREA") return;
    const key = e.key.toLowerCase();
    if (["w", "a", "s", "d"].includes(key)) {
      sendMove("stop");
    }
  };
</script>

<svelte:window onkeydown={handleGlobalKeydown} onkeyup={handleGlobalKeyup} />

<main class="app">
  <!-- Header -->
  <header class="header">
    <div class="header-brand">
      <h1 class="logo">sewbot</h1>
      <div class="status-badge" class:online={status === "Online"}>
        <span class="status-dot"></span>
        <span>{status}</span>
      </div>
    </div>
    <div class="header-meta">
      <span class="meta-label">Backend</span>
      <code class="meta-value">{backendOrigin}</code>
    </div>
  </header>

  <!-- Main Content -->
  <div class="layout">
    <!-- Left Column - Camera + Controls -->
    <div class="column-main">
      <!-- Camera Feed -->
      <section class="card camera-card">
        <div class="card-header">
          <div class="card-title">
            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <path d="m22 8-6 4 6 4V8Z"/><rect width="14" height="12" x="2" y="6" rx="2"/>
            </svg>
            Camera Feed
          </div>
          <div class="card-actions">
            <span class="live-badge">
              <span class="live-dot"></span>
              LIVE
            </span>
            <button class="btn-icon" onclick={reloadFeed} title="Refresh">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M21 12a9 9 0 1 1-9-9c2.52 0 4.93 1 6.74 2.74L21 8"/>
                <path d="M21 3v5h-5"/>
              </svg>
            </button>
          </div>
        </div>
        <div class="camera-feed">
          {#if feedError}
            <div class="feed-placeholder">
              <svg width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
                <path d="m22 8-6 4 6 4V8Z"/><rect width="14" height="12" x="2" y="6" rx="2"/>
                <line x1="2" y1="2" x2="22" y2="22"/>
              </svg>
              <span>Camera unavailable</span>
              <button class="btn-secondary" onclick={reloadFeed}>Retry Connection</button>
            </div>
          {:else}
            <img
              src={feedUrl}
              class="feed-img"
              alt="Robot camera feed"
              onerror={() => (feedError = "Feed unavailable")}
              onload={() => (feedError = "")}
            />
          {/if}
        </div>
      </section>

      <!-- Controls -->
      <section class="card controls-card">
        <div class="card-header">
          <div class="card-title">
            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <circle cx="12" cy="12" r="10"/><path d="m8 12 4 4 4-4"/><path d="M12 8v8"/>
            </svg>
            Controls
          </div>
          <span class="hint">Use WASD keys or buttons</span>
        </div>
        <div class="controls-content">
          <div class="dpad">
            <button 
              class="dpad-btn up" 
              class:active={activeDirection === "w"}
              onmousedown={() => sendMove("w")}
              onmouseup={() => sendMove("stop")}
              onmouseleave={() => activeDirection === "w" && sendMove("stop")}
            >
              <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="m18 15-6-6-6 6"/>
              </svg>
              <span class="key">W</span>
            </button>
            <button 
              class="dpad-btn left"
              class:active={activeDirection === "a"}
              onmousedown={() => sendMove("a")}
              onmouseup={() => sendMove("stop")}
              onmouseleave={() => activeDirection === "a" && sendMove("stop")}
            >
              <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="m15 18-6-6 6-6"/>
              </svg>
              <span class="key">A</span>
            </button>
            <button 
              class="dpad-btn down"
              class:active={activeDirection === "s"}
              onmousedown={() => sendMove("s")}
              onmouseup={() => sendMove("stop")}
              onmouseleave={() => activeDirection === "s" && sendMove("stop")}
            >
              <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="m6 9 6 6 6-6"/>
              </svg>
              <span class="key">S</span>
            </button>
            <button 
              class="dpad-btn right"
              class:active={activeDirection === "d"}
              onmousedown={() => sendMove("d")}
              onmouseup={() => sendMove("stop")}
              onmouseleave={() => activeDirection === "d" && sendMove("stop")}
            >
              <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="m9 18 6-6-6-6"/>
              </svg>
              <span class="key">D</span>
            </button>
            <div class="dpad-center"></div>
          </div>
          <button class="btn-danger" onclick={() => sendMove("stop")}>
            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <circle cx="12" cy="12" r="10"/><rect x="9" y="9" width="6" height="6"/>
            </svg>
            Emergency Stop
          </button>
        </div>
      </section>
    </div>

    <!-- Right Column - Logs + Terminal -->
    <div class="column-side">
      <!-- Logs -->
      <section class="card logs-card">
        <div class="card-header">
          <div class="card-title">
            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <path d="M12 12h.01"/><path d="M16 6V4a2 2 0 0 0-2-2h-4a2 2 0 0 0-2 2v2"/><path d="M22 13a18.15 18.15 0 0 1-20 0"/><rect width="20" height="14" x="2" y="6" rx="2"/>
            </svg>
            System Logs
          </div>
          <div class="card-actions">
            <span class="log-count">{logs.length}</span>
            <button class="btn-icon" onclick={clearLogs} title="Clear logs">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/>
              </svg>
            </button>
          </div>
        </div>
        <div class="logs-feed" bind:this={logsContainer}>
          {#if logs.length === 0}
            <div class="empty-state">
              <span>No logs yet</span>
            </div>
          {:else}
            {#each logs as log (log.id)}
              <div class="log-entry {log.type}">
                <span class="log-time">{log.timestamp}</span>
                <span class="log-type-badge">{log.type}</span>
                <span class="log-msg">{log.message}</span>
              </div>
            {/each}
          {/if}
        </div>
      </section>

      <!-- Terminal -->
      <section class="card terminal-card">
        <div class="card-header">
          <div class="card-title">
            <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <polyline points="4 17 10 11 4 5"/><line x1="12" x2="20" y1="19" y2="19"/>
            </svg>
            SSH Terminal
          </div>
          <div class="card-actions">
            <span class="terminal-status" class:connected={status === "Online"}>
              {status === "Online" ? "Connected" : "Disconnected"}
            </span>
            <button class="btn-icon" onclick={clearTerminal} title="Clear terminal">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/>
              </svg>
            </button>
          </div>
        </div>
        <div class="terminal">
          <div class="terminal-output" bind:this={terminalOutput}>
            {#if terminalHistory.length === 0}
              <div class="terminal-welcome">
                <span class="welcome-title">SSH Terminal Ready</span>
                <span class="welcome-hint">Enter commands to execute on the robot</span>
              </div>
            {:else}
              {#each terminalHistory as entry}
                {#if entry.type === "command"}
                  <div class="term-line cmd">
                    <span class="prompt">$</span>
                    <span>{entry.content}</span>
                  </div>
                {:else}
                  <div class="term-line out">{entry.content}</div>
                {/if}
              {/each}
            {/if}
          </div>
          <div class="terminal-input-wrapper">
            <span class="prompt">$</span>
            <input 
              type="text" 
              class="terminal-input" 
              placeholder={status === "Online" ? "Enter command..." : "Waiting for connection..."}
              bind:value={terminalInput}
              onkeydown={handleKeydown}
              disabled={status !== "Online"}
            />
          </div>
        </div>
      </section>
    </div>
  </div>
</main>

<style>
  .app {
    min-height: 100vh;
    padding: 20px;
    display: flex;
    flex-direction: column;
    gap: 20px;
    max-width: 1600px;
    margin: 0 auto;
  }

  /* Header */
  .header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 16px 20px;
    background: var(--bg-card);
    border: 1px solid var(--border);
    border-radius: var(--radius);
  }

  .header-brand {
    display: flex;
    align-items: center;
    gap: 16px;
  }

  .logo {
    margin: 0;
    font-size: 20px;
    font-weight: 600;
    letter-spacing: -0.02em;
  }

  .status-badge {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 4px 10px;
    font-size: 12px;
    font-weight: 500;
    background: var(--danger-muted);
    color: var(--danger);
    border-radius: 20px;
    transition: all 0.2s ease;
  }

  .status-badge.online {
    background: var(--accent-muted);
    color: var(--accent);
  }

  .status-dot {
    width: 6px;
    height: 6px;
    border-radius: 50%;
    background: currentColor;
  }

  .status-badge.online .status-dot {
    animation: pulse 2s ease-in-out infinite;
  }

  @keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.4; }
  }

  .header-meta {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .meta-label {
    font-size: 12px;
    color: var(--text-muted);
  }

  .meta-value {
    font-family: var(--font-mono);
    font-size: 12px;
    color: var(--text-secondary);
    padding: 4px 8px;
    background: var(--bg-elevated);
    border-radius: 6px;
  }

  /* Layout */
  .layout {
    flex: 1;
    display: grid;
    grid-template-columns: 1fr 400px;
    gap: 20px;
  }

  .column-main {
    display: flex;
    flex-direction: column;
    gap: 20px;
  }

  .column-side {
    display: flex;
    flex-direction: column;
    gap: 20px;
  }

  /* Card */
  .card {
    background: var(--bg-card);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  .card-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 14px 16px;
    border-bottom: 1px solid var(--border);
  }

  .card-title {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 13px;
    font-weight: 500;
    color: var(--text-primary);
  }

  .card-title svg {
    color: var(--text-muted);
  }

  .card-actions {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .hint {
    font-size: 12px;
    color: var(--text-muted);
  }

  /* Camera */
  .camera-card {
    flex: 1;
    min-height: 400px;
  }

  .camera-feed {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--bg-primary);
    position: relative;
  }

  .feed-img {
    width: 100%;
    height: 100%;
    object-fit: contain;
  }

  .feed-placeholder {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 16px;
    color: var(--text-muted);
  }

  .feed-placeholder span {
    font-size: 14px;
  }

  .live-badge {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 10px;
    font-weight: 600;
    letter-spacing: 0.05em;
    color: var(--danger);
    padding: 4px 8px;
    background: var(--danger-muted);
    border-radius: 4px;
  }

  .live-dot {
    width: 6px;
    height: 6px;
    background: var(--danger);
    border-radius: 50%;
    animation: pulse 1s ease-in-out infinite;
  }

  /* Controls */
  .controls-card {
    flex-shrink: 0;
  }

  .controls-content {
    padding: 24px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 32px;
  }

  .dpad {
    display: grid;
    grid-template-columns: 56px 56px 56px;
    grid-template-rows: 56px 56px 56px;
    gap: 4px;
    position: relative;
  }

  .dpad-btn {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 2px;
    background: var(--bg-elevated);
    border: 1px solid var(--border);
    color: var(--text-secondary);
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .dpad-btn:hover {
    background: var(--bg-primary);
    border-color: var(--border-hover);
    color: var(--text-primary);
  }

  .dpad-btn.active {
    background: var(--accent-muted);
    border-color: var(--accent);
    color: var(--accent);
  }

  .dpad-btn .key {
    font-size: 10px;
    font-weight: 500;
    opacity: 0.5;
  }

  .dpad-btn.up { grid-column: 2; grid-row: 1; }
  .dpad-btn.left { grid-column: 1; grid-row: 2; }
  .dpad-btn.down { grid-column: 2; grid-row: 3; }
  .dpad-btn.right { grid-column: 3; grid-row: 2; }

  .dpad-center {
    grid-column: 2;
    grid-row: 2;
    background: var(--bg-card);
    border-radius: 50%;
    border: 1px solid var(--border);
  }

  /* Buttons */
  .btn-icon {
    width: 32px;
    height: 32px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: transparent;
    border: 1px solid transparent;
    color: var(--text-muted);
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .btn-icon:hover {
    background: var(--bg-elevated);
    border-color: var(--border);
    color: var(--text-primary);
  }

  .btn-secondary {
    padding: 10px 16px;
    font-size: 13px;
    font-weight: 500;
    background: var(--bg-elevated);
    border: 1px solid var(--border);
    color: var(--text-primary);
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .btn-secondary:hover {
    background: var(--bg-primary);
    border-color: var(--border-hover);
  }

  .btn-danger {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 12px 24px;
    font-size: 13px;
    font-weight: 500;
    background: var(--danger-muted);
    border: 1px solid var(--danger);
    color: var(--danger);
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .btn-danger:hover {
    background: var(--danger);
    color: white;
  }

  /* Logs */
  .logs-card {
    flex: 1;
    min-height: 0;
  }

  .log-count {
    font-size: 11px;
    font-weight: 500;
    padding: 2px 8px;
    background: var(--bg-elevated);
    border-radius: 10px;
    color: var(--text-muted);
  }

  .logs-feed {
    flex: 1;
    overflow-y: auto;
    padding: 8px;
    display: flex;
    flex-direction: column;
    gap: 2px;
    min-height: 200px;
    max-height: 300px;
  }

  .log-entry {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 8px 10px;
    font-size: 12px;
    border-radius: 6px;
    transition: background 0.1s ease;
  }

  .log-entry:hover {
    background: var(--bg-elevated);
  }

  .log-time {
    font-family: var(--font-mono);
    font-size: 11px;
    color: var(--text-muted);
    flex-shrink: 0;
  }

  .log-type-badge {
    font-size: 9px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    padding: 2px 6px;
    border-radius: 4px;
    flex-shrink: 0;
    background: var(--bg-elevated);
    color: var(--text-muted);
  }

  .log-entry.success .log-type-badge {
    background: var(--accent-muted);
    color: var(--accent);
  }

  .log-entry.error .log-type-badge {
    background: var(--danger-muted);
    color: var(--danger);
  }

  .log-msg {
    color: var(--text-secondary);
    flex: 1;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .empty-state {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    color: var(--text-muted);
    font-size: 13px;
  }

  /* Terminal */
  .terminal-card {
    flex: 1;
    min-height: 0;
  }

  .terminal-status {
    font-size: 11px;
    font-weight: 500;
    color: var(--danger);
  }

  .terminal-status.connected {
    color: var(--accent);
  }

  .terminal {
    flex: 1;
    display: flex;
    flex-direction: column;
    background: var(--bg-primary);
    font-family: var(--font-mono);
    min-height: 200px;
    max-height: 300px;
  }

  .terminal-output {
    flex: 1;
    overflow-y: auto;
    padding: 16px;
    font-size: 12px;
    line-height: 1.6;
  }

  .terminal-welcome {
    display: flex;
    flex-direction: column;
    gap: 4px;
    color: var(--text-muted);
  }

  .welcome-title {
    color: var(--text-secondary);
  }

  .welcome-hint {
    font-size: 11px;
    opacity: 0.6;
  }

  .term-line {
    margin-bottom: 4px;
  }

  .term-line.cmd {
    display: flex;
    gap: 8px;
    color: var(--accent);
  }

  .term-line.out {
    color: var(--text-secondary);
    padding-left: 16px;
    white-space: pre-wrap;
    word-break: break-all;
  }

  .prompt {
    color: var(--accent);
    font-weight: 500;
    flex-shrink: 0;
  }

  .terminal-input-wrapper {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 12px 16px;
    border-top: 1px solid var(--border);
    background: var(--bg-card);
  }

  .terminal-input {
    flex: 1;
    background: transparent;
    border: none;
    color: var(--text-primary);
    font-family: var(--font-mono);
    font-size: 13px;
    outline: none;
  }

  .terminal-input::placeholder {
    color: var(--text-muted);
  }

  .terminal-input:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  /* Responsive */
  @media (max-width: 1024px) {
    .layout {
      grid-template-columns: 1fr;
    }

    .column-side {
      flex-direction: row;
    }

    .logs-card,
    .terminal-card {
      flex: 1;
    }
  }

  @media (max-width: 768px) {
    .app {
      padding: 12px;
      gap: 12px;
    }

    .header {
      flex-direction: column;
      align-items: flex-start;
      gap: 12px;
    }

    .column-side {
      flex-direction: column;
    }

    .controls-content {
      flex-direction: column;
      gap: 20px;
    }
  }
</style>
