---
name: chatgpt-turn-completion-gate
description: "Use when an MoA or browser-automation pipeline submits prompts to ChatGPT and must distinguish streamed thinking or partial answer text from a completed turn. Gate aggregation on an explicit terminal UI state, tolerate long high-effort runs, and never score or aggregate partial output."
version: 1.0.0
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

ChatGPT may expose a live thinking state and stream partial text for an extended period, especially at high reasoning effort. Visible text in the newest assistant message is therefore not proof that the turn is complete.

An MoA pipeline must place a barrier between reference generation and aggregation. The aggregator may run only after every required reference turn has reached a terminal state. Partial thinking, partial answer text, transient DOM stability, and network quiet are non-terminal.

## When to Use

Use this skill when:

- ChatGPT is driven through CDP, Playwright, Selenium, or another browser controller
- high-effort reasoning can run for minutes
- the interface displays thinking or streams answer text incrementally
- multiple reference agents feed an aggregator
- a premature reference snapshot would change the aggregate answer

Do not use a fixed sleep as the primary completion mechanism.

## Core Invariant

For each submitted ChatGPT turn, maintain exactly one state:

```text
queued -> submitted -> generating -> completed
                              \-> failed
                              \-> timed_out
```

Only `completed` output is eligible for aggregation. `failed` and `timed_out` are terminal but must be handled explicitly by the orchestration policy. `submitted` and `generating` output must never be passed to the aggregator.

## Reliable Completion Detection

Use multiple UI signals rather than message text alone. A turn is complete only when all required completion conditions hold for a stability window.

Recommended conditions:

1. The generation control is no longer in its active state. For example, the Stop button disappears and the Send button or normal composer control returns.
2. The newest assistant turn is no longer marked as streaming, thinking, generating, or busy.
3. The newest assistant message has a stable identity and its content has stopped changing for a short debounce window.
4. No retry, error, disconnected, or interrupted state is visible.
5. The completed assistant message belongs to the submitted user turn, not an older message.

The control-state signal is primary. Content stability is a secondary guard against transient UI transitions.

Do not treat any of these alone as sufficient:

- non-empty assistant text
- the appearance of a thinking panel
- a temporary pause in DOM mutations
- network idle
- elapsed time since submission
- the first appearance of copy or reaction controls before generation has actually stopped

## Completion Barrier

The aggregator must wait on a barrier covering all required references:

```python
TERMINAL = {"completed", "failed", "timed_out"}

async def wait_for_reference_barrier(reference_jobs):
    await asyncio.gather(*(job.wait_terminal() for job in reference_jobs))

    completed = [job for job in reference_jobs if job.state == "completed"]
    if len(completed) < MIN_COMPLETED_REFERENCES:
        raise RuntimeError("insufficient completed references")

    return [job.final_text for job in completed]
```

The aggregator is invoked once, after the barrier resolves. It must not run speculatively on partial references and then be patched with late results.

## Long-Running High-Effort Turns

High effort requires patience, not an unbounded hang.

Use two timeout layers:

1. **Progress watchdog** — reset or extend while credible progress is observed, such as continued streaming, changing thinking status, or message growth.
2. **Absolute ceiling** — a generous configurable maximum that prevents permanent deadlock.

Example policy:

```python
POLL_INTERVAL_SECONDS = 0.5
STABILITY_WINDOW_SECONDS = 2.0
NO_PROGRESS_TIMEOUT_SECONDS = 180
ABSOLUTE_TURN_TIMEOUT_SECONDS = 1800
```

These values are deployment defaults, not protocol guarantees. High-effort configurations may require a larger absolute ceiling. Avoid a short hard timeout based on normal-effort latency.

## CDP-Oriented Loop

```python
async def await_chatgpt_turn(page, submitted_turn_id):
    started = monotonic()
    last_progress = started
    last_signature = None
    stable_since = None

    while True:
        snapshot = await inspect_chat_state(page, submitted_turn_id)
        now = monotonic()

        if snapshot.error_visible:
            raise ChatTurnFailed(snapshot.error_text)

        signature = (
            snapshot.assistant_message_id,
            snapshot.text,
            snapshot.thinking_state,
            snapshot.generation_control_state,
        )

        if signature != last_signature:
            last_signature = signature
            last_progress = now
            stable_since = None

        completion_controls = (
            snapshot.generation_control_state == "idle"
            and not snapshot.is_streaming
            and not snapshot.is_thinking
        )

        if completion_controls:
            stable_since = stable_since or now
            if now - stable_since >= STABILITY_WINDOW_SECONDS:
                return snapshot.text
        else:
            stable_since = None

        if now - last_progress > NO_PROGRESS_TIMEOUT_SECONDS:
            raise ChatTurnTimedOut("no observable progress")

        if now - started > ABSOLUTE_TURN_TIMEOUT_SECONDS:
            raise ChatTurnTimedOut("absolute turn deadline exceeded")

        await asyncio.sleep(POLL_INTERVAL_SECONDS)
```

`inspect_chat_state` should use stable accessibility attributes, roles, test IDs, or structural selectors where available. Avoid selectors based only on localized button text.

## Final-Text Capture

Capture the reference text only after completion is established.

- Read the final assistant message from the DOM after the stability window.
- Normalize only transport artifacts, not substantive formatting.
- Store the message ID, completion timestamp, and final text atomically.
- Do not concatenate intermediate snapshots.
- Do not include exposed thinking text unless the application explicitly defines it as part of the final answer.

If the UI separates thinking from the final response, the MoA reference should normally contain only the final response.

## Failure Policy

When a turn fails or times out:

- Never substitute the latest partial text as a completed reference.
- Record the terminal reason and relevant diagnostics.
- Retry only according to a bounded retry policy.
- Use a fresh turn or verified recoverable session state for a retry.
- Aggregate with fewer references only when the configured minimum-completion policy permits it.

A browser disconnect, navigation, CAPTCHA, authentication prompt, or model error is a failure state, not completion.

## Session-Level Rule

When one ChatGPT conversation is assigned per Hermes session:

- serialize prompts within that conversation unless concurrent turns are explicitly supported
- bind each submitted prompt to its expected assistant turn
- do not let a later prompt overwrite or confuse completion detection for an earlier turn
- retain the conversation only after the current turn reaches a terminal state

This prevents cross-turn contamination and ensures that the aggregator receives the response associated with the correct Hermes session.

## Common Pitfalls

1. **Aggregating when text first appears**  
   Text may be a partial stream. Wait for terminal controls and stability.

2. **Using a short fixed sleep**  
   High-effort turns have variable latency. Poll explicit state instead.

3. **Treating DOM quiet as completion**  
   Thinking can pause without ending. UI control state must also be idle.

4. **Passing exposed thinking to the aggregator**  
   Unless explicitly required, capture only the final assistant response.

5. **Using partial text after timeout**  
   Mark the job timed out or retry it; partial output is ineligible.

6. **Starting aggregation before all references terminate**  
   Use a session-level barrier and a minimum-completed-reference rule.

7. **Confusing an old assistant message with the new turn**  
   Bind completion detection to the submitted user turn and newest assistant message ID.

## Verification Checklist

- [ ] Non-empty text is not used as the completion signal.
- [ ] The active generation control has returned to idle.
- [ ] Thinking and streaming indicators are absent.
- [ ] Final text is stable for a debounce window.
- [ ] The assistant message is associated with the submitted turn.
- [ ] High-effort runs have a generous, configurable absolute timeout.
- [ ] Progress extends the watchdog without allowing an infinite hang.
- [ ] Partial text is never aggregated after failure or timeout.
- [ ] Aggregation begins only after the reference barrier resolves.
- [ ] The aggregator receives only completed final responses.
