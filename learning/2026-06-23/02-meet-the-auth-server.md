# Step 2 — Meet the Bouncer (`krista-auth-server`)

> Step 1 gave you the map. Step 2 walks into the first of the three buildings — the **bouncer**. By the end of this page you will know what an "opaque bearer" is, why our brain refuses to look at a JWT, and how `krista-auth-server` quietly does the hardest job in the whole system: trust.

---

## 0. Where we are

Remember our map from Step 1:

```
   ┌──────────────────────┐    ┌──────────────────────┐    ┌──────────────────────┐
   │  krista-auth-server  │    │  krista-mcp-server   │    │  The Extension       │
   │  Port :8711          │    │  Port :8710          │    │  inside accesspoint  │
   │   ▲                  │    │  Port :8710          │    │  Port :9089          │
   │  "The Bouncer"       │    │  "The Brain"         │    │  "The Admin UI"      │
   └──────────────────────┘    └──────────────────────┘    └──────────────────────┘
        ⬆ we are here
```

This page is **only** about that first box.

---

## 1. The Problem the Bouncer Solves

Imagine the brain (`krista-mcp-server`) gets a request that says:

> *"Hi, I'm Jalpesh. Show me ACME's invoices."*

The brain has no way to verify *"are you actually Jalpesh? Are you allowed to see ACME's invoices?"* on its own. It would have to:

1. Talk to Krista's identity system (Krista IAM)
2. Check passwords / sessions / SSO
3. Look up workspace memberships
4. Resolve roles

That's a LOT of identity logic. We don't want the brain doing all of that. The brain should focus on **MCP and tools**, not identity.

**Solution:** put a small specialist process **in front of** the brain whose only job is identity. That specialist is `krista-auth-server` — the bouncer.

```
                          ┌─────────────────────┐
                          │   Krista IAM        │ ← the company's identity provider
                          │   (passwords, SSO,  │   (lives outside our project)
                          │    workspaces…)     │
                          └─────────┬───────────┘
                                    │
                          ┌─────────▼───────────┐
                          │  krista-auth-server │ ← knows IAM, speaks OAuth
                          │   "The Bouncer"     │
                          └─────────┬───────────┘
                                    │
                                    │  hands out short-lived tickets
                                    │  + verifies them on demand
                                    ▼
                          ┌─────────────────────┐
                          │  krista-mcp-server  │ ← only deals with tickets,
                          │   "The Brain"       │   never with passwords
                          └─────────────────────┘
```

That separation is the single most important architectural decision in the whole project.

---

## 2. Two Kinds of "Tokens" — Passport vs Hotel Keycard

Let's get clear on what a **token** even is. There are two flavours and they behave very differently.

### 2.1 The JWT — "the passport"

A JWT (JSON Web Token) is a string that **carries your information inside it**. Anyone who reads a JWT can see:

- Your username
- Your email
- The roles you have
- The workspaces you belong to
- When the token expires

It's signed, so it can't be forged. But it is **not encrypted**. The contents are visible to anyone who base64-decodes it.

```
   📜 JWT — a passport
   ────────────────────────────────────────
   header.PAYLOAD.signature
            ↑
            └─ Base64-decode this and you see:
               {
                 "sub": "jalpesh@kristasoft.com",
                 "roles": ["admin", "developer"],
                 "workspaces": ["ws-acme", "ws-foo"],
                 "exp": 1750000000
               }
               
   ANYONE who picks up this passport reads everything.
```

### 2.2 The Opaque Bearer — "the hotel keycard"

An opaque bearer is just a **long random string**. It carries NO information. Looking at it tells you nothing.

```
   🪪 Opaque bearer — a hotel keycard
   ────────────────────────────────────────
   xQ7t9aBcD3eFgH4iJ5kLmN6oP8rStU2vW1zY0xC
                  ↑
                  └─ Tells you literally nothing.
                     Just a random string.
                     
   To find out what it can do, you must ASK the issuing
   authority: "what does this keycard grant?"
```

The bouncer keeps a database that maps `keycard → owner, expiry, permissions`. Without that database, the keycard is meaningless.

### 2.3 Quick comparison

| | JWT (passport) | Opaque bearer (keycard) |
|---|---|---|
| **Contains your info?** | Yes, in plain view | No — just a random string |
| **Anyone can read claims?** | Yes (decode base64) | No (the bouncer keeps that secret) |
| **Need to call the issuer?** | No — verify signature offline | Yes — must "swipe" against the bouncer |
| **Can you revoke early?** | No — the `exp` field is the only deadline | Yes — bouncer deletes the row instantly |
| **Lifetime** | Usually long (hours, days) | Short (minutes — ours is **10 minutes**) |
| **Leak risk** | Catastrophic (all claims exposed) | Low (a stolen random string still expires fast) |

---

## 3. Why the Brain Refuses to Look at a JWT

Here is **the single most important rule** of the whole project:

> 🚨 **The brain (`krista-mcp-server`) never, ever, parses or verifies a JWT.**
> 🪪 **Every token the brain sees is an opaque bearer.**

Why?

1. **Leak resistance.** If a bearer leaks into a log file, into a screenshot, into an error report — the attacker just has a meaningless random string that expires in minutes and can be revoked instantly. If a JWT leaks, the attacker has your full profile + claims for hours.

2. **Single source of truth.** Only the bouncer talks to Krista IAM. If 5 services each verified JWTs independently, each one would need to know about IAM, public keys, key rotation, claim shapes, etc. With opaque bearers, only the bouncer needs that knowledge.

3. **Easy revocation.** Suppose Jalpesh gets fired at 3pm. With JWTs, his token is valid until `exp` — possibly hours later. With opaque bearers, the bouncer deletes his row and his next request fails within seconds.

4. **Less surface area for the brain.** The brain has zero auth code. It just asks the bouncer: *"is this keycard good? what does it open?"* If the bouncer says yes, the brain trusts the answer.

---

## 4. The Bouncer's Two Jobs

The bouncer does exactly two things. Memorise them.

### 4.1 Job #1 — Mint (hand out a keycard)

When someone shows up and proves who they are, the bouncer creates a new opaque bearer, stores it with the user's details, and hands it back.

```
   AI client / Admin   ──────►   Bouncer
       proof of identity
   (OAuth code, or IAM JWT)
                                    │
                                    │  generates random 256-bit string
                                    │  stores row:
                                    │  { keycard: "xQ7t…", user: "jalpesh@…",
                                    │    workspaces: […], roles: […],
                                    │    expires_at: now + 10min }
                                    │
   AI client / Admin   ◄──────   Bouncer
        "here is your keycard: xQ7t9aBcD3…"
        "valid for 10 minutes"
```

### 4.2 Job #2 — Introspect (check a keycard)

When the brain receives a request with a keycard, it asks the bouncer: *"is this still good? what does it open?"*

```
   Brain   ──────►   Bouncer
      POST /oauth2/introspect
      body: token=xQ7t9aBcD3…
                                    │
                                    │  looks up row in database
                                    │  checks expiry
                                    │
   Brain   ◄──────   Bouncer
      {
        "active": true,
        "sub": "jalpesh@kristasoft.com",
        "workspace_roles": { "ws-acme": {…}, … },
        "appliance": "uslab-07",
        …
      }
```

This call happens on **every request** the brain receives. Yes, it's a lot of traffic. The brain caches recent introspection results for 60 seconds to keep performance reasonable.

> **Note:** "introspect" is an OAuth term defined in **RFC 7662**. It's just a fancy name for "ask the issuer: is this token still good?"

---

## 5. The Bouncer Has a Friend — Krista IAM

The bouncer can't actually verify users by itself either. It doesn't know passwords. It doesn't run SSO. **Krista IAM** does all of that.

```
                                            ┌──────────────────┐
                                            │   Krista IAM     │ ← passwords, SSO,
                                            │                  │   workspaces, roles,
                                            │                  │   user accounts
                                            └────────▲─────────┘
                                                     │
                                                     │  REST API
                                                     │
   ┌─────────┐                              ┌────────┴─────────┐
   │  user   │  ── 1. logs in (OAuth) ────►│ krista-auth-     │
   │         │                              │ server (Bouncer) │
   │         │  ◄── 2. gets a keycard ──────│                  │
   └─────────┘                              └──────────────────┘
```

So the bouncer is a **translator**:

- *On the outside*, it speaks **OAuth 2.1** — the open standard everyone knows.
- *On the inside*, it speaks **Krista IAM's private REST API**.

Whoever wants to use Krista MCP only ever talks OAuth. They never know Krista IAM exists.

---

## 6. Two Different Ways to Get a Keycard

We learned in Step 1 that there are **two doors** into the system (AI clients vs human admins). Each has its own way of obtaining a keycard from the bouncer. We won't go deep — just the names today.

### 6.1 Path A — for AI clients (OAuth dance)

```
   🤖 Claude Desktop  ──► Bouncer's authorize endpoint
                          (opens browser, user logs in)
                          
                          ◄── code (one-time)
                          
                          ──► Bouncer's token endpoint
                              (code + PKCE proof)
                          
                          ◄── 🪪 keycard
```

This is **the standard OAuth 2.1 PKCE flow**. Same flow Slack, GitHub, and every modern app uses. AI clients implement it because the MCP spec says they should.

### 6.2 Path B — for admin UIs (JWT exchange)

The extension (our admin UI) already has the user's identity inside Krista Studio — Studio gives it an **IAM JWT** (a passport). The extension swaps that passport at a private door of the bouncer.

```
   📝 Extension  ──► Bouncer's internal /exchange-iam-jwt
                     (presents the IAM JWT + a shared secret)
                     
                 ◄── 🪪 keycard
```

This door is **private** — protected by a shared secret only Krista-internal services know. We'll trace this in detail in Step 7.

Both paths end with the same thing: **an opaque keycard** the brain can introspect.

---

## 7. What the Bouncer Stores in Its Database

The bouncer has its own Postgres database (schema name: `auth`). At a high level:

| What it stores | Why |
|---|---|
| **Registered clients** | "Claude Desktop", "ChatGPT", etc. — each AI client app registers itself with the bouncer first |
| **Active keycards** | The current opaque bearer rows: who they belong to, what they unlock, when they expire |
| **Authorization codes** | Short-lived one-time codes from the OAuth dance |
| **PKCE challenges** | Proofs the OAuth dance was done by the same client throughout |
| **Cached role snapshots** | "Jalpesh has admin role in ws-acme, member in ws-foo" — fetched from IAM, cached for speed |

When the brain calls "introspect this keycard," the bouncer literally does `SELECT * FROM auth_tokens WHERE access_token_hash = sha256(:keycard)`. Returns the row → returns it as JSON.

> **Important security detail:** the bouncer stores only the **hash** of the keycard, not the keycard itself. Even if someone reads the database, they can't reverse-engineer working keycards. Just like how a password database stores password hashes, not plaintext passwords.

---

## 8. The Big Picture, Updated

Here's how the bouncer fits into a full request now:

```
   1. AI client gets a keycard from the bouncer (OAuth dance)
   2. AI client sends MCP request to brain WITH keycard in header
   3. Brain asks bouncer "is this keycard good?" (introspect)
   4. Bouncer says yes + here are this user's permissions
   5. Brain runs the tool, sends answer back
```

```
   ┌────────────┐                                                  
   │ AI client  │                                                  
   └─────┬──────┘                                                  
         │                                                         
         │ 1. OAuth dance ─────────────────────►┌────────────────┐ 
         │ ◄────────────────────── 🪪 keycard ──│  krista-auth-  │ 
         │                                      │     server     │ 
         │ 2. POST /mcp                         │  (Bouncer)     │ 
         │    Authorization: Bearer xQ7t9…      │                │ 
         │                                      │                │ 
         ▼                                      │                │ 
   ┌────────────┐                               │                │ 
   │ krista-mcp │ 3. POST /oauth2/introspect ──►│                │ 
   │   -server  │                               │                │ 
   │  (Brain)   │ ◄── 4. {active:true, …} ──────│                │ 
   │            │                               └────────────────┘ 
   └─────┬──────┘                                                  
         │                                                         
         │ 5. ──── answer ─────►                                  
         ▼                                                         
   ┌────────────┐                                                  
   │ AI client  │                                                  
   └────────────┘                                                  
```

Beautiful. Two simple jobs (mint + introspect). One private friend (IAM). Zero auth logic in the brain.

---

## 9. What You Now Know

✅ The bouncer is a separate process that handles ALL identity logic, so the brain doesn't have to.
✅ There are two kinds of tokens: **JWTs** (passports — carry info openly) and **opaque bearers** (keycards — random strings, meaning lives in the bouncer's database).
✅ **The brain only ever sees opaque bearers. Never a JWT.** This is the most important rule of the system.
✅ The bouncer has two jobs: **mint** new keycards, and **introspect** old ones to say if they're still good.
✅ The bouncer is a translator: speaks **OAuth 2.1** outward, speaks **Krista IAM's REST API** inward.
✅ Keycards are short-lived (10 minutes), revocable instantly, and stored as hashes — not plaintext.
✅ There are **two ways to get a keycard**: AI clients use the OAuth dance; admin UIs use a private JWT exchange.

---

## 10. What's Next (Step 3)

In Step 3 we walk into the **second building — `krista-mcp-server`, the brain**. We will see:

- The two doors of the brain, in more detail (`/mcp` vs `/admin/**`)
- What "tools" actually are in MCP land
- The brain's persistent memory: Postgres + Infinispan cache
- How the brain stays consistent across many running pods

Approve this file → I will write Step 3.

---

*This is Step 2 of an ongoing walk-through.*
*Folder: `learning/2026-06-23/`*
*Project: krista-global-catalog / mcp-server-service*
*Author: Jalpesh's learning notes, drafted with Claude Code*
