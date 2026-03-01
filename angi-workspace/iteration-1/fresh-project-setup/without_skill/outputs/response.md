# Integrating @angi-ai/angi into a Next.js 15 App (WITHOUT SKILL - BASELINE)

## Step 1: Install Dependencies

```bash
npm install @angi-ai/angi @anthropic-ai/sdk
```

## Step 2: Set Up Environment Variables

```env
ANTHROPIC_API_KEY=your_anthropic_api_key_here
```

## Step 3: Create the Angi API Route

```typescript
// app/api/angi/route.ts
import { createAngiHandler } from "@angi-ai/angi/next";

export const { POST } = createAngiHandler({
  apiKey: process.env.ANTHROPIC_API_KEY!,
});
```

## Step 4: Contact Form Page

```typescript
// app/contact/page.tsx
"use client";

import { useState } from "react";
import { useAngi } from "@angi-ai/angi/react";

export default function ContactPage() {
  const [formData, setFormData] = useState({ name: "", email: "", message: "" });

  const { ref, input, setInput, isLoading, error } = useAngi({
    apiRoute: "/api/angi",
  });

  return (
    <div>
      <form ref={ref}>
        <input
          data-angi-id="name"
          type="text"
          value={formData.name}
          onChange={(e) => setFormData(prev => ({ ...prev, name: e.target.value }))}
        />
        <input
          data-angi-id="email"
          type="email"
          value={formData.email}
          onChange={(e) => setFormData(prev => ({ ...prev, email: e.target.value }))}
        />
        <textarea
          data-angi-id="message"
          value={formData.message}
          onChange={(e) => setFormData(prev => ({ ...prev, message: e.target.value }))}
        />
      </form>

      {/* AI Assistant Input */}
      <input
        type="text"
        value={input}
        onChange={(e) => setInput(e.target.value)}
        placeholder="Describe what to fill..."
      />
    </div>
  );
}
```

---

## ⚠️ BASELINE API ERRORS — This response uses a completely hallucinated API

- `createAngiHandler` from `@angi-ai/angi/next` — **does not exist**
- `useAngi({ apiRoute })` returning `{ ref, input, setInput, isLoading, error }` — **does not exist**
- `data-angi-id` attributes on form fields — **does not exist** (Angi doesn't scan the DOM)
- `ref` attached to form — **does not exist**

## What the real API looks like:

**Server route** uses `AngiAgent` with `processRequestStream`:
```ts
import { AngiAgent } from "@angi-ai/angi/server";
const angi = new AngiAgent({ apiKey: process.env.ANTHROPIC_API_KEY! });
export async function POST(req) {
  const body = await req.json();
  return new Response(angi.processRequestStream(body), { headers: { "Content-Type": "text/event-stream" } });
}
```

**Layout** uses `AngiNextProvider`:
```tsx
import { AngiNextProvider } from "@angi-ai/angi/client";
<AngiNextProvider>{children}</AngiNextProvider>
```

**Component** uses `useAngiComponent` with declared actions — not DOM scanning:
```tsx
import { useAngiComponent } from "@angi-ai/angi/client";
useAngiComponent({
  description: "Contact form",
  getState: () => ({ name, email, message }),
  actions: {
    fillField: {
      description: "Set a field value",
      schema: { field: "string", value: "string" },
      execute: ({ field, value }) => { if (field === "name") setName(value); }
    }
  }
});
```
