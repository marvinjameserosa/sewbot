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
  let isFullscreen = $state(false);

  const feedUrl = $derived(`${backendOrigin}/video_feed?ts=${feedNonce}`);

  const addLog = (message, type = "info") => {
    const timestamp = new Date().toLocaleTimeString("en-US", { 
      hour12: false,
      hour: "2-digit",
      minute: "2-digit"
    });
    logs = [...logs.slice(-99), { timestamp, message, type, id: Date.now() }];
    
    setTimeout(() => {
      if (logsContainer) {
        logsContainer.scrollTop = logsContainer.scrollHeight;
      }
    }, 10);
  };

  onMount(() => {
    socket.on("connect", () => {
      status = "Online";
      addLog("Connected to sewbot backend", "success");
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
      terminalHistory = [...terminalHistory, { type: "output", content: data.output, id: Date.now() + Math.random() }];
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
    terminalHistory = [...terminalHistory, { type: "command", content: cmd, id: Date.now() + Math.random() }];
    socket.emit("ssh_command", { command: cmd });
    addLog(`Executed: ${cmd}`, "info");
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

  const toggleFullscreen = () => {
    isFullscreen = !isFullscreen;
  };
</script>

<svelte:window onkeydown={handleGlobalKeydown} onkeyup={handleGlobalKeyup} />

<main class="app">
  <!-- Header -->
  <header class="header">
    <div class="header-left">
      <h1 class="logo">sewbot</h1>
      <span class="divider"></span>
      <span class="subtitle">controller</span>
    </div>
    <div class="header-right">
      <div class="connection-pill" class:online={status === "Online"}>
        <span class="connection-dot"></span>
        <span>{status}</span>
      </div>
    </div>
  </header>

  <!-- Main Layout -->
  <div class="main-layout">
    <!-- Video Player Section -->
    <div class="video-section">
      <div class="video-player" class:fullscreen={isFullscreen}>
        <div class="video-container">
          {#if feedError}
            <div class="video-offline">
              <div class="offline-icon">
                <svg width="64" height="64" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                  <rect x="2" y="6" width="14" height="12" rx="2"/>
                  <path d="m22 8-6 4 6 4V8Z"/>
                  <line x1="1" y1="1" x2="23" y2="23" stroke-width="2"/>
                </svg>
              </div>
              <p class="offline-text">Camera feed unavailable</p>
              <button class="retry-btn" onclick={reloadFeed}>
                <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                  <path d="M21 12a9 9 0 1 1-9-9c2.52 0 4.93 1 6.74 2.74L21 8"/>
                  <path d="M21 3v5h-5"/>
                </svg>
                Retry
              </button>
            </div>
          {:else}
            <img
              src={feedUrl}
              class="video-feed"
              alt="Robot camera feed"
              onerror={() => (feedError = "Feed unavailable")}
              onload={() => (feedError = "")}
            />
          {/if}
        </div>
        
        <!-- Video Controls Bar -->
        <div class="video-controls">
          <div class="controls-left">
            <div class="live-indicator">
              <span class="live-dot"></span>
              <span>LIVE</span>
            </div>
          </div>
          <div class="controls-right">
            <button class="video-btn" onclick={reloadFeed} title="Refresh feed">
              <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M21 12a9 9 0 1 1-9-9c2.52 0 4.93 1 6.74 2.74L21 8"/>
                <path d="M21 3v5h-5"/>
              </svg>
            </button>
            <button class="video-btn" onclick={toggleFullscreen} title="Fullscreen">
              <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                {#if isFullscreen}
                  <path d="M8 3v3a2 2 0 0 1-2 2H3m18 0h-3a2 2 0 0 1-2-2V3m0 18v-3a2 2 0 0 1 2-2h3M3 16h3a2 2 0 0 1 2 2v3"/>
                {:else}
                  <path d="M8 3H5a2 2 0 0 0-2 2v3m18 0V5a2 2 0 0 0-2-2h-3m0 18h3a2 2 0 0 0 2-2v-3M3 16v3a2 2 0 0 0 2 2h3"/>
                {/if}
              </svg>
            </button>
          </div>
        </div>
      </div>

      <!-- Controls Panel -->
      <div class="controls-panel">
        <div class="dpad-section">
          <p class="section-label">Movement Controls</p>
          <div class="dpad">
            <button 
              class="dpad-key up" 
              class:active={activeDirection === "w"}
              onmousedown={() => sendMove("w")}
              onmouseup={() => sendMove("stop")}
              onmouseleave={() => activeDirection === "w" && sendMove("stop")}
            >
              <span class="key-letter">W</span>
              <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <path d="m18 15-6-6-6 6"/>
              </svg>
            </button>
            <button 
              class="dpad-key left"
              class:active={activeDirection === "a"}
              onmousedown={() => sendMove("a")}
              onmouseup={() => sendMove("stop")}
              onmouseleave={() => activeDirection === "a" && sendMove("stop")}
            >
              <span class="key-letter">A</span>
              <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <path d="m15 18-6-6 6-6"/>
              </svg>
            </button>
            <div class="dpad-center-wrapper">
              <button 
                class="stop-btn"
                onclick={() => sendMove("stop")}
                title="Emergency Stop"
              >
                STOP
              </button>
            </div>
            <button 
              class="dpad-key right"
              class:active={activeDirection === "d"}
              onmousedown={() => sendMove("d")}
              onmouseup={() => sendMove("stop")}
              onmouseleave={() => activeDirection === "d" && sendMove("stop")}
            >
              <span class="key-letter">D</span>
              <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <path d="m9 18 6-6-6-6"/>
              </svg>
            </button>
            <button 
              class="dpad-key down"
              class:active={activeDirection === "s"}
              onmousedown={() => sendMove("s")}
              onmouseup={() => sendMove("stop")}
              onmouseleave={() => activeDirection === "s" && sendMove("stop")}
            >
              <span class="key-letter">S</span>
              <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5">
                <path d="m6 9 6 6 6-6"/>
              </svg>
            </button>
          </div>
          <p class="hint-text">Use WASD keys or click buttons</p>
        </div>
      </div>
    </div>

    <!-- Chat/Logs Sidebar -->
    <div class="sidebar">
      <!-- Logs as Live Chat -->
      <div class="chat-section">
        <div class="chat-header">
          <span class="chat-title">Live Activity</span>
          <div class="chat-actions">
            <span class="viewer-count">{logs.length} events</span>
            <button class="icon-btn" onclick={clearLogs} title="Clear">
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/>
              </svg>
            </button>
          </div>
        </div>
        <div class="chat-messages" bind:this={logsContainer}>
          {#if logs.length === 0}
            <div class="chat-empty">
              <p>No activity yet</p>
              <span>Events will appear here</span>
            </div>
          {:else}
            {#each logs as log (log.id)}
              <div class="chat-message {log.type}">
                <span class="msg-badge {log.type}">
                  {#if log.type === "success"}
                    <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3"><path d="M20 6 9 17l-5-5"/></svg>
                  {:else if log.type === "error"}
                    <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3"><circle cx="12" cy="12" r="10"/><path d="m15 9-6 6"/><path d="m9 9 6 6"/></svg>
                  {:else}
                    <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="3"><circle cx="12" cy="12" r="10"/><path d="M12 16v-4"/><path d="M12 8h.01"/></svg>
                  {/if}
                </span>
                <span class="msg-content">{log.message}</span>
                <span class="msg-time">{log.timestamp}</span>
              </div>
            {/each}
          {/if}
        </div>
      </div>

      <!-- Terminal as Command Input -->
      <div class="terminal-section">
        <div class="terminal-header">
          <div class="terminal-title">
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <polyline points="4 17 10 11 4 5"/><line x1="12" x2="20" y1="19" y2="19"/>
            </svg>
            <span>SSH Terminal</span>
          </div>
          <div class="terminal-actions">
            <span class="terminal-status" class:connected={status === "Online"}>
              {status === "Online" ? "connected" : "disconnected"}
            </span>
            <button class="icon-btn" onclick={clearTerminal} title="Clear">
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/>
              </svg>
            </button>
          </div>
        </div>
        <div class="terminal-body" bind:this={terminalOutput}>
          {#if terminalHistory.length === 0}
            <div class="terminal-welcome">
              <p>Welcome to SSH Terminal</p>
              <span>Enter commands to execute on the robot</span>
            </div>
          {:else}
            {#each terminalHistory as entry (entry.id)}
              <div class="terminal-line {entry.type}">
                {#if entry.type === "command"}
                  <span class="prompt">$</span>
                  <span class="cmd-text">{entry.content}</span>
                {:else}
                  <span class="output-text">{entry.content}</span>
                {/if}
              </div>
            {/each}
          {/if}
        </div>
        <div class="terminal-input-area">
          <span class="input-prompt">$</span>
          <input 
            type="text" 
            class="cmd-input" 
            placeholder={status === "Online" ? "Type a command..." : "Waiting for connection..."}
            bind:value={terminalInput}
            onkeydown={handleKeydown}
            disabled={status !== "Online"}
          />
          <button 
            class="send-btn" 
            onclick={executeCommand}
            disabled={status !== "Online" || !terminalInput.trim()}
            aria-label="Send command"
          >
            <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="m22 2-7 20-4-9-9-4Z"/><path d="M22 2 11 13"/>
            </svg>
          </button>
        </div>
      </div>
    </div>
  </div>
</main>

<style>
  :root {
    --bg-primary: #0a0a0a;
    --bg-secondary: #111111;
    --bg-tertiary: #181818;
    --bg-hover: #1f1f1f;
    --border: #262626;
    --border-light: #333333;
    --text-primary: #fafafa;
    --text-secondary: #a1a1aa;
    --text-muted: #71717a;
    --accent: #10b981;
    --accent-muted: rgba(16, 185, 129, 0.15);
    --danger: #ef4444;
    --danger-muted: rgba(239, 68, 68, 0.15);
    --info: #3b82f6;
    --info-muted: rgba(59, 130, 246, 0.15);
    --radius: 12px;
    --radius-sm: 8px;
  }

  .app {
    min-height: 100vh;
    background: var(--bg-primary);
    color: var(--text-primary);
    display: flex;
    flex-direction: column;
  }

  /* Header */
  .header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 16px 24px;
    border-bottom: 1px solid var(--border);
    background: var(--bg-secondary);
  }

  .header-left {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .logo {
    font-size: 22px;
    font-weight: 700;
    letter-spacing: -0.03em;
    margin: 0;
  }

  .divider {
    width: 1px;
    height: 20px;
    background: var(--border);
  }

  .subtitle {
    font-size: 14px;
    color: var(--text-muted);
    font-weight: 400;
  }

  .connection-pill {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 8px 14px;
    background: var(--danger-muted);
    border: 1px solid var(--danger);
    border-radius: 24px;
    font-size: 13px;
    font-weight: 500;
    color: var(--danger);
    transition: all 0.2s;
  }

  .connection-pill.online {
    background: var(--accent-muted);
    border-color: var(--accent);
    color: var(--accent);
  }

  .connection-dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: currentColor;
    animation: pulse 2s ease-in-out infinite;
  }

  @keyframes pulse {
    0%, 100% { opacity: 1; transform: scale(1); }
    50% { opacity: 0.5; transform: scale(0.9); }
  }

  /* Main Layout */
  .main-layout {
    flex: 1;
    display: grid;
    grid-template-columns: 1fr 380px;
    gap: 0;
  }

  /* Video Section */
  .video-section {
    display: flex;
    flex-direction: column;
    background: var(--bg-primary);
  }

  .video-player {
    flex: 1;
    display: flex;
    flex-direction: column;
    background: #000;
    position: relative;
    min-height: 400px;
  }

  .video-player.fullscreen {
    position: fixed;
    inset: 0;
    z-index: 100;
    min-height: 100vh;
  }

  .video-container {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    position: relative;
    overflow: hidden;
  }

  .video-feed {
    width: 100%;
    height: 100%;
    object-fit: contain;
  }

  .video-offline {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 16px;
    color: var(--text-muted);
  }

  .offline-icon {
    opacity: 0.3;
  }

  .offline-text {
    font-size: 16px;
    margin: 0;
  }

  .retry-btn {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 12px 24px;
    background: var(--bg-tertiary);
    border: 1px solid var(--border);
    color: var(--text-primary);
    border-radius: var(--radius-sm);
    font-size: 14px;
    cursor: pointer;
    transition: all 0.2s;
  }

  .retry-btn:hover {
    background: var(--bg-hover);
    border-color: var(--border-light);
  }

  /* Video Controls */
  .video-controls {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 12px 16px;
    background: linear-gradient(transparent, rgba(0,0,0,0.8));
    position: absolute;
    bottom: 0;
    left: 0;
    right: 0;
  }

  .controls-left, .controls-right {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .live-indicator {
    display: flex;
    align-items: center;
    gap: 6px;
    padding: 6px 12px;
    background: var(--danger);
    border-radius: 4px;
    font-size: 12px;
    font-weight: 700;
    letter-spacing: 0.05em;
    color: white;
  }

  .live-dot {
    width: 8px;
    height: 8px;
    background: white;
    border-radius: 50%;
    animation: pulse 1s ease-in-out infinite;
  }

  .video-btn {
    width: 40px;
    height: 40px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: rgba(255,255,255,0.1);
    border: none;
    color: white;
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: all 0.2s;
  }

  .video-btn:hover {
    background: rgba(255,255,255,0.2);
  }

  /* Controls Panel */
  .controls-panel {
    padding: 32px;
    background: var(--bg-secondary);
    border-top: 1px solid var(--border);
  }

  .section-label {
    margin: 0 0 20px;
    font-size: 14px;
    font-weight: 500;
    color: var(--text-secondary);
    text-align: center;
  }

  .dpad {
    display: grid;
    grid-template-columns: repeat(3, 72px);
    grid-template-rows: repeat(3, 72px);
    gap: 8px;
    justify-content: center;
    margin-bottom: 16px;
  }

  .dpad-key {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 4px;
    background: var(--bg-tertiary);
    border: 2px solid var(--border);
    color: var(--text-secondary);
    border-radius: var(--radius);
    cursor: pointer;
    transition: all 0.15s;
    font-family: inherit;
  }

  .dpad-key:hover {
    background: var(--bg-hover);
    border-color: var(--border-light);
    color: var(--text-primary);
    transform: scale(1.02);
  }

  .dpad-key.active {
    background: var(--accent-muted);
    border-color: var(--accent);
    color: var(--accent);
    transform: scale(0.98);
  }

  .key-letter {
    font-size: 16px;
    font-weight: 700;
    letter-spacing: 0.05em;
  }

  .dpad-key.up { grid-column: 2; grid-row: 1; }
  .dpad-key.left { grid-column: 1; grid-row: 2; }
  .dpad-key.right { grid-column: 3; grid-row: 2; }
  .dpad-key.down { grid-column: 2; grid-row: 3; }

  .dpad-center-wrapper {
    grid-column: 2;
    grid-row: 2;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .stop-btn {
    width: 100%;
    height: 100%;
    background: var(--danger-muted);
    border: 2px solid var(--danger);
    color: var(--danger);
    border-radius: var(--radius);
    font-size: 11px;
    font-weight: 800;
    letter-spacing: 0.1em;
    cursor: pointer;
    transition: all 0.15s;
  }

  .stop-btn:hover {
    background: var(--danger);
    color: white;
    transform: scale(1.02);
  }

  .hint-text {
    margin: 0;
    text-align: center;
    font-size: 12px;
    color: var(--text-muted);
  }

  /* Sidebar */
  .sidebar {
    display: flex;
    flex-direction: column;
    border-left: 1px solid var(--border);
    background: var(--bg-secondary);
  }

  /* Chat Section (Logs) */
  .chat-section {
    flex: 1;
    display: flex;
    flex-direction: column;
    min-height: 0;
  }

  .chat-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 14px 16px;
    border-bottom: 1px solid var(--border);
    background: var(--bg-tertiary);
  }

  .chat-title {
    font-size: 14px;
    font-weight: 600;
  }

  .chat-actions {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .viewer-count {
    font-size: 12px;
    color: var(--text-muted);
  }

  .icon-btn {
    width: 32px;
    height: 32px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: transparent;
    border: none;
    color: var(--text-muted);
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: all 0.15s;
  }

  .icon-btn:hover {
    background: var(--bg-hover);
    color: var(--text-primary);
  }

  .chat-messages {
    flex: 1;
    overflow-y: auto;
    padding: 12px;
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  .chat-empty {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 4px;
    color: var(--text-muted);
  }

  .chat-empty p {
    margin: 0;
    font-size: 14px;
  }

  .chat-empty span {
    font-size: 12px;
    opacity: 0.6;
  }

  .chat-message {
    display: flex;
    align-items: flex-start;
    gap: 10px;
    padding: 10px 12px;
    background: var(--bg-tertiary);
    border-radius: var(--radius-sm);
    transition: background 0.15s;
  }

  .chat-message:hover {
    background: var(--bg-hover);
  }

  .msg-badge {
    flex-shrink: 0;
    width: 24px;
    height: 24px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--info-muted);
    color: var(--info);
    border-radius: 6px;
  }

  .msg-badge.success {
    background: var(--accent-muted);
    color: var(--accent);
  }

  .msg-badge.error {
    background: var(--danger-muted);
    color: var(--danger);
  }

  .msg-content {
    flex: 1;
    font-size: 13px;
    color: var(--text-primary);
    line-height: 1.4;
  }

  .msg-time {
    font-size: 11px;
    color: var(--text-muted);
    flex-shrink: 0;
  }

  /* Terminal Section */
  .terminal-section {
    display: flex;
    flex-direction: column;
    border-top: 1px solid var(--border);
    height: 280px;
  }

  .terminal-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 12px 16px;
    background: var(--bg-tertiary);
    border-bottom: 1px solid var(--border);
  }

  .terminal-title {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 13px;
    font-weight: 500;
    color: var(--text-secondary);
  }

  .terminal-actions {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .terminal-status {
    font-size: 11px;
    color: var(--danger);
    font-weight: 500;
  }

  .terminal-status.connected {
    color: var(--accent);
  }

  .terminal-body {
    flex: 1;
    overflow-y: auto;
    padding: 16px;
    background: var(--bg-primary);
    font-family: 'JetBrains Mono', 'SF Mono', 'Fira Code', monospace;
    font-size: 13px;
    line-height: 1.6;
  }

  .terminal-welcome {
    color: var(--text-muted);
  }

  .terminal-welcome p {
    margin: 0 0 4px;
    color: var(--text-secondary);
  }

  .terminal-welcome span {
    font-size: 12px;
    opacity: 0.6;
  }

  .terminal-line {
    margin-bottom: 6px;
  }

  .terminal-line.command {
    display: flex;
    gap: 8px;
  }

  .prompt {
    color: var(--accent);
    font-weight: 600;
  }

  .cmd-text {
    color: var(--text-primary);
  }

  .output-text {
    color: var(--text-secondary);
    padding-left: 16px;
    white-space: pre-wrap;
    word-break: break-all;
  }

  .terminal-input-area {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 12px 16px;
    background: var(--bg-tertiary);
    border-top: 1px solid var(--border);
  }

  .input-prompt {
    color: var(--accent);
    font-family: 'JetBrains Mono', 'SF Mono', monospace;
    font-weight: 600;
    font-size: 14px;
  }

  .cmd-input {
    flex: 1;
    background: transparent;
    border: none;
    color: var(--text-primary);
    font-family: 'JetBrains Mono', 'SF Mono', monospace;
    font-size: 14px;
    outline: none;
  }

  .cmd-input::placeholder {
    color: var(--text-muted);
  }

  .cmd-input:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }

  .send-btn {
    width: 40px;
    height: 40px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--accent);
    border: none;
    color: white;
    border-radius: var(--radius-sm);
    cursor: pointer;
    transition: all 0.15s;
  }

  .send-btn:hover:not(:disabled) {
    background: #0ea271;
    transform: scale(1.05);
  }

  .send-btn:disabled {
    opacity: 0.3;
    cursor: not-allowed;
  }

  /* Responsive */
  @media (max-width: 1024px) {
    .main-layout {
      grid-template-columns: 1fr;
    }

    .sidebar {
      border-left: none;
      border-top: 1px solid var(--border);
      flex-direction: row;
      height: 350px;
    }

    .chat-section, .terminal-section {
      flex: 1;
      height: auto;
      border-top: none;
    }

    .terminal-section {
      border-left: 1px solid var(--border);
    }
  }

  @media (max-width: 768px) {
    .header {
      padding: 12px 16px;
    }

    .sidebar {
      flex-direction: column;
      height: auto;
    }

    .chat-section {
      max-height: 300px;
    }

    .terminal-section {
      border-left: none;
      border-top: 1px solid var(--border);
    }

    .controls-panel {
      padding: 24px 16px;
    }

    .dpad {
      grid-template-columns: repeat(3, 64px);
      grid-template-rows: repeat(3, 64px);
      gap: 6px;
    }
  }
</style>
