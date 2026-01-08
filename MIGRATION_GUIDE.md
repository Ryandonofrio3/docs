# Migration Guide: raindrop-ai to rd-mini

This guide helps you migrate from `raindrop-ai` to `rd-mini`. Most migrations take 15-30 minutes.

## Quick Reference

| raindrop-ai | rd-mini | Notes |
|-------------|---------|-------|
| `writeKey` | `apiKey` | `writeKey` still works |
| `instrumentModules` | `wrap()` | Must wrap clients explicitly |
| `trackAi()` | `begin()`/`finish()` | Or use `wrap()` for auto-tracing |
| `setUserDetails()` | `identify()` | |
| `debugLogs` | `debug` | |
| `initTracing()` | Not needed | |
| `redactPii: true` | PII plugin | See plugins section |

---

## TypeScript Migration

### Step 1: Update Imports

```typescript
// Before
import { initTracing } from "raindrop-ai/tracing";
initTracing();
import { Raindrop } from "raindrop-ai";

// After
import { Raindrop } from "rd-mini";
```

### Step 2: Update Initialization

```typescript
// Before
const raindrop = new Raindrop({
  writeKey: process.env.RAINDROP_API_KEY,
  debugLogs: true,
  redactPii: true,
  instrumentModules: {
    openAI: OpenAI,
    anthropic: AnthropicModule,
  },
});

const openai = new OpenAI();
const anthropic = new Anthropic();

// After
import { createPiiPlugin } from 'rd-mini/plugins/pii';

const raindrop = new Raindrop({
  apiKey: process.env.RAINDROP_API_KEY,  // or writeKey still works
  debug: true,
  plugins: [createPiiPlugin()],  // PII is now a plugin
});

const openai = raindrop.wrap(new OpenAI());         // Explicit wrap
const anthropic = raindrop.wrap(new Anthropic());   // Explicit wrap
```

### Step 3: Update User Identification

```typescript
// Before
raindrop.setUserDetails({
  userId: "user_123",
  traits: { name: "Jane", plan: "pro" },
});

// After
raindrop.identify("user_123", { name: "Jane", plan: "pro" });
```

### Step 4: Update Interactions (Mostly Compatible)

```typescript
// Before - still works!
const interaction = raindrop.begin({
  eventId: "evt_123",
  event: "chat_message",
  userId: "user_123",
  input: message,
  model: "gpt-4o",
  convoId: "convo_123",
});

// ... do work ...

interaction.finish({ output: response });

// After - identical API
const interaction = raindrop.begin({
  eventId: "evt_123",
  event: "chat_message",
  userId: "user_123",
  input: message,
  model: "gpt-4o",
  conversationId: "convo_123",  // Note: convoId also works
});

interaction.finish({ output: response });
```

### Step 5: Update Signals (Compatible)

```typescript
// Before - still works!
await raindrop.trackSignal({
  eventId: "evt_123",
  name: "thumbs_down",
  comment: "Answer was off-topic",
});

// After - identical API
await raindrop.trackSignal({
  eventId: "evt_123",
  name: "thumbs_down",
  comment: "Answer was off-topic",
});
```

### Step 6: Update trackAi (If Used)

If you used `trackAi()` for single-shot events:

```typescript
// Before
raindrop.trackAi({
  event: "user_message",
  userId: "user_123",
  model: "gpt-4o",
  input: "Hello",
  output: "Hi there!",
});

// After - Option 1: Use begin/finish
const interaction = raindrop.begin({
  event: "user_message",
  userId: "user_123",
  model: "gpt-4o",
  input: "Hello",
});
interaction.finish({ output: "Hi there!" });

// After - Option 2: Use withInteraction (cleaner)
await raindrop.withInteraction(
  { event: "user_message", userId: "user_123", model: "gpt-4o", input: "Hello" },
  async () => "Hi there!"  // Return value becomes output
);

// After - Option 3: Just use wrap() and let it auto-trace
const openai = raindrop.wrap(new OpenAI());
const response = await openai.chat.completions.create({
  model: "gpt-4o",
  messages: [{ role: "user", content: "Hello" }],
});
// Automatically traced!
```

### Step 7: Update withSpan/withTool (If Used)

```typescript
// Before
await interaction.withSpan(
  { name: "embedding_generation", properties: { model: "ada" } },
  async () => { /* ... */ }
);

await interaction.withTool(
  { name: "search_tool", inputParameters: { query: "test" } },
  async () => { /* ... */ }
);

// After - withTool is the unified method
await interaction.withTool(
  { name: "embedding_generation", properties: { model: "ada" } },
  async () => { /* ... */ }
);

await interaction.withTool(
  { name: "search_tool", properties: { query: "test" } },
  async () => { /* ... */ }
);

// Or use the standalone wrapTool
const searchTool = raindrop.wrapTool("search_tool", async (query: string) => {
  return await search(query);
});
```

### Step 8: Update OTEL Configuration (If Used)

```typescript
// Before
const raindrop = new Raindrop({
  writeKey: '...',
  disableTracing: true,  // Disable built-in OTEL
});
const processor = raindrop.createSpanProcessor({
  allowedInstrumentationLibraries: [],
});

// After - use OTEL plugin
import { createOtelPlugin } from 'rd-mini/plugins/otel';

const raindrop = new Raindrop({
  apiKey: '...',
  plugins: [
    createOtelPlugin({
      serviceName: 'my-service',
      includeContent: true,
    }),
  ],
});
// That's it - spans are automatically exported to your OTEL provider
```

---

## Python Migration

### Step 1: Update Imports

```python
# Before
import raindrop
raindrop.init(api_key=os.environ["RAINDROP_API_KEY"])

# After
from rd_mini import Raindrop
raindrop = Raindrop(api_key=os.environ["RAINDROP_API_KEY"])
```

### Step 2: Update Client Wrapping

```python
# Before - auto-instrumented via Traceloop
from openai import OpenAI
client = OpenAI()  # Magically traced

# After - explicit wrapping
from openai import OpenAI
from rd_mini import Raindrop

raindrop = Raindrop(api_key=os.environ["RAINDROP_API_KEY"])
client = raindrop.wrap(OpenAI())  # Explicitly traced
```

### Step 3: Update User Identification

```python
# Before
raindrop.identify("user_123", {"name": "Jane", "plan": "pro"})

# After - same API!
raindrop.identify("user_123", {"name": "Jane", "plan": "pro"})
```

### Step 4: Update Interactions

```python
# Before - decorator style
@raindrop.interaction(name="chat", version=1)
def handle_chat(message):
    return generate_response(message)

# After - context manager style
def handle_chat(message):
    with raindrop.interaction(event="chat", input=message) as ctx:
        response = generate_response(message)
        ctx.output = response
        return response

# Or use begin/finish
def handle_chat(message):
    interaction = raindrop.begin(event="chat", input=message)
    try:
        response = generate_response(message)
        interaction.finish(output=response)
        return response
    except Exception as e:
        interaction.finish(error=str(e))
        raise
```

### Step 5: Update Decorators

```python
# Before
@raindrop.task(name="embed")
def embed_text(text):
    return embeddings.create(text)

@raindrop.tool(name="search")
def search_docs(query):
    return vector_store.search(query)

# After - same API, but must use instance
@raindrop.task("embed")
def embed_text(text):
    return embeddings.create(text)

@raindrop.tool("search")
def search_docs(query):
    return vector_store.search(query)
```

### Step 6: Update task_span/tool_span

```python
# Before
with raindrop.task_span("process_data"):
    result = process(data)

with raindrop.tool_span("fetch_context"):
    context = fetch(query)

# After - use start_span with kind parameter
with raindrop.start_span("process_data", kind="task") as span:
    result = process(data)
    span.set_output(result)

with raindrop.start_span("fetch_context", kind="tool") as span:
    context = fetch(query)
    span.set_output(context)
```

### Step 7: Update track_ai (If Used)

```python
# Before
raindrop.track_ai(
    user_id="user_123",
    event="chat",
    model="gpt-4o",
    input="Hello",
    output="Hi there!",
)

# After - use begin/finish
interaction = raindrop.begin(
    user_id="user_123",
    event="chat",
    model="gpt-4o",
    input="Hello",
)
interaction.finish(output="Hi there!")

# Or use context manager
with raindrop.interaction(user_id="user_123", event="chat", input="Hello") as ctx:
    ctx.output = "Hi there!"
```

### Step 8: Update PII Redaction

```python
# Before
raindrop.init(api_key='...', redact_pii=True)

# After - use plugin
from rd_mini import Raindrop
from rd_mini.plugins.pii import create_pii_plugin

raindrop = Raindrop(
    api_key='...',
    plugins=[create_pii_plugin()],
)

# Or with options
raindrop = Raindrop(
    api_key='...',
    plugins=[
        create_pii_plugin(
            patterns=["email", "phone", "ssn"],
            redact_names=True,
        )
    ],
)
```

### Step 9: Update Shutdown

```python
# Before
raindrop.shutdown()

# After
raindrop.close()  # or raindrop.flush() to just flush without closing
```

---

## Common Issues

### "My AI calls aren't being traced"

Make sure you're wrapping your client:
```typescript
// Wrong
const openai = new OpenAI();

// Right
const openai = raindrop.wrap(new OpenAI());
```

### "I'm getting type errors with convoId"

Use `conversationId` (the new name) or `convoId` (still supported):
```typescript
raindrop.begin({
  conversationId: "convo_123",  // Preferred
  // convoId: "convo_123",      // Also works
});
```

### "PII redaction isn't working"

PII is now a plugin:
```typescript
import { createPiiPlugin } from 'rd-mini/plugins/pii';

new Raindrop({
  apiKey: '...',
  plugins: [createPiiPlugin()],
});
```

### "How do I access the trace ID?"

```typescript
// From wrapped client responses
const response = await openai.chat.completions.create({...});
console.log(response._traceId);

// From interactions
const interaction = raindrop.begin({...});
console.log(interaction.id);

// Last trace ID
console.log(raindrop.getLastTraceId());
```

---

## Feature Comparison

| Feature | raindrop-ai | rd-mini | Migration |
|---------|-------------|---------|-----------|
| Track LLM calls | Auto via instrumentModules | `wrap()` | Add `.wrap()` |
| Track interactions | `begin()`/`finish()` | Same | No change |
| Resume interaction | `resumeInteraction()` | Same | No change |
| Track signals | `trackSignal()` | Same | No change |
| Identify users | `setUserDetails()` | `identify()` | Rename method |
| PII redaction | `redactPii: true` | PII plugin | Use plugin |
| OTEL export | Built-in | OTEL plugin | Use plugin |
| Tool tracing | `withTool()` | Same | No change |
| Task tracing | `withSpan()` | `withTool()` or `startSpan()` | Use new method |
| Debug logging | `debugLogs: true` | `debug: true` | Rename option |
| Disable SDK | `disabled: true` | Same | No change |

---

## Need Help?

- Check the [Design Decisions](./DESIGN_DECISIONS.md) doc for rationale
- Review the [Quick Start](./QUICK_START.md) for examples
- Contact us on Slack or email for migration support
