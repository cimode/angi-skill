# Can `execute` be async in an Angi component?

Yes — `execute` fully supports `async/await`. The type signature for `AngiAction` is:

```ts
execute: (params: Record<string, unknown>) => void | Promise<void>
```

You can make it `async`, `await` a fetch call, update state, redirect — whatever your flow needs. Angi will await the returned promise before considering the action complete.

---

## Example: submitting to your API when Claude calls the action

Here is a contact form where Claude can fill fields and submit. The `submitForm` action POSTs to your API instead of just updating local state:

```tsx
// components/ContactForm.tsx
"use client";
import { useState } from "react";
import { useRouter } from "next/navigation";
import { useAngiComponent } from "@angi-ai/angi/client";

export function ContactForm() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [status, setStatus] = useState<"idle" | "submitting" | "success" | "error">("idle");
  const router = useRouter();

  useAngiComponent({
    description: "Contact form with name and email fields. Can be filled and submitted.",

    getState: () => ({ name, email, status }),

    actions: {
      fillField: {
        description: "Set a field value. Use field 'name' or 'email'.",
        schema: { field: "string", value: "string" },
        execute: ({ field, value }) => {
          if (field === "name") setName(value as string);
          if (field === "email") setEmail(value as string);
        },
      },

      submitForm: {
        description: "Submit the contact form to the API with the current field values.",
        schema: {},
        // async execute — Angi awaits this before marking the action done
        execute: async () => {
          setStatus("submitting");
          try {
            const res = await fetch("/api/contact", {
              method: "POST",
              headers: { "Content-Type": "application/json" },
              body: JSON.stringify({ name, email }),
            });

            if (!res.ok) throw new Error("Submission failed");

            setStatus("success");
            router.push("/thank-you");
          } catch (err) {
            setStatus("error");
          }
        },
      },

      clearForm: {
        description: "Clear all fields and reset the form to its initial state.",
        schema: {},
        execute: () => {
          setName("");
          setEmail("");
          setStatus("idle");
        },
      },
    },
  });

  return (
    <form onSubmit={(e) => e.preventDefault()}>
      <input value={name} onChange={(e) => setName(e.target.value)} placeholder="Name" />
      <input value={email} onChange={(e) => setEmail(e.target.value)} placeholder="Email" />
      {status === "error" && <p>Something went wrong. Please try again.</p>}
    </form>
  );
}
```

And in the parent page:

```tsx
// app/page.tsx
import { Angi, AngiChatBubble } from "@angi-ai/angi/client";
import { ContactForm } from "@/components/ContactForm";

export default function Page() {
  return (
    <>
      <AngiChatBubble />
      <Angi id="contact-form" permissions={["read", "write"]}>
        <ContactForm />
      </Angi>
    </>
  );
}
```

---

## How this works at runtime

When a user says "Fill the form with name Jose and email jose@example.com, then submit it", Claude will:

1. Call `fillField` with `{ field: "name", value: "Jose" }`
2. Call `fillField` with `{ field: "email", value: "jose@example.com" }`
3. Call `submitForm` — Angi awaits the async function, which POSTs to `/api/contact` and navigates to `/thank-you`

---

## Key things to note

- **Closures capture current state.** `execute` for `submitForm` closes over `name` and `email` from the component's render scope. Because actions are re-registered on each render, the values in `execute` are always fresh when Claude calls the action.
- **Set loading state inside `execute`.** Since `execute` is async, you can call `setStatus("submitting")` before the `await` and update again afterward — giving users real-time feedback.
- **Catch errors inside `execute`.** Unhandled promise rejections are not surfaced to Claude. Catch them yourself and update component state to communicate what happened.
- **No return value is used.** Even though `execute` returns `Promise<void>`, the resolved value is ignored. Use state updates (not return values) to communicate results back to the UI.
