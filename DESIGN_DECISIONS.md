# Design decisions

This guide treats an MCP (Model Context Protocol) server as a contract for an agent,
not merely as a set of API wrappers. The records below explain the choices that shape
the standard and point back to the sections where the evidence and limits are written.

The format follows the
[decision-reasoning template](https://github.com/julie-elkins/agent-systems-design-guide/blob/main/DESIGN_DECISION_TEMPLATE.md).

## 1. Decide whether the contract belongs in a server

- **Problem:** A server with many thin wrappers can add tool-choice risk without adding
  a meaningful contract.
- **Decision:** Build a server when its value is the refusal, provenance, bounds, or
  diagnostic contract; prefer a smaller access surface when it only exposes API calls.
- **Instead of:** Defending every operation as a separate tool because the underlying
  API has separate endpoints.
- **Why:** The agent chooses from the schema and can confidently narrate an incomplete
  result. A real contract needs a place to make incompleteness visible.
- **Implementation:** The opening decision rule in `README.md` separates access value
  from contract value and calls out the mixed-server case.
- **Verification:** Review the proposed tool surface for wrong-tool choice, missing
  refusal states, and whether a script would provide the same value.
- **Costs if wrong:** The server can increase selection errors, token occupancy, and
  maintenance without improving the caller's decision.
- **What this does not prove:** A small surface is not automatically a safe surface;
  payload and authorization design still matter.

## 2. Treat the schema as the agent's user interface

- **Problem:** The model chooses arguments from the schema and may never see the
  implementation details a human developer uses to fill the gaps.
- **Decision:** Put bounds, refusal conditions, destructive behaviour, and load-bearing
  guidance in the schema and payload contract.
- **Instead of:** Relying on implementation comments or a README the agent will not
  receive at call time.
- **Why:** The schema is the decision surface the model actually reads.
- **Implementation:** The standard requires typed inputs, explicit annotations,
  bounded outputs, diagnostic fields, and server-level instructions.
- **Verification:** Test serialized protocol responses and assert that the delivered
  description, not only the source constant, contains the prohibitions.
- **Costs if wrong:** A tool can be technically correct while the agent calls it
  incorrectly or reports a refused/partial result as success.
- **What this does not prove:** A complete schema cannot prevent every model mistake.

## 3. Make refusal and incompleteness first-class results

- **Problem:** Empty, capped, stale, guessed, or denied results can look identical to a
  successful empty answer.
- **Decision:** Return structured diagnostics and state what was not checked, bounded,
  or authorized.
- **Instead of:** Returning `[]`, `null`, or a generic exception and asking the agent
  to infer the meaning.
- **Why:** The result must carry enough context to stop confident narration from
  outrunning the evidence.
- **Implementation:** The practices for refusal channels, provenance, caps, and
  partial results appear throughout the fifteen-point standard.
- **Verification:** Negative tests assert the refusal code, guidance, and boundary
  metadata rather than only an exception type.
- **Costs if wrong:** A downstream user may act on a clean-looking answer that was
  never established.
- **What this does not prove:** A diagnostic is evidence of the server's state, not
  proof that the caller will interpret it correctly.

## 4. Verify over the real protocol boundary

- **Problem:** An in-process test cannot observe initialization, serialization,
  stdout/stderr collisions, or module-entrypoint failures.
- **Decision:** Keep at least one credential-free stdio (standard input/output)
  protocol test that starts the server as a real client would.
- **Instead of:** Testing only `server.call_tool()` inside the same process.
- **Why:** The delivery path can fail even when the handler function works.
- **Implementation:** The guide recommends inputs that fail validation before external
  I/O so the protocol test is safe to run in CI.
- **Verification:** The committed test checks the initialized metadata, refusal shape,
  and serialized tool surface.
- **Costs if wrong:** A release can pass unit tests and still fail to start or expose
  the contract to the actual client.
- **What this does not prove:** A credential-free refusal test does not prove live
  authorization or provider integration.

## 5. Mutation-test prose assertions

- **Problem:** A test can mention the right topic while allowing the dangerous
  instruction to remain absent or inverted.
- **Decision:** Test the imperative and the negative path, and record intentional
  surviving mutants.
- **Instead of:** Searching for a keyword and treating its presence as sufficient.
- **Why:** The failure this guide targets is often semantic: the sentence exists but
  no longer prevents the wrong behaviour.
- **Implementation:** The mutation-testing section explains how to invert prose and
  refusal assertions.
- **Verification:** Delete or invert the load-bearing assertion and confirm the suite
  fails for the intended reason.
- **Costs if wrong:** Documentation tests become decorative and let a dangerous
  contract drift while staying green.
- **What this does not prove:** Mutation coverage does not replace review of the
  underlying design.

## 6. Keep credential values out of child-process arguments

- **Problem:** A child process can complete successfully while exposing an API key
  in its process-visible argument list, where local inspection tools, wrappers, or
  diagnostics can capture it.
- **Decision:** Pass provider credentials through an explicit environment
  allow-list or a user-readable-only temporary environment file. Reject literal
  secret-valued arguments at configuration load, remove secret-valued `-e
  NAME=value` arguments before launching every intermediate child, and delete
  temporary files on every exit path. A provider-backed profile must fail
  closed if it does not select the approved transport hook.
- **Instead of:** Passing `NAME=value` through Docker-compatible command arguments
  because the container runtime accepts it.
- **Why:** Container isolation does not erase the host process boundary. A value
  that is valid for the container can still be visible before the container starts.
  The configuration boundary and the process boundary therefore need separate
  controls.
- **Implementation:** The adapter's Harbor environment handles provider values
  before Docker argument construction, and its Finch shim handles both
  Anthropic and OpenAI provider keys across direct and Compose execution paths.
  The public adapter rejects literal provider values and retains only metadata
  in receipts.
- **Verification:** Synthetic tests assert that provider values are absent from
  generated loader commands and captured Finch arguments, that the protected
  temporary file is removed after exit, that multiline values are refused, and
  that a provider-backed configuration without the approved environment hook is
  refused.
- **Costs if wrong:** A wrapper-specific path may remain unprotected, or an
  environment-file implementation may leave a readable credential artifact.
  The direct-path tests and cleanup assertion are required because a passing
  provider run alone would not detect either leak.
- **What this does not prove:** This does not protect credentials already exposed
  by the user's shell history, inherited environment inspection, provider logs, or
  third-party tooling outside the controlled child-process boundary.
