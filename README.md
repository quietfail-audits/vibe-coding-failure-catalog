# 🚨 The Vibe-Coding Failure Catalog
> **19 Production Architectural, Financial, and Security Traps in AI-Assisted Applications**  
> Maintained by the **[Quietfail Diagnostic Engine](https://quietfail.me)** (`@quietfail-audits`)

AI code generation tools (**Lovable, Cursor, Bolt, v0, Claude Code, GPT Engineer**) have fundamentally democratized software development. Non-technical and solo builders can ship complex, beautiful full-stack applications in days.

However, AI code generation creates a unique category of **invisible post-launch failure modes**: code that passes happy-path tests and looks visually flawless, but silently introduces runaway credit-card billing leaks, multi-tenant security bypasses, swallowed payment retries, and serverless runtime deadlocks.

This catalog is an open-source, community-driven reference documenting **19 verified production failure patterns** observed in real-world AI-generated codebases.

---

## 🗺️ The 4 Domain Archetypes

Every vibe-coded application falls into one or more functional archetypes, each with distinct structural vulnerabilities:

                     REPOSITORY / APPLICATION INTAKE
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        ▼                            ▼                            ▼
ARCHETYPE 1: COMMERCE        ARCHETYPE 2: REALTIME        ARCHETYPE 3: AI & VOICE
& TRANSACTIONS               & COLLABORATION             WRAPPERS
Triggers: Stripe, Razorpay,  Triggers: Supabase RLS,     Triggers: OpenAI Realtime,
Orders, Webhooks, Menus      WebSockets, Chat, Crypto    ElevenLabs, Edge Functions


---

## 📂 Archetype 1: Commerce & Transaction Systems

### Pattern #01: Swallowed Webhook Retries via Premature Idempotency
* **The AI Hallucination:** The AI writes a payment webhook handler (Stripe/Razorpay) and inserts the event ID into an idempotency table at the very start of the function, wrapped in a global `try/catch` that returns `HTTP 200 OK` on error.
* **The Production Trap:** If the database times out during the user subscription upgrade, the gateway receives a `200 OK` and permanently stops retrying. The user's credit card is charged, but their account remains locked forever with no automatic recovery.
* **The Fix:** Only commit the idempotency record inside the *same* database transaction as the subscription update. Return `HTTP 500` on failure so payment gateways retry automatically.

### Pattern #02: Premature State Reservation & Unpaid Resource Squatting
* **The AI Hallucination:** The AI generates an onboarding flow that inserts the user account, workspace entity, and custom public URL slug into the database *before* opening the payment checkout modal.
* **The Production Trap:** If the user abandons the payment modal or enters an invalid card, the account and routing slug remain reserved in PostgreSQL. Attackers can permanently squat high-value public vanity URLs without spending a dollar.
* **The Fix:** Hold pending registrations in temporary storage or signed tokens with a 15-minute TTL. Only commit the entity and slug after receiving payment gateway confirmation.

### Pattern #03: Client-Side Tenant Context Header Injection
* **The AI Hallucination:** To support administrative impersonation or multi-workspace switching, the client API wrapper reads a tenant ID from `localStorage` and injects it as an HTTP header (`x-tenant-id` or `x-impersonate-id`) into every request.
* **The Production Trap:** Any minor XSS flaw allows rogue scripts to tamper with `localStorage`, poisoning tenant context and modifying other users' resources. Exposing impersonation hooks in public client bundles also gives external testers a blueprint of backend auth logic.
* **The Fix:** Maintain tenant context strictly server-side using cryptographically signed session cookies or verified JWT claims. Never trust client-injected routing headers.

### Pattern #04: Midnight & Timezone Operational Cutoff Drift
* **The AI Hallucination:** Operational business hours (e.g., midnight delivery cutoff at 12:00 AM) are computed using native `new Date()` comparisons on serverless instances running in UTC.
* **The Production Trap:** In regions with positive UTC offsets (e.g., IST UTC+5:30), orders placed between 12:00 AM and 5:30 AM local time evaluate to the previous calendar day's batch, causing delayed or missing orders.
* **The Fix:** Enforce explicit timezone offsets (e.g., `date-fns-tz` or Luxon) pinned to the operational geography, never raw UTC server timestamps.

### Pattern #05: Phantom Route Feature Flag Desynchronization
* **The AI Hallucination:** The AI implements a frontend toggle for "Online Ordering / Delivery", switching the frontend client to call an internal `/api/orders` backend endpoint that the AI forgot to actually create.
* **The Production Trap:** Toggling the feature on the live website immediately crashes customer checkouts with 404 errors.
* **The Fix:** Implement contract tests ensuring every strategy declared in frontend feature flags maps to an active, validated backend route.

---

## 📂 Archetype 2: Multi-User Realtime & Collaboration Systems

### Pattern #06: Column-Level Cryptographic Key Custody Leakage
* **The AI Hallucination:** The AI sets up client-side end-to-end encryption and stores public keys and wrapped private keys in the same database table, creating an open Row Level Security (RLS) policy: `CREATE POLICY "read keys" ON user_keys FOR SELECT USING (true);`.
* **The Production Trap:** Any authenticated user can download the encrypted private key blobs, salts, and initialization vectors (IVs) of every user across the platform for offline brute-forcing.
* **The Fix:** Split public keys and private key envelopes into separate tables, or grant column-level `SELECT (user_id, public_key)` permissions only.

### Pattern #07: Permissive Junction-Table Membership Injection
* **The AI Hallucination:** On a bridge table (`conversation_members`), the insert policy only verifies `auth.uid() = user_id`.
* **The Production Trap:** Any logged-in user can bypass invite links and connection codes by directly sending a Supabase REST `POST` request inserting themselves into private conversations.
* **The Fix:** Restrict bridge table inserts to secure database RPC functions (e.g., `redeem_invite_code()`) rather than raw client table inserts.

### Pattern #08: Ephemeral State Leakage in Persistent Queries
* **The AI Hallucination:** "Disappearing" 24-hour stories or ephemeral chat messages are filtered only in the UI using frontend JavaScript (`story.created_at > yesterday`).
* **The Production Trap:** The database RLS policy omits the expiration check (`expires_at > now()`). Anyone querying the REST API directly can read all expired content indefinitely.
* **The Fix:** Enforce time-decay filters directly inside database RLS policies.

### Pattern #09: Unfiltered Realtime Global Fanout Over-Invalidation
* **The AI Hallucination:** The client subscribes to global table updates (`supabase.channel('public:messages')`) without filtering by `conversation_id`.
* **The Production Trap:** Every message sent by any user globally invalidates the cache on every connected client, degrading mobile browser performance and causing severe battery drain.
* **The Fix:** Always specify row-level subscription filters: `filter: 'conversation_id=eq.' + id`.

### Pattern #10: Committed Browser Secrets in Git Manifests
* **The AI Hallucination:** The AI writes a `.env` file containing third-party browser API keys (Google Maps, Algolia) and commits it to GitHub.
* **The Production Trap:** Public bots scrape the key within minutes, exhausting API quotas and incurring unauthorized billing charges.
* **The Fix:** Add strict `.gitignore` rules and restrict browser API keys by HTTP referrers in provider developer consoles.

---

## 📂 Archetype 3: AI Voice & Agentic Wrapper Systems

### Pattern #11: The $0.06/min Health Check (Runaway LLM Billing)
* **The AI Hallucination:** To test if the system is operational, the AI creates an unauthenticated `/health` or `/health-realtime` endpoint that connects to an external paid LLM (e.g., OpenAI Realtime WebSocket).
* **The Production Trap:** Automated uptime monitors (BetterStack, AWS, UptimeRobot) ping health checks every 60 seconds. Each ping opens an active audio/LLM session, running up charges 24/7 on the founder's credit card.
* **The Fix:** Health checks must only verify DNS/HTTPS reachability and internal DB pings. Never initialize paid external LLM sessions during health checks.

### Pattern #12: Unauthenticated Upstream API Proxy / Quota Theft
* **The AI Hallucination:** An edge function mints third-party signed URLs or access tokens (e.g., ElevenLabs Conversational Voice tokens) without validating user authentication headers.
* **The Production Trap:** External scripts can call the edge function in a loop, minting signed URLs and consuming the founder's voice/LLM quota without ever creating an account.
* **The Fix:** Enforce strict JWT verification before generating signed third-party tokens.

### Pattern #13: Stateless Serverless Rate-Limit Illusion
* **The AI Hallucination:** To prevent abuse, the AI writes an in-memory `Map` rate limiter inside a serverless edge function (Deno/Vercel/Cloudflare Workers).
* **The Production Trap:** Serverless isolates are stateless; cold starts wipe memory instantly, and state is not shared across geographic regions. Multi-threaded scrapers completely bypass the rate limit.
* **The Fix:** Use distributed Redis/Upstash rate limiting or edge middleware rate limiters.

### Pattern #14: Asynchronous Reconnection State Deadlock
* **The AI Hallucination:** In a realtime voice app, the reconnect state machine assumes socket `onclose` handlers will trigger retry loops even when the initial token fetch fails.
* **The Production Trap:** If a network blip occurs during token negotiation, the state machine freezes permanently in `'connecting'`, leaving the interface completely dead without an error banner.
* **The Fix:** Wrap token acquisition and WebSocket initialization in unified exponential backoff retry loops with explicit timeout triggers.

---

## 📂 Archetype 4: Universal Backend & Systems Infrastructure

### Pattern #15: Commented SQL Job Prompts
* **The AI Hallucination:** The AI writes SQL migrations for recurring maintenance (e.g., trial expiration or payment sweeps) but leaves the `pg_cron` scheduling commands as commented instructions in the migration file.
* **The Production Trap:** The recurring job is never actually scheduled. Trial accounts never expire, and unpaid users remain active forever.
* **The Fix:** Always verify automated cron jobs directly in PostgreSQL: `SELECT * FROM cron.job;`.

### Pattern #16: Unpaginated O(N) In-Memory Directory Scans
* **The AI Hallucination:** To check if an email already exists during registration, the backend calls `auth.admin.listUsers()` and searches the array in memory using `.find()`.
* **The Production Trap:** `listUsers()` defaults to a paginated limit (e.g., first 50 users). Any user registered past user #50 is invisible to the check, resulting in duplicates or failed signups.
* **The Fix:** Query the database directly via indexed SQL: `SELECT id FROM auth.users WHERE email = $1 LIMIT 1`.

### Pattern #17: Cross-Tenant Account Snooping via `auth != null`
* **The AI Hallucination:** Database access rules verify that a request is authenticated (`request.auth != null`), but fail to verify resource ownership.
* **The Production Trap:** Any logged-in student or user can query and download all user records, order histories, and phone numbers from browser DevTools.
* **The Fix:** Always enforce ownership predicates: `request.auth.uid == resource.data.user_id`.

### Pattern #18: Schema & Migration Drift
* **The AI Hallucination:** The AI updates application code to query new columns or tables, but forgets to generate or apply the corresponding SQL migration on the production database.
* **The Production Trap:** The deployed application crashes immediately in production with database schema mismatch errors.
* **The Fix:** Enforce CI/CD pipelines that validate atomic migration applications before frontend builds deploy.

### Pattern #19: Missing Idempotency on Financial Mutations
* **The AI Hallucination:** Checkout endpoints record financial transactions without unique database transaction constraints or idempotency headers.
* **The Production Trap:** Slow network retries or accidental double-clicks charge the user twice and generate duplicate orders.
* **The Fix:** Enforce database-level unique constraints on payment intent IDs.

---

## 🤝 How to Contribute

Have you caught an AI coding assistant creating a silent architectural or financial trap in your own project? We welcome contributions!

1. **Fork** this repository.
2. Add your pattern to the appropriate domain archetype.
3. Include:
   * **The AI Hallucination:** What prompt/behavior caused the issue.
   * **The Production Trap:** How it fails silently in production.
   * **The Fix / Prompt:** The clean code fix AND the plain-English prompt to paste back into your AI tool (Lovable/Cursor/Bolt).
4. Submit a **Pull Request**!

---

## 🛡️ About Quietfail

**Quietfail** is an independent post-launch diagnostic service for fast-shipped and AI-built applications. We run deep sanity checks across code, live web black-box endpoints, and database security rules to catch invisible operational risks before they cost you customers or money.

* 🌐 **Website & Intake:** [quietfail.me](https://quietfail.me)
* 📬 **Contact:** [quietfail.audits@gmail.com](mailto:quietfail.audits@gmail.com)
* 🐙 **GitHub:** [@quietfail-audits](https://github.com/quietfail-audits)

