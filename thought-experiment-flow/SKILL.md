---
name: thought-experiment-flow
description: Payload-driven thought experiment for tracing whether an API/business flow will succeed before running it. Use when the user provides or wants to provide payload values, repository names, branches, and endpoints, and asks to reason through an end-to-end flow, rerun an API safely, predict side effects, find likely bugs, or determine whether a request will succeed honestly without writing code.
---

# Thought Experiment Flow

## Overview

Use this skill to simulate an end-to-end API or business flow from a concrete payload before executing it. The goal is an honest verdict: what should happen, where it can fail, what state may be changed, and what evidence would confirm success or failure.

Do not write code unless the user separately asks for implementation. Prefer read-only inspection, precise flow tracing, and concrete verification steps.

## Required Inputs

Collect these before tracing:

- Payload values or a representative payload.
- Repository names or local paths involved in the flow.
- Branch names to inspect for each repository.
- Endpoint(s), method(s), and direction of the test, for example `POST /v1/...` from a React Frontend client to a NodeJS backend that flows to an external AWS S3 Bucket.
- Any known incident anchors: timestamps, request IDs, ray IDs, order IDs, emails, membership IDs, external references, or screenshots.

If no repositories are specified, ask for the repository names or paths and wait.

If no branches are specified, ask for the branches to inspect and wait.

If endpoints are missing but the repo and payload make the endpoint discoverable, inspect the code to infer likely endpoints and state the inference. If multiple endpoints could apply, ask the user to choose.

## Workflow

### 1. Normalize the Scenario

Restate the scenario in concrete terms:

- Payload identity fields, such as email, username, memberID, firstName, lastName, etc.
- The exact endpoint chain being reasoned about.
- The repositories and branches being inspected.
- Whether the user wants read-only reasoning, a dry-run-style trace, or actual endpoint execution.

Mask PII in user-facing output unless the user explicitly needs raw values for support escalation.

### 2. Inspect Real Code Paths

For each repository and branch:

- Confirm the branch before drawing conclusions.
- Locate route definitions, controllers, jobs, services, validators, models, external clients, queue publishers, and downstream consumers.
- Use `rg` first for endpoint paths, action names, payload keys, error codes, and side-effecting calls.
- Read the code around each matched function until the full path from request to response is understood.
- Prefer the code's behavior over comments, assumptions, or memory.

When multiple repositories participate, build a handoff map:

```text
Caller endpoint -> service A route -> service A state changes -> queue/event/API call -> service B consumer -> service B state changes -> final response/observable result
```

### 3. Run the Thought Experiment

Walk the payload through the flow in execution order. For each step, record:

- Preconditions checked by validation or lookup.
- Exact branch condition the payload takes.
- State changes that occur before the next possible failure.
- Idempotency or duplicate detection behavior.
- External side effects: Cognito, S3, SQS, CloudWatch, Step Functions, email, Notion, or third-party APIs.
- Response behavior: success payload, 4xx/5xx error mapping, thrown exceptions, retry semantics, and partial-success risks.

Pay special attention to ordering. A flow can be unsafe when it writes to system A, then fails before writing to system B, even if the final API response is an error.

### 4. Find Failure Modes

Look for bugs or operational traps that could occur with this payload:

- Mismatched identifiers, such as email vs userID, membership ID against username.
- Missing or invalid required fields.
- Case-sensitivity, enum mapping, date format, timezone, or null handling.
- Duplicate checks that are too narrow or too broad.
- Partial writes before a later failure.
- Queue publish failures after DB writes.
- Downstream consumers that reject fields the upstream accepted.
- Retry behavior that can create duplicates, collide on unique indexes, or stop at an "already exists" guard.
- Observability gaps where the caller gets a transport error but the backend may have succeeded.
- Known-noise errors that should not be mistaken for the cause.

Use logs or read-only database checks when available and relevant. If live data is queried, keep it read-only and state the time window.

### 5. Produce the Verdict

Give a clear, honest result. Do not soften uncertainty.

Use this structure:

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
<what to do next: run it, correct payload first, use update endpoint instead, trigger downstream retry, or gather missing evidence>

### Verification
- <exact logs, DB rows, API responses, or UI state to confirm after execution>
```

Verdict guidance:

- `SHOULD_SUCCEED`: All inspected preconditions pass, duplicate checks are clear, and downstream side effects are expected to run.
- `WILL_FAIL`: A deterministic validation, lookup, duplicate guard, or downstream rejection will stop the flow.
- `PARTIAL_SUCCESS_RISK`: A later failure can happen after earlier systems are already changed.
- `INCONCLUSIVE`: Required repos, branches, endpoint code, payload fields, or runtime data are unavailable.

## Important Constraints

- Do not invent missing repositories or branches. Ask for them.
- Do not assume the latest branch or deployed version when the user asks for branch-specific reasoning.
- Do not execute mutating endpoints unless the user explicitly asks.
- Do not declare success only because one system is healthy; trace every side effect needed by the business outcome.
- Be explicit about inferred facts versus confirmed facts.
