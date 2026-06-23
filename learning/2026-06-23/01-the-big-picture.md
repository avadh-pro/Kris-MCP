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

## 2. Two Kinds of Users, Two Kinds of Doors

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
 │      │  (JSON-RPC)                  │  (HTTP verbs)           │
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

> **Why two doors?** AI calls are high-frequency, machine-to-machine, and want streaming responses. Admin calls are low-frequency, human-driven, and want CRUD verbs. Mixing them would be messy. So we keep two separate front doors on the same building.

---

## 3. The Three Buildings

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

## 4. The Boot Order (Wake-Up Sequence)

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

## 5. What Each Door Looks Like to a User

**Door #1 — the AI client door** (`POST /mcp`):

```
🤖 Claude Desktop  ─────►  https://krista-mcp:8710/mcp
                            │
                            │  body: {
                            │    "jsonrpc": "2.0",
                            │    "method": "tools/call",
                            │    "params": { "name": "list_invoices", … }
                            │  }
                            ▼
                          🧠 brain runs the tool, returns the answer
```

**Door #2 — the admin UI door** (`/admin/...`):

```
👤 Admin in browser ──►  https://krista-mcp:8710/admin/governance/tools
                            │
                            │  GET → list all approved tools
                            │  POST → approve a new tool
                            │  PUT → change approval
                            │  DELETE → revoke approval
                            ▼
                          🧠 brain reads/writes its governance database
```

That's it for the first step. **No code. No file paths. Just the layout of the world.**

---

## 6. What You Now Know

✅ MCP is a standard protocol that lets AI assistants talk to business systems.
✅ Our project is the Krista-side implementation of MCP.
✅ It has **two doors**: one for AI clients (`/mcp`), one for human admins (`/admin/**`).
✅ It is split into **three independent processes**: auth-server, mcp-server, extension.
✅ They **boot in a specific order**: auth → brain → extension.

---

## 7. What's Next (Step 2)

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
