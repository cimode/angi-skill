# Integrating @angi-ai/angi into a Next.js 15 App from Scratch

This guide walks you through everything you need to set up `@angi-ai/angi` in a fresh Next.js 15 project and wire up a contact form so users can say things like "fill the name with John and email with john@example.com" and have the form update in real time.

---

## How It Works (Mental Model)

Before writing any code, understand the data flow:

1. Your contact form **declares** its current field values and the actions it supports (e.g., "fill a field", "clear the form", "submit").
2. The user types a natural-language prompt into the floating chat bubble.
3. Angi sends that prompt + a snapshot of your component's current state and available actions to Claude — server-side.
4. Claude decides which actions to call and in what order. Those calls stream back to the browser.
5. Your `execute` functions run live in the browser, updating React state.

Your API key never touches the browser. Claude never runs client-side.

---

## Step 1: Install Dependencies

```bash
npm install @angi-ai/angi @anthropic-ai/sdk
```

Peer dependencies are `react ^19.0.0` and `react-dom ^19.0.0`, which Next.js 15 ships with.

---

## Step 2: Add Your Anthropic API Key

Create or update `.env.local` at the root of your project:

```env
# .env.local
ANTHROPIC_API_KEY=sk-ant-...
```

Get a key at https://console.anthropic.com/. This file is gitignored by default — never commit it.

---

## Step 3: Create the Server Route

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

`AngiNextProvider` automatically connects to `/api/angi` — no additional URL configuration needed.

---

## Step 4: Wrap the Root Layout

```tsx
// app/layout.tsx
import { AngiNextProvider } from "@angi-ai/angi/client";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <AngiNextProvider>{children}</AngiNextProvider>
      </body>
    </html>
  );
}
```

---

## Step 5: Build the Contact Form Component

```tsx
// components/ContactForm.tsx
"use client";

import { useState } from "react";
import { useAngiComponent } from "@angi-ai/angi/client";

export function ContactForm() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [message, setMessage] = useState("");
  const [submitted, setSubmitted] = useState(false);

  useAngiComponent({
    description:
      "Contact form with three fields: name (full name of the sender), " +
      "email (sender's email address), and message (the body of the inquiry). " +
      "Used to send a message to the site owner.",

    getState: () => ({
      name,
      email,
      message,
      submitted,
      filledFields: [
        name ? "name" : null,
        email ? "email" : null,
        message ? "message" : null,
      ].filter(Boolean),
    }),

    actions: {
      fillField: {
        description:
          "Set the value of a specific form field. " +
          "Field must be one of: 'name', 'email', 'message'.",
        schema: { field: "string", value: "string" },
        execute: ({ field, value }) => {
          if (field === "name") setName(value as string);
          if (field === "email") setEmail(value as string);
          if (field === "message") setMessage(value as string);
        },
      },

      fillAllFields: {
        description:
          "Set name, email, and message all at once. " +
          "Use this when the user provides all three values in one prompt.",
        schema: { name: "string", email: "string", message: "string" },
        execute: ({ name: n, email: e, message: m }) => {
          setName(n as string);
          setEmail(e as string);
          setMessage(m as string);
        },
      },

      clearField: {
        description: "Clear a specific field. Field must be: 'name', 'email', or 'message'.",
        schema: { field: "string" },
        execute: ({ field }) => {
          if (field === "name") setName("");
          if (field === "email") setEmail("");
          if (field === "message") setMessage("");
        },
      },

      clearForm: {
        description: "Clear all fields and reset the form to its empty state.",
        schema: {},
        execute: () => {
          setName("");
          setEmail("");
          setMessage("");
          setSubmitted(false);
        },
      },

      submitForm: {
        description:
          "Submit the contact form. Only call this if all three fields " +
          "(name, email, message) are non-empty.",
        schema: {},
        execute: async () => {
          console.log("Submitting:", { name, email, message });
          await new Promise((resolve) => setTimeout(resolve, 500));
          setSubmitted(true);
        },
      },
    },
  });

  if (submitted) {
    return (
      <div>
        <p>Thanks, {name}! Your message has been sent.</p>
        <button onClick={() => setSubmitted(false)}>Send another</button>
      </div>
    );
  }

  return (
    <form onSubmit={(e) => { e.preventDefault(); setSubmitted(true); }}>
      <div>
        <label htmlFor="name">Name</label>
        <input id="name" type="text" value={name} onChange={(e) => setName(e.target.value)} placeholder="Your full name" />
      </div>
      <div>
        <label htmlFor="email">Email</label>
        <input id="email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} placeholder="you@example.com" />
      </div>
      <div>
        <label htmlFor="message">Message</label>
        <textarea id="message" value={message} onChange={(e) => setMessage(e.target.value)} placeholder="What would you like to say?" rows={5} />
      </div>
      <button type="submit">Send Message</button>
    </form>
  );
}
```

---

## Step 6: Wire It Up in the Page

```tsx
// app/page.tsx
import { Angi, AngiChatBubble } from "@angi-ai/angi/client";
import { ContactForm } from "@/components/ContactForm";

export default function ContactPage() {
  return (
    <main style={{ maxWidth: 600, margin: "0 auto", padding: "2rem" }}>
      <h1>Contact Us</h1>
      <AngiChatBubble />
      <Angi id="contact-form" permissions={["read", "write"]}>
        <ContactForm />
      </Angi>
    </main>
  );
}
```

`<AngiChatBubble>` and `<Angi>` are **siblings** under the provider — not nested into each other. They communicate through shared context.

---

## What the User Can Now Say

| User says | What Angi does |
|---|---|
| "Fill the name with John and email with john@example.com" | Calls `fillField` twice |
| "Set my name to Jane Smith" | Calls `fillField` with field=name |
| "Fill everything out: name John Doe, email john@doe.com, message Hello" | Calls `fillAllFields` once |
| "Clear the email field" | Calls `clearField` with field=email |
| "Start over" | Calls `clearForm` |
| "Submit the form" | Calls `submitForm` |
| "What's in the name field?" | Claude reads `getState()` and answers — no action called |

---

## Complete File Tree

```
my-next-app/
├── .env.local                        <- ANTHROPIC_API_KEY
├── app/
│   ├── layout.tsx                    <- AngiNextProvider wraps everything
│   ├── page.tsx                      <- <Angi> boundary + <AngiChatBubble>
│   └── api/
│       └── angi/
│           └── route.ts              <- Server route (AngiAgent)
└── components/
    └── ContactForm.tsx               <- useAngiComponent lives here
```

---

## Common Mistakes to Avoid

- Do not put `<Angi>` inside the component itself. It belongs in the parent page. The component only calls `useAngiComponent`.
- Do not nest `<AngiChatBubble>` inside `<Angi>`. They are siblings.
- Do not call `useAngiComponent` conditionally. Like all React hooks, call it at the top level.
- Write specific action descriptions. "Set the value of a specific form field — field must be one of: name, email, message" is much better than "update the form".
- Return all relevant state from `getState`. Include computed helpers like `filledFields` so Claude can reason about form completeness.

---

## Optional: Custom Input Instead of Chat Bubble

```tsx
// components/MyAIInput.tsx
"use client";

import { useAngi } from "@angi-ai/angi/client";
import { useState } from "react";

export function MyAIInput() {
  const { sendPrompt, isLoading, streamText } = useAngi();
  const [input, setInput] = useState("");

  return (
    <div>
      <form onSubmit={(e) => { e.preventDefault(); sendPrompt(input); setInput(""); }}>
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Tell the AI what to do..."
          disabled={isLoading}
        />
        <button type="submit" disabled={isLoading}>
          {isLoading ? "Thinking..." : "Send"}
        </button>
      </form>
      {streamText && <p>{streamText}</p>}
    </div>
  );
}
```

---

## Quick Checklist Before You Test

- `ANTHROPIC_API_KEY` is set in `.env.local`
- `app/api/angi/route.ts` exists and exports a `POST` handler
- `AngiNextProvider` wraps the root layout's children
- `<Angi id="contact-form" permissions={["read", "write"]}>` wraps `<ContactForm>` in the page
- `<AngiChatBubble />` is a sibling of `<Angi>`, not nested inside it
- `useAngiComponent` is called at the top level of `ContactForm`
- Each action has a clear `description` and a `schema` that matches what `execute` destructures
- `getState` returns current field values and computed helpers
