---
name: strict-response-contracts
description: "Use when a user imposes an exact output contract such as an exact string, one-word answer, fixed sentence count, strict format, or parser-consumed response. Prioritize literal compliance, suppress unsolicited framing, and verify the final output mechanically before sending."
version: 1.0.0
author: Hermes Skill Share
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [instruction-following, exact-output, formatting, evaluation]
    related_skills: [chatgpt-turn-completion-gate]
---

# Strict Response Contracts

## Overview

Some prompts test whether the agent can obey a narrow output contract. Examples include “Say exactly: smoke-test-ok,” “Answer in one word,” “Use two sentences,” and “Return only valid JSON.”

These tasks have two independent success conditions:

1. The payload is semantically correct.
2. The surface form exactly matches the request.

When the contract is explicit, extra helpful prose is an error.

## When to Use

Use this skill for:

- exact-string or sentinel requests
- one-word, one-line, or fixed-sentence limits
- “return only” and “no explanation” constraints
- parser-consumed JSON, XML, CSV, or code output
- smoke tests, wrapper tests, and benchmark probes
- any request where extra text breaks downstream processing

Do not use it to make ordinary answers artificially terse when no strict contract exists.

## Workflow

1. Extract the required payload.
2. Record all count, case, punctuation, wrapper, and exclusivity constraints.
3. Draft directly in the required format.
4. Remove acknowledgments, headings, explanations, citations, code fences, and follow-up offers unless explicitly required.
5. Verify the final response mechanically before sending.

## Mechanical Verification

Check:

- **Payload:** Is the requested content correct?
- **Exclusivity:** Did any extra text leak in?
- **Count:** Are word, sentence, line, and item counts exact?
- **Literal form:** Are case, spacing, punctuation, hyphens, and underscores preserved?
- **Parser validity:** Would a strict parser accept the output?
- **Boundary:** Are there accidental quotes, backticks, or code fences?

For an exact literal, compare the draft character-for-character with the requested payload.

## Examples

Prompt:

```text
Say exactly: smoke-test-ok
```

Correct:

```text
smoke-test-ok
```

Prompt:

```text
What is 2+2? Answer in one word.
```

Correct:

```text
Four
```

The numeral `4` is mathematically correct but violates the one-word contract.

## Common Pitfalls

1. Adding “Sure,” “Here you go,” or another preamble.
2. Correcting capitalization or punctuation inside an exact literal.
3. Wrapping a required token in Markdown.
4. Returning valid content in the wrong grammar.
5. Explaining a test instead of passing it.

## Verification Checklist

- [ ] The response is semantically correct.
- [ ] Every explicit formatting constraint is satisfied.
- [ ] “Exactly” and “only” outputs contain no extra text.
- [ ] Counts are exact.
- [ ] Literal characters are preserved.
- [ ] Structured output parses successfully.
