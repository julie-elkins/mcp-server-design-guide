# Designing MCP Servers for Agent Callers

*A working standard, and the four failures you only find by audit.*

by Julie Elkins

---

Most guidance on building Model Context Protocol servers reads like API design
guidance, because MCP looks like API design. It isn't, quite. An HTTP API is called
by code that a human wrote against a spec they read. An MCP tool is called by a
model that read only your schema, chose arguments from it, and will turn whatever
comes back into a confident sentence for a person who cannot see the payload.

That single asymmetry generates most of what follows. The failure I design against
is not the exception — an exception is loud and someone fixes it. It's the
**incomplete, capped, stale or refused result being reported as a complete one.**
The tool worked. The agent summarized. The answer was wrong and nothing anywhere
said so.

These are the fifteen practices I apply to every new server by default, followed by
four things that no tool will ever complain about and that I have only ever found by
sitting down and auditing. But first, the question that comes before all of them.

## Should it be a server at all?

There is a recurring argument that MCP servers are a token-inefficiency trap — that
every tool definition rides along in the prompt on every turn, so cost grows with the
size of your surface, and you would be better off with a single `execute_code` tool
that discovers capabilities at runtime. The argument is half right, and the right half
is not fixed by building the server well. So settle it before writing a schema.

**The mechanism claim is out of date.** I measured my own setup on 2026-09-21: 26
configured servers publishing roughly 690 tools. Tool *schemas* were not resident. Only
the names were, and a schema had to be fetched before its tool could be called — the
client had already implemented the progressive disclosure the critique asks for. What
stays resident is the name list plus each server's `instructions` block. That is a real
cost and a much smaller one, and a critique aimed at "the whole surface travels every
turn" is arguing with a version that has been fixed.

**The billing claim is also wrong, and it matters that you know why.** A static tool
surface sits in the cached prefix of the prompt, where re-reads bill at a fraction of
input price. "Linear growth with each turn" describes context *occupancy*, not the
invoice. If you argue about surface size on token grounds you will get a caching
rebuttal and you will deserve it.

**What does not get cheaper is the model's ability to choose.** Every tool you add is
one more candidate to be chosen wrongly among, and a wrong-tool choice is silent in
exactly the way this whole document is about — the call succeeds, the agent
summarizes, nobody sees the payload. There is a cruder version of the same hazard:
tool names are capped in length, and a server whose name is long enough pushes its
longest tool names past the cap, where they simply stop existing with no error
anywhere. Surface size degrades correctness before it degrades your bill. Argue it
there.

So the decision rule is not about tokens. It is about what the server's value
actually is:

- **If the value is access** — authenticated reach into an API that a script could
  have called anyway — then one code-execution tool beats one tool per operation. This
  is not hypothetical; at least one major cloud provider ships precisely that shape,
  a single `run_script` tool with the SDK importable inside it, and it displaces
  dozens of per-operation tools at the cost of one name.
- **If the value is the contract** — a refusal channel, a list of the checks that were
  skipped, a stated provenance on every threshold, a "here is what this result does not
  show" attached to every payload — then it has to be a server, because the contract
  *is* the product. Hand an agent raw access instead and it reaches the same verdicts
  with nothing left to stop it reporting an empty finding list as a clean result.

Which means the interesting case is the mixed server: mostly thin wrapper, with one or
two tools carrying a real contract. Split it. Do not defend it whole.

**One measurement trap, because I walked into it.** If you want to know which of your
servers earn their surface, count invocations — and be careful how. Matching tool names
across 220 of my own session transcripts ranked my *least*-used server first: one
publishing around 130 tools, with 65 real calls in its life, scored nearly 118,000
apparent "uses." Every session's tool listing names every tool a server publishes, so a
wide idle server outscores a narrow busy one by construction. Parse the actual tool-call
records instead. When I did, nine servers accounted for about 330 of the 690 names
against 87 calls in all of recorded history.

A wide surface manufactures its own evidence of being needed. That is the same failure
this document is built around — an absent signal read as a present one — committed one
level up, against your own architecture instead of against your users.

## The schema is your entire user interface

**1. Write parameter prose that says what a *wrong* value does.** Not what the type
is — the type is already in the schema. An agent choosing between two plausible
arguments needs the consequence, not the restatement. Compare:

```
queue: str   # The routing queue name.
```
```
queue: str   # Match VERBATIM, including live typos -- queues get created with
             # misspelled names ('infrastucture') and nobody renames them once
             # things route there. This tool does not normalize, lowercase or
             # spell-correct, and a corrected spelling returns zero rows while
             # looking exactly like a genuine absence of data.
```

The second one prevents a class of silent wrong answer. The first one is decoration.

**2. Return structured output models, never raw dictionaries.** A signature that
says `-> Dict[str, Any]` publishes an empty schema, which means the only way for the
agent to learn the response shape is to call the tool and look. Type it. Pydantic,
dataclasses, whatever your stack gives you — the point is that the shape is
discoverable before the call, not after.

**3. Write error messages that name the correction, not the rule.** "Person not
found" tells an agent nothing it can act on. "Not found — call `find_person` with a
display name instead, then retry with an alias it returns" gets the next call right.
Every refusal should contain its own remedy.

**4. Sensible defaults, with overrides.** The common call should work with the
required argument alone. Optional parameters that are effectively mandatory are a
trap: the agent omits them and gets a result that is technically correct and not
what anyone wanted.

**5. Validate at the boundary, and assume adversarial-by-accident inputs.** Agents
pass empty strings, `None`, depth 100, aliases with an `@` and an email domain
attached. Cap and reject explicitly, because the alternative is a query that
technically executes.

**6. Refusal is a result, not a failure.** This is the load-bearing one. Return a
structured "I cannot answer this reliably" rather than a plausible-looking
placeholder. The agent will format whatever you hand it, so bad data must not be
*able* to reach it. Give refusals their own type, and give each distinct cause its
own channel — because different causes have different remedies, and the remedy is
the useful part. A channel whose fix is *local configuration* cannot be merged with
one whose fix is *retry with a different argument*.

**7. Synthetic test fixtures, so tests never touch live systems.** Four shapes, at
minimum: a known-good record, a nested/multi-level one, an error case, and one large
enough to exercise your caps.

**8. Every magic number carries its provenance.** A citation, a measurement with the
date it was taken, or an explicit design decision. An unlabeled threshold is a
defect, and the reason is social rather than technical: an unexplained gate gets
lowered by the next person under time pressure, while a gate that says
*MEASURED 2026-08-30, below this the false-positive rate exceeds 12%* gets met.

**9. Logging, never `print` — but be precise about the hazard.** stdout *is* the MCP
transport on stdio. The usual advice stops there, which leaves people believing any
stray print kills the session. Measured against the 2.x Python SDK: `stdio_server`
claims fd 1 and points it at stderr, so a *buffered* print — and even a log handler
aimed at `sys.stdout` — gets retargeted and the session survives. What actually
corrupts the handshake is a **flushed** write before that diversion happens:
`print(..., flush=True)`, or anything at all running under `python -u`. Name
`sys.stderr` explicitly anyway. That buys you portability across SDK versions and
direct library use, not survival on this particular transport.

**10. Diagnostics travel *with* the result.** Caveats belong in the payload, not in a
docstring the agent never sees at call time. Include data freshness and source, not
only warnings — "this is 15 minutes stale" changes how an answer should be
presented, and only the payload can say it.

**11. A README with a real call and its real reply.** Not an API reference. One
concrete round trip teaches more than a parameter table.

**12. Pin major versions, commit your lockfile.**

**13. Ship a CLAUDE.md** (or equivalent agent-facing doc): what the server does in
plain language, when to use each tool, what order to call them in, and the gotchas.

**14. A server fronting an expiring credential must publish the remaining lifetime,
and its refusal must name the *cheapest* remedy.** Three parts, and the third is the
one everybody skips.

The setup that taught me this: a short-lived posture cookie sitting underneath a
much longer session. An expiry mid-run looks exactly like a broken tool. And a
single blanket "re-authenticate" charged the user a hardware-key touch for the
failure that happens several times a day, when a cheap refresh command fixed it in a
second.

So: (a) put the deadline in *every* payload, so the refresh can happen **before** a
call fails rather than after; (b) branch the refusal by *which* credential lapsed,
checking the expensive tier first, so a genuinely dead session can't be sent round
the cheap loop forever; and (c) give every branch a fallback, including one that
admits the local credential state does not explain the failure.

Two corollaries that cost me real debugging time. If the server caches, **do not
gate the expiry warning on cache origin** the way you gate freshness — the run being
served from cache is precisely the one that still has time to act. And invalidate a
loaded credential on **file mtime**, not only on a refusal: if you advised a refresh
and the user took it, the renewal arrives with no refusal to invalidate anything.

Read expiry *metadata* and never a credential *value*. Not in a log line, not in a
payload. If you need a non-credential field out of a credential file — a username,
say — take the exception deliberately: classify the safe names explicitly, expose a
**named** accessor rather than a general `get_field(name)` helper, and have it assert
its own target is on the safe list before reading. The generalized version is how
the next value out of that file becomes a session token.

**15. If callers will ask about *themselves*, resolve identity in a tool — and
publish how it was resolved beside the answer, never folded into it.** "Who is my
manager" is among the most common questions any directory-shaped server gets, and
it's the one kind the addressing tools cannot answer without a fact the user
shouldn't have to supply. The alternative is an agent inferring an identifier from a
display name, which **resolves to a real and wrong person.**

Order your sources explicitly: caller-supplied override first, then
session-authoritative, then inferred. An override that a "better" source can outvote
is not an override. Label every result with its source and an `inferred` boolean.
And note which case is dangerous — it is not the failure. A guess that *fails* is
harmless. A guess that *succeeds* returns a genuine record belonging to somebody
else, so it needs a warning in words, in the payload.

## The four things only an audit finds

None of these produce an error. Nothing warns you. I found all four by deliberately
sitting down and looking.

- **Tool annotations.** `readOnlyHint` decides whether a client prompts the user
  before every single call — get it wrong on a read-only tool and you've made your
  server exhausting to use. `openWorldHint` says the answer can change over time.
  Leave `destructiveHint` and `idempotentHint` **unset** when `readOnlyHint` is
  true; the spec only defines them for the false case.
- **Per-tool `title`.** Unset means the client displays your raw function name to a
  human.
- **Server-level metadata on the constructor**, not only per tool. `title`,
  `description`, `version`, `instructions`. Note that `version` defaults to the
  **empty string**, so an unset one renders as a blank space next to every other
  server in the client's list. `instructions` is sent once at `initialize` and is
  the only text you can write that isn't attached to a tool — put cross-tool
  guidance there (call order, what to do with diagnostics) and keep it short,
  because it is prepended to the system prompt of every session that loads you.
- **Pin published bounds against their constants with a test.** Any limit you state
  in prose rather than as a schema keyword will drift away from the code silently.

## Test it as a real client, over stdio, in a committed test

Anything that calls `server.call_tool()` in-process shares a process with the
server, and therefore *structurally cannot* observe:

- the `initialize` response, where your name, version and instructions live
- a stdout/stderr collision
- an import that only breaks under `python -m`
- a schema your SDK refuses to serialize

Those are also, in my experience, the exact four failures a new colleague hits in
their first ten minutes. So spawn the real thing over stdio in a committed test.

Make it credential-free by only calling tools with inputs that fail their own bounds
checks — those refuse before any I/O, so the file still runs in CI with no secrets.
This has a design consequence worth stating: **check argument bounds before
resolving identity**, so the protocol test can exercise a refusal without
credentials at all.

## Mutation-test the tests, including the prose assertions

A green suite tells you the tests ran, not that they can fail. Mutation testing tells
you the second thing.

It is most valuable exactly where you'd least expect to bother: assertions about
prose. One of mine checked that a server's `instructions` "mention `find_person`."
It passed. It also passed on instructions that mentioned `find_person` **and then
told the caller to guess an identifier anyway** — the precise behavior the
instructions existed to prevent.

Pin the imperative, not the topic. And document by-design surviving mutants in place,
so the next person doesn't spend an afternoon chasing an equivalent one.

## Why all of this points the same direction

Read back through and nearly every item is aimed at one failure mode: **a result
that is partial, capped, stale, guessed or refused being presented as complete and
correct.** The agent is a confident narrator. It cannot tell the difference unless
your payload tells it, in the payload, in words.

Items 2, 6 and 10 are the ones to get right first, because they are the hardest to
retrofit — each changes the shape of every response you return.

---

*Part two covers the layer above this one — orchestrating multiple agents and tools:
role boundaries, typed worker contracts, and the tension between failing fast on an
upstream refusal and degrading gracefully on a lateral one.*

---

**Julie Elkins** is a senior solutions architect and technical educator working on
AI-assisted software development. More at
[youtube.com/@julieelkins](https://youtube.com/@julieelkins).

Corrections and disagreements welcome — open an issue.
