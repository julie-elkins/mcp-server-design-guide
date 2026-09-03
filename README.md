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
sitting down and auditing.

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
