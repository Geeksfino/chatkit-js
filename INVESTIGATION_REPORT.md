# ChatKit.js Deep Dive Investigation Report

## Executive Summary

ChatKit.js is a framework-agnostic, drop-in chat component library that provides a complete, production-ready chat interface for AI-powered applications. The project consists of TypeScript type definitions and React bindings that wrap a web component (`<openai-chatkit>`). Despite having "not much code" in this repository, it achieves its extensive feature set through a clever architecture: the actual implementation resides in a bundled JavaScript file loaded from a CDN, while this repository provides the type-safe interfaces and React integration layer.

**Key Architecture Insight**: This repository is essentially a TypeScript type definitions package plus React bindings. The actual ChatKit implementation is delivered as a pre-compiled web component from `https://cdn.platform.openai.com/deployments/chatkit/chatkit.js`.

---

## 1. Deep UI Customization Properties

ChatKit achieves deep customization through a comprehensive `ChatKitOptions` interface. Here are **all** the customization properties:

### 1.1 Theme & Visual Appearance (`theme`)

**Color Scheme:**

- `colorScheme`: `'light' | 'dark'`

**Typography:**

- `typography.baseSize`: `14 | 15 | 16 | 17 | 18` (base font size in pixels)
- `typography.fontSources`: Array of `FontObject` for custom web fonts
  - `family`: CSS font-family name
  - `src`: URL of font file
  - `weight`: Font weight (string or number)
  - `style`: `'normal' | 'italic' | 'oblique'`
  - `display`: `'auto' | 'block' | 'swap' | 'fallback' | 'optional'`
  - `unicodeRange`: Unicode range descriptor
- `typography.fontFamily`: Primary font family
- `typography.fontFamilyMono`: Monospace font family

**Spacing & Layout:**

- `radius`: `'pill' | 'round' | 'soft' | 'sharp'` - controls corner roundness
- `density`: `'compact' | 'normal' | 'spacious'` - controls overall spacing

**Color Customization:**

- `color.grayscale`:
  - `hue`: 0-360 (hue in degrees)
  - `tint`: `0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9`
  - `shade`: `-4 | -3 | -2 | -1 | 0 | 1 | 2 | 3 | 4`
- `color.accent`:
  - `primary`: Color string (hex, rgb, etc.)
  - `level`: `0 | 1 | 2 | 3` - accent intensity level
- `color.surface`:
  - `background`: Surface background color
  - `foreground`: Surface foreground color

### 1.2 Header Configuration (`header`)

- `header.enabled`: `boolean` - show/hide entire header
- `header.title.enabled`: `boolean` - show/hide title area
- `header.title.text`: `string` - static title text (overrides dynamic thread title)
- `header.leftAction`: Custom button on left side
  - `icon`: One of 50+ `HeaderIcon` types (e.g., 'settings-cog', 'home', 'sidebar-left')
  - `onClick`: Callback function
- `header.rightAction`: Custom button on right side
  - Same properties as `leftAction`

### 1.3 History Panel (`history`)

- `history.enabled`: `boolean` - enable/disable history panel
- `history.showDelete`: `boolean` - show delete button for threads
- `history.showRename`: `boolean` - show rename button for threads

### 1.4 Start Screen (`startScreen`)

- `startScreen.greeting`: `string` - greeting text in new thread view (default: "What can I help with today?")
- `startScreen.prompts`: Array of starter prompts
  - `label`: Display text
  - `prompt`: Text inserted into composer
  - `icon`: Optional icon from 30+ available icons

### 1.5 Composer Configuration (`composer`)

- `composer.placeholder`: `string` - placeholder text (default: "Message the AI")
- `composer.attachments`:
  - `enabled`: `boolean` - enable file attachments
  - `maxSize`: `number` - max file size in bytes (default: 100MB)
  - `maxCount`: `number` - max attachments per message (default: 10)
  - `accept`: `Record<string, string[]>` - MIME types and extensions (e.g., `{"application/pdf": [".pdf"]}`)
- `composer.tools`: Array of `ToolOption`
  - `id`: Unique identifier
  - `label`: Display label
  - `shortLabel`: Optional compact label when selected
  - `icon`: Icon from available set
  - `placeholderOverride`: Override composer placeholder when tool selected
  - `pinned`: `boolean` - pin tool outside menu
- `composer.models`: Array of `ModelOption`
  - `id`: Model identifier
  - `label`: Display label
  - `description`: Optional helper text
  - `disabled`: `boolean` - visible but not selectable
  - `default`: `boolean` - default selection

### 1.6 Thread Item Actions (`threadItemActions`)

- `threadItemActions.feedback`: `boolean` - show thumbs up/down buttons
- `threadItemActions.retry`: `boolean` - show retry button
- `threadItemActions.share`: `boolean` - show share button

### 1.7 Disclaimer (`disclaimer`)

- `disclaimer.text`: `string` - markdown text below composer
- `disclaimer.highContrast`: `boolean` - increase text contrast

### 1.8 Localization (`locale`)

- `locale`: One of 70+ supported locales (e.g., 'en', 'es-ES', 'zh-CN', 'ja-JP')
- Falls back to browser locale if not specified

### 1.9 Initial State

- `initialThread`: `string | null` - initial thread ID to load (null for new thread)

**Total Count**: Over **40+ direct customization properties** with hundreds of possible combinations when accounting for nested options, icon choices, and locale selections.

---

## 2. Visualizing Agentic Actions

### 2.1 How It Works

ChatKit visualizes agentic actions through **two primary mechanisms**:

#### A. Client Tools (`onClientTool`)

Client tools allow the backend agent to delegate work to the browser:

**Protocol:**

1. Backend agent calls a client tool during response generation
2. ChatKit receives the tool call and pauses the response
3. ChatKit invokes `onClientTool({ name, params })` handler
4. Handler executes in browser and returns a JSON-serializable result
5. ChatKit forwards the result back to the backend
6. Agent continues processing with the client tool result

**Expected Format:**

```typescript
onClientTool?: (toolCall: {
  name: string;  // Tool identifier
  params: Record<string, unknown>;  // Tool parameters
}) => Promise<Record<string, unknown>> | Record<string, unknown>;
```

**Use Cases:**

- Accessing browser-only APIs (geolocation, local storage)
- Updating UI state in sync with server changes
- Triggering client-side actions (opening tabs, sending emails)

**Error Handling:**

- Throwing an error in the handler fails the tool call
- Error message is sent to the agent
- Response stream halts on tool failure

#### B. Widget Actions (`widgets.onAction`)

Widgets can trigger custom actions that represent agentic steps:

**Protocol:**

1. Server sends widget (Card or ListView) in response
2. Widget contains interactive elements (buttons, forms) with `ActionConfig`
3. User interacts with widget element
4. ChatKit invokes `widgets.onAction(action, widgetItem)`
5. Handler processes action (client-side state updates, server API calls)
6. Optionally use `sendCustomAction()` to send action back to server

**Expected Format:**

```typescript
widgets?: {
  onAction?: (
    action: { type: string; payload?: Record<string, unknown> },
    widgetItem: { id: string; widget: Card | ListView }
  ) => Promise<void>;
}
```

**Workflow Visualization:**

- Each action can represent a step in an agentic workflow
- Widget status can show current step: `status: { text: string; icon?: string }`
- Actions can update widget state to show progress
- Multiple widgets can show chain-of-thought reasoning steps

### 2.2 No Built-in Protocol for Agent Updates

**Important Finding**: ChatKit does **not** have a built-in protocol for streaming agentic updates or chain-of-thought reasoning. Instead:

- The backend (via OpenAI's ChatKit SDK or custom implementation) controls what gets streamed
- Widgets are the primary visualization mechanism for multi-step processes
- The streaming occurs at the **message/response level**, not at a granular "agent step" level
- Developers must structure their agent outputs as:
  1. Text content (streamed naturally)
  2. Widget items (for interactive visualizations)
  3. Tool calls (for client-side actions)

### 2.3 Agent Action Visualization Strategy

Based on the architecture, here's how agentic actions are visualized:

1. **Text Streaming**: Natural language explanations of agent thinking
2. **Tool Indicators**: Client tool invocations pause the stream visibly
3. **Widget Cards**: Display structured information about agent actions
   - Card with status: "Searching database..."
   - ListView showing search results
   - Form widgets for user input in the workflow
4. **Transitions**: Widgets support transition animations for state changes
5. **Status Badges**: Badges in widgets show step completion states

**Key Insight**: The visualization happens through **composition** of these primitives rather than a specialized "agent step" protocol.

---

## 3. Interactive Widget Rendering

### 3.1 Widget Type System

ChatKit expects widgets to follow a **strongly-typed, declarative JSON structure**. There are **three root widget types**:

1. **`Card`**: A container for rich content
2. **`ListView`**: A list of items with optional interactions
3. **`BasicRoot`**: A simple container with flexible layout

### 3.2 Widget Component Library

ChatKit provides **25 widget component types**:

#### Layout Components (7)

- `Box`: Flexible container with direction (row/col)
- `Row`: Horizontal layout container
- `Col`: Vertical layout container
- `Form`: Form container with submit action
- `Spacer`: Empty space with min size
- `Divider`: Visual separator with customizable appearance
- `Transition`: Animated state transition wrapper

#### Text Components (4)

- `Text`: Basic text with extensive styling options
  - Supports: size, weight, color, alignment, truncation, line limits
  - Can be editable with form integration
  - Supports streaming for real-time updates
- `Title`: Heading text (5 sizes: sm to 5xl)
- `Caption`: Small descriptive text (3 sizes)
- `Markdown`: Formatted markdown content with streaming support

#### Content Components (3)

- `Badge`: Labeled badge with color variants
  - Colors: secondary, success, danger, warning, info, discovery
  - Variants: solid, soft, outline
  - Sizes: sm, md, lg
  - Can be pill-shaped
- `Icon`: Icon from 60+ available widget icons
- `Image`: Image with fit modes, positioning, and optional frame

#### Interactive Components (11)

- `Button`: Clickable button
  - Styles: primary, secondary
  - Colors: 8 options (primary, danger, success, etc.)
  - Variants: solid, soft, outline, ghost
  - Sizes: 3xs to 3xl
  - Can have icons, be pill-shaped, block-level, disabled
  - Triggers `ActionConfig` on click
- `Input`: Text input field
  - Types: text, number, email, password, tel, url
  - Form integration with validation
  - Supports autofocus, autoselect, patterns
- `Textarea`: Multi-line text input
  - Auto-resize option
  - Max rows configuration
- `Select`: Dropdown selection
  - Options with label/value pairs
  - Can trigger actions on change
  - Clearable option
- `DatePicker`: Date selection control
  - ISO datetime format
  - Min/max date constraints
  - Position and alignment options
- `Checkbox`: Boolean checkbox
  - Form integration
  - Label and validation support
- `RadioGroup`: Multiple choice selection
  - Row or column layout
  - Option-level disable support
- `Label`: Form field label
  - Links to form field by name

### 3.3 Widget Convention and Protocol

#### Structure Convention

Widgets follow a **hierarchical component tree**:

```typescript
{
  type: 'Card',  // Root type
  id?: string,   // Optional identifier
  key?: string,  // React-like reconciliation key
  children: [    // Child components
    { type: 'Title', value: 'Hello' },
    { type: 'Button', label: 'Click', onClickAction: {...} }
  ],
  // Type-specific props...
}
```

#### Action Protocol

Interactive widgets use `ActionConfig`:

```typescript
type ActionConfig = {
  type: string; // Custom action type
  payload?: Record<string, unknown>; // Optional data
};
```

**Action Flow:**

1. Widget definition includes action: `{ type: 'refresh-dashboard', payload: { page: 1 } }`
2. User interacts (clicks button, changes select, etc.)
3. ChatKit calls `widgets.onAction(action, widgetItem)`
4. Client handles action (update state, call API)
5. Optionally forward to server via `sendCustomAction()`

#### Form Protocol

Forms collect structured input:

```typescript
{
  type: 'Form',
  onSubmitAction: { type: 'submit-form', payload: {...} },
  children: [
    { type: 'Input', name: 'email', required: true },
    { type: 'Button', submit: true, label: 'Submit' }
  ]
}
```

Form data is collected by field `name` and included in the submit action.

#### Theming Support

Widgets support dual themes:

- `theme?: 'light' | 'dark'` - Override ChatKit's theme per widget
- `ThemeColor`: Colors can be theme-aware: `{ dark: '#fff', light: '#000' }`

#### State Management

Widget state is managed through:

- **Collapse states**: Cards can be collapsed
- **Status indicators**: Show loading/processing states
- **Transitions**: Smooth animations between states
- **Streaming**: Text/Markdown can stream content updates
- **Default values**: Form controls have default values

### 3.4 Widget Design Tool

ChatKit provides **Widget Studio** (https://widgets.chatkit.studio/):

- Visual editor for designing widgets
- Live preview
- Export to JSON
- Copy generated code directly into backend

**Key Finding**: Widgets are **server-rendered** (backend generates JSON), not client-rendered. The client only interprets and displays the widget structure.

---

## 4. Thread and Message Management

### 4.1 Thread Concept

**Threads** are conversation containers:

- Each thread has a unique ID (string)
- Thread can be new (ID = null) or existing (ID = string)
- Threads have titles (auto-generated or user-renamed)
- History panel shows all threads
- Threads persist across sessions (server-managed)

### 4.2 Thread Operations

#### Imperative Methods

```typescript
interface OpenAIChatKit {
  // Switch threads
  setThreadId(threadId: string | null): Promise<void>;

  // Send message in current thread
  sendUserMessage(params: {
    text: string;
    reply?: string; // Quote for context
    attachments?: Attachment[];
    newThread?: boolean; // Force new thread
  }): Promise<void>;

  // Set composer without sending
  setComposerValue(params: {
    text: string;
    reply?: string;
    attachments?: Attachment[];
  }): Promise<void>;

  // Manually sync with server
  fetchUpdates(): Promise<void>;
}
```

#### Thread Events

```typescript
type ChatKitEvents = {
  // Thread selection changed
  'chatkit.thread.change': CustomEvent<{ threadId: string | null }>;

  // Thread loading lifecycle
  'chatkit.thread.load.start': CustomEvent<{ threadId: string }>;
  'chatkit.thread.load.end': CustomEvent<{ threadId: string }>;
};
```

### 4.3 Message Structure

Messages are called "thread items" and include:

1. **User Messages**:
   - Text content
   - Optional reply/quote context
   - Attachments (files/images)
   - Entity tags (mentions)

2. **Assistant Messages**:
   - Text content (streamed)
   - Widgets (interactive cards/lists)
   - Source annotations
   - Entity references

3. **System/Tool Messages**: (Implied, not directly exposed to client)

### 4.4 Message Management Features

#### Thread Item Actions

```typescript
threadItemActions?: {
  feedback?: boolean;  // Thumbs up/down on responses
  retry?: boolean;     // Retry last user message
  share?: boolean;     // Share content
}
```

**Feedback Flow:**

- User clicks thumbs up/down
- Feedback sent to server automatically
- Server can log/process feedback

**Retry Flow:**

- User clicks retry
- Server receives retry event
- Server removes failed items and restarts generation

**Share Flow:**

- User clicks share
- `message.share` event emitted (not in public types, likely internal)
- Event contains shared content, item IDs, thread ID

#### Attachment Handling

**Upload Process** (for custom backends):

1. User selects files in composer
2. Files uploaded via `uploadStrategy`:
   - `two_phase`: Upload then attach
   - `direct`: Upload to specified URL
3. Server returns attachment IDs
4. Attachments included in message submission

**Attachment Types:**

```typescript
type Attachment =
  | {
      type: 'file';
      id: string;
      name: string;
      mime_type: string;
    }
  | {
      type: 'image';
      id: string;
      preview_url: string; // For thumbnail
      name: string;
      mime_type: string;
    };
```

#### Reply Context

Messages can include quoted context:

- `reply` parameter in `sendUserMessage` and `setComposerValue`
- Displays quoted text above new message
- Helps maintain conversation context

### 4.5 Thread State Management

**Initialization:**

```typescript
{
  initialThread: 'thread_123' | null;
}
```

- `null`: Start with new thread view
- `string`: Load existing thread

**Persistence:**

- Listen to `chatkit.thread.change` event
- Store `threadId` in app state/storage
- Restore on next mount via `initialThread`

**Synchronization:**

- ChatKit polls server for updates (automatic)
- Manual sync: `fetchUpdates()` after external changes
- Real-time updates via server events (WebSocket/SSE implied)

### 4.6 History Management

```typescript
history?: {
  enabled?: boolean;      // Show/hide history panel
  showDelete?: boolean;   // Allow deleting threads
  showRename?: boolean;   // Allow renaming threads
}
```

**Features:**

- Sidebar/panel listing all threads
- Thread titles (auto-generated or custom)
- Delete threads (with confirmation)
- Rename threads (inline editing)
- Search/filter threads (implied by UI, not in API)

### 4.7 Response Streaming

```typescript
type ChatKitEvents = {
  'chatkit.response.start': CustomEvent<void>; // Agent starts responding
  'chatkit.response.end': CustomEvent<void>; // Agent finishes
};
```

**Streaming Behavior:**

- Text appears incrementally
- Widgets appear when complete
- Tool calls pause the stream
- Streaming state prevents certain operations

**Client State Management:**

```typescript
onResponseStart: () => setIsResponding(true),
onResponseEnd: () => setIsResponding(false)
```

Use `isResponding` to disable UI elements during generation.

---

## 5. Source Annotations and Entity Tagging

### 5.1 Entity Concept

**Entities** are structured, referenceable objects that can appear in conversations:

```typescript
type Entity = {
  title: string; // Display name: "Harry Potter", "Claim #A-1023"
  id: string; // Unique identifier
  icon?: string; // Optional icon URL
  interactive?: boolean; // Can be clicked/previewed
  group?: string; // Grouping: "People", "Documents"
  data?: Record<string, string>; // Arbitrary metadata
};
```

**Examples of Entities:**

- **Documents**: PDFs, articles, wiki pages
- **People**: Users, customers, team members
- **Business Objects**: Tickets, orders, claims
- **Locations**: Addresses, places
- **Dates/Events**: Calendar entries

### 5.2 Source Annotations

Source annotations are entities used as **citations** in assistant responses.

#### How It Works

1. **Backend Generates Response** with entity references:
   - Assistant text includes entities as sources
   - Entities sent as structured data alongside text

2. **ChatKit Renders Citations**:
   - Source entities appear as inline annotations
   - Hovering shows preview popover
   - Clicking triggers `entities.onClick`

3. **Preview Generation**:

```typescript
entities?: {
  onRequestPreview?: (entity: Entity) => Promise<{
    preview: Widgets.BasicRoot | null
  }>
}
```

**Preview Widget Example:**

```typescript
onRequestPreview: async (entity) => ({
  preview: {
    type: 'Card',
    children: [
      { type: 'Title', value: entity.title },
      { type: 'Text', value: 'Document summary here...' },
      { type: 'Button', label: 'Open', onClickAction: {...} }
    ]
  }
})
```

#### Use Cases for Source Annotations

- **RAG Systems**: Show which documents informed the answer
- **Data Queries**: Link to source tables/records
- **Multi-Source Synthesis**: Display all contributing sources
- **Transparency**: Let users verify information origin
- **Navigation**: Jump to original source with one click

### 5.3 Entity Tagging (@-mentions)

Entity tagging allows users to **mention entities** in their messages using @-syntax.

#### Tag Search Configuration

```typescript
entities?: {
  onTagSearch?: (query: string) => Promise<Entity[]>;
  onClick?: (entity: Entity) => void;
  onRequestPreview?: (entity: Entity) => Promise<{...}>;
}
```

#### Tag Flow

1. **User Types `@`** in composer
2. **Autocomplete Triggers**: `onTagSearch('')` called
3. **User Types Query**: `onTagSearch('query')` called for each keystroke
4. **Results Displayed**: Grouped by `entity.group`
5. **User Selects Entity**: Tag inserted into message
6. **Message Sent**: Entity metadata included in submission

#### Tagged Entity Rendering

**In User Messages:**

- Tags appear as highlighted mentions
- Clicking tag triggers `entities.onClick(entity)`
- Preview on hover via `onRequestPreview`

**In Assistant Messages:**

- Same rendering as user messages
- Source annotations use same entity type
- Consistent interaction model

#### Entity Data Flow

```
User types @query
    ↓
onTagSearch(query)
    ↓
Return Entity[]
    ↓
User selects entity
    ↓
Tag rendered in composer
    ↓
Message sent with entity.data
    ↓
Server receives full entity context
```

**Server-Side Use:**

- Entity IDs and metadata sent to backend
- Server can retrieve full entity details
- Can influence agent behavior (e.g., search specific user's documents)

### 5.4 Entity Interaction Patterns

#### Preview Popover

- Appears on hover over entity
- Uses Widget system for rendering
- Can show rich information (text, images, buttons)
- Cached to avoid repeated calls

#### Click Actions

```typescript
entities?: {
  onClick: (entity: Entity) => void
}
```

- Navigate to entity page
- Open entity in modal
- Copy entity reference
- Perform custom action

#### Interactive vs. Non-Interactive

```typescript
interactive?: boolean
```

- `true`: Entity is clickable and shows preview
- `false` / `undefined`: Entity is display-only
- Useful for distinguishing actionable vs. informational entities

### 5.5 Entity Metadata

```typescript
data?: Record<string, string>
```

**Purposes:**

1. **Server Communication**: Metadata sent back to server in message
2. **Client Actions**: Used by `onClick` and `onRequestPreview` handlers
3. **Contextual Information**: Store URLs, IDs, types without separate lookup

**Example:**

```typescript
{
  id: 'doc_123',
  title: 'Q4 Report',
  data: {
    url: 'https://example.com/docs/123',
    type: 'pdf',
    author: 'jane@example.com',
    lastModified: '2024-01-15'
  }
}
```

### 5.6 Combining Source Annotations and Tags

**Unified Entity System:**

- Same `Entity` type for both sources and tags
- Same preview system for both contexts
- Consistent interaction model

**Workflow Example:**

1. User tags `@document-123` in message
2. Agent searches document and cites it as source
3. Response shows document as both:
   - Tag in quoted user message
   - Source annotation in assistant response
4. Both instances link to same entity
5. Consistent preview/interaction experience

### 5.7 Implementation Notes

**Custom Backends Only:**
Per the docs: "Entities are supported only in custom integrations. If you are using ChatKit.js with a hosted backend, entity features are not available."

**Backend Responsibilities:**

- Maintain entity database
- Implement search (for tag autocomplete)
- Generate source annotations in responses
- Handle entity clicks/previews (custom backends)

**Client Responsibilities:**

- Implement `onTagSearch` (query entity DB)
- Implement `onClick` (handle entity activation)
- Implement `onRequestPreview` (generate preview widgets)
- Pass entity data to server in messages

---

## 6. Architecture Summary

### 6.1 Component Hierarchy

```
Repository Structure:
├── packages/chatkit/         # Type definitions
│   ├── types/
│   │   ├── index.d.ts       # Main ChatKitOptions interface
│   │   ├── widgets.d.ts     # Widget type system (25 components)
│   │   └── dom-augment.d.ts # Web component registration
│   └── src/index.ts          # Re-exports types
├── packages/chatkit-react/   # React bindings
│   ├── src/
│   │   ├── ChatKit.tsx      # React wrapper component
│   │   ├── useChatKit.ts    # React hook
│   │   └── useStableOptions.ts # Memoization helper
│   └── package.json
└── packages/docs/            # Documentation site
    └── src/content/docs/
        ├── guides/           # Feature guides
        └── index.mdx         # Main docs
```

### 6.2 Design Patterns

1. **Web Component First**: Core is a custom element (`<openai-chatkit>`)
2. **Framework Adapters**: React bindings wrap the web component
3. **Type-Safe Configuration**: Strong TypeScript typing for all options
4. **Declarative Widgets**: JSON-based widget definitions
5. **Event-Driven**: CustomEvents for lifecycle hooks
6. **Imperative Methods**: Direct control for advanced use cases
7. **Server-Driven UI**: Backend controls what widgets appear
8. **Progressive Enhancement**: Start simple, add features as needed

### 6.3 Integration Points

**With Backend:**

- Client token / custom API authentication
- Message submission and streaming
- Widget JSON generation
- Entity search and metadata
- Tool invocations (client and server)
- File upload handling

**With Host Application:**

- Options configuration
- Event listeners
- Imperative method calls
- Custom styling (CSS)
- Entity providers
- Action handlers

### 6.4 Limitations and Constraints

**What ChatKit Doesn't Provide:**

- Backend implementation (use OpenAI ChatKit SDK or custom)
- Entity database (implement `onTagSearch`)
- File storage (implement upload strategy)
- Authentication system (implement token generation)
- Widget JSON generation (use Widget Studio or code)
- Database for threads/messages (server-managed)

**What ChatKit Does Provide:**

- Complete UI components
- Type-safe interfaces
- React integration
- Widget rendering engine
- Attachment handling UI
- Thread management UI
- Localization (70+ languages)
- Theme system
- Accessibility features (implied by semantic HTML)

---

## 7. Key Findings and Insights

### 7.1 Architecture Insights

1. **Thin Client, Rich Types**: This repo is mostly TypeScript definitions. The actual implementation is closed-source and delivered via CDN.

2. **Server-Driven Philosophy**: The backend has significant control over what appears in the UI (widgets, tools, entities), making the client a rendering engine.

3. **Framework Agnostic**: Web component base allows use in any framework (React, Vue, Svelte, vanilla JS).

4. **Opt-In Complexity**: Start with minimal config (just `api`), add features incrementally.

### 7.2 Agentic Actions Without a Protocol

**Critical Finding**: ChatKit does NOT have a specialized protocol for "agentic actions" or "chain-of-thought reasoning." Instead:

- Text streaming shows agent thinking
- Client tools represent agent steps that need client execution
- Widgets visualize agent outputs and intermediate states
- Developers must structure their agent to output appropriate combinations

This is a **design choice** rather than a limitation—it keeps ChatKit general-purpose rather than tightly coupled to one agent architecture.

### 7.3 Widget System as Universal UI

The widget system is remarkably comprehensive:

- 25 component types
- Declarative JSON format
- Form handling built-in
- Action system for interactivity
- Theming support
- Streaming text updates
- Visual design tool (Widget Studio)

This makes widgets suitable for:

- Data visualization
- Form collection
- Multi-step workflows
- Progress indicators
- Rich previews
- Interactive documentation

### 7.4 Entity System Design

Entities are a **unified abstraction** for:

- Source citations (agent → user)
- @-mentions (user → agent)
- Clickable references
- Contextual previews

This unification provides:

- Consistent UX for related concepts
- Shared implementation
- Metadata flow in both directions
- Rich interactivity

### 7.5 Customization Depth

With 40+ direct properties and nested options, ChatKit can be tailored to nearly any design system:

- Full color control (grayscale, accent, surfaces)
- Typography system (fonts, sizes)
- Spacing system (density, radius)
- Component visibility toggles
- Locale support
- Icon customization
- Layout options

Yet it maintains simplicity—most apps need only 5-10 options.

---

## 8. Answers to Specific Questions

### Q1: What are the properties to allow customization? List all of them.

**Answer**: See Section 1 for the complete list. Summary:

- **Theme**: 15+ properties (color scheme, typography, colors, spacing, radius, density)
- **Header**: 5 properties (enable, title, left/right actions)
- **History**: 3 properties (enable, delete, rename)
- **Start Screen**: 2 properties (greeting, prompts array)
- **Composer**: 10+ properties (placeholder, attachments, tools, models)
- **Thread Actions**: 3 properties (feedback, retry, share)
- **Disclaimer**: 2 properties (text, high contrast)
- **Entities**: 3 properties (tag search, click, preview)
- **Widgets**: 1 property (action handler)
- **Locale**: 1 property (70+ options)
- **Initial State**: 1 property (initial thread)

**Total**: 40+ direct properties with hundreds of nested options.

### Q2: How does it visualize agentic actions? Explain if there is a protocol for it to interpret, how does it expect updates from an agent.

**Answer**: See Section 2. Key points:

- **No specialized protocol** for agentic actions
- Uses **composition** of: text streaming + client tools + widgets
- **Client tools** represent agent steps requiring browser execution
- **Widgets** visualize agent outputs (cards, lists, forms)
- **Protocol**: Standard message streaming with embedded widgets and tool calls
- **Updates**: Via server events (not directly exposed in client API)
- Backend controls visualization by structuring response appropriately

### Q3: How does it render interactive widgets? Does it expect a certain type or convention of widgets?

**Answer**: See Section 3. Key points:

- **25 component types** in declarative JSON format
- **Three root types**: Card, ListView, BasicRoot
- **Convention**: Hierarchical tree structure with `type`, optional `id`/`key`, and `children`
- **Action protocol**: `ActionConfig` with `type` and optional `payload`
- **Form protocol**: Named inputs collected on submit
- **Server-rendered**: Backend generates widget JSON, client renders
- **Widget Studio**: Visual design tool to generate JSON
- **Strong typing**: Full TypeScript definitions for all widget types

### Q4: Drill down to thread and message management.

**Answer**: See Section 4. Key points:

- **Threads**: Conversation containers with unique IDs
- **Operations**: `setThreadId()`, `sendUserMessage()`, `setComposerValue()`, `fetchUpdates()`
- **Events**: Thread change, load start/end, response start/end
- **Messages**: User messages (text + attachments + tags), Assistant messages (text + widgets + sources)
- **Features**: Feedback, retry, share, reply context, attachments
- **State**: Initialize via `initialThread`, persist via `thread.change` event
- **History**: Panel with delete/rename options
- **Streaming**: Real-time text updates with start/end events

### Q5: What is source annotation and entity tagging?

**Answer**: See Section 5. Key points:

- **Entities**: Structured objects (documents, people, business objects) with ID, title, metadata
- **Source Annotations**: Entities cited by agent as information sources
  - Appear as inline citations in responses
  - Hover shows preview popover
  - Click triggers custom action
- **Entity Tagging**: @-mentions in user messages
  - Type `@` to trigger autocomplete
  - `onTagSearch` provides suggestions
  - Selected entity included in message metadata
- **Unified System**: Same entity type for sources and tags
- **Previews**: Custom widgets via `onRequestPreview`
- **Actions**: Custom behavior via `onClick`
- **Custom Backends Only**: Not available with hosted OpenAI backend

---

## 9. Recommendations

### For Implementation

1. **Start Simple**: Begin with minimal config (just `api`)
2. **Add Incrementally**: Enable features as needed
3. **Use Widget Studio**: Design widgets visually before coding
4. **Implement Entity Search Early**: If using custom backend, provides significant UX value
5. **Monitor Events**: Use `onLog` and `onError` for observability
6. **Plan for Streaming**: Account for `response.start`/`response.end` in UI state
7. **Test Widget Actions**: Ensure client/server action flow works correctly

### For Further Investigation

1. **Server-Side SDK**: Examine OpenAI ChatKit Python SDK for backend implementation patterns
2. **Widget Rendering**: Investigate how the bundled JS renders widgets (performance, accessibility)
3. **Real-Time Protocol**: Understand the WebSocket/SSE protocol for server events
4. **Security Model**: Review domain key verification and CORS handling
5. **Mobile Behavior**: Test on mobile browsers (focus, keyboard, file upload)

---

## 10. Conclusion

ChatKit achieves its extensive feature set through a carefully designed architecture that separates concerns:

- **Type Definitions**: This repository (comprehensive, type-safe)
- **Implementation**: CDN-delivered web component (proprietary)
- **Backend**: Server SDK or custom implementation (flexible)

The "not much code" observation is correct for _this_ repository, but the system as a whole is sophisticated. The value here is in:

1. **Excellent TypeScript types** that document the entire API
2. **Clean React integration** that works seamlessly
3. **Well-designed patterns** for customization, entities, widgets, and actions
4. **Comprehensive documentation** that explains each feature

The lack of a specialized "agentic actions protocol" is intentional—ChatKit provides primitives (streaming text, widgets, tools) that can be composed to visualize any agent architecture, rather than prescribing one specific approach.

This design makes ChatKit a **general-purpose chat UI framework** suitable for RAG systems, traditional chatbots, agentic workflows, and hybrid approaches alike.
