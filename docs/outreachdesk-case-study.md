# OutreachDesk: building and operating a production SaaS solo, with agents in the loop

This is the engineering companion to the [agentic pentest case study](../README.md). One product, built and run by one person on one VPS, then attacked with the same agentic tooling I sell as a security capability.

The interesting part is not that AI wrote code. Anyone can get an agent to emit code. The interesting part is the machinery that kept an 83,000-line codebase coherent, deployable and auditable while most of the typing was done by agents.

---

## 1. What the thing actually is

A multi-tenant outreach platform: sequences, prospect CRM, unified reply inbox, deliverability controls, consent and suppression, social scheduling, ticket helpdesk, and a public marketing site. It replaces tools like Instantly or Smartlead, self-hosted, with the data staying on the customer's own infrastructure.

| Dimension | Reality |
|---|---|
| Application code | ~83,000 lines (41k Python, 41k TypeScript) across 227 source files |
| Commit history | 480 commits, May to October 2026 |
| Backend | FastAPI on Python 3.12, SQLAlchemy, APScheduler with 24 in-process jobs |
| Frontend | React 19, TypeScript, Vite, React Router 7, Tailwind |
| Data | PostgreSQL, 48 tables |
| Identity | Keycloak 26.1, realm groups mapped to product entitlements |
| Infra | One 8-core / 31 GB VPS, Traefik, Cloudflare Tunnel, Cloudflare Pages for the static frontend, ClamAV sidecar |
| Model layer | OpenRouter (zero-retention) as primary, local Ollama profile available, per-tenant bring-your-own-key supported |

I am the only engineer. There is no QA team, no on-call rotation, and no one else to blame.

---

## 2. Architecture in one paragraph

Cloudflare terminates TLS and serves the static SPA from the edge. API traffic goes through a Cloudflare Tunnel to Traefik, which routes to a single FastAPI container. That container is both the API and the scheduler: 24 APScheduler jobs run in-process, coordinated by a database leader lock so a second instance cannot double-fire. PostgreSQL holds everything. Keycloak holds identity, and a sync job maps Keycloak group membership to the product's feature tier on every login. A ClamAV sidecar scans uploaded CSV and XLSX before ingest.

The deliberate constraint: **no Redis, no Celery, no message broker.** Everything that would normally need one has an in-process fallback. That is a bad choice at 50 tenants and a good choice at five, and I chose it knowing which one I was in.

---

## 3. Five decisions I would defend in a design review

### 3.1 Exactly-once sends via an outbox row

Cold email has one unforgivable failure: the same prospect receives the same email twice. The naive implementation sends, then writes the audit row. If the process dies between the two, the next scheduler tick sees an unsent row and sends again.

Instead, a row with a unique idempotency key is committed to `outbox_events` **before** the SMTP call. Success flips the row to processed and the email log to sent in one transaction. A crash after SMTP still leaves the outbox row, so the retry is a no-op, and the duplicate never happens. Cost: an extra table and one more write per email. Worth it.

### 3.2 Keyword fast path with LLM override

Reply classification drives autonomous action: unsubscribes cancel every sequence and write a consent record, hostile replies pause and block, out-of-office pauses for a week. Routing that on a model call means routing breaks when the model is slow, rate limited, or down.

So classification runs twice. A keyword pass gives an instant, deterministic answer. If the model is available, its answer can override. If it is not, the conservative keyword decision stands. Degradation instead of failure.

### 3.3 Async tracking queue, because pixels are a write storm

Open and click tracking do not write to the database on request. The endpoint enqueues an event in memory, returns a 1x1 GIF or a redirect in under 5 ms, and a 60-second job batches the writes. A campaign that spikes to thousands of opens in a minute produces thousands of enqueues and a handful of batched updates instead of thousands of row locks.

### 3.4 Pure ASGI middleware, not BaseHTTPMiddleware

The first version used Starlette's `BaseHTTPMiddleware` for CORS, rate limiting and security headers. The failure mode is subtle and expensive: when an unhandled exception fires, that middleware swallows the response and the CORS headers with it, so the browser reports a CORS error and hides the actual 500. Debugging incidents that do not exist is how you lose a weekend.

Rewrote both as pure ASGI middleware. Now every error response keeps its headers, and the real error is visible in the client.

### 3.5 Consent as an append-only audit, not a boolean

Consent is stored with a type (express or implied), a timestamp and an expiry. Implied consent expires after two years, and an expired prospect is blocked automatically rather than being emailed because nobody remembered. Every change appends to `consent_audit_logs`; application code never updates or deletes from it. Unsubscribe is RFC 8058 one-click with a signed token, no login required, because a recipient who has to log in to opt out will just hit "report spam" instead.

---

## 4. What broke, and what I changed

Four failures that shaped the codebase. All four are real, all four are in the product's history.

| Failure | Root cause | Fix |
|---|---|---|
| Duplicate sends after a container restart | Send and audit write were not atomic | Outbox pattern with idempotency key, committed before SMTP |
| Browser showed CORS errors on every 500 | `BaseHTTPMiddleware` swallowed the error response | Pure ASGI middleware for CORS, rate limit and security headers |
| Database contention during large campaign opens | Per-request writes to `email_logs` | In-memory tracking queue, batched flush every 60 seconds |
| Sequences kept firing after "out of office" replies | Auto-replies treated as engagement | Auto-reply suppression, out-of-office pauses the sequence for seven days, hostile replies block the prospect |

I did not find these by reading code. Three of them were found by running the product against real mailboxes and watching what happened, and the fourth was found by my own pentest pipeline. That is the argument for building the verification loop before building more features.

---

## 5. AI-assisted development: what actually worked

The honest version, because the dishonest version is useless to anyone hiring.

**What agents were genuinely good at**

- Breadth. Forty-plus API routers, forty-plus frontend routes, a 48-table schema. Work that is mostly typed correctly or not at all.
- Mechanical refactors across many files with a consistent pattern.
- First drafts of services with a clear contract: a bounce checker, a spam scorer, an SPF analyzer.
- Documentation that stays in sync, because regenerating it is cheap.

**What agents were bad at, consistently**

- Inventing plausible configuration. A default that looks reasonable and silently changes behaviour is worse than a crash, because nothing tells you.
- Tests that pass without asserting anything that matters. Coverage without verification is a confidence trick you play on yourself.
- Contract drift between frontend and backend: a field renamed in one place, still expected in the other, discovered in production.
- Security smells. Trusting a user-supplied URL, logging a token, skipping authorization on one route out of forty. This is exactly why the pentest pipeline exists.

**The rules that made it work**

1. Spec first, in writing, before any agent touches code. Ambiguity is what agents fill in with invention.
2. One concern per change. Agents happily do four things at once and then you cannot tell which one broke it.
3. Verify live, never by claim. An agent saying "implemented and tested" is a hypothesis. A smoke test against the running container is evidence.
4. Read the diff. Every time. The cost of review is the price of the leverage.
5. Freeze infrastructure files (Dockerfile, compose, env, proxy config) behind explicit approval, so a helpful agent cannot quietly rewrite the deployment path.
6. Keep an audit trail of what the model was asked and what it produced. Mine stores a SHA-256 fingerprint of every prompt rather than the prompt itself, which proves processing without retaining customer data.

---

## 6. What I would rebuild

- **Split the scheduler out of the API container.** It works today, and it is the first thing that breaks under real load. A separate worker process with its own scaling story is the correct shape.
- **Stateless OAuth.** The container runs a single worker because OAuth state lives in memory. That constraint is invisible until someone tries to scale it.
- **Redis from day one.** The in-memory fallbacks were a deliberate simplification and they now cost more than the dependency would have.
- **Event-sourced send state.** The outbox fixed duplicates. A proper state machine for every email would have made the incident class disappear rather than being patched.
- **Contract-first API.** Generate the TypeScript client from the OpenAPI schema instead of maintaining two definitions by hand.

---

## 7. The part that is genuinely unusual

The same asset appears twice in this portfolio: once as a product I built and operate, and once as the target of my agentic penetration testing pipeline, with the findings published.

That loop is the whole argument. Build it, attack it with your own tooling, publish what the tooling found, fix it, attack it again. Most portfolios show either product work or security work. The interesting claim is that the two make each other better, and the evidence is the report.

Source is private (live product, real tenant data, real credentials). Read-only architecture walkthrough, redacted code excerpts and a live product tour are available on request: **a.aly@live.com**
