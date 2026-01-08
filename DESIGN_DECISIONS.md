# rd-mini Design Decisions

This document explains the architectural decisions behind rd-mini and why they differ from raindrop-ai.

## Philosophy: 90% of Value, 10% of Complexity

Most customers need three things:
1. **Track AI calls** - What did the model see? What did it output?
2. **Group related calls** - This RAG query involved retrieval + generation
3. **Collect feedback** - User clicked thumbs up/down

Everything else (OTEL export, PII redaction, custom spans) is valuable but not essential. rd-mini delivers the core experience with minimal API surface, then offers advanced features via plugins.

---

## Decision 1: Explicit Wrapping over Auto-Instrumentation

### raindrop-ai approach
```typescript
import { initTracing } from "raindrop-ai/tracing";
initTracing(); // Must call before other imports!

import { Raindrop } from "raindrop-ai";
import OpenAI from "openai";

const raindrop = new Raindrop({
  writeKey: '...',
  instrumentModules: { openAI: OpenAI }
});

const client = new OpenAI(); // Magically instrumented
```

### rd-mini approach
```typescript
import { Raindrop } from "rd-mini";
import OpenAI from "openai";

const raindrop = new Raindrop({ apiKey: '...' });
const client = raindrop.wrap(new OpenAI()); // Explicitly wrapped
```

### Why we changed this

| Problem with auto-instrumentation | How explicit wrapping solves it |
|-----------------------------------|--------------------------------|
| Module import order matters (`initTracing` must be first) | No import order requirements |
| Bundlers (webpack, esbuild) can break monkey-patching | Standard proxy pattern works everywhere |
| Hard to debug ("why isn't this being traced?") | Clear: if it's wrapped, it's traced |
| Next.js requires `serverExternalPackages` config | No special config needed |
| Anthropic required `import * as` namespace import | Normal imports work |

**Trade-off**: Customers must add `.wrap()` to their client initialization. We believe this is worth it for predictability.

---

## Decision 2: Fewer Ways to Do the Same Thing

### raindrop-ai had multiple patterns for tracking AI calls

```typescript
// Pattern 1: Auto-instrumentation (via instrumentModules)
const client = new OpenAI();
await client.chat.completions.create({...}); // Auto-traced

// Pattern 2: Single-shot tracking
raindrop.trackAi({ input, output, model, ... });

// Pattern 3: Interaction flow
const interaction = raindrop.begin({...});
// ... do stuff ...
interaction.finish({ output });

// Pattern 4: Manual spans
await interaction.withSpan({ name: 'task' }, async () => {...});
```

### rd-mini consolidates to two patterns

```typescript
// Pattern 1: Automatic (for LLM calls)
const client = raindrop.wrap(new OpenAI());
await client.chat.completions.create({...}); // Auto-traced

// Pattern 2: Manual (for grouping or custom operations)
await raindrop.withInteraction({ userId, event }, async () => {
  // All wrapped calls inside are grouped
  await client.chat.completions.create({...});
  return output;
});
```

### Why we removed `trackAi()` from server SDK

The raindrop-ai docs say:
> "Heads-up: We recommend migrating to begin() -> finish() for all new code"

We followed this recommendation. Server-side tracking should use either:
- `wrap()` for automatic LLM call tracing
- `begin()`/`finish()` or `withInteraction()` for complex flows

The browser SDK retains `trackAi()` because client-side single-shot events are a valid use case.

---

## Decision 3: Plugins over Built-in Features

### raindrop-ai approach
```typescript
new Raindrop({
  writeKey: '...',
  redactPii: true, // Built-in
  // OTEL is always on via Traceloop
});
```

### rd-mini approach
```typescript
import { createPiiPlugin } from 'rd-mini/plugins/pii';
import { createOtelPlugin } from 'rd-mini/plugins/otel';

new Raindrop({
  apiKey: '...',
  plugins: [
    createPiiPlugin({ patterns: ['email', 'phone'] }),
    createOtelPlugin({ serviceName: 'my-app' }),
  ],
});
```

### Why plugins are better

1. **Smaller bundle size** - Don't pay for features you don't use
2. **Configurable** - PII plugin can specify which patterns to use
3. **Extensible** - Customers can write their own plugins
4. **No hidden dependencies** - OTEL only loads if you use the plugin
5. **Testable** - Plugins can be unit tested in isolation

### Plugin API

```typescript
interface RaindropPlugin {
  name: string;
  onInteractionStart?(ctx: InteractionContext): void;
  onInteractionEnd?(ctx: InteractionContext): void;
  onSpan?(span: SpanData): void;
  onTrace?(trace: TraceData): void;
  flush?(): Promise<void>;
  shutdown?(): Promise<void>;
}
```

---

## Decision 4: Instance-based over Global State

### raindrop-ai Python approach
```python
import raindrop

raindrop.init(api_key='...')  # Global initialization
raindrop.track_ai(...)        # Module-level function
```

### rd-mini Python approach
```python
from rd_mini import Raindrop

raindrop = Raindrop(api_key='...')  # Instance
raindrop.wrap(client)                # Method on instance
```

### Why instance-based is better

1. **Testable** - Create isolated instances for testing
2. **Multiple configurations** - Different API keys for different services
3. **No import side effects** - Importing doesn't change global state
4. **Explicit dependencies** - Functions receive `raindrop` as parameter
5. **Type safety** - IDE knows what methods are available

**Trade-off**: Customers need to pass the instance around or use a singleton pattern. This is standard Python practice.

---

## Decision 5: Backward Compatibility via Aliases

We support legacy config names to ease migration:

```typescript
// Both work
new Raindrop({ apiKey: '...' });      // New style
new Raindrop({ writeKey: '...' });    // Legacy style (deprecated)
```

```python
# Both work
Raindrop(api_key='...')     # New style
Raindrop(write_key='...')   # Legacy style (deprecated)
```

We chose `apiKey`/`api_key` because:
- It's the industry standard (OpenAI, Anthropic, etc.)
- It's clearer than "writeKey" (what's a write key?)

---

## Decision 6: What We Intentionally Removed

| Removed Feature | Replacement | Rationale |
|-----------------|-------------|-----------|
| `instrumentModules` | `wrap()` | Explicit > magic |
| `initTracing()` | Not needed | No initialization step |
| `createSpanProcessor()` | OTEL plugin | Plugin is simpler |
| `withSpan()` | `withTool()` / `startSpan()` | Span vs Tool distinction was confusing |
| `tracer()` | `begin()` with batch ID | Same functionality, clearer API |
| Traceloop dependency | Optional OTEL plugin | Smaller bundle, optional OTEL |
| `disableTracing` config | `disabled: true` | One flag for everything |
| `bufferSize` / `bufferTimeout` | Internal defaults | Fewer config options to understand |

---

## Data Model

The core data model is simple:

```
Event (an AI interaction)
├── input          - What the user/system sent
├── output         - What the model returned
├── model          - Which model was used
├── userId         - Who triggered this
├── conversationId - Groups related events
├── properties     - Custom metadata
├── Spans[]        - Child operations (tools, tasks)
├── Attachments[]  - Rich content (code, images, docs)
└── latencyMs, tokens, error, etc.

Signal (feedback on an event)
├── eventId        - Which event this is about
├── name           - "thumbs_up", "thumbs_down", custom
├── type           - "default", "feedback", "edit"
├── sentiment      - "POSITIVE", "NEGATIVE"
└── comment, after, properties
```

**Key insight**: A Span is just an Event with a parent. An Interaction is just a container that groups Events. We don't need separate concepts - just parent/child relationships.

---

## Summary

rd-mini is opinionated. We believe:

1. **Explicit is better than implicit** - `wrap()` over auto-patching
2. **Simple is better than flexible** - Two patterns, not seven
3. **Optional is better than built-in** - Plugins over monolithic features
4. **Instance is better than global** - Testable, composable code

These opinions are informed by real customer pain points with raindrop-ai: bundler issues, import order bugs, debugging difficulty, and conceptual overhead.
