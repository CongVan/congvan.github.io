---
title: "The Gather Item: Collecting Structured Data Through Conversation"
date: 2026-04-11T13:00:00+07:00
lastmod: 2026-04-11T13:00:00+07:00
draft: false
tags: ["AI Research", "Backend"]
categories: ["Building a Conversational AI Platform"]
series: ["Building a Conversational AI Platform"]
summary: "Gather is where the platform earns its keep — turning a messy free-form conversation into clean structured data. Asked vs inferred outputs, regex validation, a field-level state machine, pre-population from prior context, and the stale threshold that stops the AI from asking the same question forever."
ShowToc: true
weight: 6
seriesTotal: 12
---

{{< series-nav >}}

*Day 6 of 12. [Day 5](/posts/convey-item-scripted-dynamic-messages/) was Convey — the simple item that just sends a message. Today: Gather, the item that does the actual data collection work.*

---

## Why Gather Is the Hardest Simple Item

On paper, Gather looks trivial: "ask the user for data, save it." In practice, it's the item I spent the most time on. The reason is that a five-line conversation like:

> AI: What's your email?
> User: it's on my LinkedIn
> AI: I need to confirm — could you type it directly?
> User: jane@startup.com
> AI: Thanks!

...involves a tiny pile of decisions: is "it's on my LinkedIn" an answer or a dodge? Does "jane@startup.com" pass regex? If it fails, how many retries before I give up? Can I ask about two fields in one turn, or does that overwhelm the user? If the user already mentioned their email five messages ago in Q&A mode, should I ask again?

Every one of those decisions is handled inside a single Gather item. Here's how.

## Asked vs Inferred Outputs

Gather items have a list of **outputs** (named fields to collect). Each output is either *asked* or *inferred*.

![Asked vs inferred outputs](/images/blog/gather-asked-vs-inferred.svg)

An **asked** output is one the AI explicitly prompts for: "What's your email?" The answer is the user's direct response. Validation is regex or type-based. The flow is linear — question, answer, validate, move on.

An **inferred** output is extracted from the conversation without a direct question. Example: the user mentions "we raised pre-seed from a small fund last year" and the system extracts `funding_stage = "pre-seed"`. No question was asked — the LLM read the message and pulled the structured value out.

A single Gather item can mix both:

```python
class GatherItem(BaseAgendaItem):
    outputs: list[GatherOutput]

class GatherOutput(BaseModel):
    name: str                       # e.g. "email"
    kind: Literal['asked', 'inferred']
    required: bool
    description: str                # natural language for the LLM
    validator: str | None = None    # regex pattern, only for asked
    confirmation_needed: bool = False
```

Example config for a founder screening Gather item:

```yaml
type: gather
outputs:
  - name: founder_name
    kind: asked
    required: true
    description: "Full legal name of the founder"
    confirmation_needed: true

  - name: email
    kind: asked
    required: true
    description: "Email address"
    validator: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'

  - name: funding_stage
    kind: inferred
    required: false
    description: "Current funding stage (pre-seed, seed, series A, etc.)"
```

Asked outputs drive question generation. Inferred outputs are scanned from the transcript after every user turn. The distinction matters because it changes when the LLM is invoked and what prompt it sees.

## The Field State Machine

Every output has a state. The item only completes when all *required* outputs reach a terminal state.

![Field state machine](/images/blog/gather-field-state-machine.svg)

```python
class FieldState(str, Enum):
    PENDING = 'pending'         # Not collected yet
    PRE_FILLED = 'pre_filled'   # Found in conversation history before item started
    PROVIDED = 'provided'       # User answered, validation passed
    INVALID = 'invalid'         # User answered, validation failed
    REFUSED = 'refused'         # User explicitly declined
    STALE = 'stale'             # Failed 3 times, giving up
```

Terminal states: `PRE_FILLED`, `PROVIDED`, `REFUSED`, `STALE`. Non-terminal: `PENDING`, `INVALID`. The item loops until every required output is terminal.

```python
@property
def should_end(self) -> bool:
    return all(
        f.state in {FieldState.PROVIDED, FieldState.PRE_FILLED,
                    FieldState.REFUSED, FieldState.STALE}
        for f in self.outputs if f.required
    )
```

The `STALE` state is the escape hatch. Without it, the AI asks "what's your email?" three times, the user keeps saying "I'd rather not", and the conversation loops forever. With `STALE`, after three failed attempts the system marks the field as stale and moves on. That stale marker flows through to the completion webhook — the downstream system can decide whether to follow up out-of-band.

## Pre-Population From Prior Context

The best user experience is not asking questions the user already answered. Before the first question, Gather scans the recent conversation for anything that matches its outputs.

```python
async def on_start(self):
    # Scan the last 20 messages for values matching output names
    recent = self.state.messages[-20:]

    for output in self.outputs:
        inferred = await self.try_extract_from_history(output, recent)
        if inferred is not None:
            self.store_output(output.name, inferred)
            output.state = FieldState.PRE_FILLED

    # Now start asking for anything still PENDING
    await self.ask_next_pending_field()
```

The `try_extract_from_history` helper is a small LLM call — it gives the model the output definition (name + description + optional validator) and the recent messages, and asks for the value if it's clearly present. If not, it returns `None`.

In practice, this catches 30-50% of the fields users mention during Q&A mode before triggering the agenda. When a user says "I want to apply, my name is Jane Chen and I'm the founder of StackShift" and the agenda's first Gather item needs `founder_name` and `startup_name`, both get pre-filled and the item moves straight to the next missing field.

## Regex Validation for Asked Outputs

Regex is the simplest validation layer. For emails, phone numbers, URLs, dates — regex catches 95% of the bad cases.

```python
async def on_user_response(self, message: str):
    field = self.current_field  # the one we just asked about

    if field.validator:
        import re
        if not re.match(field.validator, message.strip()):
            field.state = FieldState.INVALID
            field.attempts += 1
            if field.attempts >= 3:
                field.state = FieldState.STALE
            else:
                await self.re_ask_with_feedback(field, message)
            return

    # Validation passed
    self.store_output(field.name, message.strip())
    field.state = FieldState.PROVIDED

    if field.confirmation_needed:
        await self.confirm_with_user(field)
```

The `re_ask_with_feedback` method generates a clarifying question rather than just repeating the original. Instead of "What's your email?" a second time, the AI says "That doesn't look like a valid email — could you double-check?" The difference feels small, but users hate being asked the exact same question verbatim.

Regex isn't the only option — for more complex validation (international phone numbers, structured dates), I call a real validator (libphonenumber, date parser). The state machine doesn't care which — it only cares whether validation returned true or false.

## Confirmation Step

Some outputs are too important to trust without confirmation. Names, email addresses, phone numbers — if the AI misheard and the downstream system fires a webhook with bad data, it's embarrassing.

```python
async def confirm_with_user(self, field: GatherOutput):
    value = self.state.agenda.outputs[field.name]
    question = f'Just to confirm: {field.name} is "{value}" — is that right?'
    self.emit(MessageEvent(content=question))
    # Next user response flows into confirm_response()

async def confirm_response(self, message: str):
    if self.detect_affirmative(message):
        self.current_field.state = FieldState.PROVIDED
        self.move_to_next_field()
    else:
        # User said no — re-ask
        self.current_field.state = FieldState.PENDING
        self.current_field.attempts += 1
        await self.ask_next_pending_field()
```

The `detect_affirmative` helper is a small LLM classification call. "Yes", "yep", "that's correct", "sounds right", "confirmed" — all positive. "No", "nope", "wait", "that's wrong" — all negative. Edge cases like "the name is right but fix the email" are handled by re-scanning the response for any output value.

Confirmation is opt-in per-field via the `confirmation_needed` flag. I turn it on for outputs that go into CRM systems and off for everything else — it adds a conversation turn, which is cost for the user.

## Inferred Output Extraction

Inferred outputs don't get asked — they get extracted after every user turn. The extraction is a dedicated LLM call that reads the latest message and the output definitions.

```python
async def on_user_response(self, message: str):
    # Handle the asked field first (if any)
    if self.current_field:
        await self.handle_asked_response(message)

    # Then try to extract any inferred outputs
    inferred_outputs = [o for o in self.outputs
                        if o.kind == 'inferred' and o.state == FieldState.PENDING]

    if inferred_outputs:
        extracted = await self.extract_inferred(message, inferred_outputs)
        for name, value in extracted.items():
            if value is not None:
                self.store_output(name, value)
                next(o for o in self.outputs if o.name == name).state = FieldState.PROVIDED
```

The key insight: inferred extraction runs **after every user turn**, not only when the LLM explicitly asks about a field. So if the user volunteers information — "we're pre-seed and looking to close next quarter" — the `funding_stage` output gets populated even though the AI never asked.

## When Gather Ends

Gather completes when every required output is terminal. The item emits `endAgendaItem` with the collected data, the state machine persists it to [Redis](/posts/conversation-state-in-redis/), and the next item starts.

The emit looks like this:

```python
async def on_end(self):
    outputs = {
        f.name: self.state.agenda.outputs.get(f.name)
        for f in self.outputs
    }
    states = {f.name: f.state for f in self.outputs}

    self.emit(EndAgendaItemEvent(
        item_id=self.id,
        outputs=outputs,
        output_states=states,
    ))
```

Both the values and the states flow downstream. Completion webhooks can distinguish between "user provided this" (`PROVIDED`), "we had this already" (`PRE_FILLED`), "user declined" (`REFUSED`), and "we gave up asking" (`STALE`). Different downstream systems treat those differently — some auto-disqualify on `REFUSED`, others flag `STALE` fields for a human to follow up.

## Gather vs Debate

Gather is for specific fields. Debate (coming next, [Day 7](/posts/debate-item-ai-follow-up-questions/)) is for open-ended exploration. The quick heuristic:

- "I need the user's email, phone, and company name" → Gather
- "I want to understand how they think about their market" → Debate

Gather hits a clear completion state (all fields terminal). Debate runs for a fixed number of follow-up rounds. They look similar on the surface but their state machines are different, and the prompts they build for the LLM are different too.

## Closing

Gather is the item that turns conversations into data. Most of the "is my AI chatbot reliable?" question lives inside this one item: can it extract structured values from messy replies, validate them, re-ask intelligently, and give up gracefully? The state machine above is the answer.

Next up: [Day 7 — The Debate Item](/posts/debate-item-ai-follow-up-questions/), which is Gather's open-ended cousin.

---

### Related in this series

- [Day 1 — The Agenda Engine](/posts/agenda-engine-deterministic-ai-conversations/) — where Gather fits in the state machine
- [Day 4 — Conversation State in Redis](/posts/conversation-state-in-redis/) — where Gather outputs land
- [Day 5 — The Convey Item](/posts/convey-item-scripted-dynamic-messages/) — the item before Gather in most agendas
- [Day 7 — The Debate Item](/posts/debate-item-ai-follow-up-questions/) — Gather's open-ended cousin

### References

- [OpenAI Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs) — the mechanism behind inferred extraction
- [Python `re` module](https://docs.python.org/3/library/re.html) — for regex validators
- [libphonenumber](https://github.com/google/libphonenumber) — the real-world phone number validator
