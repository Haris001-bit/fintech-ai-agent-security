# Deep dive: MCP tokens and the third-party boundary

[Back to README](../README.md)

Detail on §8 of the README, and on the minting edge case in §4.

---

## What is actually being granted

The in-app research agent runs on the platform's server, inside a signed-in session, on the company's own model-provider account. A user's own assistant is none of those things. It is software the user chose, running wherever they run it, holding a credential that outlives any browser session, and reaching in over MCP from outside the platform.

| | In-app agent | User's own assistant |
|---|---|---|
| Runs | On the platform's server | Wherever the user runs it |
| Credential | The current session | A long-lived MCP token |
| Model provider | The company's own account, a service provider | Whatever the user chose: a third party |
| Bank data | With the customer's consent | Not until the aggregator agrees in writing |

Every decision below follows from treating that token as the most exposed credential in the system.

---

## The token

- **Minted by the signed-in user.** Where the owner holds a verified second factor, a password-only session cannot mint one; that is part of the database rule in README §4.
- **Stored as a hash**, never as the token itself.
- **Revocable by its owner.** A revoked token returns nothing.
- **Stamped on use.** A successful call records the token's last-used time, and the smoke test asserts the stamp.
- **Records the assurance level it was minted at.**

That last property exists for one edge case. A connection minted before its owner enrolled a second factor would otherwise be a password-only credential that quietly survives the enrolment. The resolver refuses a password-only-minted connection from an owner who now holds a factor, and the harness for that case pins 28 assertions.

---

## One resolver, re-checked on every call

Every tool call starts at the resolver, which lives in the database. It checks revocation, the second-factor rule and the per-user rate limit at call time.

**Why every call, rather than once at minting:** a token's standing changes after minting. It can be revoked, or its owner can enrol a second factor. If the check happened only when the token was created, each change would leave a working credential behind, inside software nobody at the platform can reach.

The model on the other end of the connection asks for a tool by name; who it is asking as comes from the token, never from the model.

---

## A deliberately small surface

- **Public key only.** The MCP server has no service-role dependency.
- **No writes.** Every tool is read-only.
- **No arbitrary table or query proxy.** Tools are named and fixed in shape.

The obvious MCP build is a generic "query my data" tool, since the assistant on the other end is perfectly capable of writing queries. That is also an arbitrary query proxy behind a long-lived token held by a third party. Named tools with fixed shapes are less flexible, and that is the point: the set of things a leaked token can do is the set of tools, and it is short.

---

## The bank-data decision

Once bank connections go live, should bank data be readable over this connection? Two options were on the table:

| Option | For | Against |
|---|---|---|
| Keep bank data out of MCP for now | Matches what the privacy policy says; keeps the aggregator application clean; costs nothing while no bank data exists | The user's own assistant sees less |
| Let it through from day one | More useful to the user | The aggregator's terms require its written permission before bank data passes to a third party, and a user's own assistant is a third party we do not control. Declaring that in an application is the kind of thing that slows one down or gets it refused |

I chose the first. A user's own assistant sees everything the user entered themselves and nothing that came from their bank. The aggregator gets asked in writing, and if it agrees, opening the connection to bank data is a small change.

This is separate from the in-app agent, which runs on the company's own model-provider account. That counts as a service provider, so bank data can go there with the customer's consent (README §5).

**The general point: "the user asked for it" does not turn a third party into the user.** The assistant acts for the user, but its provider, its logs and its retention sit outside the platform's control and outside its agreements. The access design has to price that in even when the user is the one who connected it.

---

## The route over the network

The database covers authority. The MCP route itself has its own behaviour to prove: the two protocol eras it accepts, origin refusal, header-mismatch refusal, and the expected 404s.

Pull requests get no preview deployment, so the CI job builds the app and starts it inside the runner rather than depending on one.

---

## Testing it in CI

The smoke test is written to walk the path a real user walks: mint a token on demo data, call the tool as the anonymous role with that token, and assert both that data comes back and that the last-used time was stamped. The full walk covers the two-tenant proof, revocation, the last-used stamp, the network walk and the rate-limit burst; it needs a login that can complete a second factor (the last decision below).

Three decisions shaped it.

**Serialised across the whole repository.** The rate-limit window is counted per user, and every run signs in as the same engineer account. Two smoke runs on different branches whose bursts land within a minute of each other share one budget, and the second reads as a failure. A per-branch concurrency group cannot see the other branch, so the smoke test has its own repository-wide group. In-progress runs are not cancelled, because cancelling the runner does not un-spend the window; it only loses the run that had already paid for it.

**Armed only when the database can pass it.** A check that fails against a database behaving exactly as its applied migrations say is how people learn to ignore red builds. So an arm of the smoke test, such as the rate-limit burst, switches on the day the migration it tests is applied, and the job becomes a required check only once the migrations it depends on are in place: a required check that cannot pass blocks the very fix that makes it pass.

**No personal authenticator secret in CI.** Minting at the second-factor level needs a code. Copying a person's authenticator secret into CI would make a setup key that is meant to exist once exist twice, somewhere any collaborator able to edit a workflow could reach it. The design answer is a CI-only login with no second factor, seeded with demo data only, so a leaked CI credential reaches nothing real.
