# ChatKit.js Investigation - Quick Reference

This is a quick summary of the detailed [INVESTIGATION_REPORT.md](./INVESTIGATION_REPORT.md).

## Key Finding

**ChatKit.js is a TypeScript type definitions package + React bindings that wraps a CDN-delivered web component.**

- This repository: Types and React integration (~few hundred lines)
- Actual implementation: Pre-compiled JavaScript from OpenAI's CDN
- Backend: Separate (OpenAI ChatKit SDK or custom implementation)

## Quick Answers to Investigation Questions

### 1. What properties allow customization?

**40+ direct properties** across these categories:

- **Theme** (15+ props): colors, typography, spacing, radius, density
- **Header** (5 props): visibility, title, left/right actions
- **History** (3 props): enable, delete, rename
- **Start Screen** (2 props): greeting, prompts
- **Composer** (10+ props): placeholder, attachments, tools, models
- **Thread Actions** (3 props): feedback, retry, share
- **Disclaimer** (2 props): text, contrast
- **Entities** (3 props): tag search, click, preview
- **Widgets** (1 prop): action handler
- **Locale** (1 prop): 70+ language options
- **Initial State** (1 prop): thread ID

### 2. How does it visualize agentic actions?

**No specialized protocol** - uses composition of:

- **Text Streaming**: Natural language explanation
- **Client Tools**: Browser-executed agent steps
- **Widgets**: Structured visualizations (cards, lists, forms)

Protocol:

1. Backend generates response with text + widgets + tool calls
2. Client tools pause streaming for browser execution
3. Widgets display structured information
4. Text streaming shows thinking process

### 3. How does it render interactive widgets?

**25 widget component types** in declarative JSON:

- **Root types**: Card, ListView, BasicRoot
- **Layout**: Box, Row, Col, Form, Spacer, Divider, Transition (7)
- **Text**: Text, Title, Caption, Markdown (4)
- **Content**: Badge, Icon, Image (3)
- **Interactive**: Button, Input, Textarea, Select, DatePicker, Checkbox, RadioGroup, Label (11)

**Convention**: Hierarchical JSON tree with `type`, optional `id`/`key`, and `children`.

**Action Protocol**: `{ type: string, payload?: object }` → `widgets.onAction(action, widgetItem)`

**Server-rendered**: Backend generates widget JSON, client renders it.

### 4. Thread and message management

**Threads** = conversation containers with unique IDs

**Operations**:

- `setThreadId(id)`: Switch threads
- `sendUserMessage({text, reply, attachments, newThread})`: Send message
- `setComposerValue({text, reply, attachments})`: Draft without sending
- `fetchUpdates()`: Manual sync with server

**Events**:

- `thread.change`: Thread switched
- `thread.load.start/end`: Loading lifecycle
- `response.start/end`: Streaming lifecycle

**Features**:

- Attachments (files/images with upload strategy)
- Reply context (quoting)
- Feedback (thumbs up/down)
- Retry failed messages
- Share content

### 5. What is source annotation and entity tagging?

**Entities** = structured objects (documents, people, business objects) with ID, title, icon, metadata.

**Two uses**:

1. **Source Annotations** (agent → user):
   - Assistant cites entities as sources
   - Appear as inline citations
   - Hover shows preview (custom widget)
   - Click triggers custom action

2. **Entity Tagging** (user → agent):
   - User types `@` to mention entities
   - `onTagSearch(query)` provides autocomplete
   - Selected entity included in message metadata
   - Backend receives full entity context

**Unified system**: Same entity type and preview mechanism for both.

## Architecture Patterns

1. **Web Component First**: `<openai-chatkit>` custom element
2. **Framework Adapters**: React wrapper available
3. **Type-Safe**: Full TypeScript definitions
4. **Declarative Widgets**: Server-generated JSON
5. **Event-Driven**: CustomEvents for lifecycle
6. **Server-Driven UI**: Backend controls content
7. **Progressive Enhancement**: Start simple, add features as needed

## File Structure

```
packages/
├── chatkit/              # Type definitions
│   ├── types/
│   │   ├── index.d.ts   # Main ChatKitOptions (943 lines)
│   │   └── widgets.d.ts # Widget types (562 lines)
│   └── src/index.ts     # Re-exports
├── chatkit-react/        # React bindings
│   └── src/
│       ├── ChatKit.tsx  # React wrapper (90 lines)
│       └── useChatKit.ts # Hook (107 lines)
└── docs/                 # Documentation site
    └── src/content/docs/
        └── guides/       # Feature guides
```

## What ChatKit Provides

✅ Complete UI components
✅ Type-safe interfaces
✅ React integration
✅ Widget rendering engine
✅ Attachment handling UI
✅ Thread management UI
✅ Localization (70+ languages)
✅ Theme system

## What You Must Implement

❌ Backend (use OpenAI SDK or custom)
❌ Entity database (`onTagSearch`)
❌ File storage (upload strategy)
❌ Authentication (token generation)
❌ Widget JSON generation
❌ Thread/message persistence

## Development Commands

```bash
pnpm install          # Install dependencies
pnpm build           # Build all packages
pnpm test            # Run tests (16 tests pass)
pnpm lint            # Lint code
pnpm format          # Format with prettier
pnpm check           # Run all checks
```

## Next Steps

1. **For Basic Chat**: Just configure `api` with token/URL
2. **For Customization**: Add theme, composer, header options
3. **For Entities**: Implement `onTagSearch` and `onRequestPreview`
4. **For Widgets**: Design with Widget Studio, generate JSON
5. **For Agentic AI**: Structure backend responses with text + widgets + tools

---

For detailed information on any topic, see [INVESTIGATION_REPORT.md](./INVESTIGATION_REPORT.md).
