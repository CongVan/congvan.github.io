---
title: "Streaming AI Responses: From LLM to Redis Streams"
date: 2026-04-11T10:00:00+07:00
lastmod: 2026-04-11T10:00:00+07:00
draft: false
tags: ["Backend"]
categories: ["Building a Conversational AI Platform"]
series: ["Building a Conversational AI Platform"]
summary: "How a single AI response travels from OpenAI through a Python AI service, into a Node.js backend via gRPC, and ends up in Redis Streams — decoupling LLM generation from client delivery so a dropped connection never kills a response mid-stream."
ShowToc: true
weight: 3
seriesTotal: 12
---

{{< series-nav >}}

*Day 3 of 12. In [Day 1](/posts/agenda-engine-deterministic-ai-conversations/) I built the state machine that makes AI conversations deterministic. In [Day 2](/posts/qa-mode-rag-knowledge-chat/) I built the Q&A mode that answers from a knowledge base. Today I build the **pipe** — how text moves from the LLM to somewhere a client can read.*

---

## Why Streaming, and Why It's Harder Than It Looks

A user sends a message. The response has to feel instant.

Waiting 8 seconds for a full response is a non-starter. ChatGPT set the expectation — [OpenAI's streaming API](https://platform.openai.com/docs/api-reference/streaming) returns tokens as they're generated, and users now expect every AI chat to do the same.

The naive approach is to pipe those tokens straight through the backend to the browser. It works. It's 15 lines of code. And it has a brutal failure mode:

**The browser disconnects halfway through. The whole response is lost.**

The user closes the tab, the laptop goes to sleep, a flaky Wi-Fi drops the TCP connection — and the LLM generation, which is still running, has nowhere to go. The backend catches the client disconnect, cancels the generation, and throws away whatever was already produced. The user refreshes, triggers a new generation, pays the token cost twice, and waits again.

At a handful of requests per day, nobody notices. At 250K+ monthly interactions across 20+ enterprise clients, I was watching dropped connections burn real money.

## The Fix: Decouple Generation From Delivery

Stop piping the LLM directly to the browser. Instead, write every chunk to a buffer the backend owns, and let the client read from that buffer on its own schedule.

If the client drops, the buffer keeps filling. When the client reconnects, it reads whatever accumulated while it was gone and picks up where it left off. The LLM never knows anything changed.

The buffer, in my case, is **[Redis Streams](https://redis.io/docs/latest/develop/data-types/streams/)** — an append-only log data structure with built-in consumer groups and resume semantics. Upstash's [resumable LLM streams article](https://upstash.com/blog/resumable-llm-streams) covers the same pattern. Redis also has an [official tutorial on streaming LLM output](https://redis.io/tutorials/howtos/solutions/streams/streaming-llm-output/).

## The Pipeline

A chunk of text travels through four systems before it's safe:

![Streaming pipeline: LLM → AI Service → Backend → Redis Streams](/images/blog/streaming-pipeline.svg)

Each hop uses a different transport. Each transport is picked for what it's good at.

| Hop | Transport | Why |
|---|---|---|
| LLM → AI Service | OpenAI SDK async iterator | Built into the SDK, native Python `async for` |
| AI Service → Backend | gRPC server streaming | Typed messages, HTTP/2 multiplexing, backpressure |
| Backend → Redis Streams | `XADD` append-only | Persistent, multi-consumer, replayable |

The client consumer (browser, WhatsApp webhook, admin dashboard) reads from Redis separately. That's a future post — this one stops at `XADD`.

## Hop 1: LLM to AI Service

Inside the Python AI service, each call to OpenAI returns an async iterator. The service loops over it. Each iteration yields either a text chunk or a partial tool call.

```python
async def stream_llm_response(messages, tools):
    stream = await client.chat.completions.create(
        model="gpt-4",
        messages=messages,
        tools=tools,
        stream=True,
    )

    tool_buffer = ToolCallBuffer()

    async for chunk in stream:
        delta = chunk.choices[0].delta

        # Text: yield each token immediately
        if delta.content:
            yield ChunkMessageEvent(text=delta.content)

        # Tool calls: arrive in fragments, reassemble first
        if delta.tool_calls:
            tool_buffer.feed(delta.tool_calls)
            if tool_buffer.is_complete():
                yield ToolEvent(
                    name=tool_buffer.name,
                    arguments=tool_buffer.args,
                )
                tool_buffer.reset()
```

Text chunks stream token-by-token. Tool calls are different — they arrive in fragments (name first, then arguments character-by-character as JSON) and only mean something once fully assembled. The `tool_buffer` collects fragments until `is_complete()` returns true, then emits one `ToolEvent`.

This distinction matters because tool calls drive agenda triggers and output extraction (details in [Day 1](/posts/agenda-engine-deterministic-ai-conversations/) and the item posts coming up).

## Hop 2: AI Service to Backend (gRPC)

The AI service doesn't talk to the browser directly. It streams events to the backend over [gRPC server streaming](https://grpc.io/docs/what-is-grpc/core-concepts/#server-streaming-rpc).

The contract is a single protobuf file shared between Python and TypeScript:

```protobuf
service ConversationService {
  rpc Chat(ChatPayload) returns (stream EventResponse) {}
}

message ChatPayload {
  string conversation_id = 1;
  string user_message = 2;
  ConversationState state = 3;
  repeated Message history = 4;
}

message EventResponse {
  string conversation_id = 1;
  int64 timestamp_ms = 2;

  oneof event {
    ChunkMessageEvent chunk = 10;
    ToolEvent tool = 11;
    StartAgendaEvent start_agenda = 12;
    StartAgendaItemEvent start_item = 13;
    EndAgendaItemEvent end_item = 14;
    EndAgendaEvent end_agenda = 15;
    ErrorEvent error = 16;
  }
}
```

The `oneof` keyword is the trick. One stream carries every event kind — text chunks, tool calls, agenda lifecycle, errors — in order. No separate channels for text versus lifecycle. No race conditions between a "chunk" channel and a "state" channel. See the [protobuf `oneof` docs](https://protobuf.dev/programming-guides/proto3/#oneof) for the full spec.

Server streaming also gives me **backpressure for free**: if the backend can't keep up, the AI service's send blocks, which backs up the `async for` loop in Hop 1, which backs up the OpenAI SDK. Everything throttles together.

## Hop 3: Backend to Redis Streams

This is the hop that makes the whole thing resilient.

```typescript
import { createClient } from 'redis'

const redis = createClient()
await redis.connect()

async function forwardToRedis(conversationId: string, payload: ChatPayload) {
  const streamKey = `conversation:${conversationId}:stream`

  // Call the AI service over gRPC
  const events = aiService.chat(payload)  // server-streaming RPC

  for await (const event of events) {
    // Append every event to the Redis stream
    // '*' tells Redis to auto-generate a timestamp-based ID
    await redis.xAdd(streamKey, '*', {
      event: JSON.stringify(event),
    })

    // Notify any waiting consumers that new data is ready
    await redis.publish(`${streamKey}:notify`, '1')
  }

  // Mark end-of-stream so consumers know to stop waiting
  await redis.xAdd(streamKey, '*', {
    event: JSON.stringify({ type: 'done' }),
  })
}
```

Three things worth noting:

**`XADD` with `*` as the ID** — Redis generates a timestamp-based ID for every entry. Entries come out of the stream in the exact order they went in. See the [`XADD` reference](https://redis.io/docs/latest/commands/xadd/).

**The pub/sub notify** — Consumers block on `XREAD BLOCK`, which wakes up when new data arrives. The extra `PUBLISH` lets me also notify pub/sub subscribers who might be doing something cheaper than a blocking read. It's optional but makes multi-consumer fan-out trivial.

**The `done` sentinel** — Redis Streams don't have a native "end of stream" concept. I add an explicit marker so consumers know the generation is finished and they can stop waiting.

## Why Redis Over Piping Direct

![Naive pipe vs Redis pipe: scenario comparison](/images/blog/streaming-redis-resume.svg)

The side-by-side is the whole argument. Naive pipe: client drops, generation is lost. Redis pipe: client drops, generation keeps filling the buffer, client reconnects and reads the backlog.

But resumability is only the headline benefit. The same architecture gives me four more things for free:

**Multi-consumer.** Admin dashboard and user chat window can both read the same stream. No extra code — just a second `XREAD` from a different consumer.

**Decoupling.** LLM generation runs independently of client delivery. A slow consumer doesn't back-pressure the LLM (unlike the naive pipe, where the backend's `res.write()` blocks until the browser acks). I can push to 1000 active conversations without any of them affecting each other.

**Replay.** `XREAD` can start from any stream ID, including `0`. That means a consumer that missed the first 80% of a response can read the whole thing from the beginning.

**Observability.** Every chunk is persisted for a few minutes. I can log into Redis and literally see what the LLM produced for a specific conversation. Debugging is immediate.

## The Event Catalog

Day 3 is about the **pipe**. The payloads — what each event actually means and how each one drives the agenda — are detailed in later posts. Here's the map:

![EventResponse oneof tree](/images/blog/streaming-event-types.svg)

| Event | What it means | Detailed in |
|---|---|---|
| `ChunkMessageEvent` | A token or phrase from the LLM | This post |
| `ToolEvent` | LLM called a function | [Day 1 — Agenda Engine](/posts/agenda-engine-deterministic-ai-conversations/) |
| `StartAgendaEvent` | User confirmed, agenda begins | [Day 1 — Agenda Engine](/posts/agenda-engine-deterministic-ai-conversations/) |
| `StartAgendaItemEvent` | Next item starts executing | Day 5 (Convey), Day 6 (Gather) |
| `EndAgendaItemEvent` | Item done, outputs collected | Day 6 (Gather) |
| `EndAgendaEvent` | All items complete, webhooks fire | Day 11 (State Machine) |
| `ErrorEvent` | Something broke | This post |

Every event carries a `conversation_id` and a `timestamp_ms`. That's enough to reconstruct the full timeline of a conversation from the Redis stream alone.

## Edge Cases I Had to Handle

**Connection drops mid-generation.** Handled by design. Generation keeps running, chunks keep hitting Redis, client reconnects and reads from its last known ID.

```typescript
// Consumer picks up where it left off
const entries = await redis.xRead(
  [{ key: streamKey, id: lastReadId }],
  { BLOCK: 0 },  // block forever until new data
)
```

**Multi-message grouping on messaging channels.** On WhatsApp and Messenger, users sometimes send 3 messages in 500ms. If I spawn an AI call per message, I get 3 racing responses. Instead, the backend debounces incoming messages by conversation ID for ~1 second, groups them into one `ChatPayload`, and sends one response. This lives in the channel adapter layer — more in Day 12.

**Stream cleanup.** Redis Streams cost memory. I cap each stream at 1000 entries with `XTRIM` and set a 1-hour TTL on the key:

```typescript
await redis.xAdd(streamKey, '*', { event: '...' })
await redis.xTrim(streamKey, 'MAXLEN', 1000, { approximateLength: true })
await redis.expire(streamKey, 3600)  // 1 hour
```

`MAXLEN ~ 1000` is approximate trimming — Redis trims in chunks for performance instead of exactly at the threshold. For chat, approximate is fine.

**Errors mid-generation.** If the LLM raises an error halfway through, the AI service catches it and emits an `ErrorEvent` as the last event in the stream before closing the gRPC channel. The backend appends it to the Redis stream and then writes the `done` sentinel. Consumers see the error, show it to the user, and can retry the whole message.

## What's Next: The Consumer Side

This post stops at "chunks are safely in Redis." Everything from Redis onward — SSE endpoint, browser rendering, reconnection logic — is a follow-up post I haven't written yet.

The plan for the consumer side: a `GET /api/chat/stream/:conversationId` endpoint that opens a [consumer group](https://redis.io/docs/latest/commands/xreadgroup/) on the stream, reads unseen entries with `XREADGROUP`, and writes each one to the browser as an SSE `data:` line. Upstash's [resumable LLM streams article](https://upstash.com/blog/resumable-llm-streams) is the reference implementation I'm borrowing from. I'll publish it as an update when the frontend side lands.

## Closing

This is the plumbing for chat **text**. One pipe, four hops, zero wasted tokens on dropped connections.

But text is only half of what a conversation needs. The other half is **state** — which agenda is running, which item is active, what data got collected, whether the conversation is paused or completed. That lives in Redis too, just in different data structures. That's the next post.

Next: **Day 4 — Conversation State in Redis: Hashes, Streams, and a Single Source of Truth**.

---

### Related in this series

- [Day 1 — The Agenda Engine](/posts/agenda-engine-deterministic-ai-conversations/) — the source of every event type streamed through this pipe
- [Day 2 — How Q&A Mode Works: RAG](/posts/qa-mode-rag-knowledge-chat/) — the text being streamed is often RAG-grounded

### References

- [OpenAI Streaming API](https://platform.openai.com/docs/api-reference/streaming)
- [gRPC server streaming concept](https://grpc.io/docs/what-is-grpc/core-concepts/#server-streaming-rpc)
- [Protocol Buffers `oneof`](https://protobuf.dev/programming-guides/proto3/#oneof)
- [Redis Streams data type](https://redis.io/docs/latest/develop/data-types/streams/)
- [Redis `XADD` command](https://redis.io/docs/latest/commands/xadd/)
- [Redis `XREAD` command](https://redis.io/docs/latest/commands/xread/)
- [Upstash: Resumable LLM Streams with Redis](https://upstash.com/blog/resumable-llm-streams)
- [Redis tutorial: Streaming LLM Output](https://redis.io/tutorials/howtos/solutions/streams/streaming-llm-output/)
