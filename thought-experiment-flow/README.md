# thought-experiment-flow

A Claude/Codex agent skill that simulates an end-to-end API or business flow from a concrete payload *before* it is executed — producing an honest verdict on whether the request will succeed, where it can fail, what state it will change, and what evidence would confirm the outcome.

Use it when you want to reason through a flow safely instead of "just running it and seeing what happens."

## When to use

- You have (or can construct) a representative payload and want to know what would happen if you sent it.
- You need to rerun a previously-failed API call and want to predict side effects before doing so.
- You're investigating an incident and want to trace the request path across repos without writing code.
- You want a second opinion on whether a flow has partial-success risk (writes to system A, then fails before system B).
- You need to map an endpoint chain across multiple services and branches.

Do **not** use it to generate or modify code — the skill is read-only by design.

## Required inputs

Before tracing, the skill will ask for whatever is missing:

- **Payload** — the values (or a representative payload) that will be sent.
- **Repositories** — names or local paths of every service involved.
- **Branches** — the exact branch to inspect per repo. The skill will not assume `main` or "latest deployed."
- **Endpoint chain** — method, path, and direction (e.g. `POST /v1/...` from React frontend → Node backend → AWS S3).
- **Incident anchors** *(optional)* — timestamps, request IDs, ray IDs, order IDs, emails, membership IDs, screenshots.

If repos or branches are missing, the skill stops and asks. If endpoints are discoverable from the code, it infers them and states the inference.

## What it does

1. **Normalize the scenario** — restates payload, endpoint chain, repos/branches, and intent (read-only trace vs. dry-run vs. real execution). Masks PII in output by default.
2. **Inspect real code paths** — uses `rg` and targeted reads across the specified branches to find routes, controllers, jobs, services, validators, models, external clients, and downstream consumers. Builds a handoff map across services.
3. **Run the thought experiment** — walks the payload through the flow in execution order, recording preconditions, branch conditions taken, state changes, idempotency behavior, external side effects, and response mapping.
4. **Find failure modes** — looks for mismatched identifiers, validation gaps, case/enum/timezone bugs, narrow or broad duplicate checks, partial writes, queue-after-DB ordering risks, downstream field rejections, retry collisions, observability gaps, and known-noise errors.
5. **Produce a verdict** — returns a structured markdown report.

## Output shape

```markdown
## Thought Experiment Verdict

**Scenario:** <one-sentence payload + endpoint summary>
**Repos / branches inspected:** <repo@branch list>
**Endpoint chain:** <caller -> service -> downstream>

### Verdict: <SHOULD_SUCCEED / WILL_FAIL / PARTIAL_SUCCESS_RISK / INCONCLUSIVE>

### Execution Trace
1. <Step> — <what happens with this payload>
2. <Step> — <what happens next>

### Failure Points
- `<condition>` — <why it may fail, expected response or side effect>

### State Changes If Run
- <system/table/service> — <create/update/delete/queue/API side effect>

### Best Resolution
<run it, correct payload first, use update endpoint instead, trigger downstream retry, or gather missing evidence>

### Verification
- <exact logs, DB rows, API responses, or UI state to confirm after execution>
```

### Verdict meanings

- **`SHOULD_SUCCEED`** — All inspected preconditions pass, duplicate guards are clear, and downstream side effects are expected to run.
- **`WILL_FAIL`** — A deterministic validation, lookup, duplicate guard, or downstream rejection will stop the flow.
- **`PARTIAL_SUCCESS_RISK`** — A later failure can occur after earlier systems are already mutated.
- **`INCONCLUSIVE`** — Required repos, branches, endpoint code, payload fields, or runtime data are unavailable.

## Constraints

- Never invents missing repos or branches — asks instead.
- Never assumes the latest branch or deployed version when branch-specific reasoning was requested.
- Never executes mutating endpoints unless the user explicitly asks.
- Never declares success based on one healthy system; traces every side effect the business outcome depends on.
- Always distinguishes inferred facts from confirmed facts.

## Installation

Drop this directory into your agent's skills location, e.g.:

```bash
cp -r thought-experiment-flow ~/.claude/skills/
```

The agent will pick up the skill from `SKILL.md` and invoke it automatically when a user request matches its `description`, or you can reference it explicitly by name.

## Example trigger phrases

- "Walk this payload through the create-membership flow and tell me if it'll succeed."
- "If I rerun this `POST /v1/orders` with the same body, what changes?"
- "Trace what happens between `service-a@main` and `service-b@release-2025-05` for this payload."
- "Is there partial-success risk if step 3 fails here?"
