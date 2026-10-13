# Deep dive: proving database controls in CI

[Back to README](../README.md)

Detail on §4, §5 and §10 of the README.

---

## Why the database needs its own tests

Every control that matters in this system lives in SQL: the row-level policies, the triggers, the security-definer functions, the second-factor rule, the token resolver, the consent records. Those controls ship as migrations, outside the application's deploy pipeline.

A SQL harness that runs only on a laptop proves something once. A harness that has quietly stopped passing can sit green in the repository indefinitely, and nothing will say so. Application unit tests cannot help here, because the property under test is what the database refuses when the application is not in the path at all.

---

## The harness shape

Each security migration ships with a harness that runs against a throwaway in-process Postgres inside the CI runner, and a harness joins CI only once it has been shown to pass on a clean runner. The harness tool is installed into the runner's temporary directory at a pinned version rather than into the package manifest, under a standing rule that no production dependency is added for testing.

Over time the harnesses converged on one order:

```
1. reproduce   apply the naive version and show the hole is real
2. install     apply the migration, then apply it again: it must be repeatable
3. refuse      prove each refusal, by name
4. preserve    prove that what must still work, still works
5. mutate      plant loosened variants of the guards; each must turn the check red
6. pin         print a PASS line with an exact assertion count; CI requires it
```

### Why reproduce first

A harness that only proves the fix will pass just as happily on a database that never had the hole. Reproducing the failure first proves the test can see it at all. Some examples of what the harnesses show before proving anything:

| Harness | Reproduced first |
|---|---|
| Database two-step rule | Password-only reads, writes, invite claims and connector minting |
| Consent records | A naive own-rows table accepting a backdated, an unpublished, an edited and a deleted acceptance |
| Admin writes | Logins with no seat, no profile or a disabled profile, an engineer and a shared demo login, all writing into another tenant's data through an admin check that is not NULL-safe |
| Rate-limit cap | Two checks at the same moment both passing, and the day ending above its cap |
| Gaps report | A definer function granted only to the anonymous role, invisible to a naive version of the report |

### Why preserve

A fix that refuses everything passes every refusal test. The admin-writes harness, after proving each of those identities is refused, proves that admins, an engineer on demo data and the SQL editor itself can still write. Without that half, "secure" and "broken" look identical in the log.

### Why mutate

A test that asserts too little passes against a weakened guard. So the harnesses plant loosened versions of the rules they protect and require each one to turn the check red. The consent harness plants four. The two-step harness plants loosened variants of the database rule. If a loosened guard stays green, the harness is measuring the wrong thing.

### Why pin the count

**Exit 0 is not enough, so the pass line is required.** A harness that runs only when it is the entry file depends on that check being right. One that compares a resolved path with an unresolved one is wrong in any workspace reached through a symlink or a junction: the harness runs no assertion at all and exits 0, and CI cannot tell that apart from success.

So CI requires the PASS line, and for most harnesses it requires the exact count, pinned both in CI and inside the harness. A skipped phase or an early return changes the count and turns the run red. The shell runs with pipefail, so a real failure stays red when its output is piped through to a log.

Pinned counts for the controls in this write-up:

| Control | Pinned assertions |
|---|---|
| Two-step sign-in, enforced in the database | 394 |
| Versioned consent records | 508 |
| NULL-safe admin writes | 153 |
| Atomic rate-limit cap | 60 |
| Connections minted before enrolment | 28 |

A count is a blunt instrument, and it is meant to be. It turns "the harness passed" from a claim into something CI can check.

---

## Fixtures that tell the truth about the platform

A harness that runs as a superuser proves very little about production, where a migration runs as an ordinary owner with the hosted platform's grants. The newer harnesses paste their migrations as a non-superuser owner, and their fixtures grant what the platform grants and nothing more.

Faithful fixtures matter in both directions. A migration that takes a row lock the hosted owner may not be allowed to take passes as a superuser; a fixture that gives the function owner read access on the platform's user-accounts table and nothing beyond it surfaces that before production does. In the other direction, a harness proves its fixture grants what the platform grants before it trusts any "the anonymous role cannot reach this" check, because a fixture that grants less than production makes every refusal test pass for the wrong reason.

---

## Hazards that only a new migration can change

Some harness findings describe how SQL that has already run would behave if an older migration were re-applied out of order. Those print as warnings rather than failing the build, because no code change can alter SQL that has already run; only a new migration can.

The harness also applies every later migration that touches the same object. So when the fix ships as a new file, the hazard's check turns red instead of leaving a warning in place for a hazard the new migration has already removed.

One such hazard is ordering. When a later migration tightens something an earlier one created, re-applying the earlier one reopens the hole until the later one is applied again after it, and the harness proves both halves: the reopening, and the repair. **When migrations ship outside a deploy pipeline, their order is a runtime property, not a property of the file names.**

---

## The live isolation proof

The harnesses test SQL in isolation. The isolation proof tests the real database. It signs in as the demo engineer with the public key, the same key the browser ships, and asserts that row-level security returns zero rows belonging to the owner's tenant.

The proof also fails on any gap in the second-factor gaps report, which ties the two halves of the programme together: a definer function that skips the second-factor gate is treated as an isolation failure, not a style issue.

When its secrets are not configured, the proof skips with a titled warning instead of failing, so CI is useful before the secrets exist and stronger once they do. The tradeoff is that a gate able to skip can be off without anyone noticing, which is why the skip announces itself on the run rather than passing silently.

---

## The rest of the gate

**Build with no environment.** The build step runs with no database environment on purpose. Every client factory returns null when its environment is absent, so a green build also proves nothing reaches for live data at build time.

**Secret scan on incoming commits.** It scans the pull request's own commits. The full history was audited once by hand, and re-running that on every pull request would be waste. The job gets explicit least-privilege read scopes, because a default workflow token may not be able to read the commit list, and a scanner that cannot read crashes instead of scanning. A crashed scanner and a clean one can look alike at a glance, which is the reason to treat any scanner's first green run with suspicion.

**Dependency audit, scoped to what a pull request changes.** It reads the lockfile for high or critical advisories in production dependencies. A pull request that changes dependencies must leave none; any other pull request gets a warning, and a weekly run on the main branch fails and sends an email. If the registry is unreachable, a dependency pull request fails and the rest warn. Fixes ship as their own pull request, and an automatic fix is never forced without reading what it would downgrade.

**Pull requests only.** The main branch requires a branch to be current before it can merge, so the tree CI tests on the pull request is the tree that becomes main. Running the same jobs again on the merge commit would re-test code that passed minutes earlier.

---

## What this buys

The gate turns "the database refuses this" from a sentence in a document into a count in a log line that CI checks on every pull request. When a reviewer asks how we know a control holds, the answer is a harness that showed the hole, showed the fix, showed the loosened fix failing, and printed a number.
