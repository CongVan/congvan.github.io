---
title: "Four Channels, One Engine: Webchat, Widget, WhatsApp, Messenger"
date: 2026-04-11T19:40:00+07:00
lastmod: 2026-04-11T19:40:00+07:00
draft: false
tags: ["Backend", "DevOps", "AI Research"]
categories: ["Building a Conversational AI Platform"]
series: ["Building a Conversational AI Platform"]
summary: "The same agent runs on a website, an embeddable widget, WhatsApp, and Messenger. Each channel has its own quirks — multi-message grouping, 24-hour windows, presence pings. Here's the adapter pattern that keeps the engine clean and the channels fed."
description: "How one Agenda Engine serves four channels — webchat, widget, WhatsApp, Messenger — through a thin adapter layer that normalizes each channel's quirks into a unified message format."
ShowToc: true
weight: 12
seriesTotal: 12
---

{{< series-nav >}}

*Day 12 of 12. The final post in the series. Across the last 11 days I built an [Agenda Engine](/posts/agenda-engine-deterministic-ai-conversations/) that runs structured AI conversations. Today: how the same engine serves four very different channels — and the adapter layer that makes it possible.*

> **TL;DR**
> - **One engine, four channels**: webchat (full page), widget (embedded script), WhatsApp Business, Facebook Messenger.
> - **Adapter pattern**: each channel has a thin adapter that normalizes its quirks into a unified message format. The Agenda Engine never sees channel-specific code.
> - **Real channel quirks**: WhatsApp users send 3 messages in 500ms (debounce + group). Messenger has a 24-hour window for unsolicited messages. Widgets need to handle iframe parent-window communication. Webchat needs to handle file uploads.
> - **Channel-aware persona**: the same persona (from [Day 9](/posts/multi-level-prompt-architecture/)) renders slightly differently per channel — more emoji on WhatsApp, more formality on email, and so on.
> - **Series wrap-up**: this post ties together all 11 prior posts and shows the full architecture diagram.

---

## The Goal: One Agent, Many Doorways

Picture a fictional vacation rental company. They want their AI booking concierge available wherever guests already are:

- **Webchat** on `book.rentalco.example.com` — full-page chat for users browsing their site
- **Widget** embedded on partner travel blogs — a floating chat icon that opens an iframe
- **WhatsApp** for guests outside the US who prefer messaging apps
- **Messenger** for Facebook page visitors

Same AI persona ("Maya, a friendly booking concierge"). Same agendas (room search, reservation, payment). Same knowledge base (properties, amenities, cancellation policies). But four wildly different transport layers, each with its own constraints.

If I built four separate engines, I'd have 4× the bugs, 4× the deployments, and 4× the prompt drift. Instead, I built **one engine and four adapters**.

![Four channels feed into thin adapters that normalize messages for one engine](/images/blog/four-channels-architecture.svg)

## The Unified Message Format

Every adapter, regardless of source, produces messages in the same shape:

```python
@dataclass
class IncomingMessage:
    conversation_id: str           # stable per (channel, user)
    user_id: str                   # external ID from the channel
    channel: Channel               # 'webchat' | 'widget' | 'whatsapp' | 'messenger'
    text: str | None
    attachments: list[Attachment]
    timestamp: datetime
    metadata: dict                 # channel-specific bits (e.g., wa_phone_number)
```

The Agenda Engine accepts only this format. It has zero awareness of HTTP, webhooks, or channel-specific APIs. The adapters do the translation.

```python
class ChannelAdapter(ABC):
    @abstractmethod
    async def to_unified(self, raw: dict) -> IncomingMessage:
        """Convert this channel's raw payload into a unified message."""

    @abstractmethod
    async def from_unified(self, message: OutgoingMessage) -> None:
        """Send an outgoing message back through this channel."""
```

Four implementations, one interface. Let's walk through each.

## Channel 1: Webchat

The webchat lives at `book.rentalco.example.com`. Users browse there and click the chat panel that opens fullscreen.

**Transport**: Server-Sent Events (SSE) for streaming responses, plain REST for sending messages.

**Quirks**:
- Users see typing indicators in real time
- File attachments upload via multipart POST
- Sessions can last hours — users browse other tabs and come back

```python
class WebchatAdapter(ChannelAdapter):
    async def handle_post(self, request):
        body = await request.json()
        msg = IncomingMessage(
            conversation_id=body['session_id'],
            user_id=body['user_id'] or generate_anon_id(),
            channel='webchat',
            text=body['text'],
            attachments=[],
            timestamp=datetime.now(),
            metadata={'user_agent': request.headers.get('user-agent')},
        )
        await engine.process(msg)
        return Response('ok')

    async def stream_response(self, conversation_id: str):
        # SSE endpoint — clients consume events from Redis Streams
        # (the same stream from Day 3)
        async for event in redis.xRead(f'conversation:{conversation_id}:stream', '0'):
            yield f'data: {json.dumps(event)}\n\n'
```

**Real conversation on webchat**:

```
[user opens chat panel on book.rentalco.example.com]
🤖 Maya: Hi! Looking for a place to stay? Tell me where and when.
👤 user: Lisbon, mid-June, 2 adults
🤖 Maya: Got it — Lisbon, mid-June, 2 guests. How many nights?
👤 user: 5 nights
🤖 Maya: I have a few options. Want me to walk you through them, 
        or do you have specific must-haves like a kitchen or sea view?
```

Standard webchat. The user has time and visual context, so Maya's responses can be longer and more conversational.

## Channel 2: Widget

The widget is the same conversation experience, but embedded on a third-party site (a travel blog, a partner agency). It loads as a script tag and renders an iframe.

**Transport**: Same SSE + REST as webchat (the iframe is just a sandboxed webchat).

**Quirks**:
- The iframe needs to communicate with the parent page (resize on open/close, pass URL context, send analytics events)
- Lazy-loaded — script doesn't fire until user scrolls past 50% or 30 seconds idle
- Pre-collects nothing — visitors might not be logged in to the host site

```html
<!-- The embeddable script tag -->
<script src="https://chat.rentalco.example.com/widget.js" 
        data-agent-id="maya-booking" 
        data-position="bottom-right"
        defer></script>
```

The widget JS injects an iframe pointing at `chat.rentalco.example.com/embed?agent=maya-booking`. Inside the iframe, the same webchat adapter handles everything. The only extra layer is `postMessage` between iframe and parent for resize events.

**Real conversation on a partner blog**:

```
[user reading "Top 10 Lisbon neighborhoods" on travelblog.example.com]
[after 30 seconds, the widget opens with a soft chime]
🤖 Maya: Hi! See anything you like in this Lisbon guide? I can help 
        you find a place to stay — just tell me the dates.
👤 user: Yeah I love Alfama. June 12-17, 2 people
🤖 Maya: Alfama is gorgeous. I have 6 properties available those 
        dates. Want them ranked by price, by reviews, or by walking 
        distance to the metro?
```

Same engine. The widget context lets Maya open with a contextual hook ("See anything you like in this Lisbon guide?") because the parent page passed its URL when initializing the widget. That URL is in `metadata.parent_url` — Maya's first Convey item references it via a placeholder.

## Channel 3: WhatsApp Business

WhatsApp is the most constrained channel. It uses the WhatsApp Business Platform's webhook API.

**Transport**: Webhook for incoming, REST POST for outgoing.

**Quirks**:
- **24-hour window**: outside this window, businesses can only send template messages (pre-approved by Meta)
- **Multi-message grouping**: users casually fire 3 messages in 2 seconds — the system has to debounce them
- **No typing indicators**: the user only sees the response when it arrives
- **Attachment types**: voice notes, images, location pins, documents

```python
class WhatsAppAdapter(ChannelAdapter):
    async def handle_webhook(self, request):
        body = await request.json()
        for entry in body['entry']:
            for change in entry['changes']:
                if change['field'] != 'messages':
                    continue
                for raw_msg in change['value'].get('messages', []):
                    msg = await self.to_unified(raw_msg)
                    
                    # Debounce: hold for 1 second to group rapid messages
                    await self.debounce_and_process(msg)

    async def debounce_and_process(self, msg: IncomingMessage):
        debounce_key = f'wa:debounce:{msg.conversation_id}'
        
        # Push this message into a Redis list
        await redis.rPush(debounce_key, msg.text)
        await redis.expire(debounce_key, 5)
        
        # Schedule a delayed job to flush the list after 1 second
        # (BullMQ delayed job, deduplicated by conversation_id)
        await queue.add('flush_wa_debounce', {
            'conversation_id': msg.conversation_id,
        }, {'delay': 1000, 'jobId': debounce_key})

    async def flush_debounce(self, conversation_id: str):
        debounce_key = f'wa:debounce:{conversation_id}'
        messages = await redis.lRange(debounce_key, 0, -1)
        await redis.delete(debounce_key)
        
        if not messages:
            return
        
        # Combine into one logical message
        combined_text = '\n'.join(messages)
        unified = IncomingMessage(
            conversation_id=conversation_id,
            user_id=lookup_user_for_conversation(conversation_id),
            channel='whatsapp',
            text=combined_text,
            attachments=[],
            timestamp=datetime.now(),
            metadata={'message_count': len(messages)},
        )
        await engine.process(unified)
```

**Real WhatsApp conversation** showing the debounce in action:

```
👤 user: hi
👤 user: I'm looking for a place
👤 user: Lisbon, June 12-17, 2 people
[1 second debounce window passes]
[adapter combines all 3 into one message: "hi\nI'm looking for a place\nLisbon, June 12-17, 2 people"]
🤖 Maya: Hi! I'd love to help. Lisbon, June 12-17, 2 guests — 
        let me check availability... 6 properties match. 
        Want them ranked by price or location?
```

Without the debounce, Maya would have replied to "hi" with a generic greeting, then to "I'm looking for a place" asking what kind, then to the third message with the actual search. Three messages where one would do — and a frustrating experience for a user who already typed everything they needed.

The 24-hour window also matters. After Maya's reply, the user has 24 hours to respond freely. If they ghost for 25 hours, Maya can only reach back out via a pre-approved template ("Hi! Just checking in on your Lisbon search — want to pick up where we left off?"). Outside the window, the engine is read-only for that conversation.

## Channel 4: Facebook Messenger

Messenger is similar to WhatsApp in shape (webhook + REST) but with different constraints.

**Transport**: Webhook for incoming, Send API for outgoing.

**Quirks**:
- **24-hour standard messaging window** (same as WhatsApp)
- **Message tags** for sending outside the window (reservation updates, account updates)
- **Quick replies** are a native UI element — buttons for common responses
- **Persistent menu** at the bottom — always available even mid-conversation

```python
class MessengerAdapter(ChannelAdapter):
    async def to_unified(self, raw: dict) -> IncomingMessage:
        return IncomingMessage(
            conversation_id=f"messenger:{raw['sender']['id']}",
            user_id=raw['sender']['id'],
            channel='messenger',
            text=raw.get('message', {}).get('text'),
            attachments=self._extract_attachments(raw),
            timestamp=datetime.fromtimestamp(raw['timestamp'] / 1000),
            metadata={'page_id': raw['recipient']['id']},
        )

    async def from_unified(self, message: OutgoingMessage):
        payload = {
            'recipient': {'id': message.user_id},
            'message': {'text': message.text},
            'messaging_type': 'RESPONSE',
        }
        
        # Add quick replies if the engine emitted suggested options
        if message.suggested_replies:
            payload['message']['quick_replies'] = [
                {'content_type': 'text', 'title': r, 'payload': r}
                for r in message.suggested_replies[:13]
            ]
        
        await self.send_to_messenger_api(payload)
```

**Real Messenger conversation** with quick replies:

```
👤 user: I want to book a place
🤖 Maya: Sure! Where are you headed?
        [Quick replies: Lisbon | Barcelona | Rome | Other]
👤 user: [taps "Lisbon"]
🤖 Maya: Lisbon it is. When?
        [Quick replies: This weekend | Next month | Custom dates]
👤 user: [taps "Custom dates"]
🤖 Maya: Got it. What dates?
👤 user: June 12-17
```

The Gather item collecting `destination` had its `quick_replies` field set to common destinations. The Messenger adapter rendered them as native buttons. The webchat adapter would have ignored that field (webchat doesn't have quick replies as native UI). Same item, channel-appropriate rendering.

## How the Channel Affects the Persona

The [persona](/posts/multi-level-prompt-architecture/) from Day 9 had `channel_adjustments`. Here's how Maya's prompt actually changes per channel:

**Webchat** (default formality, moderate detail):
> Hi! I'd love to help you find a place. Tell me where you're going and when, and I'll narrow it down. If you have any must-haves — sea view, kitchen, walkability — mention those too and I can filter for them.

**WhatsApp** (concise, more emoji):
> Hey! 👋 Where to and when?

**Messenger** (concise, with quick replies):
> Where are you headed?
> [Lisbon] [Barcelona] [Rome] [Other]

**Email** (formal, thorough — if Maya is configured for an email channel later):
> Hello,
>
> Thanks for reaching out about your trip. I'd be happy to help you find a place to stay. Could you share your destination, travel dates, number of guests, and any specific requirements (kitchen, ocean view, walking distance to landmarks, etc.)?
>
> I'll put together a few options based on what you tell me.
>
> Best,
> Maya

Same persona config, four different opening lines. The agenda items (Convey, Gather, etc.) don't change — only the persona's `channel_adjustments` block adjusts how the LLM is instructed to format output.

## Cross-Channel Conversation Continuity

A nice side effect of the unified format: a user can start a conversation in one channel and continue in another, as long as the system can identify them across channels.

```python
async def lookup_user_across_channels(channel_user_id: str, channel: str) -> str | None:
    """Try to find a unified user_id across channels."""
    # Direct hit
    user = await db.users.find_one({f'{channel}_id': channel_user_id})
    if user:
        return user['unified_id']
    
    # Phone-based linking (WhatsApp ↔ webchat phone field)
    if channel == 'whatsapp':
        phone = await whatsapp.lookup_phone(channel_user_id)
        user = await db.users.find_one({'phone': phone})
        if user:
            return user['unified_id']
    
    return None
```

When a user starts a webchat conversation and provides their phone number, the system stores `phone → unified_id`. Later, if the same person sends a WhatsApp message from that phone, the lookup matches. The new WhatsApp message lands in the same `unified_id` and can read all prior outputs from `:agenda:outputs` ([Day 4](/posts/conversation-state-in-redis/)).

Real example: a guest starts a booking on the website ("Lisbon, June 12-17, 2 people, $200/night max"), gets distracted, leaves. Two days later they message the same brand on WhatsApp asking "did I have something pending?" — and Maya, with access to the prior conversation's outputs, can pick up exactly where it left off without making them re-enter anything.

## Live Chat Takeover

Sometimes the AI hits its limits and a human agent needs to step in. Each channel adapter exposes a `pause_ai` and `resume_ai` hook:

```python
async def takeover(conversation_id: str, agent_id: str):
    await redis.hSet(f'conversation:{conversation_id}:state', {
        'mode': 'live_chat',
        'live_agent_id': agent_id,
        'paused_at': datetime.now().isoformat(),
    })
    # Future incoming messages route to the agent's queue, not the AI service
```

The agenda engine checks `state.mode` on every incoming message. If `mode == 'live_chat'`, the message bypasses the AI entirely and lands in the human agent's queue. When the agent hits "release", the AI resumes — and importantly, the [resume-agenda event](/posts/state-machine-event-driven-agenda-transitions/) fires, so the AI gets a contextual re-ask of whatever item was paused.

This is the same paused-state machinery from [Day 8](/posts/qa-item-knowledge-base-mid-agenda/) — Q&A interrupts and live chat interrupts both use the same pause/resume hooks. One mechanism, two use cases.

## The Full Architecture (Series Wrap-Up)

This is the last post in the series. Here's what got built across 12 days:

**Infrastructure** (Days 1-4):
- Day 1: [The Agenda Engine](/posts/agenda-engine-deterministic-ai-conversations/) — hybrid state machine
- Day 2: [Q&A Mode with RAG](/posts/qa-mode-rag-knowledge-chat/) — knowledge-grounded default chat
- Day 3: [Streaming AI Responses to Redis](/posts/streaming-ai-responses-llm-to-redis/) — resilient gRPC + Redis Streams pipe
- Day 4: [Conversation State in Redis](/posts/conversation-state-in-redis/) — hashes, streams, MULTI/EXEC

**The four core item types** (Days 5-8):
- Day 5: [Convey](/posts/convey-item-scripted-dynamic-messages/) — static and dynamic messages
- Day 6: [Gather](/posts/gather-item-structured-data-collection/) — structured data with field state machine
- Day 7: [Debate](/posts/debate-item-ai-follow-up-questions/) — AI-generated follow-ups
- Day 8: [Q&A Item](/posts/qa-item-knowledge-base-mid-agenda/) — pause and resume mid-flow

**Cross-cutting glue** (Days 9-11):
- Day 9: [Multi-Level Prompt Architecture](/posts/multi-level-prompt-architecture/) — Persona → Agenda → Item
- Day 10: [Conditional Logic](/posts/conditional-logic-branching-agenda-items/) — JSON ops and AI eval
- Day 11: [State Machine & Events](/posts/state-machine-event-driven-agenda-transitions/) — 8 events that drive everything

**Channels** (Day 12, this post):
- Webchat, Widget, WhatsApp, Messenger — one engine, four adapters

Every layer was built around the same two principles from Day 1:

1. **AI for language, code for structure**. The LLM handles natural conversation. The state machine, condition evaluator, output validator, and event handlers all run as code with deterministic semantics.

2. **Persistence at every boundary**. Redis Streams for chat events, Redis Hashes for state, Postgres for the durable record of completed agendas. Every state change is auditable, replayable, and recoverable.

The result: 250K+ monthly interactions across 20+ enterprise clients, on a platform that handles ordering, screening, scheduling, and support — with one AI agent per client and a small engineering team.

Thanks for following along.

---

### Related in this series

- [Day 1 — The Agenda Engine](/posts/agenda-engine-deterministic-ai-conversations/) — the foundation
- [Day 4 — Conversation State in Redis](/posts/conversation-state-in-redis/) — where channel-bridged state lives
- [Day 9 — Multi-Level Prompt Architecture](/posts/multi-level-prompt-architecture/) — how channel-aware persona rendering works
- [Day 11 — State Machine & Events](/posts/state-machine-event-driven-agenda-transitions/) — the event log every channel writes to

### References

- [WhatsApp Business Platform docs](https://developers.facebook.com/docs/whatsapp/cloud-api)
- [Messenger Send API](https://developers.facebook.com/docs/messenger-platform/send-messages)
- [Server-Sent Events (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events) — the webchat transport
- [BullMQ Delayed Jobs](https://docs.bullmq.io/guide/jobs/delayed) — for the WhatsApp debounce flush
