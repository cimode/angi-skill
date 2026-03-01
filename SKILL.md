---
name: angi
description: >
  Expert developer guide for @angi-ai/angi — the AI-native React UI library that lets end users control interface components through natural language. Use this skill whenever a developer mentions Angi, wants to make a React component AI-addressable, wants Claude or an LLM to read or operate a form/table/section/dashboard, or asks how to expose component state and actions to an LLM. Also trigger for: "I want an AI component", "I want Claude to fill this form", "how do I expose state to Claude", "make this component readable by Claude", "set up Angi in my project", "I want users to control my UI with natural language", "how do I use useAngiComponent", or any question about @angi-ai/angi integration, setup, or component design.
---

# Angi Developer Guide

`@angi-ai/angi` is an AI-native UI runtime for React. It lets end users say what they want in plain language, and Angi translates that into real component actions — no manual clicking, no form-filling by hand.

**Core mental model:**
- Your components **declare** their current state and what they can do (actions)
- The user types a natural language prompt (via `AngiChatBubble` or your own input)
- Angi sends the prompt + component metadata to Claude server-side
- Claude calls the appropriate actions; results stream back and execute live in the browser

The LLM never runs in the browser. Your API key stays server-side.

---

## Step 1: Understand the Situation

Before generating code, figure out where the developer is:

- **Starting from scratch** → walk through [Full Setup](#full-setup)
- **Angi already installed, want to register a component** → jump to [Make a Component AI-Addressable](#make-a-component-ai-addressable)
- **Want Claude to read a component's state without modifying it** → jump to [Read-Only Components](#read-only-components)
- **Want to design the right actions for a component** → jump to [Designing Custom Actions](#designing-custom-actions)

Generate actual working code, not just explanations. Include enough context in comments that the developer understands what each piece does and why.

---

## Full Setup

### 1. Install

```bash
npm install @angi-ai/angi @anthropic-ai/sdk
```

Peer dependencies: `react ^19.0.0`, `react-dom ^19.0.0`.

### 2. API Key

```env
# .env.local
ANTHROPIC_API_KEY=sk-ant-...
```

Get a key at [console.anthropic.com](https://console.anthropic.com/).

### 3. Create the server route

This is the only server file needed. It receives prompts from the UI, calls Claude, and streams the response back.

```ts
// app/api/angi/route.ts
import { AngiAgent } from "@angi-ai/angi/server";
import { NextRequest } from "next/server";

const angi = new AngiAgent({ apiKey: process.env.ANTHROPIC_API_KEY! });

export async function POST(req: NextRequest) {
  const body = await req.json();
  const stream = angi.processRequestStream(body);

  return new Response(stream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      Connection: "keep-alive",
    },
  });
}
```

### 4. Wrap the layout

```tsx
// app/layout.tsx
import { AngiNextProvider } from "@angi-ai/angi/client";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <AngiNextProvider>{children}</AngiNextProvider>
      </body>
    </html>
  );
}
```

`AngiNextProvider` sets up the shared React context and connects automatically to `/api/angi`.

### 5. Add the chat widget

```tsx
import { AngiChatBubble } from "@angi-ai/angi/client";

// Drop anywhere in your page tree — renders a floating AI button
<AngiChatBubble />
```

`AngiChatBubble` and your components are **siblings** under the provider — not nested. They communicate through shared context, not through JSX nesting.

---

## Make a Component AI-Addressable

Any React component can be made AI-addressable in two steps: wrap it with an `<Angi>` boundary in the parent, and call `useAngiComponent` inside the component itself.

### 1. Add the `<Angi>` boundary in the parent

```tsx
// app/page.tsx
import { Angi, AngiChatBubble } from "@angi-ai/angi/client";

export default function Page() {
  return (
    <>
      <AngiChatBubble />

      {/* The boundary assigns an id and declares permissions */}
      <Angi id="contact-form" permissions={["read", "write"]}>
        <ContactForm />
      </Angi>
    </>
  );
}
```

The boundary lives in the **parent or page**, not inside the component itself. This keeps the component unaware of Angi's existence — it just calls a hook.

If you omit `id`, Angi derives one from the component's description. Prefer explicit IDs for anything Claude might refer to by name.

### 2. Call `useAngiComponent` inside the component

```tsx
// components/ContactForm.tsx
"use client";
import { useState } from "react";
import { useAngiComponent } from "@angi-ai/angi/client";

export function ContactForm() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");

  useAngiComponent({
    // What this component is — Claude uses this as context
    description: "Contact form with name and email fields",

    // Current state snapshot — called fresh at the moment a prompt is sent
    getState: () => ({ name, email }),

    // Actions Claude is allowed to invoke
    actions: {
      fillField: {
        description: "Set a field value by field name",
        schema: { field: "string", value: "string" },
        execute: ({ field, value }) => {
          if (field === "name") setName(value as string);
          if (field === "email") setEmail(value as string);
        },
      },
      clearForm: {
        description: "Clear all fields",
        schema: {},
        execute: () => { setName(""); setEmail(""); },
      },
    },
  });

  return (
    <form>
      <input value={name} onChange={(e) => setName(e.target.value)} placeholder="Name" />
      <input value={email} onChange={(e) => setEmail(e.target.value)} placeholder="Email" />
    </form>
  );
}
```

The user can now say _"Fill the name as José and set email to jose@example.com"_ and Claude will call `fillField` twice — once per field — and both inputs update in real time.

### `useAngiComponent` options

| Option | Type | Required | Description |
|---|---|---|---|
| `description` | `string` | ✅ | Sent to Claude as context. Be specific — Claude uses this to understand what the component does and holds. |
| `getState` | `() => unknown` | ✅ | Returns current state. Called fresh at prompt time, so it reflects the latest values. |
| `actions` | `Record<string, AngiAction>` | ❌ | The actions Claude may invoke. Omit or pass `{}` for read-only components. |

---

## Read-Only Components

If you only want Claude to **see** a component's current data (but not modify it), use `permissions={["read"]}` and omit actions. This is useful for summary panels, dashboards, and status displays — things Claude should be able to reference when answering the user's questions.

```tsx
// Parent
<Angi id="order-summary" permissions={["read"]}>
  <OrderSummary />
</Angi>
```

```tsx
// OrderSummary.tsx
function OrderSummary() {
  const { items, total, status } = useOrderStore();

  useAngiComponent({
    description: "Current order summary showing items, total price, and order status",
    getState: () => ({ items, total, status }),
    // No actions — Claude can read this state but cannot change anything
  });

  return <div>...</div>;
}
```

When a user asks "What's in my cart?" or "What's my order total?", Claude answers from live component state — no separate API needed.

---

## Permissions: Read vs Write

Each `<Angi>` boundary declares what Claude is allowed to do with that component:

| Permission | Effect |
|---|---|
| `"read"` | `getState()` is included in Claude's system prompt — Claude can reference it |
| `"write"` | Claude may call the component's registered actions |

Use `["read", "write"]` (the default) for components Claude should fully control. Use `["read"]` for components you want Claude to reference but not modify.

**Why this matters:** permissions are enforced in two places — before the payload is sent to the server (write-only components' action schemas are omitted entirely, so Claude never sees them as callable tools), and again on the client when action chunks arrive back. Claude cannot invoke an action on a read-only component even if it tries.

---

## Designing Custom Actions

Actions are entirely developer-defined. Angi has no predefined actions — you name them, describe them, define their parameters, and implement what they do.

### Anatomy of an action

```ts
{
  description: string;                    // What this action does — Claude reads this to decide when to call it
  schema: Record<string, string>;         // Parameter names → types ("string", "number", "boolean")
  execute: (params: Record<string, unknown>) => void | Promise<void>;  // Your code, runs in the browser
}
```

### Keep actions granular and composable

Claude picks and combines actions, so smaller and more focused actions give it more flexibility than one large catch-all:

```ts
// ✅ Granular — Claude can compose these freely
actions: {
  setSearchQuery:     { schema: { query: "string" }, ... },
  filterByCategory:   { schema: { category: "string" }, ... },
  setSortOrder:       { schema: { by: "string" }, ... },
  clearAllFilters:    { schema: {}, ... },
}

// 🚫 Too coarse — Claude has to guess what instruction to pass
actions: {
  updateFilters: { schema: { instruction: "string" }, ... }
}
```

### Write descriptions Claude can reason about

The description is how Claude decides whether to call an action. Make it unambiguous:

```ts
// ✅ Clear
description: "Select a category from the filter dropdown"

// 🚫 Vague
description: "Change the dropdown"
```

### `execute` is plain JavaScript — anything goes

```ts
// React local state
execute: ({ value }) => setName(value as string)

// Async API call
execute: async () => {
  await fetch("/api/submit", { method: "POST", body: JSON.stringify({ name, email }) });
  router.push("/thank-you");
}

// External store (Zustand, Redux, etc.)
execute: ({ tab }) => useAppStore.getState().setActiveTab(tab as string)

// Multiple state updates at once
execute: ({ category, minPrice, maxPrice }) => {
  setCategory(category as string);
  setPriceRange([Number(minPrice), Number(maxPrice)]);
}
```

### A fuller example — product search page

```tsx
function ProductSearch() {
  const [query, setQuery] = useState("");
  const [category, setCategory] = useState("all");
  const [sortBy, setSortBy] = useState("relevance");
  const [results, setResults] = useState<Product[]>([]);

  useAngiComponent({
    description: "Product search page with keyword search, category filter, and sort",
    getState: () => ({ query, category, sortBy, resultCount: results.length }),
    actions: {
      search: {
        description: "Search products by keyword",
        schema: { query: "string" },
        execute: ({ query }) => setQuery(query as string),
      },
      filterByCategory: {
        description: "Filter results to a specific product category",
        schema: { category: "string" },
        execute: ({ category }) => setCategory(category as string),
      },
      sortResults: {
        description: "Sort results — accepted values: 'relevance', 'price_asc', 'price_desc', 'rating'",
        schema: { by: "string" },
        execute: ({ by }) => setSortBy(by as string),
      },
      clearFilters: {
        description: "Reset all filters and search query to defaults",
        schema: {},
        execute: () => { setQuery(""); setCategory("all"); setSortBy("relevance"); },
      },
    },
  });

  return <div>...</div>;
}
```

Now a user can say _"Search for sneakers under $100, sorted by price"_ and Claude will call `search`, `filterByCategory`, and `sortResults` in sequence.

---

## Programmatic Usage (No Chat Bubble)

If you want to trigger Angi from your own UI instead of the floating bubble, use `useAngi()`:

```tsx
import { useAngi } from "@angi-ai/angi/client";

function MyCustomInput() {
  const { sendPrompt, isLoading, streamText } = useAngi();

  return (
    <div>
      <button onClick={() => sendPrompt("Clear the form and start over")} disabled={isLoading}>
        Reset with AI
      </button>
      {streamText && <p>{streamText}</p>}
    </div>
  );
}
```

`sendPrompt` takes a plain string and triggers the full Angi flow — registry snapshot, server call, streaming actions — without needing the chat bubble.

---

## Import Reference

```ts
// Client-side
import {
  Angi,
  AngiNextProvider,
  AngiChatBubble,
  useAngi,
  useAngiComponent,
} from "@angi-ai/angi/client";

import type { AngiAction, AngiPermission } from "@angi-ai/angi/client";

// Server-side
import { AngiAgent } from "@angi-ai/angi/server";
```

---

## Quick Checklist

When making a component AI-addressable, verify:

1. `<Angi id="..." permissions={[...]}>` wraps the component **in the parent/page**, not inside the component itself
2. `useAngiComponent` is called **inside** the component that renders the UI
3. `description` clearly describes what the component is and what data it holds
4. `getState` returns the values Claude needs to understand the component's current situation
5. Each action has a clear `description` Claude can reason about, and a `schema` that matches what `execute` expects
6. If Claude should only read (not modify), `permissions={["read"]}` and no actions are passed
