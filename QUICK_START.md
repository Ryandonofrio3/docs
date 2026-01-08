# rd-mini Quick Start

Get AI observability in 5 minutes. No configuration, no complexity.

## Install

```bash
# TypeScript
npm install rd-mini

# Python
pip install rd-mini
```

## The 90% Use Case

Most apps need three things: **trace AI calls**, **group related calls**, and **collect feedback**.

### TypeScript

```typescript
import { Raindrop } from 'rd-mini';
import OpenAI from 'openai';

// 1. Initialize and wrap your client
const raindrop = new Raindrop({ apiKey: process.env.RAINDROP_API_KEY });
const openai = raindrop.wrap(new OpenAI());

// 2. Use your client normally - calls are auto-traced
const response = await openai.chat.completions.create({
  model: 'gpt-4o',
  messages: [{ role: 'user', content: 'Hello!' }],
});

// Access the trace ID
console.log('Trace ID:', response._traceId);

// 3. Collect feedback
await raindrop.feedback(response._traceId, {
  type: 'thumbs_up',
  comment: 'Great response!',
});

// Remember to flush before exit
await raindrop.close();
```

### Python

```python
import os
from rd_mini import Raindrop
from openai import OpenAI

# 1. Initialize and wrap your client
raindrop = Raindrop(api_key=os.environ["RAINDROP_API_KEY"])
openai = raindrop.wrap(OpenAI())

# 2. Use your client normally - calls are auto-traced
response = openai.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello!"}],
)

# Access the trace ID
print(f"Trace ID: {response._trace_id}")

# 3. Collect feedback
raindrop.feedback(response._trace_id, {
    "type": "thumbs_up",
    "comment": "Great response!",
})

# Remember to flush before exit
raindrop.close()
```

**That's it.** Your AI calls are now tracked in Raindrop.

---

## Grouping Related Calls (RAG, Agents, etc.)

When multiple AI calls are part of one user interaction, group them:

### TypeScript

```typescript
// Option 1: withInteraction (recommended)
const result = await raindrop.withInteraction(
  {
    userId: 'user_123',
    event: 'rag_query',
    input: userQuestion,
  },
  async (ctx) => {
    // Step 1: Retrieve context
    const docs = await searchDocs(userQuestion);

    // Step 2: Generate response (auto-traced, linked to this interaction)
    const response = await openai.chat.completions.create({
      model: 'gpt-4o',
      messages: [
        { role: 'system', content: `Context: ${docs.join('\n')}` },
        { role: 'user', content: userQuestion },
      ],
    });

    return response.choices[0].message.content;
  }
);
// result is the return value, output is auto-captured

// Option 2: begin/finish (more control)
const interaction = raindrop.begin({
  userId: 'user_123',
  event: 'rag_query',
  input: userQuestion,
});

try {
  const docs = await searchDocs(userQuestion);
  const response = await openai.chat.completions.create({...});

  interaction.finish({ output: response.choices[0].message.content });
} catch (error) {
  interaction.finish({ error: error.message });
  throw error;
}
```

### Python

```python
# Option 1: Context manager (recommended)
with raindrop.interaction(
    user_id="user_123",
    event="rag_query",
    input=user_question,
) as ctx:
    # Step 1: Retrieve context
    docs = search_docs(user_question)

    # Step 2: Generate response (auto-traced, linked to this interaction)
    response = openai.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": f"Context: {docs}"},
            {"role": "user", "content": user_question},
        ],
    )

    ctx.output = response.choices[0].message.content

# Option 2: begin/finish (more control)
interaction = raindrop.begin(
    user_id="user_123",
    event="rag_query",
    input=user_question,
)

try:
    docs = search_docs(user_question)
    response = openai.chat.completions.create(...)
    interaction.finish(output=response.choices[0].message.content)
except Exception as e:
    interaction.finish(error=str(e))
    raise
```

---

## Tracing Tools

If your agent uses tools (search, calculator, API calls), trace them:

### TypeScript

```typescript
// Option 1: Wrap a function
const searchDocs = raindrop.wrapTool('search_docs', async (query: string) => {
  const results = await vectorStore.search(query);
  return results;
});

// Now searchDocs() is automatically traced
const docs = await searchDocs('what is love');

// Option 2: Inline within interaction
await raindrop.withInteraction({ event: 'agent' }, async (ctx) => {
  // Trace a tool call inline
  const docs = await ctx.withTool({ name: 'search' }, async () => {
    return await vectorStore.search(query);
  });

  // Continue with LLM call...
});
```

### Python

```python
# Option 1: Decorator
@raindrop.tool("search_docs")
def search_docs(query: str) -> list[str]:
    return vector_store.search(query)

# Now search_docs() is automatically traced
docs = search_docs("what is love")

# Option 2: wrap_tool function
def original_search(query: str) -> list[str]:
    return vector_store.search(query)

search_docs = raindrop.wrap_tool("search_docs", original_search)
```

---

## Identifying Users

Associate events with users:

```typescript
// TypeScript
raindrop.identify('user_123', {
  name: 'Jane Doe',
  email: 'jane@example.com',
  plan: 'pro',
});
```

```python
# Python
raindrop.identify("user_123", {
    "name": "Jane Doe",
    "email": "jane@example.com",
    "plan": "pro",
})
```

---

## Signals (Feedback, Edits, Custom)

Track user signals on AI outputs:

```typescript
// TypeScript

// Thumbs up/down
await raindrop.trackSignal({
  eventId: traceId,
  name: 'thumbs_up',
  sentiment: 'POSITIVE',
});

// User edited the output
await raindrop.trackSignal({
  eventId: traceId,
  name: 'edit',
  type: 'edit',
  after: 'The corrected response text',
});

// Custom signal
await raindrop.trackSignal({
  eventId: traceId,
  name: 'shared_response',
  sentiment: 'POSITIVE',
  properties: { platform: 'slack' },
});

// Convenience method for feedback
await raindrop.feedback(traceId, {
  type: 'thumbs_down',
  comment: 'Response was too long',
});
```

```python
# Python

# Thumbs up/down
raindrop.track_signal(
    event_id=trace_id,
    name="thumbs_up",
    sentiment="POSITIVE",
)

# Convenience method
raindrop.feedback(trace_id, {
    "type": "thumbs_down",
    "comment": "Response was too long",
})
```

---

## Attachments

Include rich content in your events:

```typescript
const interaction = raindrop.begin({ event: 'code_review' });

interaction.addAttachments([
  // Code the user submitted
  {
    type: 'code',
    role: 'input',
    language: 'typescript',
    name: 'user_code.ts',
    value: userCode,
  },
  // Generated review
  {
    type: 'text',
    role: 'output',
    name: 'Review',
    value: reviewText,
  },
  // Screenshot
  {
    type: 'image',
    role: 'output',
    value: 'https://example.com/screenshot.png',
  },
]);

interaction.finish();
```

---

## Supported Providers

rd-mini auto-traces these providers when wrapped:

| Provider | TypeScript | Python |
|----------|------------|--------|
| OpenAI | `raindrop.wrap(new OpenAI())` | `raindrop.wrap(OpenAI())` |
| Anthropic | `raindrop.wrap(new Anthropic())` | `raindrop.wrap(Anthropic())` |
| Vercel AI SDK | `raindrop.wrap(openai('gpt-4o'))` | N/A |

For other providers, use `begin()`/`finish()` to manually track calls.

---

## Configuration

```typescript
const raindrop = new Raindrop({
  apiKey: '...',           // Required
  debug: true,             // Log events to console
  disabled: true,          // Disable all tracking (for tests)
  baseUrl: 'https://...',  // Custom API endpoint
  plugins: [...],          // Add plugins (PII, OTEL, custom)
});
```

---

## Plugins (Advanced)

### PII Redaction

```typescript
import { createPiiPlugin } from 'rd-mini/plugins/pii';

const raindrop = new Raindrop({
  apiKey: '...',
  plugins: [
    createPiiPlugin({
      patterns: ['email', 'phone', 'ssn', 'creditCard'],
      redactNames: true,
    }),
  ],
});
```

### OpenTelemetry Export

```typescript
import { createOtelPlugin } from 'rd-mini/plugins/otel';

const raindrop = new Raindrop({
  apiKey: '...',
  plugins: [
    createOtelPlugin({
      serviceName: 'my-ai-service',
    }),
  ],
});
```

---

## Next Steps

- View your traces in the [Raindrop Dashboard](https://app.raindrop.ai)
- Set up [Signals](https://docs.raindrop.ai/platform/signals) for quality monitoring
- Configure [Alerts](https://docs.raindrop.ai/platform/alerts) for anomalies
- Read [Design Decisions](./DESIGN_DECISIONS.md) to understand the architecture
