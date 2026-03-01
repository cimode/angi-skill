# Can the `execute` function be async in an Angi component?

Yes, the `execute` function in an Angi component fully supports being async. You can mark it with `async` and use `await` inside it to call your API before returning a result to Claude.

## Basic Pattern

```typescript
import { AngiComponent, createAction } from "@angi-ai/angi";

const submitOrderAction = createAction({
  name: "submit_order",
  description: "Submit an order to the API",
  parameters: {
    type: "object",
    properties: {
      productId: {
        type: "string",
        description: "The ID of the product to order",
      },
      quantity: {
        type: "number",
        description: "The quantity to order",
      },
    },
    required: ["productId", "quantity"],
  },
  execute: async ({ productId, quantity }) => {
    const response = await fetch("/api/orders", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ productId, quantity }),
    });

    if (!response.ok) {
      throw new Error(`Order submission failed: ${response.statusText}`);
    }

    const result = await response.json();

    return {
      success: true,
      orderId: result.id,
      message: `Order ${result.id} submitted successfully`,
    };
  },
});

export const OrderComponent: AngiComponent = {
  id: "order-component",
  actions: [submitOrderAction],
  render: ({ state }) => {
    return `Order status: ${state?.message ?? "Ready"}`;
  },
};
```

## Key Points

**Async is fully supported.** Mark `execute` with `async` and use `await` freely. Angi awaits the returned promise before passing the result back to Claude.

**Return value matters.** Whatever you return from `execute` is given back to Claude as the action result. Return a descriptive object so Claude can communicate meaningful feedback to the user.

**Error handling.** If you throw inside `execute`, Angi surfaces the error to Claude, which can inform the user of the failure. Throw with a clear message.

**Combining API calls and local state updates.** You can do both in the same `execute`:

```typescript
execute: async ({ productId, quantity }, { setState }) => {
  setState({ status: "submitting" });

  try {
    const response = await fetch("/api/orders", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ productId, quantity }),
    });

    const result = await response.json();
    setState({ status: "success", orderId: result.id });

    return { success: true, orderId: result.id };
  } catch (error) {
    setState({ status: "error" });
    throw error;
  }
},
```

Note: The availability of `setState` (and other context utilities) as the second argument to `execute` depends on your specific version of `@angi-ai/angi`. Verify against your version's API docs if needed.

## Summary

Making `execute` async is straightforward: add `async`, use `await` for your API calls, and return a result object. This is the correct approach when you need to hit a backend endpoint rather than only updating local state.
