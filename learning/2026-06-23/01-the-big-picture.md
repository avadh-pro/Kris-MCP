# Step 1 — The Big Picture

> The very first step. Forget the code for a moment. Let's understand **why this project exists** and **who the players are**. We will not look at a single file yet. Just the shape of the world.

---

## 1. The Problem We Are Solving

Imagine you have an AI assistant — like **Claude Desktop**, **ChatGPT**, **Cursor**, or **Microsoft Teams**.

You ask it: *"Hey, what are the open invoices for ACME Corp in our Krista system?"*

The AI does not know. It cannot reach into your company's Krista platform. It only has its training data and the conversation you are typing in. **It needs a bridge.**

That bridge is called **MCP — the Model Context Protocol**.

```
         "What are ACME's open invoices?"
                       │
                       ▼
           ┌─────────────────────┐
           │   Your AI client    │   ← Claude Desktop, ChatGPT, Cursor, Teams
           └──────────┬──────────┘
                      │
                      │   speaks MCP (a standard "language")
                      │
                      ▼
           ┌─────────────────────┐
           │   Krista MCP Server │   ← what we are building
           └──────────┬──────────┘
                      │
                      │   speaks Krista's internal language
                      │
                      ▼
           ┌─────────────────────┐
           │   Krista Platform   │   ← invoices, customers, automations…
           └─────────────────────┘
```

**One sentence:** MCP is the standardised "USB plug" that lets any AI client talk to any business system. Our project is the Krista-side end of that plug.

---

## 2. How the AI Client Talks to the Brain (Transport)

MCP is just a **language** (JSON-RPC 2.0). But every language needs a **medium** to travel over. MCP supports two:

### 2.1 The language — JSON-RPC 2.0

Every MCP message looks like this — regardless of medium:

```jsonc
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "list_invoices",
    "arguments": { "customer": "ACME" }
  }
}
```

Always JSON. Always has `jsonrpc`, `id`, `method`, `params`. Same envelope, no matter how it gets there.

### 2.2 The two pipes — stdio vs HTTP

```
   ┌────────────────────────┐       ┌────────────────────────┐
   │  STDIO transport       │       │  HTTP transport        │
   │  (server runs locally) │       │  (server runs remote)  │
   └────────────────────────┘       └────────────────────────┘

   🤖 Claude Desktop                  🤖 Claude Desktop
        │                                  │
        │  spawns child process            │  POST https://krista-mcp:8710/mcp
        │  stdin/stdout pipes              │  Content-Type: application/json
        │                                  │  Authorization: Bearer <opaque>
        ▼                                  ▼
   📂 filesystem MCP                  🧠 krista-mcp-server (remote)
   📂 git MCP                         🐙 GitHub MCP (hosted)
   📂 sqlite MCP                      🌍 Atlassian MCP (hosted)
   (local processes)                  (HTTPS endpoints)
```

| Transport | Server lives… | Used for |
|---|---|---|
| **stdio** | as a child process of the AI client | local helpers (filesystem, git, sqlite) |
| **HTTP** | as a remote HTTPS endpoint | **our Krista MCP server**, GitHub MCP, Atlassian MCP |

Our project is the **HTTP** flavour — Krista runs the MCP server centrally, and any AI client connects to `POST /mcp` over HTTPS.

### 2.3 Why HTTP needs a "streaming" mode (SSE)

Normal HTTP is like sending an **SMS**: one message, one reply, line closes.

```
   You: "What's the weather?"   ──────►   Server
   Server: "Sunny, 22°C"        ◄──────   (line closes)
```

That's fine for *"one question, one answer."* But our brain has TWO situations where one-shot HTTP isn't enough.

**Situation A — long answers in chunks**

You ask Claude: *"Read my last 100 emails and summarise."* The brain takes 8 seconds. With plain HTTP, Claude stares at a blank screen and then suddenly gets a wall of text. We'd rather Claude show *"reading email 1… reading email 2…"* progressively.

**Situation B — the brain has news, unprompted**

The admin (in a different browser tab) clicks **"approve the new `delete_invoice` tool"**. The brain now has a new tool. How does it tell every connected AI client *"hey, refresh your tool list"*? With plain HTTP, the brain cannot speak unless asked. No open channel exists.

**The fix: SSE (Server-Sent Events).** It's a feature of HTTP that says *"keep this connection open; the server keeps writing until it decides to close."*

```
                   normal HTTP                          SSE-style HTTP

   You:  "tools/call"   ───►                    You: "tools/call"   ───►
                                                                          (line stays open)
   Server: "answer"     ◄───                    Server: chunk 1     ◄───
   (connection closes)                          Server: chunk 2     ◄───
                                                Server: notification ◄─── (brain pushed this on its own!)
                                                Server: chunk 3     ◄───
                                                Server: "done"      ◄───
                                                (now the line closes)
```

**One request from the client. Many messages from the server.** Same `POST /mcp` URL — the response just has `Content-Type: text/event-stream` and the brain writes message after message into that pipe.

### 2.4 Why stdio doesn't need SSE

Stdio is already a **permanently open phone line**. The pipes between Claude Desktop and a local MCP server are always open until one side quits. Either side can write a message anytime.

```
     Claude Desktop  ◄═══════════════════════════►  Local MCP server
                      stdin/stdout pipes, both
                      sides can write anytime,
                      line never closes
```

So stdio gets "the server can push messages anytime" **for free**. HTTP had to invent SSE to fake the same behaviour.

### 2.5 What this means for our brain

Our `krista-mcp-server` does this for every connected AI client:

1. Claude Desktop sends `POST /mcp` (one request).
2. Brain assigns it an `Mcp-Session-Id` and keeps the SSE channel open.
3. The connection stays open — minutes, hours.
4. The brain pushes down it:
   - **Streaming tool responses** (long answers in chunks).
   - **`notifications/tools/list_changed`** when an admin approves/revokes a tool.
   - **Status updates** during long tool runs.
5. Claude closes the channel when the user quits.

So when you hear "**persistent SSE channel**" or "**audit fan-out via `notifications/tools/list_changed`**" — that's literally the brain whispering down this open SSE pipe to all connected AI clients in real time.

**TL;DR:** *HTTP normally closes after one reply. SSE keeps it open so the brain can stream long answers AND push surprise notifications. Stdio is already always-open, so it doesn't need SSE.*

---

## 3. Two Kinds of Users, Two Kinds of Doors

Our project does NOT only serve AI clients. It also serves **human admins** who need to decide *which tools the AI is allowed to use*, *for which user*, *in which workspace*.

So there are two front doors:

```
 ┌───────────────────────────────────────────────────────────────┐
 │                                                               │
 │   Door #1                          Door #2                    │
 │   ━━━━━━━                          ━━━━━━━                    │
 │                                                               │
 │     🤖                              👤                         │
 │   AI client                       Human admin                 │
 │   (Claude Desktop,                (uses Krista Studio         │
 │    Cursor, Teams…)                 in a browser)              │
 │                                                               │
 │      │                              │                         │
 │      │  speaks MCP                  │  speaks REST            │
 │      │  (JSON-RPC + SSE)            │  (HTTP verbs, no SSE)   │
 │      ▼                              ▼                         │
 │   POST /mcp                      /admin/governance/...        │
 │                                  /admin/skills/...            │
 │                                  /admin/refresh/...           │
 │                                                               │
 └───────────────────────────────────────────────────────────────┘
                          │
                          │  same backend serves both
                          ▼
                ┌──────────────────────┐
                │   krista-mcp-server  │
                └──────────────────────┘
```

> **Why two doors?** AI calls are high-frequency, machine-to-machine, and want streaming responses (SSE). Admin calls are low-frequency, human-driven, and want CRUD verbs. Mixing them would be messy. So we keep two separate front doors on the same building.

---

## 4. The Three Buildings

The project is actually **three separate processes** running side by side. Each is a small Java application. They talk to each other over HTTP.

```
   ┌──────────────────────┐    ┌──────────────────────┐    ┌──────────────────────┐
   │  krista-auth-server  │    │  krista-mcp-server   │    │  The Extension       │
   │                      │    │                      │    │  (runs inside        │
   │  Port :8711          │    │  Port :8710          │    │   "accesspoint"      │
   │                      │    │                      │    │   on Port :9089)     │
   │  "The Bouncer"       │    │  "The Brain"         │    │                      │
   │                      │    │                      │    │  "The Admin UI"      │
   └──────────────────────┘    └──────────────────────┘    └──────────────────────┘
```

Just to get the vibe — we will go deeper in later steps:

| | What it does in ONE line | Think of it like… |
|---|---|---|
| **`krista-auth-server`** | Checks "is this person who they claim to be?" | The bouncer at the club entrance |
| **`krista-mcp-server`** | Speaks MCP, runs tools, stores governance rules | The brain doing the actual work |
| **The Extension** | Shows the admin UI and talks to the brain | The receptionist in front of the brain |

> **Important:** The three buildings **share no code**. They are independent. They only talk via HTTP. You could swap any one of them out tomorrow as long as the HTTP contract stays the same.

---

## 5. The Boot Order (Wake-Up Sequence)

You cannot start them in any order. There is a dependency chain.

```
    Step 1:   krista-auth-server wakes up      ▶  (needs Postgres)
                       │
                       ▼  "I'm ready to check identities"
    Step 2:   krista-mcp-server wakes up       ▶  (needs auth-server)
                       │
                       ▼  "I can verify bearers and serve tools"
    Step 3:   The Extension loads inside accesspoint
                       │
                       ▼  "Admin UI is now live"
```

If you skip Step 1, Step 2 will keep returning errors because it cannot verify any user. If you skip Step 2, Step 3 will load but every button click fails. Always boot in order: **auth → brain → extension**.

---

## 6. What Each Door Looks Like to a User

**Door #1 — the AI client door** (`POST /mcp`, JSON-RPC over HTTP + SSE):

```
🤖 Claude Desktop  ─────►  https://krista-mcp:8710/mcp
                            │
                            │  body: {
                            │    "jsonrpc": "2.0",
                            │    "method": "tools/call",
                            │    "params": { "name": "list_invoices", … }
                            │  }
                            ▼
                          🧠 brain runs the tool, streams the answer
                              back over the same open SSE channel
```

**Door #2 — the admin UI door** (`/admin/...`, plain REST):

```
👤 Admin in browser ──►  https://krista-mcp:8710/admin/governance/tools
                            │
                            │  GET → list all approved tools
                            │  POST → approve a new tool
                            │  PUT → change approval
                            │  DELETE → revoke approval
                            ▼
                          🧠 brain reads/writes its governance database
                              (and pushes `tools/list_changed` over
                              SSE to every connected AI client)
```

Notice the lovely connection between the two doors: when an admin uses Door #2 to approve a tool, the brain automatically whispers down every open SSE channel on Door #1 to tell connected AI clients *"refresh your tool list."* That's how the system stays consistent in real time.

That's it for the first step. **No code. No file paths. Just the layout of the world.**

---

## 7. What You Now Know

✅ MCP is a standard protocol that lets AI assistants talk to business systems.
✅ MCP is **JSON-RPC 2.0** carried over either **stdio** (local servers) or **HTTP** (remote servers).
✅ HTTP MCP uses **SSE** so the server can stream answers and push notifications down a long-lived connection. Stdio doesn't need SSE because the pipe is always open.
✅ Our project is the **HTTP** flavour: a remote MCP server Krista hosts centrally.
✅ It has **two doors**: one for AI clients (`/mcp`), one for human admins (`/admin/**`).
✅ It is split into **three independent processes**: auth-server, mcp-server, extension.
✅ They **boot in a specific order**: auth → brain → extension.
✅ Admin writes on Door #2 fan out as `notifications/tools/list_changed` on every open SSE channel on Door #1 — real-time consistency.

---

## 8. What's Next (Step 2)

In Step 2 we will zoom into the **first building — `krista-auth-server`**. We will learn:
- What an "opaque bearer" is and why the brain only accepts those (never JWTs)
- How the auth-server is connected to *Krista IAM* (the platform's identity provider)
- The single most important rule: **the brain never parses a JWT**

Approve this file → I will write Step 2.

---

*This is Step 1 of an ongoing walk-through.*
*Folder: `learning/2026-06-23/`*
*Project: krista-global-catalog / mcp-server-service*
*Author: Jalpesh's learning notes, drafted with Claude Code*
