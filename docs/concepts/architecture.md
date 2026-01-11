---
summary: "WebSocket gateway architecture, components, and client flows"
read_when:
  - Working on gateway protocol, clients, or transports
---
# Gateway architecture

Last updated: 2026-01-11

## System architecture diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    CLAWDBOT                                         │
│                              Personal AI Assistant                                  │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              MESSAGING PROVIDERS                                    │
│                                                                                     │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│   │ WhatsApp │ │ Telegram │ │  Slack   │ │ Discord  │ │  Signal  │ │ iMessage │   │
│   │ (Baileys)│ │ (grammY) │ │  (Bolt)  │ │(discord- │ │  (CLI)   │ │ (native) │   │
│   │          │ │          │ │          │ │   js)    │ │          │ │          │   │
│   └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘   │
│        │            │            │            │            │            │         │
│        └────────────┴────────────┴─────┬──────┴────────────┴────────────┘         │
│                                        │                                           │
│                                        ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │                         PROVIDER ADAPTERS                                    │  │
│  │                    (src/providers/, src/{provider}/)                         │  │
│  │                                                                              │  │
│  │    Inbound handling · Outbound delivery · Media processing · Auth           │  │
│  └──────────────────────────────────────┬──────────────────────────────────────┘  │
└─────────────────────────────────────────┼───────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           GATEWAY (Control Plane)                                   │
│                              (src/gateway/)                                         │
│                                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                              │   │
│  │   ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐            │   │
│  │   │  WS API    │  │   HTTP     │  │  OpenAI-   │  │  Control   │            │   │
│  │   │  Server    │  │   REST     │  │  compat    │  │    UI      │            │   │
│  │   │  :18789    │  │   API      │  │   API      │  │  (Web)     │            │   │
│  │   └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘            │   │
│  │         └───────────────┴───────────────┴───────────────┘                   │   │
│  │                                    │                                         │   │
│  │   ┌────────────────────────────────┴────────────────────────────────────┐   │   │
│  │   │                      Core Services                                   │   │   │
│  │   │                                                                      │   │   │
│  │   │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    │   │   │
│  │   │  │   Router    │ │   Session   │ │   Config    │ │   Cron      │    │   │   │
│  │   │  │  (routing)  │ │   Manager   │ │   Reload    │ │  Service    │    │   │   │
│  │   │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘    │   │   │
│  │   │                                                                      │   │   │
│  │   │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    │   │   │
│  │   │  │  Provider   │ │   Bridge    │ │  Presence   │ │   Hooks     │    │   │   │
│  │   │  │  Connector  │ │   (Nodes)   │ │  Tracker    │ │  (Webhooks) │    │   │   │
│  │   │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘    │   │   │
│  │   │                                                                      │   │   │
│  │   └──────────────────────────────────────────────────────────────────────┘   │   │
│  │                                                                              │   │
│  └──────────────────────────────────────┬──────────────────────────────────────┘   │
│                                         │                                           │
└─────────────────────────────────────────┼───────────────────────────────────────────┘
                                          │
          ┌───────────────────────────────┼───────────────────────────────┐
          │                               │                               │
          ▼                               ▼                               ▼
┌──────────────────────┐   ┌──────────────────────────────┐   ┌──────────────────────┐
│    AGENT ENGINE      │   │         NODES                │   │    CLI & TOOLS       │
│   (src/agents/)      │   │   (Bridge :18790)            │   │   (src/cli/)         │
│                      │   │                              │   │                      │
│ ┌──────────────────┐ │   │  ┌────────┐  ┌────────┐     │   │ ┌──────────────────┐ │
│ │  Pi Embedded     │ │   │  │ macOS  │  │  iOS   │     │   │ │  Commands        │ │
│ │   Runner         │ │   │  │  App   │  │  App   │     │   │ │  ─────────       │ │
│ │                  │ │   │  └────────┘  └────────┘     │   │ │  gateway         │ │
│ │  • Tool calling  │ │   │                              │   │ │  agent           │ │
│ │  • Streaming     │ │   │  ┌────────┐  ┌────────┐     │   │ │  message send    │ │
│ │  • Context mgmt  │ │   │  │Android │  │ WebChat│     │   │ │  configure       │ │
│ │                  │ │   │  │  App   │  │        │     │   │ │  status          │ │
│ └──────────────────┘ │   │  └────────┘  └────────┘     │   │ │  doctor          │ │
│                      │   │                              │   │ │  sessions        │ │
│ ┌──────────────────┐ │   │  Node Capabilities:          │   │ │  cron            │ │
│ │  Tool Registry   │ │   │  • camera.capture            │   │ │  ...             │ │
│ │                  │ │   │  • screen.record             │   │ └──────────────────┘ │
│ │  • browser       │ │   │  • canvas.*                  │   │                      │
│ │  • canvas        │ │   │  • location.get              │   │ ┌──────────────────┐ │
│ │  • nodes         │ │   │  • audio.play                │   │ │  TUI             │ │
│ │  • sessions      │ │   │  • talk mode                 │   │ │  (Interactive)   │ │
│ │  • bash          │ │   │  • voice wake                │   │ └──────────────────┘ │
│ │  • code sandbox  │ │   │                              │   │                      │
│ └──────────────────┘ │   └──────────────────────────────┘   └──────────────────────┘
│                      │
│ ┌──────────────────┐ │
│ │  Model Selection │ │
│ │                  │ │
│ │  • Anthropic     │ │
│ │  • OpenAI        │ │
│ │  • Gemini        │ │
│ │  • Fallback      │ │
│ └──────────────────┘ │
└──────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              INFRASTRUCTURE                                          │
│                              (src/infra/)                                            │
│                                                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │   Daemon    │  │  Bonjour    │  │  Sessions   │  │   Config    │  │   Media    │ │
│  │  (launchd/  │  │  Discovery  │  │   Store     │  │   Parser    │  │ Processing │ │
│  │  systemd)   │  │   (mDNS)    │  │   (JSONL)   │  │  (JSON5)    │  │  (Sharp)   │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  └────────────┘ │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              CANVAS HOST                                             │
│                              (src/canvas-host/)                                      │
│                                                                                      │
│         ┌───────────────────────────────────────────────────────────────┐           │
│         │                    A2UI Canvas (:18793)                        │           │
│         │                                                                │           │
│         │    Agent-driven visual workspace · Live HTML rendering         │           │
│         │    Real-time WebSocket updates · Eval/snapshot/push            │           │
│         └───────────────────────────────────────────────────────────────┘           │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘


                              ═══════════════════════
                                  DATA FLOWS
                              ═══════════════════════

    ┌──────────────────────────────────────────────────────────────────────────────┐
    │                           INBOUND MESSAGE FLOW                                │
    │                                                                               │
    │   Provider  ──▶  Adapter  ──▶  Gateway  ──▶  Router  ──▶  Agent  ──▶  Reply  │
    │  (WhatsApp)    (auto-reply)   (server)   (resolve)    (Pi runner)  (provider)│
    └──────────────────────────────────────────────────────────────────────────────┘

    ┌──────────────────────────────────────────────────────────────────────────────┐
    │                           OUTBOUND MESSAGE FLOW                               │
    │                                                                               │
    │   CLI/API  ──▶  Command  ──▶  Provider  ──▶  Adapter  ──▶  Delivery Tracking │
    │   (send)      (handler)     (selector)    (outbound)                          │
    └──────────────────────────────────────────────────────────────────────────────┘

    ┌──────────────────────────────────────────────────────────────────────────────┐
    │                           AGENT EXECUTION FLOW                                │
    │                                                                               │
    │   Trigger  ──▶  Context  ──▶  Model  ──▶  Reasoning  ──▶  Tools  ──▶  Result │
    │  (message/      (setup)    (selection)  (streaming)    (browser,    (collect │
    │   cron/hook)                                            canvas,     & deliver)│
    │                                                         nodes...)             │
    └──────────────────────────────────────────────────────────────────────────────┘


                              ═══════════════════════
                                 KEY DIRECTORIES
                              ═══════════════════════

    src/
    ├── cli/            CLI wiring, parsers, progress UI
    ├── commands/       Command implementations (agent, message, configure...)
    ├── gateway/        Central control plane server
    ├── agents/         Pi agent execution, tools, model selection
    ├── providers/      Provider adapters and registry
    ├── whatsapp/       WhatsApp-specific (Baileys)
    ├── telegram/       Telegram-specific (grammY)
    ├── slack/          Slack-specific (Bolt)
    ├── discord/        Discord-specific (discord.js)
    ├── signal/         Signal-specific (signal-cli)
    ├── imessage/       iMessage-specific (native)
    ├── config/         Config schema, validation
    ├── sessions/       Session state (JSONL)
    ├── infra/          Daemon, bonjour, ports, migrations
    ├── canvas-host/    A2UI canvas server
    ├── routing/        Route resolution
    ├── cron/           Scheduled tasks
    ├── hooks/          Webhook integrations
    ├── media/          Media processing
    └── terminal/       UI utilities, tables, theming

    apps/
    ├── macos/          SwiftUI menu bar app
    ├── ios/            SwiftUI mobile app
    ├── android/        Kotlin mobile app
    └── shared/         Common Swift code (ClawdbotKit)
```

## Overview

- A single long‑lived **Gateway** owns all messaging surfaces (WhatsApp via
  Baileys, Telegram via grammY, Slack, Discord, Signal, iMessage, WebChat).
- All clients (macOS app, CLI, web UI, automations) connect to the Gateway over
  **one transport: WebSocket** on the configured bind host (default
  `127.0.0.1:18789`).
- One Gateway per host; it is the only place that opens a WhatsApp session.
- A **bridge** (default `18790`) is used for nodes (macOS/iOS/Android).
- A **canvas host** (default `18793`) serves agent‑editable HTML and A2UI.

## Components and flows

### Gateway (daemon)
- Maintains provider connections.
- Exposes a typed WS API (requests, responses, server‑push events).
- Validates inbound frames against JSON Schema.
- Emits events like `agent`, `chat`, `presence`, `health`, `heartbeat`, `cron`.

### Clients (mac app / CLI / web admin)
- One WS connection per client.
- Send requests (`health`, `status`, `send`, `agent`, `system-presence`).
- Subscribe to events (`tick`, `agent`, `presence`, `shutdown`).

### Nodes (macOS / iOS / Android)
- Connect to the **bridge** (TCP JSONL) rather than the WS server.
- Pair with the Gateway to receive a token.
- Expose commands like `canvas.*`, `camera.*`, `screen.record`, `location.get`.

### WebChat
- Static UI that uses the Gateway WS API for chat history and sends.
- In remote setups, connects through the same SSH/Tailscale tunnel as other
  clients.

## Connection lifecycle (single client)

```
Client                    Gateway
  |                          |
  |---- req:connect -------->|
  |<------ res (ok) ---------|   (or res error + close)
  |   (payload=hello-ok carries snapshot: presence + health)
  |                          |
  |<------ event:presence ---|
  |<------ event:tick -------|
  |                          |
  |------- req:agent ------->|
  |<------ res:agent --------|   (ack: {runId,status:"accepted"})
  |<------ event:agent ------|   (streaming)
  |<------ res:agent --------|   (final: {runId,status,summary})
  |                          |
```

## Wire protocol (summary)

- Transport: WebSocket, text frames with JSON payloads.
- First frame **must** be `connect`.
- After handshake:
  - Requests: `{type:"req", id, method, params}` → `{type:"res", id, ok, payload|error}`
  - Events: `{type:"event", event, payload, seq?, stateVersion?}`
- If `CLAWDBOT_GATEWAY_TOKEN` (or `--token`) is set, `connect.params.auth.token`
  must match or the socket closes.
- Idempotency keys are required for side‑effecting methods (`send`, `agent`) to
  safely retry; the server keeps a short‑lived dedupe cache.

## Protocol typing and codegen

- TypeBox schemas define the protocol.
- JSON Schema is generated from those schemas.
- Swift models are generated from the JSON Schema.

## Remote access

- Preferred: Tailscale or VPN.
- Alternative: SSH tunnel
  ```bash
  ssh -N -L 18789:127.0.0.1:18789 user@host
  ```
- The same handshake + auth token apply over the tunnel.

## Operations snapshot

- Start: `clawdbot gateway` (foreground, logs to stdout).
- Health: `health` over WS (also included in `hello-ok`).
- Supervision: launchd/systemd for auto‑restart.

## Invariants

- Exactly one Gateway controls a single Baileys session per host.
- Handshake is mandatory; any non‑JSON or non‑connect first frame is a hard close.
- Events are not replayed; clients must refresh on gaps.
