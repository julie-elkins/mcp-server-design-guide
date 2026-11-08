# MCP evaluation checklist

Use this checklist to evaluate a Model Context Protocol (MCP) server as an
agent-facing tool surface. The purpose is not only to confirm that the tool
returns a response; it is to determine whether an agent can use the tool
correctly, safely, and repeatedly to reach the declared task outcome.

## Tool contract

- [ ] Is the server needed because it provides a meaningful contract, or is it
      only a thin wrapper around access that code could call directly?
- [ ] Does every tool have a typed input and output schema?
- [ ] Do descriptions explain what a wrong value does, not only the parameter type?
- [ ] Are required arguments actually required?
- [ ] Are bounds, defaults, enums, and nullable values explicit?
- [ ] Does the result expose freshness, provenance, completeness, and refusal state?
- [ ] When meaning affects the result, does the payload expose the semantic context
      used—definitions, relationships, scope, and source limits—or state that this
      check is not applicable?
- [ ] Can an agent distinguish an empty result from a successful result with no data?
- [ ] Does every refusal state the cheapest safe correction?

## Agent choice and interaction

- [ ] Are similarly named tools tested for wrong-tool selection?
- [ ] Are tool descriptions checked at the host's delivered length, not only in source?
- [ ] Are call order, prerequisites, and cross-tool dependencies explicit?
- [ ] Are read-only, destructive, idempotent, and open-world annotations correct?
- [ ] Does the server expose identity and inference sources when a caller asks
      about itself or another person?
- [ ] Are ambiguity, missing context, conflicting evidence, and unauthorized
      requests represented as structured outcomes?

## State, policy, and side effects

- [ ] Is success determined from the resulting state, not merely from a returned
      message or HTTP status?
- [ ] Are policy rules evaluated before a side effect?
- [ ] Is confirmation required before high-risk or irreversible actions?
- [ ] Are retries safe, bounded, and explicit about idempotency?
- [ ] Does the server preserve operation identifiers and terminal state?
- [ ] Are partial completion, timeout, stale data, and unavailable dependencies visible?

## Test environment and negative controls

- [ ] Does the test use the real MCP protocol boundary, including initialization
      and serialized schemas?
- [ ] Are tests credential-free and based on synthetic fixtures?
- [ ] Do child-process arguments avoid literal credential values?
- [ ] Are credential files or environment handoffs permission-restricted and
      removed after the child exits?
- [ ] Is there a known-good case?
- [ ] Is there an empty or incomplete-result case that must not pass?
- [ ] Is there a wrong-tool or wrong-argument case?
- [ ] Is there an authorization or policy-violation case?
- [ ] Is there a tool/environment failure and recovery case?
- [ ] Is there a case where the final text looks plausible but the state is wrong?

## Repeated reliability

- [ ] Are the same scenarios run repeatedly to expose inconsistent behavior?
- [ ] Is a pass^k-style measure useful? Pass^k means the probability that the
      agent succeeds across k independent trials.
- [ ] Are tool choice, argument correctness, state correctness, and final answer
      reported separately?
- [ ] Are latency, cost, retries, and refusal rates visible?
- [ ] Are results separated by tool, policy, environment, and risk slice?
- [ ] Is the evaluator itself checked against human-reviewed or independently
      verified cases?
- [ ] Are position, verbosity, and self-preference biases tested when an
      LLM-as-a-judge grades the result?

## Release boundary

- [ ] Does the report state the exact task and decision it supports?
- [ ] Does it distinguish protocol success from semantic task success?
- [ ] Are unavailable and not-evaluated checks prevented from appearing as passes?
- [ ] Is a human owner responsible for release, hold, rollback, or escalation?
- [ ] Does the report state what it does not prove about production or users?

## Research references

- [Agentic Benchmark Checklist](https://arxiv.org/abs/2507.02825)
- [Agent Evaluation Reliability](https://arxiv.org/abs/2610.00651)
- [Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena](https://arxiv.org/abs/2306.05685)
- [G-Eval](https://arxiv.org/abs/2303.16634)
- [Quantifying Variance in Evaluation Benchmarks](https://arxiv.org/abs/2406.10229)
- [AgentBench](https://arxiv.org/abs/2308.03688)
- [τ-bench](https://arxiv.org/abs/2406.12045)
