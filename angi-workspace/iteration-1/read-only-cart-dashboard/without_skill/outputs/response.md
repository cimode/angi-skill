# Read-Only Cart Dashboard with @angi-ai/angi

## Overview

You want Claude to be able to **read** your `OrderSummaryCard` state (items, subtotal, discount, total) and answer questions about it, but never modify the cart. In `@angi-ai/angi`, this is controlled entirely by the `permissions` prop on the `<Angi>` boundary wrapper and by the `actions` you register (or deliberately omit) in `useAngiComponent`.

The SDK enforces read-only behavior at two independent layers:

1. **Client layer** (`AngiProvider` / `handleChunk`): Even if an `action` chunk arrives, it is silently dropped when `component.permissions` does not include `"write"`.
2. **Server layer** (`buildTools`): Tools are only generated for components that include `"write"` in their permissions array. A read-only component produces zero tools, so the model is never even offered a way to call an action against it.

Setting `permissions={["read"]}` and providing no `actions` is the complete, idiomatic solution.

---

## How Permissions Work in the SDK

### `<Angi permissions={[...]}>` (boundary wrapper)

```tsx
// packages/angi/src/client/components/Angi.tsx
export function Angi({
  id,
  permissions = ["read", "write"],  // default is full access
  children,
}: AngiProps) { ... }
```

The `permissions` array is stored in React context and picked up by `useAngiComponent` inside the wrapped component. It is then serialised into `ComponentPayload.permissions` and sent to the server with every request.

### Server: `buildTools` skips read-only components

```ts
// packages/angi/src/server/core/buildTools.ts
for (const comp of components) {
  if (!comp.permissions.includes("write")) continue;  // no tools generated
  // ...
}
```

Because no tools are generated for your cart component, the model **cannot call any action on it**, regardless of what the user asks.

### Server: `buildSystemPrompt` still exposes state

```ts
// packages/angi/src/server/core/buildSystemPrompt.ts
if (comp.permissions.includes("read") && comp.state != null) {
  parts.push(`Current state: ${JSON.stringify(comp.state, null, 2)}`);
}
```

The current cart state is included in the system prompt so Claude can read it and answer questions. This is the read side of "read-only".

### Client: `handleChunk` double-checks write permission

```ts
// packages/angi/src/client/components/AngiProvider.tsx
if (!component.permissions.includes("write")) {
  console.warn(`[Angi] Component "${componentId}" does not allow write.`);
  return;
}
```

This is a second line of defence — even if a rogue action chunk arrived, the client would refuse to execute it.

---

## Complete Working Example

### 1. The `OrderSummaryCard` component

```tsx
// app/components/OrderSummaryCard.tsx
"use client";

import { useAngiComponent } from "@angi-ai/angi";

export interface CartItem {
  id: string;
  name: string;
  quantity: number;
  unitPrice: number;
}

interface OrderSummaryCardProps {
  items: CartItem[];
  subtotal: number;
  discount: number;
  total: number;
}

export function OrderSummaryCard({ items, subtotal, discount, total }: OrderSummaryCardProps) {
  useAngiComponent({
    description:
      "Order summary card showing the user's current cart items, subtotal, discount, and total",
    getState: () => ({ items, subtotal, discount, total }),
    // actions intentionally omitted — defaults to {}
  });

  return (
    <div className="rounded-xl border p-6 shadow-sm">
      <h2 className="mb-4 text-xl font-semibold">Order Summary</h2>
      <ul className="mb-4 space-y-2">
        {items.map((item) => (
          <li key={item.id} className="flex justify-between text-sm">
            <span>{item.name} x {item.quantity}</span>
            <span>${(item.unitPrice * item.quantity).toFixed(2)}</span>
          </li>
        ))}
      </ul>
      <div className="border-t pt-4 space-y-1 text-sm">
        <div className="flex justify-between">
          <span>Subtotal</span><span>${subtotal.toFixed(2)}</span>
        </div>
        {discount > 0 && (
          <div className="flex justify-between text-green-600">
            <span>Discount</span><span>-${discount.toFixed(2)}</span>
          </div>
        )}
        <div className="flex justify-between font-bold text-base mt-2">
          <span>Total</span><span>${total.toFixed(2)}</span>
        </div>
      </div>
    </div>
  );
}
```

### 2. The dashboard that wraps the component

```tsx
// app/components/CartDashboard.tsx
"use client";

import { Angi, useAngi } from "@angi-ai/angi";
import { OrderSummaryCard } from "./OrderSummaryCard";
import { useState } from "react";

const LIVE_CART = {
  items: [
    { id: "1", name: "Wireless Headphones", quantity: 1, unitPrice: 89.99 },
    { id: "2", name: "USB-C Cable (3-pack)", quantity: 2, unitPrice: 12.49 },
  ],
  subtotal: 114.97,
  discount: 10.0,
  total: 104.97,
};

function ChatInput() {
  const { sendPrompt, isLoading, streamText } = useAngi();
  const [prompt, setPrompt] = useState("");

  const handleSend = () => {
    if (!prompt.trim()) return;
    sendPrompt(prompt);
    setPrompt("");
  };

  return (
    <div className="mt-6 space-y-3">
      <div className="flex gap-2">
        <input
          className="flex-1 rounded border px-3 py-2 text-sm"
          placeholder="Ask about your cart... e.g. 'What's my total?'"
          value={prompt}
          onChange={(e) => setPrompt(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && handleSend()}
        />
        <button
          className="rounded bg-blue-600 px-4 py-2 text-sm text-white disabled:opacity-50"
          onClick={handleSend}
          disabled={isLoading}
        >
          {isLoading ? "..." : "Ask"}
        </button>
      </div>
      {streamText && (
        <div className="rounded bg-gray-50 p-3 text-sm text-gray-700">
          {streamText}
        </div>
      )}
    </div>
  );
}

export function CartDashboard() {
  return (
    <div className="max-w-md mx-auto py-10">
      {/* permissions={["read"]} is the critical line */}
      <Angi id="order-summary-card" permissions={["read"]}>
        <OrderSummaryCard {...LIVE_CART} />
      </Angi>
      <ChatInput />
    </div>
  );
}
```

### 3. The Next.js page

```tsx
// app/dashboard/page.tsx
import { AngiNextProvider } from "@angi-ai/angi";
import { CartDashboard } from "../components/CartDashboard";

export default function DashboardPage() {
  return (
    <AngiNextProvider endpoint="/api/angi">
      <CartDashboard />
    </AngiNextProvider>
  );
}
```

### 4. The API route

```ts
// app/api/angi/route.ts
import { AngiAgent } from "@angi-ai/angi/server";
import type { AngiRequestBody } from "@angi-ai/angi/server";

const agent = new AngiAgent({ apiKey: process.env.ANTHROPIC_API_KEY! });

export async function POST(request: Request) {
  const body = (await request.json()) as AngiRequestBody;
  const stream = agent.processRequestStream(body);
  return new Response(stream, {
    headers: {
      "Content-Type": "text/event-stream",
      "Cache-Control": "no-cache",
      Connection: "keep-alive",
    },
  });
}
```

---

## The Key Line

```tsx
<Angi id="order-summary-card" permissions={["read"]}>
  <OrderSummaryCard {...cartData} />
</Angi>
```

That single `permissions={["read"]}` prop does all three things:

- Includes the live cart state in Claude's system prompt (so it can answer "what's in my cart?").
- Prevents any write tools from being generated on the server (`buildTools` skips the component entirely).
- Causes the client to silently discard any action chunk that somehow arrives (`handleChunk` checks `"write"` before executing).

No additional configuration, no custom guards, no server-side middleware needed.
