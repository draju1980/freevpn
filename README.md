# FreeVPN - Chrome Extension using Tor Network

A free VPN Chrome extension that routes browser traffic through the Tor/onion network. Fast, cheap to run, no user-side installs required. Supports future mobile apps (Android/iOS).

## Architecture

**Hybrid** — remote Deno relay server (default) + optional local mode for power users.

```
Chrome Extension (thin client)
        ↓ WebSocket (control channel only)
Deno Relay Server (remote or localhost)
        ↓ HTTP CONNECT proxy (data channel)
Chrome Extension ←──────────────────┘
        ↓ (chrome.proxy points to relay's HTTP proxy port)
Deno Relay Server
        ↓ SOCKS5
Tor Daemon (port 9050, on same machine as relay)
        ↓
Tor Network → Internet
```

**Key insight**: WebSocket is control-only (auth, country selection, circuit management). Actual traffic flows through an HTTP CONNECT proxy exposed by the relay, which internally bridges to Tor's SOCKS5. Simpler and faster than tunneling all data through WebSocket.

## MVP Features

1. **On/off toggle** — one-click connect/disconnect
2. **Country/exit node selection** — dropdown to pick exit country
3. **Split tunneling** — per-domain bypass rules (which sites skip Tor)
4. **Leak prevention** — WebRTC + DNS leak protection

## Node Discovery & Health

Each extension instance is a **named node** in the network. The relay server acts as a registry/coordinator.

```
Extension "node-alpha"  ──┐
Extension "node-beta"   ──┤  WebSocket
Extension "node-gamma"  ──┤
                          ▼
                   Relay Server
                  ┌─────────────┐
                  │ Node Registry│  ← tracks all connected nodes
                  │  - name      │
                  │  - status    │
                  │  - latency   │
                  │  - country   │
                  │  - last_seen │
                  └─────────────┘
```

**How it works:**

1. **Unique node name** — each extension generates a persistent unique name on install (e.g. `node-a3f8b2`), stored in `chrome.storage.local`. This is the node's identity in the network.
2. **Registration** — on connect, the extension sends its node name to the relay. The relay adds it to the node registry.
3. **Health heartbeat** — every 60 seconds, each node sends a `node_health` message containing:
   - `nodeName` — unique identifier
   - `status` — `healthy` | `degraded` | `error`
   - `latencyMs` — round-trip time to relay
   - `uptime` — seconds since connect
4. **Network awareness** — on connect (and periodically), the relay broadcasts the full node list to all connected nodes. Each extension knows every other node in the network.
5. **Stale node cleanup** — if the relay doesn't receive a health ping from a node within 120s, it marks the node as `offline` and notifies other nodes.

**Use cases for node awareness:**
- Display "X nodes online" in the popup UI
- Future: peer-to-peer relay fallback if the main relay goes down
- Future: load balancing across multiple relay servers
- Future: node reputation scoring based on uptime/health

## File Structure

```
/
├── deno.json                          # Workspace root
├── .gitignore
├── .env.example
│
├── shared/
│   ├── deno.json
│   ├── mod.ts                         # Barrel export
│   ├── types.ts                       # ConnectionStatus, ExitCountry, SplitTunnelRule, ExtensionSettings, NodeInfo
│   ├── protocol.ts                    # ClientMessage / ServerMessage discriminated unions
│   └── constants.ts                   # Ports, country list, ping interval
│
├── server/
│   ├── deno.json
│   ├── main.ts                        # Entry: Deno.serve(), WebSocket upgrade, /health
│   ├── tor-control.ts                 # Tor Control Protocol (port 9051): auth, setExitCountry, newCircuit
│   ├── proxy-bridge.ts                # HTTP CONNECT → SOCKS5 bridge to Tor port 9050
│   ├── session.ts                     # Session manager (Map<sessionId, ClientSession>)
│   ├── node-registry.ts              # Node registry: tracks connected nodes, health, stale cleanup
│   ├── ws-handler.ts                  # WebSocket message dispatcher
│   ├── tor-control_test.ts
│   ├── proxy-bridge_test.ts
│   ├── ws-handler_test.ts
│   └── session_test.ts
│
├── extension/
│   ├── deno.json
│   ├── build.ts                       # esbuild bundler script
│   ├── manifest.json                  # Manifest V3
│   ├── popup.html
│   ├── popup.css                      # Dark theme
│   ├── icons/                         # 16/48/128 on/off variants
│   ├── dist/                          # gitignored build output
│   └── src/
│       ├── background/
│       │   ├── service-worker.ts      # Main entry: orchestrates all modules
│       │   ├── connection-manager.ts  # WebSocket lifecycle + reconnect w/ backoff
│       │   ├── proxy-manager.ts       # PAC script gen, chrome.proxy, chrome.privacy
│       │   ├── message-bus.ts         # Popup ↔ background messaging
│       │   └── leak-guard.ts          # Monitor proxy errors, validate settings
│       └── popup/
│           ├── popup.ts               # Popup entry point
│           └── ui-renderer.ts         # DOM update functions
│
└── tests/
    └── integration/
        └── e2e_test.ts                # Full flow with real Tor
```

## Implementation Plan

### Step 1: Scaffolding + Shared Types

- [ ] Create `deno.json` workspace with members: `server`, `shared`, `extension`
- [ ] Implement `shared/types.ts` — `ConnectionStatus`, `ExitCountry`, `SplitTunnelRule`, `ExtensionSettings`, `ClientSession`, `NodeInfo`, `NodeHealth`
- [ ] Implement `shared/protocol.ts` — `ClientMessage` (auth, connect, disconnect, set_country, new_circuit, get_countries, node_health, ping) and `ServerMessage` (auth_ok, connected, disconnected, countries_list, nodes_list, node_updated, etc.) + `PopupMessage`/`PopupResponse` for extension internal messaging
- [ ] Implement `shared/constants.ts` — ports, country list, ping interval, node health interval (60s), node stale timeout (120s)
- [ ] Implement `shared/mod.ts` — barrel export

### Step 2: Deno Relay Server

- [ ] **`tor-control.ts`** — TCP client to Tor control port 9051. Commands: `AUTHENTICATE`, `SETCONF ExitNodes={cc}`, `SIGNAL NEWNYM`, `GETINFO circuit-status`. Parses multi-line responses (250- continuation, 250 terminator).
- [ ] **`proxy-bridge.ts`** — HTTP CONNECT proxy on port 8118. For each connection: parse CONNECT request → SOCKS5 handshake to 127.0.0.1:9050 → bidirectional pipe. SOCKS5 handshake: greeting [05,01,00] → method [05,00] → connect [05,01,00,03,len,domain,portHi,portLo] → response.
- [ ] **`session.ts`** — Token-based auth, session lifecycle, rate limiting (max 5 per token), periodic cleanup of expired sessions.
- [ ] **`node-registry.ts`** — Tracks all connected nodes. Handles `node_health` messages, marks stale nodes after 120s, broadcasts `nodes_list` to all connected clients on changes.
- [ ] **`ws-handler.ts`** — Dispatch `ClientMessage` types to `TorController`/`SessionManager`/`NodeRegistry`. Auth guard on privileged messages.
- [ ] **`main.ts`** — `Deno.serve()`, upgrade `/ws` to WebSocket, `/health` endpoint, CORS origin validation. Env vars: `PORT`, `PROXY_PORT`, `TOR_SOCKS_PORT`, `TOR_CONTROL_PORT`, `AUTH_TOKENS`, `TOR_CONTROL_PASSWORD`.

### Step 3: Chrome Extension Background

- [ ] **`extension/deno.json`** — workspace member config with esbuild dependency
- [ ] **`connection-manager.ts`** — WebSocket to relay with exponential backoff reconnect (1s → 30s max). Use `chrome.alarms` for heartbeat (MV3 service workers can't use `setInterval` reliably).
- [ ] **`proxy-manager.ts`** — Generate PAC script with split tunnel rules. Apply via `chrome.proxy.settings.set({ value: { mode: "pac_script", pacScript: { data: script } } })`. WebRTC leak prevention: `chrome.privacy.network.webRTCIPHandlingPolicy.set({ value: "disable_non_proxied_udp" })`. DNS leaks prevented by proxy (relay does DNS through Tor).
- [ ] **`message-bus.ts`** — Handle popup messages: `getStatus`, `connect`, `disconnect`, `setCountry`, `newCircuit`, `addSplitRule`, `removeSplitRule`, `getSettings`, `updateSettings`.
- [ ] **`leak-guard.ts`** — Listen `chrome.proxy.onProxyError`, periodic verify proxy still applied.
- [ ] **`service-worker.ts`** — Main entry point: initialize all modules, handle extension lifecycle (install, startup).

### Step 4: Chrome Extension UI

- [ ] **`manifest.json`** — MV3, permissions: `proxy`, `privacy`, `storage`, `alarms`. Background service worker.
- [ ] **`popup.html` + `popup.css`** — Dark theme, 360px wide. Sections: status indicator + toggle button, country dropdown, split tunnel rules list + add input, "New Identity" button.
- [ ] **`popup.ts`** — On load fetch status from background, wire up event listeners for toggle/country/rules/new-circuit.
- [ ] **`ui-renderer.ts`** — Pure DOM update functions: `renderStatus()`, `renderCountrySelect()`, `renderSplitRules()`.
- [ ] **`build.ts`** — esbuild: bundle `service-worker.ts` (esm) and `popup.ts` (iife) to `dist/`. Inline shared types at build time.
- [ ] **`icons/`** — Placeholder SVG icons: 16/48/128px, on/off variants (green shield = connected, gray shield = disconnected).

### Step 5: Security & Polish

- [ ] No fallback to `DIRECT` in PAC script (prevents leaks on proxy failure)
- [ ] Rate limiting on server (max 5 connections per token) — already in session.ts
- [ ] CORS validation on WebSocket upgrade — already in main.ts
- [ ] Error states in UI: Tor not running, relay unreachable, auth failure
- [ ] Icon badge shows connection state (green/gray/red)
- [ ] Graceful disconnect on extension unload

### Step 6: Tests

- [ ] **`server/session_test.ts`** — Auth, rate limiting, session expiry, cleanup
- [ ] **`server/ws-handler_test.ts`** — Message dispatch, auth guard, error handling
- [ ] **`server/proxy-bridge_test.ts`** — SOCKS5 handshake byte construction, CONNECT request parsing
- [ ] **`server/tor-control_test.ts`** — Control protocol command formatting, response parsing
- [ ] **PAC script generation tests** — Various split tunnel rule combinations, wildcard matching
- [ ] **`tests/integration/e2e_test.ts`** — Full flow: relay + WebSocket client, auth → connect → country set → disconnect

## Key Technical Decisions

| Decision | Rationale |
|---|---|
| **WebSocket = control channel only** | Traffic flows through HTTP proxy, not tunneled through WS frames. Simpler, faster. |
| **HTTP CONNECT proxy (not raw SOCKS5)** | `chrome.proxy` PAC scripts work most reliably with `PROXY` type. Relay translates to SOCKS5 internally. |
| **PAC script over fixed_servers** | Enables flexible per-domain split tunneling with wildcard support. |
| **`chrome.alarms` for keepalive** | MV3 service workers die after ~30s idle. Alarms (min 0.5 min) keep WebSocket alive. |
| **Single shared proxy port** | All clients share one Tor daemon. For multi-user production, use Tor's `IsolateSOCKSAuth` for circuit isolation per user. |
| **Optional local mode** | Same server code, user runs `deno task start` locally, extension points to `ws://localhost:8080`. |

## Verification

1. Start Tor daemon: `tor` (ensure ports 9050/9051 are listening)
2. Start relay: `deno task start` in `/server`
3. Build extension: `deno task build` in `/extension`
4. Load unpacked extension from `/extension` in Chrome
5. Click toggle → should connect → verify at https://check.torproject.org
6. Test country selection → verify exit IP matches country
7. Test split tunnel → bypassed domains should show real IP
8. Test leaks at https://browserleaks.com/webrtc and https://dnsleaktest.com
