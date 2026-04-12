---
title: "Conditional Logic: Branching Agenda Items by AI or JSON Rules"
date: 2026-04-11T19:20:00+07:00
lastmod: 2026-04-11T19:20:00+07:00
draft: false
tags: ["AI Research", "Backend"]
categories: ["Building a Conversational AI Platform"]
series: ["Building a Conversational AI Platform"]
summary: "Not every user should go through every agenda item. Conditional logic skips irrelevant items based on prior outputs — two modes: AI-evaluated natural language for subjective calls and JSON operators for deterministic ones."
description: "Conditional logic routes users through different agenda paths based on collected data. AI-evaluated natural language for subjective calls, JSON operators for deterministic ones."
ShowToc: true
weight: 10
seriesTotal: 12
---

{{< series-nav >}}

*Day 10 of 12. So far every agenda item in the examples has run in sequence for every user. In real workflows, you want **different users to go through different paths** depending on what they've already said.*

> **TL;DR**
> - **Every item has an optional condition**. If it evaluates to `false`, the item is skipped entirely and the next item's condition is checked.
> - **Two modes**: AI-evaluated (natural language, good for subjective checks) and JSON operators (deterministic, good for numeric and exact-match checks).
> - **JSON operators** I ended up using: `$eq`, `$ne`, `$gt`, `$lt`, `$ct` (contains), `$fz` (fuzzy match), `$and`, `$or` with nesting.
> - **Real example in this post**: a job screening agenda that branches into a "senior" path (deep Debate) or a "junior" path (portfolio review + polite decline).

---

## The Scenario: One Agenda, Two Candidate Paths

A hiring manager set up a job screening agenda for a mid-level backend role. Every applicant goes through the same trigger ("I want to apply"), but **what happens next depends on their background**:

- **Senior candidates (5+ years, prior senior role)** → jump into a deep technical Debate item, then schedule a call with the hiring manager.
- **Junior candidates (under 5 years)** → fill out a portfolio form and get a polite "we'll review in 5 business days" message. The hiring manager reviews async.

Without conditional logic, the agenda has one shape and everyone goes through it. Either seniors waste time on junior-appropriate questions, or juniors get grilled on system design they haven't done yet. Neither is good.

With conditional logic, one agenda serves both paths cleanly.

![Conditional branching: senior goes to Debate, junior goes to portfolio upload](/images/blog/conditional-branching.svg)

Here's the config for this branching agenda:

```yaml
name: backend_engineer_screening
trigger_phrase: "I want to apply for the backend role"

items:
  - type: convey
    name: welcome
    mode: static
    message: "Thanks for applying. I'll ask a few quick questions first."

  - type: gather
    name: basics
    outputs:
      - name: full_name
        kind: asked
        required: true
      - name: email
        kind: asked
        required: true
        validator: '^[^@]+@[^@]+\.[^@]+$'
      - name: years_experience
        kind: asked
        required: true
      - name: current_role_level
        kind: asked
        required: true
        description: "Junior, Mid, or Senior"

  # Senior path — only runs if years >= 5 OR role == senior
  - type: debate
    name: senior_technical_probe
    condition:
      mode: json
      expression:
        $or:
          - { years_experience: { $gte: 5 } }
          - { current_role_level: { $eq: "Senior" } }
    starting_question: "Walk me through a system you've designed that handled significant scale."
    num_follow_ups: 4

  - type: convey
    name: senior_next_steps
    mode: dynamic
    condition:
      mode: json
      expression:
        $or:
          - { years_experience: { $gte: 5 } }
          - { current_role_level: { $eq: "Senior" } }
    message: |
      Write a warm message acknowledging the candidate's background and letting 
      them know the next step is a 45-minute call with the hiring manager. 
      Reference one specific thing they said in the technical probe.

  # Junior path — only runs if years < 5 AND role != senior
  - type: gather
    name: junior_portfolio
    condition:
      mode: json
      expression:
        $and:
          - { years_experience: { $lt: 5 } }
          - { current_role_level: { $ne: "Senior" } }
    outputs:
      - name: github_url
        kind: asked
        validator: '^https?://github\.com/.+'
      - name: portfolio_url
        kind: asked
        required: false
      - name: years_in_school
        kind: asked

  - type: convey
    name: junior_thanks
    mode: static
    condition:
      mode: json
      expression:
        $and:
          - { years_experience: { $lt: 5 } }
          - { current_role_level: { $ne: "Senior" } }
    message: |
      Thanks for the details! The hiring manager reviews portfolios async.
      You'll hear back within 5 business days.
```

Every item after `basics` has a condition. When the engine hits each item, it evaluates the condition against the current outputs. If `true`, the item runs. If `false`, the engine skips to the next item.

## How Conditions Are Evaluated

The engine's execution loop from [Day 1](/posts/agenda-engine-deterministic-ai-conversations/) looks like this with conditions added:

```python
async def run_agenda(agenda: Agenda, state: ConversationState):
    for item in agenda.items:
        if item.condition and not await evaluate_condition(item.condition, state.outputs):
            # Skip this item
            continue

        # Run the item (Convey, Gather, Debate, or Q&A)
        await item.on_start()
        while not item.should_end:
            await wait_for_user_response()
        await item.on_end()
```

The `evaluate_condition` function handles both JSON and AI-evaluated modes:

```python
async def evaluate_condition(condition: dict, outputs: dict) -> bool:
    if condition['mode'] == 'json':
        return evaluate_json(condition['expression'], outputs)
    elif condition['mode'] == 'ai':
        return await evaluate_ai(condition['prompt'], outputs)
    else:
        raise ValueError(f"Unknown condition mode: {condition['mode']}")
```

Each mode has different strengths.

## Mode 1: JSON Operators (for deterministic checks)

When the decision is based on concrete values — numbers, string equality, list membership — JSON operators are the right tool. They're deterministic (no LLM call, no cost, no hallucination risk), fast (micro-seconds), and auditable.

```python
OPERATORS = {
    '$eq': lambda a, b: a == b,
    '$ne': lambda a, b: a != b,
    '$gt': lambda a, b: a > b,
    '$gte': lambda a, b: a >= b,
    '$lt': lambda a, b: a < b,
    '$lte': lambda a, b: a <= b,
    '$in': lambda a, b: a in b,
    '$nin': lambda a, b: a not in b,
    '$ct': lambda a, b: b in a,               # contains (string)
    '$fz': lambda a, b: fuzzy_match(a, b),    # fuzzy match
}

LOGICAL_OPS = {
    '$and': lambda results: all(results),
    '$or': lambda results: any(results),
    '$not': lambda results: not results[0],
}


def evaluate_json(expression: dict, outputs: dict) -> bool:
    # Logical operators take a list of sub-expressions
    for logical_op, sub_exprs in expression.items():
        if logical_op in LOGICAL_OPS:
            results = [evaluate_json(sub, outputs) for sub in sub_exprs]
            return LOGICAL_OPS[logical_op](results)

    # Otherwise it's a field-level check: { field: { $op: value } }
    for field, ops in expression.items():
        actual = outputs.get(field)
        for op_name, expected in ops.items():
            if op_name not in OPERATORS:
                raise ValueError(f"Unknown operator: {op_name}")
            if not OPERATORS[op_name](actual, expected):
                return False
    return True
```

Real examples of JSON conditions I've used:

**Filter by email domain** (enterprise signup gate):
```yaml
condition:
  mode: json
  expression:
    email: { $ct: "@" }  # has @
```

**Age threshold for a financial product**:
```yaml
condition:
  mode: json
  expression:
    age: { $gte: 18 }
```

**Nested condition for a multi-region filter**:
```yaml
condition:
  mode: json
  expression:
    $and:
      - { country: { $in: ["US", "CA", "UK"] } }
      - { email_verified: { $eq: true } }
      - { $or:
          - { plan: { $eq: "enterprise" } }
          - { company_size: { $gt: 500 } }
        }
```

This reads as: "Country is US, CA, or UK, AND email is verified, AND (plan is enterprise OR company size is over 500)". Nested Boolean logic in config, no code changes.

## Mode 2: AI-Evaluated (for subjective checks)

Some decisions aren't reducible to operators. "Did the founder sound confident about their go-to-market?" is a subjective judgment that needs an LLM.

```yaml
condition:
  mode: ai
  prompt: |
    Based on the gathered outputs, should we proceed with the deep dive?
    
    Skip if:
    - The founder's problem statement was vague or generic
    - They couldn't name any specific competitors
    - Their funding stage is clearly mismatched (Series B founder 
      applying for pre-seed)

    Run a detailed evaluation against those criteria.
    
    Return a boolean reasoning first, then your answer.
```

The evaluator calls the LLM with a structured response schema:

```python
async def evaluate_ai(prompt: str, outputs: dict) -> bool:
    context = '\n'.join(f'{k}: {v}' for k, v in outputs.items())

    schema = {
        'type': 'object',
        'properties': {
            'reasoning': {'type': 'string'},
            'should_run': {'type': 'boolean'},
        },
        'required': ['reasoning', 'should_run'],
    }

    response = await llm.parse(
        system_prompt=prompt,
        user_message=f'Outputs collected so far:\n{context}',
        response_schema=schema,
    )

    # Log the reasoning for audit trails
    logger.info('ai_condition_eval', extra={
        'prompt': prompt,
        'outputs': outputs,
        'reasoning': response['reasoning'],
        'result': response['should_run'],
    })

    return response['should_run']
```

The reasoning goes into structured logs so I can audit any decision the AI made. When a user complains "why didn't I get scheduled for a call?", I can look up the condition evaluation for their specific conversation and see what the LLM saw.

## When to Use Which

Start with JSON. Add AI only when you can't express the check as operators.

| Scenario | Mode | Why |
|---|---|---|
| `years_experience >= 5` | JSON | Pure numeric comparison |
| `email_domain in [gmail, yahoo]` | JSON | Exact match |
| `"the founder sounded confident"` | AI | Subjective |
| `country == "US" AND age >= 18` | JSON | Compound deterministic |
| `"the problem statement was vague"` | AI | Requires LLM judgment |
| `score > 75` | JSON | Numeric threshold on a computed field |

A hybrid pattern also works: **use AI to compute a score, then JSON to threshold it**. The AI condition in the Debate summary produces a `confidence_score: 0-10` output. The next item's condition is `{ confidence_score: { $gte: 7 } }`. AI does the subjective part; JSON does the deterministic part.

## Real Conversation: Senior Path

Here's what the senior path looks like for a real-feeling applicant:

```
🤖 Screener: Thanks for applying. I'll ask a few quick questions first.
🤖 Screener: First — what's your full name?
👤 Taylor: Taylor Kim
🤖 Screener: Email?
👤 Taylor: taylor.kim@gmail.com
🤖 Screener: How many years of engineering experience do you have?
👤 Taylor: 9 years
🤖 Screener: And your current role level — junior, mid, or senior?
👤 Taylor: Senior
[Condition for debate item evaluates: years>=5 OR role==senior → TRUE → run]
🤖 Screener: Walk me through a system you've designed that handled 
            significant scale.
👤 Taylor: [long answer about a payment processing system]
🤖 Screener: Interesting — you mentioned you moved from synchronous 
            to async under load. What was the trigger? A specific 
            incident, or gradual?
👤 Taylor: There was an incident — a 3am page when the queue backed up.
🤖 Screener: What would you do differently if you were designing 
            it from scratch today?
...
[Debate completes after 4 follow-ups]
[Condition for senior_next_steps evaluates: TRUE → run]
🤖 Screener: Taylor, this is a great conversation. The payment system 
            story — especially how you balanced strong consistency 
            with the async pattern — lines up exactly with what the 
            team is tackling. The next step is a 45-minute call with 
            Sarah, our hiring manager. I'll send a Calendly link by email.
[Condition for junior path evaluates: FALSE → skipped]
[Agenda completes]
```

## Real Conversation: Junior Path

Same agenda, junior applicant:

```
🤖 Screener: Thanks for applying. I'll ask a few quick questions first.
🤖 Screener: First — what's your full name?
👤 Riley: Riley Martinez
🤖 Screener: Email?
👤 Riley: riley.m@school.edu
🤖 Screener: How many years of engineering experience do you have?
👤 Riley: 1 year — just graduated
🤖 Screener: And your current role level — junior, mid, or senior?
👤 Riley: Junior
[Condition for debate item: years<5 AND role!=senior → TRUE for skip]
[senior_next_steps: same condition → also skipped]
[Condition for junior_portfolio: TRUE → run]
🤖 Screener: Great. Can you share your GitHub URL?
👤 Riley: https://github.com/rileymz
🤖 Screener: Thanks. Any other portfolio link? (optional)
👤 Riley: https://rileyportfolio.dev
🤖 Screener: Last one — how many years were you in school before graduating?
👤 Riley: 4 years for bachelor's, plus 2 internships in the last two summers
[junior_thanks: condition TRUE → run]
🤖 Screener: Thanks for the details! The hiring manager reviews 
            portfolios async. You'll hear back within 5 business days.
[Agenda completes]
```

Same agenda. Totally different path. No duplicated items, no duplicated prompts. The conditional checks route each candidate to the right questions.

## Gotchas I Hit

**Missing outputs in conditions**. If a condition references `country` but the Gather item didn't collect it, `outputs.get('country')` returns `None`. All operators against `None` return `False`, so the item is skipped silently. Fix: validate condition references against declared outputs at agenda-load time.

**AI condition drift**. Early on, the same AI condition evaluated differently on different runs because I wasn't passing `temperature=0`. Setting temperature to 0 doesn't fully eliminate drift but cuts it by ~90%. For audit-critical conditions (compliance, gating), prefer JSON mode.

**Implicit order assumptions**. Conditions depend on outputs from **prior** items in the agenda. If you reorder items and a condition now references something that hasn't been collected yet, the check evaluates to `False` and the path silently changes. Fix: the engine now flags forward references at load time and refuses to start the agenda.

**The "skip everything" trap**. If every remaining item's condition is `False`, the agenda ends without any Convey telling the user what happened. At minimum, include a catch-all Convey at the end with no condition — it always runs as the closing message.

## What's Next

Conditions decide which items run. The next post wires up the **events** that drive the whole state machine — how `startAgenda`, `endAgendaItem`, `exitAgenda`, and the other 5 events coordinate everything we've covered so far. [Day 11 — State Machine & Event Handlers](/posts/state-machine-event-driven-agenda-transitions/).

---

### Related in this series

- [Day 1 — The Agenda Engine](/posts/agenda-engine-deterministic-ai-conversations/) — the execution loop that checks conditions
- [Day 4 — Conversation State in Redis](/posts/conversation-state-in-redis/) — where condition evaluations read their inputs
- [Day 6 — The Gather Item](/posts/gather-item-structured-data-collection/) — the item that produces most of the values conditions check
- [Day 9 — Multi-Level Prompt Architecture](/posts/multi-level-prompt-architecture/) — the AI-evaluated condition mode reuses the prompt stack

### References

- [MongoDB query operators](https://www.mongodb.com/docs/manual/reference/operator/query/) — the `$eq`, `$gt`, `$and` syntax I borrowed
- [JSONLogic](https://jsonlogic.com/) — an alternative JSON-based rule format
- [OpenAI structured outputs](https://platform.openai.com/docs/guides/structured-outputs) — for AI-evaluated conditions
