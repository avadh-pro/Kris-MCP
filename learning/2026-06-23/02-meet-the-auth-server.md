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
                          │  krista-auth-server │ ← knows IAM, speaks OAuth   {Q1: Why only OAuth?}
                          │   "The Bouncer"     │
                          └─────────┬───────────┘
                                    │
                                    │  hands out short-lived tickets   {Q2: Is this the opaque bearer token?}
                                    │  + verifies them on demand
                                    ▼
                          ┌─────────────────────┐
                          │  krista-mcp-server  │ ← only deals with tickets,
                          │   "The Brain"       │   never with passwords
                          └─────────────────────┘
```

> **A1 — Why only OAuth?**
>
> OAuth 2.1 isn't "only" — it's the ONE auth protocol every modern AI client (Claude Desktop, Cursor, ChatGPT, Perplexity, …) already knows how to speak out of the box. By choosing it we get three things for free:
>
> 1. **Interoperability.** RFCs 6749 (core), 7636 (PKCE), 7662 (introspection) are all standards. Any client that follows them can talk to Krista without writing Krista-specific code. The MCP spec itself says "if you need auth, use OAuth."
> 2. **Battle-tested security.** PKCE, state, audience, token rotation — every edge case in OAuth has been pen-tested at scale. Inventing our own protocol would mean re-discovering those bugs.
> 3. **A clean separation of concerns.** OAuth is what the world sees on the *outside*; Krista IAM is our private system on the *inside*. The auth-server translates between them. If we ever swap Krista IAM for a different identity store, the OAuth surface clients see does NOT change.
>
> So "only OAuth" is really "OAuth as the one public-facing language; whatever internals we like behind the curtain."

> **A2 — Is this the opaque bearer token?**
>
> Yes. "Short-lived ticket" / "hotel keycard" / "opaque bearer" are three names for the same thing in this doc. The auth-server hands one out after a successful login; the brain swipes it on every request to verify the user is still allowed in. Section 2.2 below explains what makes it "opaque."

That separation is the single most important architectural decision in the whole project.

---

{Q3: Who produces the JWT token the first time for any user? And where is it stored — IAM or krista-auth-server?}

> **A3 — Who issues the JWT, and where does it live?**
>
> **Krista IAM issues the JWT.** IAM is Krista's identity system — it owns user accounts, passwords, SSO, workspace memberships, and roles. When a human signs in to Krista Studio (the appliance), IAM authenticates them and mints a signed JWT carrying their identity claims.
>
> **The JWT is NOT persistently stored by any of our three services.** Specifically:
>
> | Component | Stores the JWT? | What it does instead |
> |---|---|---|
> | **Krista IAM** | No (it issues a fresh one each time the user signs in) | Stores the *underlying* user account + workspace/role facts in its own DB |
> | **Studio (browser session)** | Yes — short-lived, in the user's session / cookie only | Forwards it to the extension on every UI call as the `X-Krista-Access-Token` header |
> | **The Extension** | **No** — reads the JWT off the inbound HTTP header and immediately exchanges it for an opaque bearer | Caches only the resulting *opaque bearer* in memory for 9 minutes (`BearerCache`) |
> | **`krista-auth-server`** | **No** — validates the JWT's signature + claims, then throws it away | Stores only the resulting *opaque bearer hash* in its `auth` schema |
> | **`krista-mcp-server` (the brain)** | **No** — it never sees a JWT at all | Only receives opaque bearers |
>
> Said simply: **JWTs live in transit, never at rest in our services.** They are passports carried in the user's pocket and shown at doors — never stored by the doormen.

## 2. Two Kinds of "Tokens" — Passport vs Hotel Keycard

Let's get clear on what a **token** even is. There are two flavours and they behave very differently.

### 2.1 The JWT — "the passport"

A JWT (JSON Web Token) is a string that **carries your information inside it**. Anyone who reads a JWT can see:   {Q4: Who reads it, and where is it stored?}

- Your username
- Your email
- The roles you have
- The workspaces you belong to
- When the token expires

> **A4 — Who reads the JWT, and where is it stored?**
>
> **Readers (in order, on one request from a Studio user):**
>
> 1. **Studio (browser side)** reads the JWT to set up the user's session — primarily to know who the user is for UI display.
> 2. **The Extension** reads the JWT off the inbound `X-Krista-Access-Token` HTTP header (see `IamJwtExtractor` in the extension's `phase2/` package). It extracts `accountId` + `workspaceId` claims, then forwards the JWT to step 3.
> 3. **`krista-auth-server`** reads the JWT during the `/internal/admin/exchange-iam-jwt` call. In production it verifies the signature, checks expiry, checks the issuer, then extracts claims to populate the new opaque-bearer's row. **(In today's codebase the default mode is `PERMISSIVE` — see the reality-check under A5 for why.)**
> 4. **`krista-mcp-server` (the brain)** **NEVER reads the JWT.** Hard rule. That's the whole point of having an auth-server.
>
> **Storage:** as covered in A3 above, the JWT is **never persisted at rest** by any of our three services. It lives in transit only — browser session memory, an HTTP request header, a method-local variable. The moment auth-server is done validating it, the JWT is GC-collected and the only thing that survives is the opaque bearer hash in `auth_tokens`.

It's signed   {Q5: What does "signed" mean, and who signed it?}, so it can't be forged. But it is **not encrypted**. The contents are visible to anyone who base64-decodes it.

> **A5 — What does "signed" mean, and who signed it?**
>
> A JWT has three parts joined by dots: `header.payload.signature`.
>
> "Signed" means the **signature** part was computed with a **private key** held by the issuer. The math goes:
>
> ```
> signature = HMAC_or_RSA_or_EC( header + "." + payload,  IAM's_private_key )
> ```
>
> **Who signed it:** Krista IAM. IAM holds the private key — that's its secret. Nobody else has it.
>
> **Why this matters:** anybody who has IAM's *public* key can RE-COMPUTE the signature over the header + payload they received and check it against the signature in the JWT. If even one bit of the payload has been tampered with, the recomputed signature won't match and the JWT is rejected.
>
> So "signed" doesn't mean "secret" — it means **tamper-evident**. The contents are still readable to anyone (you just base64-decode the payload). But you can't *change* a claim without invalidating the signature. Think of it as a wax seal on a letter: anyone can read the letter, but breaking the seal proves the letter has been opened or modified.
>
> In our system, `krista-auth-server` is the only service that holds IAM's public key and runs the signature check. That's why the brain never sees JWTs — it would have to know about IAM's public key, key rotation, allowed algorithms, etc. We push all of that into the auth-server.
>
> ---
>
> **🔍 Reality check — what today's codebase actually does**
>
> The auth-server has **two verifier modes**, controlled by the property `krista.iam.jwt.verification-mode`:
>
> | Mode | Behaviour | Trust anchor |
> |---|---|---|
> | **`PERMISSIVE`** (current default) | Parses the JWT's payload but **does NOT verify the signature**. | The `X-Internal-Secret` header on `/internal/admin/exchange-iam-jwt` — only services that know the shared secret can call this endpoint at all. |
> | **`SIGNED`** (production target) | Verifies the JWT's signature against IAM's published JWKS (public keys), checks issuer + audience + expiry. | The cryptographic signature itself. |
>
> Code references: `IamJwtVerifierProperties.Mode { PERMISSIVE, SIGNED }`, `PermissiveIamJwtVerifier`, `NimbusIamJwtVerifier`.
>
> **Why two modes?** Krista's IAM didn't always publish a stable JWKS endpoint. `PERMISSIVE` lets the JWT-exchange flow work safely *today* (gated by the internal secret) while the team finishes the JWKS plumbing. Once that lands, the property flips to `SIGNED` and the internal-secret remains as a **defense-in-depth** second layer.
>
> **For your mental model:** in this doc we describe what JWTs "are supposed to" guarantee (tamper-evident signature). In production code right now, the internal-secret is the actual gate. Both are true at different layers of the conversation.

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

An opaque bearer is just a **long random string**. It carries NO information. Looking at it tells you nothing.   {Q6: So the bearer is *generated from* the JWT?}

> **A6 — Is the bearer generated *from* the JWT?**
>
> **No — the JWT is not an input to the bearer's value.** The bearer is fresh random bytes, completely independent of the JWT's contents. The auth-server does:
>
> ```
> bearer = base64url( SecureRandom.nextBytes(32) )       // 256 cryptographic random bits
> ```
>
> No hashing of the JWT, no derivation, no mathematical relationship.
>
> **What IS related:** the JWT is *validated* before the bearer is minted. The flow is:
>
> 1. Auth-server receives the JWT.
> 2. Verifies signature + expiry + issuer + audience.
> 3. Extracts identity claims (`sub`, `workspace_roles`, etc.).
> 4. Generates a fresh random bearer (step above).
> 5. Stores a row: `(bearer_hash, identity_from_JWT, expiry_now_plus_10min)`.
> 6. Returns the plaintext bearer to the caller.
>
> So the JWT is the **proof-of-identity** that *unlocks* a new bearer — but the bearer's bytes are randomly generated. The relationship is "presented, validated, identity copied over to a new row" — not "derived from."

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

The bouncer keeps a database that maps `keycard → owner, expiry, permissions`. Without that database, the keycard is meaningless.   {Q7: Where is the opaque bearer first stored with its identity? Is it during the OAuth 2.1 flow via krista-auth-server? Does IAM hand back the JWT, which auth-server then trades for the bearer?}

> **A7 — First storage point + flow of identity → JWT → bearer**
>
> Yes to all three. Here's the precise sequence, broken out for both doors into the system:
>
> **🚪 Path A — AI clients (Claude Desktop, ChatGPT, Cursor, …) — pure OAuth 2.1 / PKCE flow:**
>
> ```
> 1. AI client redirects user's browser → auth-server's /oauth2/authorize
> 2. auth-server bounces user to Krista IAM's login page
> 3. User logs in (password / SSO)
> 4. IAM redirects back → auth-server with proof of who the user is
> 5. auth-server hands AI client a one-time `code`
> 6. AI client → auth-server's /oauth2/token (sends code + PKCE proof)
> 7. ★ auth-server NOW generates the random bearer, stores it in
>    auth_tokens(bearer_hash, identity, expiry), and returns the
>    plaintext bearer to the AI client
> ```
> The bearer's **first storage** is step 7, in auth-server's `auth_tokens` table. Note: in this path the JWT is not even visible to our services — IAM owns the user's auth session internally and only hands back a `code` to the auth-server.
>
> **🚪 Path B — Admin UIs (the Extension, called from Studio) — JWT exchange:**
>
> ```
> 1. User signs in to Studio; IAM mints an IAM JWT
> 2. Studio calls the Extension with X-Krista-Access-Token: <IAM JWT>
> 3. Extension's IamJwtExtractor reads the JWT off the header
> 4. Extension's JwtExchangeClient → auth-server's
>    POST /internal/admin/exchange-iam-jwt
>    (sends the JWT + a shared secret only Krista-internal services know)
> 5. auth-server verifies the JWT, extracts identity claims
> 6. ★ auth-server generates the random bearer, stores it in
>    auth_tokens(bearer_hash, identity_from_JWT, expiry), returns plaintext
> 7. Extension caches the bearer in BearerCache (in-memory, 9-min TTL)
>    keyed on (accountId, workspaceId)
> 8. Extension uses the bearer on every Phase 2 admin call to the brain
> ```
> Here the bearer's **first storage** is step 6, again in `auth_tokens`. This is the path your sentence describes: "IAM provides you with the JWT and auth-server leverages that JWT to mint the bearer." Exactly right.
>
> **In both paths, the persistent storage of the bearer happens in exactly one place: `auth_tokens` inside `krista-auth-server`'s Postgres schema `auth`.** And what's stored is the **hash** (SHA-256) of the bearer, never the plaintext — see Section 7 of this doc.

### 2.3 Quick comparison

|                              | JWT (passport)                            | Opaque bearer (keycard)                         |
|------------------------------|-------------------------------------------|-------------------------------------------------|
| **Contains your info?**      | Yes, in plain view                        | No — just a random string                       |
| **Anyone can read claims?**  | Yes (decode base64)                       | No (the bouncer keeps that secret)              |
| **Need to call the issuer?** | No — verify signature offline             | Yes — must "swipe" against the bouncer          |
| **Can you revoke early?**    | No — the `exp` field is the only deadline | Yes — bouncer deletes the row instantly         |
| **Lifetime**                 | Usually long (hours, days)                | Short (minutes — ours is **10 minutes**)        |
| **Leak risk**                | Catastrophic (all claims exposed)         | Low (a stolen random string still expires fast) |

---

## 3. Why the Brain Refuses to Look at a JWT

Here is **the single most important rule** of the whole project:

> 🚨 **The brain (`krista-mcp-server`) never, ever, parses or verifies a JWT.**
> 🪪 **Every token the brain sees is an opaque bearer.**   {Q8: How is the opaque bearer generated the first time? Is it per user? Who makes it?}

> **A8 — How, per-user, and who?**
>
> **WHO:** `krista-auth-server` makes it. Nobody else. The brain doesn't make tokens; IAM doesn't make opaque bearers (IAM makes JWTs); the extension doesn't make tokens — it only *holds* them. Only the auth-server mints.
>
> **HOW:** at the moment of mint (either step 7 of Path A or step 6 of Path B in A7 above), the auth-server runs roughly:
>
> ```java
> byte[] randomBytes = new byte[32];                 // 256 bits
> SecureRandom.getInstanceStrong().nextBytes(randomBytes);
> String bearer = Base64.getUrlEncoder()
>                       .withoutPadding()
>                       .encodeToString(randomBytes);
>
> String hashForStorage = sha256(bearer);            // never store the plaintext
>
> jdbc.update(
>     "INSERT INTO auth.auth_tokens(token_hash, sub, workspace_roles, expires_at) "
>         + "VALUES (?, ?, ?, ?)",
>     hashForStorage,
>     identityFromValidatedJWTorOAuthFlow.sub(),
>     identityFromValidatedJWTorOAuthFlow.workspaceRoles(),
>     Instant.now().plus(Duration.ofMinutes(10)));
>
> return bearer;   // plaintext flows OUT to the caller, never stored
> ```
>
> Two security properties fall out of this:
>
> 1. **Cryptographic randomness** — 256 bits from `SecureRandom` means an attacker has 1-in-2^256 odds of guessing a valid bearer. That's not a number; that's a "this will never happen before the heat death of the universe" guarantee.
> 2. **Hash-at-rest** — even if someone steals the `auth_tokens` table, they get hashes, not bearers. Hashes cannot be replayed against the brain — the brain wants the plaintext bearer to do introspect.
>
> **PER USER:** yes — every bearer is bound to **exactly one user's identity** at the moment of mint. The row stores who it belongs to (`sub`), which workspaces they're an admin in (`workspace_roles`), which appliance they're on, and so on. If Jalpesh asks for a bearer twice in a row, he gets two different bearers, two different rows. They share his identity but are otherwise independent tickets.
>
> So: 256 random bits, made by the auth-server, bound to one user, stored as a hash, returned in plaintext exactly once.

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

| What it stores            | Why                                                                                            |
|---------------------------|------------------------------------------------------------------------------------------------|
| **Registered clients**    | "Claude Desktop", "ChatGPT", etc. — each AI client app registers itself with the bouncer first |
| **Active keycards**       | The current opaque bearer rows: who they belong to, what they unlock, when they expire         |
| **Authorization codes**   | Short-lived one-time codes from the OAuth dance                                                |
| **PKCE challenges**       | Proofs the OAuth dance was done by the same client throughout                                  |
| **Cached role snapshots** | "Jalpesh has admin role in ws-acme, member in ws-foo" — fetched from IAM, cached for speed     |

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
