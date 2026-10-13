# Securing AI Agents on Private Financial Data

Engineering notes from my work as the founding AI engineer at an early-stage fintech handling users' financial data: a research agent with guardrails and a streaming safety filter, an MCP server with hashed, revocable access tokens, and the security programme around users' financial data and bank-data access. The platform is read-only by design. It never moves money.

**The non-obvious part is that the browser holds a key that can talk to the database directly. Once that is true, any rule enforced only in the application is a suggestion, and an AI agent is just one more client holding that key. So every control that matters, from two-step sign-in to the agent's identity, is enforced in the database and re-checked on every call, with CI proving it rather than a document asserting it.**

Design notes and reasoning only. No client name, no product features beyond the security surface, no source code, no SQL, no configuration, no credentials, no customer data and no data-vendor names.

**Deep dives:** [MCP tokens and the third-party boundary](docs/01-own-assistant-over-mcp.md) · [Proving database controls in CI](docs/02-proving-database-controls-in-ci.md)

---

## Contents

1. [Problem shape](#1-problem-shape)
2. [Architecture: the database is the boundary](#2-architecture-the-database-is-the-boundary)
3. [Row-level security as the trust boundary](#3-row-level-security-as-the-trust-boundary)
4. [Two-step sign-in, enforced in the database](#4-two-step-sign-in-enforced-in-the-database)
5. [Versioned consent records](#5-versioned-consent-records)
6. [The request guard](#6-the-request-guard)
7. [The research agent](#7-the-research-agent)
8. [MCP tokens and the third-party boundary](#8-mcp-tokens-and-the-third-party-boundary)
9. [Monitoring that does not leak](#9-monitoring-that-does-not-leak)
10. [The CI security gate](#10-the-ci-security-gate)
11. [Failure modes](#11-failure-modes)
12. [What I'd do differently](#12-what-id-do-differently)

---

## 1. Problem shape

The platform reads a person's financial data and never moves money. Read-only sounds like it should make security easy. It doesn't, because the asset at risk is confidentiality, and confidentiality fails without a sound: a leaked row raises no error.

Three kinds of reader needed the same private data:

| Reader | Acts for | What makes it hard |
|---|---|---|
| The signed-in person, in the browser | Themselves | The browser holds a public database key by design |
| The in-app research agent | The signed-in person | It must act with exactly the person's authority, never more |
| The person's own AI assistant, over MCP | The person, from outside | A long-lived credential in software nobody here controls |

Engineers see demo data only, and the whole thing had to stand up to a bank-data aggregator's security review; the dependency audit in §10 exists because of a question in it.

---

## 2. Architecture: the database is the boundary

The intuitive build is an app server that checks the session, the role and the second factor, then queries the database with a privileged key. That was ruled out deliberately: **the app holds no privileged key that can reach customer data.** Every client it creates against customer data uses the public key the browser ships, and row-level security is the real boundary.

The consequence is the whole architecture. Anyone with a valid session can call the database's data API directly and skip every check the app makes. A role check in a route handler protects the route, not the data.

```
 browser (public key + session) ------------------+
      |                                           |  direct data-API calls
      v                                           |  skip the app entirely
 request guard: origin, second factor, rate limit |
      |                                           |
 app routes + research agent                      |
      |        user's own assistant -> MCP route  |
      v                               v           v
 +-----------------------------------------------------------+
 | database: row-level security on every private table,      |
 | second-factor rule in policies and definer functions,     |
 | token resolver, append-only consent records               |
 +-----------------------------------------------------------+
        ^ CI: isolation proof, SQL harnesses,      | error and uptime
        |     secret scan, dependency audit        v monitoring
```

The guard still matters, but the last line is the database: the only layer every path goes through.

**The test I applied to every control: if the person opened the browser console and called the data API with their own session, would it still hold? If not, the control was not finished.**

Holding no privileged key has a cost: anything that writes rows a user cannot write directly becomes a security-definer function, running with its owner's rights, and each one must restate the rules below by hand.

---

## 3. Row-level security as the trust boundary

Every private financial table scopes its reads through one visibility predicate: one place to audit, one place to fix.

**Team access is demo-only, by trigger.** No path exists for an engineer or any other non-owner seat to see a real customer's data; support access, if ever needed, must be designed with an audit trail first.

**The audit rule.** The quiet failure in row-level security is an over-broad policy. One that gives the anonymous and signed-in roles every operation under an always-true condition leaves a table readable and writable with no login, and nothing raises an error. Probing the obvious financial tables with the public key says nothing about the rest. **Every audit must enumerate every table, not the obvious ones**, and every probe should expect permission denied.

---

## 4. Two-step sign-in, enforced in the database

Two-step sign-in is the rule for team logins and an option for customers.

An app-level check that refuses a team session without a completed second factor closes the front door, not the data API: a password-only session is still a valid session, and it can call the data API directly. So the rule lives in the database. For a login holding a verified second factor, a password-only session is refused in the table policies, at the top of every security-definer function (which would otherwise bypass the policies), when claiming an invitation, and when minting an MCP connection. Even the function that provisions a new account refuses a session that still owes its code.

The harness reproduces each bypass on a database without the rule first, then proves the rule and plants loosened variants that must turn red: 394 pinned assertions.

**The gaps report.** Definer functions are where the rule is easiest to forget, so a report reviews them for a missing second-factor gate, and the RLS isolation proof fails on any gap. An audit tool can have a blind spot of its own: a definer granted only to the anonymous role is easy to leave out of a review. So the report's harness plants that case and requires the report to see it. **An audit tool needs the same adversarial testing as the thing it audits.**

One more edge: each MCP connection records the assurance level it was minted at, so one minted before its owner enrolled is refused afterwards. See [docs/01](docs/01-own-assistant-over-mcp.md).

---

## 5. Versioned consent records

Consent is evidence. When someone asks what a customer agreed to, the answer must come from a record nobody could have edited: not the customer, and not us.

The naive design is an own-rows table where each user manages their own acceptance. The harness reproduces what that accepts before proving the design below: a backdated acceptance, one of a version never published, an edit after the fact, and a deletion.

| Property | Why |
|---|---|
| Server-clock timestamp | A client-supplied time can be backdated |
| Registered versions only, stored with a hash of the text | Proves which wording was agreed, and that it was published |
| Written only in the agreeing person's own session | Nobody, the team included, accepts on someone's behalf |
| Append-only, for the owner too | The history only grows |
| A removed login leaves a tombstone; retention runs from removal | Deleting an account does not erase what it agreed to |
| The two-step rule applies | Consent follows the same rule as every other private write |

Four loosened guards are planted and must each turn red: 508 pinned assertions. It matters for the AI side too: the in-app agent runs on the company's own model-provider account, a service provider, and bank data will reach it with the customer's consent.

---

## 6. The request guard

Before a request reaches a route, a guard runs a cross-site check (an origin the app does not accept is refused), a second-factor check, and per-user rate limits. The second-factor check duplicates the database rule on purpose: the database is the boundary, the guard is the fast, legible refusal.

**Rate limits that hold under concurrency.** "Count recent rows, then insert if under the cap" has a race: two simultaneous checks both see room, both pass, and the day ends above its cap. The harness reproduces that race on a count-then-insert limiter, then proves the design I used instead, which holds an advisory lock on the limit's key until commit. It also pins that a refusal writes nothing and exactly which rows count toward the cap. One side effect of per-user windows: everything signed in as one account shares one budget, which is why the CI smoke test is serialised.

---

## 7. The research agent

The agent's guardrails are mostly about what it is never given, not what it is told.

- **Identity comes from the session, never from the model.** Who the agent acts for is taken from the signed-in session on the server.
- **Nothing to exfiltrate.** Conversations are not persisted, so there is no archive to ask for.
- **Read-only tools.** Every tool the agent can call reads; none writes, and like the rest of the platform it never moves money.

**The streaming filter.** Every answer passes a shared output gate between the model's stream and the user, because text already shown cannot be withdrawn.

---

## 8. MCP tokens and the third-party boundary

The MCP server is the one path by which software outside the platform reads private data: an AI assistant the user chose, reading that user's own data with a long-lived token. That makes the token the most exposed credential in the system. Tokens are stored as hashes and revocable by their owner; a revoked token returns nothing. One resolver in the database starts every call, so revocation, the second-factor rule and the per-user rate limit are checked at call time, never cached in the token. The server uses the public key only, with no write and no arbitrary query proxy.

The decision I made here was about bank data. A user's own assistant is a third party we do not control, and the aggregator's terms require its written permission before bank data passes to one. So the assistant sees what the user entered and nothing from the bank until the aggregator agrees in writing. It cost nothing while no bank data existed, and opening it later is a small change.

Full detail: [docs/01-own-assistant-over-mcp.md](docs/01-own-assistant-over-mcp.md)

---

## 9. Monitoring that does not leak

A default error event is close to a privacy incident. So events are rebuilt, not scrubbed: the original is discarded and a minimal one built with only the error type, scrubbed stack locations, release and runtime metadata, and a fixed set of permitted tags. Request data, page addresses, user data, prompts, breadcrumbs, context, amounts and tracing are disabled. **Build from a permitted list, don't strip a forbidden one:** a scrubber leaks the field nobody thought of.

Browser events pass through a bounded relay that sends a fresh request without visitor headers, and content-security-policy reports are reduced to fixed categories. An uptime check runs every minute from outside the deployment, and scheduled jobs send check-ins, with a high-priority email rule.

Two rules for job monitoring:

- **Alert lag is a design parameter.** The missed-run margin and the maximum runtime decide how long a failure stays silent, so set them on purpose. And a monitor exists only after its job's first check-in.
- **Read it the day it ships.** A monitor nobody reads is a claim, not a control. Confirm the alert path with evidence rather than asserting it.

---

## 10. The CI security gate

A security claim in a document is only as good as the check behind it. The gate's job is to make a sentence like "a pull request reaching for financial data fails CI" true, and keep it true. The main branch rejects direct pushes, admins included, and gates merging on the required checks.

| Check | Proves | Decision inside it |
|---|---|---|
| Secret scan | No new secret in the pull request's commits | Explicit read scopes; a crashed scan must not pass for a clean one |
| Dependency audit | No high or critical advisory in production dependencies | Blocks only pull requests that change dependencies; others warn |
| RLS isolation proof | A demo engineer with the public key sees zero owner rows | Fails on any second-factor gap |
| SQL harnesses | Each security migration refuses what it should | Reproduce first; pinned counts; required pass line |
| MCP smoke test | A connection minted without the second factor is refused, on demo data | Serialised repository-wide |
| Build with no environment | Nothing reaches for live data at build time | Client factories return null without env |

The dependency audit's scoping is the subtle one: advisories appear under an untouched lockfile and core framework packages are pinned exactly, so failing every pull request would freeze main. A weekly run on main fails and emails instead.

Full detail: [docs/02-proving-database-controls-in-ci.md](docs/02-proving-database-controls-in-ci.md)

---

## 11. Failure modes

| Failure | Symptom | Mitigation |
|---|---|---|
| Password-only session calls the data API | App two-step check bypassed | Rule in policies, definers, invites and minting |
| Definer skips the second-factor gate | Bypass no policy catches | Gaps report; isolation proof fails on any gap |
| Admin check not NULL-safe | No-seat, disabled, engineer and demo logins can write | NULL-safe checks; harness reproduces each identity first |
| Over-broad policy on an unaudited table | Open with no login | Enumerate every table |
| Simultaneous rate-limit checks | Cap exceeded | Advisory lock until commit |
| Error event carries personal data | Leak to a third party | Event rebuilt from a permitted list |
| Harness runs nothing, exits 0 | Green build, nothing tested | Required pass line, pinned count |

**A check that can pass without checking is worse than none: it removes the worry along with the protection.**

---

## 12. What I'd do differently

**Wire every security claim to a check the day it is written.** Until the check exists, the sentence is a hope that reads like a fact.

**Generate audit scope from the database catalogue.** A hand-picked list covers the tables someone thought of; the catalogue covers the tables that exist.

**Test audit tools as hard as controls.** Plant the case a tool is meant to catch, and require it to catch it, before trusting its silence.

**Require the pass line from the first harness.** A harness can exit 0 without running a single assertion, and nothing else in the log tells those two apart.

**Read the monitor on day one.**

**Give CI its own login.** A full end-to-end walk needs a login that can complete a second factor. Copying a person's authenticator secret into CI is the wrong answer; a CI-only login with no factor, seeded with demo data only, is the clean one.

---

## Stack

Next.js · TypeScript · Supabase (Postgres, row-level security, security-definer functions) · Claude API · MCP · Vercel

---

*Engineering notes only, published with the client's permission. Contains no source code, SQL, configuration, credentials, endpoints, customer data, data-vendor names, product features beyond the security surface, or client-identifying information.*
