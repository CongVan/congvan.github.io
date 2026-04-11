---
title: "Convey Item: Static vs Dynamic AI Messages (85/15 Rule)"
date: 2026-04-11T12:00:00+07:00
lastmod: 2026-04-11T16:00:00+07:00
draft: false
tags: ["AI Research", "Backend"]
categories: ["Building a Conversational AI Platform"]
series: ["Building a Conversational AI Platform"]
summary: "The simplest agenda item. Convey delivers one message, then ends. Static vs dynamic, placeholder substitution, and why I use static 85% of the time."
description: "The simplest agenda item. Convey delivers one message, then ends. Static vs dynamic, placeholder substitution, and why I use static 85% of the time."
ShowToc: true
weight: 5
seriesTotal: 12
---

{{< series-nav >}}

*Day 5 of 12. The [Agenda Engine](/posts/agenda-engine-deterministic-ai-conversations/) is built from four core item types. Today: the simplest one — Convey. It sends one message, then it's done.*

> **TL;DR**
> - **What it does**: Convey speaks one line and ends. No data collected, no follow-up questions.
> - **Two modes**: static (exact text, zero LLM calls, instant) or dynamic (LLM generates from a prompt).
> - **The 85/15 rule**: across every production agenda I've built, static covers ~85% of Convey uses. Dynamic is reserved for personalized closings where the message genuinely depends on what just happened.
> - **Placeholders** (`{{ user_name }}`, `{{ startup_name }}`) make "static" flexible without paying the LLM cost.

---

## What Convey Does

Convey is the item with the fewest moving parts. It speaks one line to the user and ends. No data collected, no follow-up questions, no waiting for a response.

You'd think it's too simple to be worth its own post. But it's also the most-used item in every single agenda I've ever built. Every welcome message, every progress update, every closing line is a Convey item. When an agenda has six items and five of them are Gather/Debate/Q&A doing the real work, the first and last are usually Convey.

It comes in two flavors: **static** (exact text) and **dynamic** (LLM-generated from a prompt).

## Static or Dynamic Convey: Which Should You Use?

![Static vs dynamic Convey: two paths to one outgoing message](/images/blog/convey-static-vs-dynamic.svg)

| | Static | Dynamic |
|---|---|---|
| **Stored as** | Exact text | A prompt |
| **LLM call** | None | Yes |
| **Cost** | Zero tokens | ~200-500 tokens |
| **Latency** | Instant | 1-3 seconds |
| **Use for** | Disclaimers, confirmations, legal text | Personalized closings, context-aware summaries |

The instinct when you first build a conversational platform is to make everything dynamic. "It's AI-powered, let the AI write all the words." I did that for about two weeks before reversing course.

The problem: dynamic Convey messages are **slow, expensive, and sometimes wrong**. For a "Thanks for applying, we'll get back to you in 3 days" message, letting the LLM generate it means 1-3 seconds of latency, 300 tokens burned, and a non-zero chance of the AI making up a different timeframe. The static version is zero tokens, zero milliseconds, zero hallucination.

Rule of thumb: **use static unless the content genuinely depends on what just happened in the conversation**.

## Static Convey

Static Convey stores the exact text to send. The agenda item's config has a `message` field with the literal string.

```python
class ConveyItem(BaseAgendaItem):
    mode: Literal['static', 'dynamic']
    message: str  # for static: exact text; for dynamic: a prompt

    async def on_start(self):
        if self.mode == 'static':
            rendered = self.render_placeholders(self.message)
            self.emit(MessageEvent(content=rendered))
        else:
            await self.generate_and_emit()

    @property
    def should_end(self) -> bool:
        return True  # Convey always ends after one message
```

The `should_end` property returning `True` is the reason Convey is the simplest item. The moment the message is emitted, the item is done. The [state machine from Day 4](/posts/conversation-state-in-redis/) writes `endAgendaItem` and moves to the next item. No waiting for a user response, no tracking intermediate state.

## Placeholders: The Static Mode's Escape Hatch

"Static" doesn't mean "completely fixed." Most static messages use **placeholders** — named references to values that come from earlier items or the user profile.

```python
def render_placeholders(self, template: str) -> str:
    """Replace {{ placeholder }} with values from conversation context."""
    context = {
        'user_name': self.state.user.name or 'there',
        'user_email': self.state.user.email,
        'agenda_name': self.state.agenda.name,
        # All outputs collected so far in this agenda
        **self.state.agenda.outputs,
    }

    def replace(match):
        key = match.group(1).strip()
        return str(context.get(key, f'{{{{ {key} }}}}'))

    return re.sub(r'\{\{\s*(\w+)\s*\}\}', replace, template)
```

A welcome message like:

```
Hi {{ user_name }}, welcome to the founder screening. I'll ask you about
{{ startup_name }} and your experience before we move on.
```

Gets rendered, at runtime, using whatever the user's name and startup name are. No LLM involved. The placeholders are resolved from the outputs hash in Redis ([Day 4](/posts/conversation-state-in-redis/)) and from the user profile — same mechanism, same data structure, just string substitution.

This covers 80% of the cases where you'd otherwise reach for dynamic Convey.

## Dynamic Convey

Dynamic Convey is for the 20% of cases where the content really does depend on what happened in the conversation in a way a template can't capture. Personalized summaries, closing notes that reference something the user said, adaptive recommendations — these are the real dynamic jobs.

The config stores a prompt instead of a literal message:

```python
class ConveyItem(BaseAgendaItem):
    async def generate_and_emit(self):
        # Build the system prompt from the item config
        system_prompt = self.prompt_builder.compose(
            role=self.persona.role,
            style=self.persona.communication_style,
            task=self.message,  # the prompt, e.g. "Write a personalized closing..."
            outputs=self.state.agenda.outputs,  # prior collected data
        )

        # Stream the LLM response chunk by chunk
        async for chunk in llm.chat_stream(system_prompt=system_prompt):
            if chunk.content:
                self.emit(MessageChunkEvent(text=chunk.content))

        self.emit(EndMessageEvent())
```

An example dynamic prompt for a founder screening closing:

```
Write a personalized closing message for the candidate based on their
responses. Acknowledge one specific strength they demonstrated. Keep it
to 2-3 sentences. Don't promise a specific timeline.
```

The LLM sees all the prior outputs (from Gather and Debate items earlier in the agenda, covered in [Day 6](/posts/gather-item-structured-data-collection/) and [Day 7](/posts/debate-item-ai-follow-up-questions/)) as context. The "strength to acknowledge" is whatever the AI decides from reading the conversation — that's exactly the kind of judgment that can't live in a template.

## Streaming the Dynamic Output

Dynamic Convey streams tokens through the same pipe as any other LLM response — see [Day 3](/posts/streaming-ai-responses-llm-to-redis/). The user sees the text appear word by word, same as ChatGPT. The chunks land in Redis Streams, the state machine emits `startAgendaItem` → [chunks] → `endMessage` → `endAgendaItem`, and the next item starts.

One detail that matters: **dynamic Convey doesn't use tool calls**. It's a plain chat completion. That's one of the few places in the agenda engine where tool calls are off — Convey's job is to *say something*, not to decide what to do next. Tool calls come back in Gather and Debate items.

## When to Use Each Mode

Static works when the message:
- Is the same for every user (welcome, instructions, disclaimer)
- Can be parameterized with placeholders (personalized greeting, confirmation)
- Must not vary (legal text, exact dates, regulatory language)

Dynamic works when the message:
- References things the user just said in open-ended ways
- Needs to acknowledge or summarize multiple prior outputs
- Is the final closing of an agenda and you want it to feel personal

My rough split across all the Convey items in production agendas: **~85% static, ~15% dynamic**. The 15% is almost entirely closing messages after a Debate item.

## Common Patterns

### Welcome message (static + placeholder)

```yaml
type: convey
mode: static
message: |
  Hi {{ user_name }}, thanks for applying to {{ program_name }}.
  I'll ask you a few questions about your background. Should take
  about 5 minutes.
```

### Progress update mid-agenda (static)

```yaml
type: convey
mode: static
message: |
  Great, we've covered the basics. Next I'll ask a few questions
  about your technical experience.
```

### Confirmation summary (static + placeholder + list)

```yaml
type: convey
mode: static
message: |
  Just to confirm:
  - Name: {{ founder_name }}
  - Email: {{ email }}
  - Stage: {{ funding_stage }}

  Let me know if anything looks wrong.
```

### Personalized closing (dynamic)

```yaml
type: convey
mode: dynamic
message: |
  Write a 2-3 sentence closing message for this candidate. Acknowledge
  one specific strength from their responses. Mention that the team will
  review within 3 business days. Don't commit to a specific outcome.
```

## Why This Item Matters

Convey is boring. It ends after one message. It has no state. Most of the interesting logic in the series happens in Gather, Debate, Q&A, and the state machine.

But Convey is also the item that shapes the *feel* of every conversation. The welcome message sets the tone. The progress updates keep users oriented. The closing leaves the last impression. And because Convey is cheap (mostly static, zero LLM cost), I use it liberally — sometimes 3-4 Convey items in a 6-item agenda, just to narrate the flow for the user.

Next up: [Day 6 — The Gather Item](/posts/gather-item-structured-data-collection/), which actually collects data.

---

### Related in this series

- [Day 1 — The Agenda Engine](/posts/agenda-engine-deterministic-ai-conversations/) — how items fit into the state machine
- [Day 3 — Streaming AI Responses](/posts/streaming-ai-responses-llm-to-redis/) — how dynamic Convey chunks reach Redis
- [Day 4 — Conversation State in Redis](/posts/conversation-state-in-redis/) — where placeholder values come from
- [Day 9 — Multi-Level Prompt Architecture](/posts/multi-level-prompt-architecture/) *(coming later)* — how the persona's style flows into dynamic Convey prompts

### References

- [OpenAI Chat Completions API](https://platform.openai.com/docs/api-reference/chat)
- [Mustache template syntax](https://mustache.github.io/mustache.5.html) — the `{{ placeholder }}` convention I borrowed from
