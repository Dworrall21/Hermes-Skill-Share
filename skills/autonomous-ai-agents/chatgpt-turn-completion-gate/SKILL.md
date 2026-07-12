---
name: chatgpt-turn-completion-gate
description: "Use when an MoA or browser-automation pipeline submits prompts to ChatGPT and must distinguish surfaced thinking or streamed partial text from a completed turn. Block aggregation until ChatGPT reaches an explicit terminal UI state, tolerate long high-effort runs, and never treat provisional output as a reference answer."
version: 1.0.1
author: Hermes Skill Share
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [moa, chatgpt, cdp, browser-automation, streaming, synchronization]
    related_skills: [strict-response-contracts]
---

# ChatGPT Turn Completion Gate

## Overview

ChatGPT can surface an extended thinking process and stream provisional answer text, especially at high reasoning effort. Visible assistant text is not proof that the turn is complete.

The MoA must enforce a hard barrier between ChatGPT reference generation and aggregation. The aggregator must not begin reading, weighing, scoring, summarizing, or combining ChatGPT output until the ChatGPT turn has fully completed.

Thinking text, partial answer text, temporary pauses, network quiet, and transient DOM stability are non-terminal states.

## Hard Requirement

```text
NO CHATGPT TURN COMPLETION -> NO AGGREGATOR EXECUTION
```

This is a scheduling invariant:

- Do not launch the aggregator while ChatGPT is thinking.
- Do not let the aggregator inspect a growing assistant message.
- Do not score a partial snapshot.
- Do not start a speculative aggregate and patch it later.
- Do not use surfaced thinking as the ChatGPT reference answer.
- Release the aggregator only after the ChatGPT job records `completed`.

## When to Use

Use this skill when:

- ChatGPT is driven through CDP, Playwright, Selenium, or another browser controller
- high-effort reasoning may take a long time
- the interface exposes thinking or streams answer text incrementally
- ChatGPT supplies a reference to a Mixture-of-Agents pipeline
- premature aggregation could omit or distort the final answer

Do not use a fixed sleep as the primary completion mechanism.

## State Machine

Maintain one authoritative state per ChatGPT turn:

```text
queued -> submitted -> generating -> completed
                              |-> failed
                              |-> timed_out
```

| State | Eligible for aggregation? |
|---|---|
| `queued` | No |
| `submitted` | No |
| `generating` | No |
| `completed` | Yes |
| `failed` | No |
| `timed_out` | No |

Only `completed` may expose `final_text`. Intermediate snapshots may be retained for diagnostics but must never be returned through the reference-result interface.

## Completion Detection

A turn is complete only when all required terminal signals hold for a stability window:

1. The generation control has returned to idle. For example, the Stop control is gone and Send is available.
2. The newest assistant turn is no longer marked thinking, streaming, generating, or busy.
3. Any surfaced thinking region is no longer active.
4. The final assistant message identity and content remain stable during a debounce interval.
5. No retry, interrupted, disconnected, authentication, or error state is visible.
6. The final assistant message belongs to the submitted user turn.

The active/idle generation control is the primary signal. Message stability is secondary.

Never use any of these alone as proof of completion:

- non-empty assistant text
- visible thinking text
- a final-looking paragraph
- a pause in DOM mutations
- network idle
- elapsed time
- copy or reaction controls appearing
- an older completed message matching a broad selector

## Aggregation Barrier

The aggregator must be structurally downstream of the reference barrier:

```python
async def collect_completed_references(reference_jobs):
    await asyncio.gather(*(job.wait_terminal() for job in reference_jobs))

    completed = [job for job in reference_jobs if job.state == "completed"]
    if len(completed) < MIN_COMPLETED_REFERENCES:
        raise RuntimeError("insufficient completed references")

    return [job.final_text for job in completed]

async def run_moa(reference_jobs, aggregator):
    references = await collect_completed_references(reference_jobs)
    return await aggregator.generate(references)
```

The aggregator receives immutable final-reference objects, not live DOM handles, observers, or mutable strings.

When ChatGPT is mandatory, check its named dependency explicitly:

```python
if chatgpt_job.state != "completed":
    raise RuntimeError("ChatGPT reference is not complete")
```

A numeric quorum must not bypass an unfinished mandatory ChatGPT reference.

## Long High-Effort Runs

High-effort turns require a long-lived wait loop and generous limits.

Use two timeout layers:

1. **Progress watchdog** — refreshed by credible activity such as changing surfaced thinking, text growth, status transitions, or control changes.
2. **Absolute ceiling** — a configurable upper bound that prevents permanent deadlock.

Example defaults:

```python
POLL_INTERVAL_SECONDS = 0.5
STABILITY_WINDOW_SECONDS = 2.0
NO_PROGRESS_TIMEOUT_SECONDS = 300
ABSOLUTE_TURN_TIMEOUT_SECONDS = 3600
```

These are deployment defaults, not guarantees. Configure them according to model and reasoning effort. Do not use a timeout calibrated from low-effort turns.

## CDP-Oriented Wait Loop

```python
async def await_chatgpt_turn(page, submitted_turn_id):
    started = monotonic()
    last_progress = started
    last_signature = None
    stable_since = None

    while True:
        state = await inspect_chat_state(page, submitted_turn_id)
        now = monotonic()

        if state.error_visible:
            raise ChatTurnFailed(state.error_text)

        signature = (
            state.assistant_message_id,
            state.final_text,
            state.thinking_text,
            state.is_thinking,
            state.is_streaming,
            state.generation_control_state,
        )

        if signature != last_signature:
            last_signature = signature
            last_progress = now
            stable_since = None

        terminal_ui = (
            state.generation_control_state == "idle"
            and not state.is_thinking
            and not state.is_streaming
            and state.assistant_message_id is not None
            and state.belongs_to_turn(submitted_turn_id)
        )

        if terminal_ui:
            stable_since = stable_since or now
            if now - stable_since >= STABILITY_WINDOW_SECONDS:
                return state.final_text
        else:
            stable_since = None

        if now - last_progress > NO_PROGRESS_TIMEOUT_SECONDS:
            raise ChatTurnTimedOut("no observable progress")

        if now - started > ABSOLUTE_TURN_TIMEOUT_SECONDS:
            raise ChatTurnTimedOut("absolute turn deadline exceeded")

        await asyncio.sleep(POLL_INTERVAL_SECONDS)
```

Use stable accessibility attributes, roles, test IDs, and structural relationships where possible. Avoid volatile CSS classes or localized labels as the only signal.

## Surfaced Thinking Versus Final Answer

Treat surfaced thinking as activity evidence only:

- It may refresh the progress watchdog.
- It proves the turn is still generating.
- It must not be passed to the aggregator.
- It must not be concatenated with the final answer.
- Its disappearance alone does not prove completion.

Capture only the final assistant response after the terminal gate opens.

## Final-Text Commit

Once completion is established:

1. Re-read the final assistant message.
2. Verify that its identity belongs to the submitted turn.
3. Store `final_text`, message ID, completion timestamp, and terminal state atomically.
4. Freeze the reference object.
5. Release the aggregator dependency.

Do not release the aggregator first and populate `final_text` afterward.

## Failure and Timeout Policy

When the ChatGPT turn fails or times out:

- Never promote partial text to `final_text`.
- Record the terminal reason and diagnostics.
- Retry only through a bounded retry policy.
- Use a fresh turn or verified recoverable session state.
- Aggregate without ChatGPT only when degraded mode is explicitly permitted.
- If ChatGPT is mandatory, fail the MoA run rather than proceeding early.

Browser disconnection, navigation, CAPTCHA, authentication prompts, interruption, and model errors are failures, not completion.

## Session-Level Rule

When one ChatGPT conversation is assigned per Hermes session:

- serialize prompts unless concurrent turns are explicitly supported
- bind each prompt to its expected assistant message
- wait for the current turn to become terminal before submitting another prompt
- retain the Hermes session ID with the conversation and final reference
- do not let a later turn satisfy an earlier completion waiter

## Common Pitfalls

1. **Starting aggregation when text first appears**  
   The text is provisional. Wait for terminal controls and stability.

2. **Treating surfaced thinking as the answer**  
   Thinking indicates ongoing work; only the completed final response is eligible.

3. **Using a short fixed delay**  
   High-effort latency varies. Poll explicit state.

4. **Using DOM quiet as completion**  
   Thinking can pause. Require idle controls and absent activity indicators.

5. **Starting the aggregator speculatively**  
   The aggregator must run after the completion barrier, not in parallel with ChatGPT.

6. **Using partial text after timeout**  
   Mark the turn timed out or retry it. Partial output remains ineligible.

7. **Allowing a quorum to bypass ChatGPT**  
   When ChatGPT is mandatory, check its named job state explicitly.

8. **Reading an older completed message**  
   Bind the message to the submitted turn ID.

## Verification Checklist

- [ ] Surfaced thinking is classified as `generating`, never `completed`.
- [ ] Non-empty or final-looking text is not a completion signal.
- [ ] Generation controls have returned to idle.
- [ ] Thinking and streaming indicators are inactive.
- [ ] Final text is stable for the debounce window.
- [ ] The final assistant message belongs to the submitted turn.
- [ ] The aggregator is structurally downstream of the completion barrier.
- [ ] The aggregator cannot read mutable or partial ChatGPT output.
- [ ] High-effort turns have generous progress and absolute timeout policies.
- [ ] Partial output is never promoted after failure or timeout.
- [ ] A mandatory ChatGPT reference cannot be bypassed by a numeric quorum.
- [ ] Only the completed final response is supplied to aggregation.
