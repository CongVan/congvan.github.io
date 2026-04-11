---
title: "Agenda State Machine: 8 Events That Drive Every Conversation"
date: 2026-04-11T19:30:00+07:00
lastmod: 2026-04-11T19:30:00+07:00
draft: false
tags: ["Backend"]
categories: ["Building a Conversational AI Platform"]
series: ["Building a Conversational AI Platform"]
summary: "The 8 events that move a conversation through its lifecycle: select-agenda, start-agenda, start-item, update-item, end-item, end-agenda, exit-agenda, resume-agenda. Each one has a real scenario."
description: "The 8 events that move a conversation through its full lifecycle, from trigger detection to completion or abandon, each tied to a real scenario."
ShowToc: true
weight: 11
seriesTotal: 12
---

{{< series-nav >}}

*Day 11 of 12. Past posts covered the items themselves. This post is about the **events that move a conversation between items** — the state machine plumbing that makes agendas reliable across thousands of edge cases.*

> **TL;DR**
> - **8 events** drive every agenda lifecycle: `select-agenda`, `start-agenda`, `start-item`, `update-item`, `end-item`, `end-agenda`, `exit-agenda`, `resume-agenda`. (Plus `timeout` as a special triggered event.)
> - **Every event is logged to Redis Streams** ([Day 3](/posts/streaming-ai-responses-llm-to-redis/)) for replay and audit.
> - **Every event is persisted to Redis Hashes** ([Day 4](/posts/conversation-state-in-redis/)) for the next message to read.
> - **Each event ties to a real user scenario** — this post walks through them with the actual messages that fire each one.

---

## What an Event Is, Concretely

An event is a small typed message the agenda engine emits when something meaningful happens in the conversation. Each event has:

- A **type** (one of the 8 below)
- A **conversation_id** so you can group events by conversation
- A **timestamp**
- A **payload** specific to the event type (e.g., `agenda_id`, `item_id`, `outputs`)

```python
class AgendaEvent:
    type: Literal[
        'select-agenda',
        'start-agenda',
        'start-item',
        'update-item',
        'end-item',
        'end-agenda',
        'exit-agenda',
        'resume-agenda',
    ]
    conversation_id: str
    timestamp_ms: int
    payload: dict
```

Events are emitted by the AI service (running the agenda items), forwarded through gRPC to the backend, and the backend writes them to Redis (covered in [Day 3](/posts/streaming-ai-responses-llm-to-redis/) and [Day 4](/posts/conversation-state-in-redis/)). The state machine doesn't *do* anything beyond emit events — the listeners decide how to react.

![State machine: 8 events drive transitions between Q&A, Confirm, In-Agenda, and terminal states](/images/blog/state-machine-events.svg)

## Event 1: `select-agenda`

**Fires when**: the user says something the trigger classifier matches to an agenda's trigger phrase.

**Real scenario**: a user has been chatting in Q&A mode, asking about pricing and features. They eventually say:

```
👤 user: ok this is great. I want to sign up for the Pro plan.
```

The agenda engine, watching every message during Q&A mode, classifies this against the configured triggers. "Pro plan signup" matches the `signup_pro` agenda. It emits:

```json
{
  "type": "select-agenda",
  "conversation_id": "conv-4821",
  "timestamp_ms": 1744377823145,
  "payload": {
    "agenda_id": "signup_pro",
    "matched_trigger": "I want to sign up for the Pro plan",
    "confidence": 0.94
  }
}
```

What the listeners do:
- **Backend**: writes to Redis stream + updates `:state` hash with `mode = "confirm_agenda"`
- **AI service**: starts building the confirmation message ("Got it — want me to walk you through Pro signup?")
- **Admin dashboard**: a new "agenda triggered" badge appears next to this conversation

## Event 2: `start-agenda`

**Fires when**: the user confirms they want to proceed with the agenda (or auto-start kicks in).

**Real scenario**: after the confirmation prompt, the user says yes:

```
🤖 AI: Got it — want me to walk you through Pro signup?
👤 user: yeah let's do it
```

The agenda engine detects affirmative intent and emits:

```json
{
  "type": "start-agenda",
  "conversation_id": "conv-4821",
  "timestamp_ms": 1744377828012,
  "payload": {
    "agenda_id": "signup_pro",
    "first_item_id": "convey_welcome",
    "pre_populated_outputs": {
      "user_email": "alex@acme.io"
    }
  }
}
```

Note `pre_populated_outputs` — the engine scanned the prior 20 Q&A messages and found Alex already mentioned their email earlier. That value is pre-loaded so the upcoming Gather item won't re-ask.

What the listeners do:
- **Backend**: `:state.mode = "in_agenda"`, `:state.agenda_id = "signup_pro"`, `:state.current_item_id = "convey_welcome"`. Pre-populated outputs land in `:agenda:outputs`.
- **AI service**: invokes the first item's `on_start()` handler.

## Event 3: `start-item`

**Fires when**: an agenda item begins executing — either the first item after `start-agenda` or any subsequent item after the previous one ended.

**Real scenario**: the welcome Convey item just finished. The next item is a Gather collecting full name and company. The engine fires:

```json
{
  "type": "start-item",
  "conversation_id": "conv-4821",
  "timestamp_ms": 1744377831205,
  "payload": {
    "item_id": "gather_basics",
    "item_type": "gather",
    "item_index": 1
  }
}
```

This event is mostly for observability — admins can see exactly which item is active for any conversation, in real time. It's also what the [conditional logic check](/posts/conditional-logic-branching-agenda-items/) hooks into: if the item's condition evaluates to false, the engine emits `end-item` immediately with `status = skipped` instead of `start-item`.

## Event 4: `update-item`

**Fires when**: an item's internal state changes mid-execution (a new field provided, a follow-up question generated, an output extracted).

**Real scenario**: the Gather item is collecting fields one at a time. After the user provides their full name:

```
🤖 AI: What's your full name?
👤 user: Alex Rivera
```

The Gather item validates the response and emits:

```json
{
  "type": "update-item",
  "conversation_id": "conv-4821",
  "timestamp_ms": 1744377835880,
  "payload": {
    "item_id": "gather_basics",
    "delta": {
      "outputs": {
        "full_name": {
          "value": "Alex Rivera",
          "state": "PROVIDED"
        }
      }
    }
  }
}
```

`update-item` is the most frequent event during a Gather or Debate. It fires every time a field gets filled in, every time a follow-up question gets generated, every time the AI extracts an inferred output from a free-form response.

The admin dashboard uses these events to render a live progress bar — "3 of 4 fields collected, currently asking for company". Without `update-item`, the dashboard would only know the item started, not how far along it is.

## Event 5: `end-item`

**Fires when**: an item completes — either because it finished naturally (`should_end == True`) or because its condition evaluated to false at the start.

**Real scenario**: the Gather item just collected the last required field. Its `should_end` flips true and the engine fires:

```json
{
  "type": "end-item",
  "conversation_id": "conv-4821",
  "timestamp_ms": 1744377854221,
  "payload": {
    "item_id": "gather_basics",
    "status": "completed",
    "outputs": {
      "full_name": "Alex Rivera",
      "email": "alex@acme.io",
      "company": "Acme Analytics",
      "team_size": "12"
    },
    "duration_ms": 23016
  }
}
```

`end-item` is the event the [Redis fan-out from Day 4](/posts/conversation-state-in-redis/) reacts to — it writes the outputs to `:agenda:outputs`, appends to `:agenda:history`, and updates `:state.current_item_id` to the next item's id.

If the item was skipped because of a condition, the payload looks like:

```json
{
  "type": "end-item",
  "conversation_id": "conv-4821",
  "timestamp_ms": 1744377854221,
  "payload": {
    "item_id": "debate_technical",
    "status": "skipped",
    "skip_reason": "condition false: years_experience < 5"
  }
}
```

Same event, different status. Listeners distinguish on `payload.status`.

## Event 6: `end-agenda`

**Fires when**: the last item in the agenda completes (or all remaining items get skipped).

**Real scenario**: the signup agenda has 5 items. The last one — a Convey saying "all set, check your email" — just finished. The engine fires:

```json
{
  "type": "end-agenda",
  "conversation_id": "conv-4821",
  "timestamp_ms": 1744377889441,
  "payload": {
    "agenda_id": "signup_pro",
    "status": "completed",
    "total_duration_ms": 64296,
    "items_run": 4,
    "items_skipped": 1,
    "final_outputs": {
      "full_name": "Alex Rivera",
      "email": "alex@acme.io",
      "company": "Acme Analytics",
      "team_size": "12",
      "selected_plan": "pro_annual"
    }
  }
}
```

This is the event that fires **completion actions** — webhooks to external systems, CRM creates, email confirmations. The listener pattern:

```python
@on_event('end-agenda')
async def fire_completion_actions(event: AgendaEvent):
    agenda = await load_agenda(event.payload['agenda_id'])

    for action in agenda.completion_actions:
        if action.type == 'webhook':
            await fire_webhook(action.url, event.payload['final_outputs'])
        elif action.type == 'hubspot':
            await create_hubspot_deal(event.payload['final_outputs'])
        elif action.type == 'email':
            await send_email(action.template, event.payload['final_outputs'])

    # Mark conversation back to Q&A mode for the next user message
    await redis.hSet(f'conversation:{event.conversation_id}:state', {
        'mode': 'qa',
        'last_completed_agenda': event.payload['agenda_id'],
    })

    # Flush durable record to Postgres
    await flush_to_postgres(event.conversation_id, event.payload)
```

After `end-agenda`, the conversation drops back to Q&A mode. The user can keep chatting, ask follow-up questions, or trigger another agenda.

## Event 7: `exit-agenda`

**Fires when**: the user explicitly quits the agenda before completion.

**Real scenario**: mid-Gather, the user gets frustrated:

```
🤖 AI: What's your team size?
👤 user: actually never mind, I changed my mind
```

The exit-detection classifier (same one from [Day 8](/posts/qa-item-knowledge-base-mid-agenda/)) returns `intent: exit`. The engine fires:

```json
{
  "type": "exit-agenda",
  "conversation_id": "conv-4821",
  "timestamp_ms": 1744377896113,
  "payload": {
    "agenda_id": "signup_pro",
    "status": "exited",
    "exit_reason": "user_explicit",
    "items_completed": ["convey_welcome", "gather_basics"],
    "items_remaining": ["select_plan", "convey_confirmation"],
    "partial_outputs": {
      "full_name": "Alex Rivera",
      "email": "alex@acme.io",
      "company": "Acme Analytics"
    }
  }
}
```

The partial outputs are still saved. Why? Because if Alex comes back tomorrow and triggers signup again, the engine can pre-populate from this partial state — they won't have to retype their email.

Some agendas configure exit hooks that fire on `exit-agenda` with `status = exited`:

```python
@on_event('exit-agenda')
async def handle_exit(event: AgendaEvent):
    # Trigger an abandoned-cart-style follow-up email
    if event.payload.get('partial_outputs', {}).get('email'):
        await schedule_followup_email(
            email=event.payload['partial_outputs']['email'],
            template='abandoned_signup',
            send_at=datetime.now() + timedelta(hours=24),
        )
```

One of my favorite real-world wins: an enterprise client added an `exit-agenda` hook that sent abandoned-cart emails 24 hours after a partial signup. Recovery rate: ~12%.

## Event 8: `resume-agenda`

**Fires when**: a paused agenda resumes — typically after a [Q&A item](/posts/qa-item-knowledge-base-mid-agenda/) interruption ended.

**Real scenario**: from the Day 8 pizza ordering example. Mid-Gather, the user asks an off-topic question. The Q&A item kicks in, answers, and the user signals they're done:

```
🤖 Pizza X: We're open until 11pm tonight. Anything else?
👤 user: nope, no more questions
[Q&A item detects exit, control returns to Gather]
```

The engine fires:

```json
{
  "type": "resume-agenda",
  "conversation_id": "conv-9912",
  "timestamp_ms": 1744377912008,
  "payload": {
    "resumed_item_id": "gather_order_details",
    "resumed_field": "delivery_address",
    "pause_duration_ms": 18420,
    "interrupt_type": "qa_item"
  }
}
```

`resume-agenda` is what triggers the **contextual re-ask** — the listener generates a new prompt that acknowledges the detour ("So back to your order — pepperoni and mushroom medium. What's your delivery address?") rather than repeating the original question verbatim.

## The Special Case: `timeout`

`timeout` isn't in the 8 main events because it's not emitted by the engine — it's emitted by a separate scheduled job. But it's the same shape:

```json
{
  "type": "timeout",
  "conversation_id": "conv-4821",
  "timestamp_ms": 1744378400000,
  "payload": {
    "agenda_id": "signup_pro",
    "last_activity_ms": 1744377896113,
    "idle_duration_ms": 503887
  }
}
```

The scheduled job (a BullMQ delayed job, set during `start-agenda`) fires after the configured idle timeout (default 30 minutes). It's handled identically to `exit-agenda` — the partial outputs are saved, the agenda is marked `status = timed_out`, and any timeout webhook fires.

## Event Order in a Real Conversation

Here's the full event sequence for the Pro signup scenario above:

```
t=0     select-agenda      (user said "I want to sign up for Pro")
t=5s    start-agenda       (user confirmed)
t=5s    start-item         (convey_welcome)
t=8s    end-item           (convey_welcome, status: completed)
t=8s    start-item         (gather_basics)
t=12s   update-item        (full_name provided)
t=18s   update-item        (email provided)
t=24s   update-item        (company provided)
t=31s   update-item        (team_size provided)
t=31s   end-item           (gather_basics, status: completed)
t=31s   start-item         (select_plan)
t=42s   update-item        (selected_plan provided)
t=42s   end-item           (select_plan, status: completed)
t=42s   start-item         (convey_confirmation)
t=45s   end-item           (convey_confirmation, status: completed)
t=45s   end-agenda         (status: completed)
                           ↓
                           webhook fires → CRM created → confirmation email sent
```

15 events for one ~45-second conversation. Every one is in the Redis stream, which means I can replay any conversation step-by-step, see exactly when state changed, and build dashboards on top of the same event log.

## Why Events Instead of Direct Mutations

The first version of the agenda engine had no events. Item handlers wrote directly to Redis. It worked, but three things broke at scale:

1. **No replayability**. If a webhook listener failed, I couldn't replay the failed action without reconstructing state from chat messages.
2. **No observability**. The admin dashboard had to poll Redis to know what was happening. With events, it subscribes to a Redis stream and gets pushed updates.
3. **Tightly coupled writes**. Every new feature (admin dashboard, audit log, webhook system, analytics) had to add code inside the item handlers. With events, each feature is a listener that consumes the same event stream.

Refactoring to events took ~2 weeks. After that, every new listener was 50 lines of code instead of 500.

## What's Next

That's the full agenda lifecycle. The last post in the series ([Day 12](/posts/four-channels-one-engine/)) covers how all this works across **four different channels** — webchat, widget, WhatsApp, and Messenger — with a single unified conversation engine.

---

### Related in this series

- [Day 3 — Streaming AI Responses to Redis Streams](/posts/streaming-ai-responses-llm-to-redis/) — where these events get serialized
- [Day 4 — Conversation State in Redis](/posts/conversation-state-in-redis/) — what the listeners write
- [Day 8 — Q&A Item](/posts/qa-item-knowledge-base-mid-agenda/) — the source of `resume-agenda` events
- [Day 10 — Conditional Logic](/posts/conditional-logic-branching-agenda-items/) — produces `end-item` with `status: skipped`

### References

- [Event Sourcing pattern (Martin Fowler)](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/) — the event log
- [BullMQ Delayed Jobs](https://docs.bullmq.io/guide/jobs/delayed) — how the timeout event gets scheduled
