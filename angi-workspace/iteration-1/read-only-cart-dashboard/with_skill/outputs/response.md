# Read-Only Cart Dashboard with Angi

This is exactly the use case that Angi's read-only permission mode is designed for. You want Claude to be able to answer questions like "what's in my cart?" or "what's my total?" from live component state — without ever being allowed to modify the cart.

The key insight: use `permissions={["read"]}` on the `<Angi>` boundary and omit `actions` entirely from `useAngiComponent`. Angi enforces this in two places — the action schemas are never sent to the server (Claude never sees them as callable tools), and the client also discards any action chunks that arrive back.

---

## What You Need

```bash
npm install @angi-ai/angi @anthropic-ai/sdk
```

```env
# .env.local
ANTHROPIC_API_KEY=sk-ant-...
```

---

## Step 1: Server Route

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

---

## Step 2: Layout Provider

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

---

## Step 3: Dashboard Page

The `<Angi>` boundary lives in the **parent/page**, not inside the component. `permissions={["read"]}` is the enforcement declaration.

```tsx
// app/dashboard/page.tsx
"use client";
import { Angi, AngiChatBubble } from "@angi-ai/angi/client";
import { OrderSummaryCard } from "@/components/OrderSummaryCard";

const cartData = {
  items: [
    { id: "1", name: "Wireless Headphones", quantity: 1, price: 89.99 },
    { id: "2", name: "USB-C Cable (3-pack)", quantity: 2, price: 12.49 },
    { id: "3", name: "Laptop Stand", quantity: 1, price: 34.99 },
  ],
  subtotal: 149.96,
  discount: 15.00,
  total: 134.96,
};

export default function DashboardPage() {
  return (
    <>
      <AngiChatBubble />

      <main className="p-8">
        <h1 className="text-2xl font-bold mb-6">Your Dashboard</h1>

        {/*
          permissions={["read"]} means Claude may read getState() output
          but cannot invoke any actions. Enforced server-side (schemas omitted)
          and client-side (action chunks discarded).
        */}
        <Angi id="order-summary-card" permissions={["read"]}>
          <OrderSummaryCard
            items={cartData.items}
            subtotal={cartData.subtotal}
            discount={cartData.discount}
            total={cartData.total}
          />
        </Angi>
      </main>
    </>
  );
}
```

---

## Step 4: OrderSummaryCard Component

`useAngiComponent` is called inside the component. No `actions` key — Claude can read state but has nothing to call to modify the cart.

```tsx
// components/OrderSummaryCard.tsx
"use client";
import { useAngiComponent } from "@angi-ai/angi/client";

interface CartItem {
  id: string;
  name: string;
  quantity: number;
  price: number; // unit price
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
      "Order summary card showing the user's current shopping cart. " +
      "Contains items with names, quantities, and unit prices, " +
      "the subtotal before discounts, the discount amount, and the final total.",

    // Called fresh at prompt time — Claude always sees current live data.
    getState: () => ({
      items: items.map((item) => ({
        name: item.name,
        quantity: item.quantity,
        unitPrice: item.price,
        lineTotal: item.quantity * item.price,
      })),
      itemCount: items.reduce((sum, item) => sum + item.quantity, 0),
      subtotal,
      discount,
      total,
    }),

    // No actions — Claude has read access but nothing to invoke.
  });

  return (
    <div className="rounded-lg border border-gray-200 p-6 max-w-md shadow-sm">
      <h2 className="text-lg font-semibold mb-4">Order Summary</h2>

      <ul className="space-y-3 mb-6">
        {items.map((item) => (
          <li key={item.id} className="flex justify-between text-sm">
            <span>
              {item.name}
              {item.quantity > 1 && (
                <span className="text-gray-500 ml-1">x{item.quantity}</span>
              )}
            </span>
            <span className="font-medium">
              ${(item.price * item.quantity).toFixed(2)}
            </span>
          </li>
        ))}
      </ul>

      <div className="border-t border-gray-100 pt-4 space-y-2 text-sm">
        <div className="flex justify-between text-gray-600">
          <span>Subtotal</span>
          <span>${subtotal.toFixed(2)}</span>
        </div>
        {discount > 0 && (
          <div className="flex justify-between text-green-600">
            <span>Discount</span>
            <span>-${discount.toFixed(2)}</span>
          </div>
        )}
        <div className="flex justify-between font-semibold text-base pt-2 border-t border-gray-100">
          <span>Total</span>
          <span>${total.toFixed(2)}</span>
        </div>
      </div>
    </div>
  );
}
```

---

## What Claude Can and Cannot Do

**Can do** (via `getState` snapshot):
- "What's in my cart?" — lists items with quantities and line totals
- "What's my total?" — reads `total`
- "Do I have any discounts?" — reads `discount`
- "How many items am I ordering?" — reads `itemCount`

**Cannot do** (no actions registered, `permissions={["read"]}`):
- Add or remove items
- Change quantities or prices
- Apply or clear discounts
- Clear the cart

The enforcement is structural: action schemas are never sent to the server, so Claude never sees them as callable tools. Even if it tried, the client would discard any incoming action chunks.

---

## Using a Real Store (Zustand Example)

```tsx
// app/dashboard/page.tsx — live store data
"use client";
import { Angi, AngiChatBubble } from "@angi-ai/angi/client";
import { OrderSummaryCard } from "@/components/OrderSummaryCard";
import { useCartStore } from "@/store/cartStore";

export default function DashboardPage() {
  const { items, subtotal, discount, total } = useCartStore();

  return (
    <>
      <AngiChatBubble />
      <main className="p-8">
        <Angi id="order-summary-card" permissions={["read"]}>
          <OrderSummaryCard
            items={items}
            subtotal={subtotal}
            discount={discount}
            total={total}
          />
        </Angi>
      </main>
    </>
  );
}
```

`getState` closes over the component's current props, so it always reflects the latest store values at prompt time — no additional wiring needed.

---

## Quick Checklist

1. `<Angi id="order-summary-card" permissions={["read"]}>` wraps the component **in the parent page**, not inside `OrderSummaryCard`
2. `useAngiComponent` is called **inside** `OrderSummaryCard`
3. `description` clearly explains what the component holds
4. `getState` returns all the data Claude needs to answer cart questions
5. **No `actions` key** — omitting it means nothing is callable
6. `permissions={["read"]}` on the boundary is the enforcement layer
