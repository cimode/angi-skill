# Making ProductFilterPanel AI-Addressable with Angi

Since you already have Angi set up, you only need to do two things:

1. Wrap `ProductFilterPanel` with an `<Angi>` boundary in its parent/page
2. Call `useAngiComponent` inside `ProductFilterPanel` itself

---

## Step 1: Wrap the component in the parent/page

The `<Angi>` boundary lives in the **parent** — not inside the component itself.

```tsx
// app/page.tsx (or wherever ProductFilterPanel is rendered)
import { Angi, AngiChatBubble } from "@angi-ai/angi/client";
import { ProductFilterPanel } from "@/components/ProductFilterPanel";

export default function Page() {
  return (
    <>
      {/* The floating AI chat widget — users type natural language requests here */}
      <AngiChatBubble />

      {/* The boundary assigns an ID and grants read+write access to Claude */}
      <Angi id="product-filter-panel" permissions={["read", "write"]}>
        <ProductFilterPanel />
      </Angi>
    </>
  );
}
```

---

## Step 2: Register the component with `useAngiComponent`

```tsx
// components/ProductFilterPanel.tsx
"use client";
import { useState } from "react";
import { useAngiComponent } from "@angi-ai/angi/client";

export function ProductFilterPanel() {
  const [category, setCategory] = useState<string>("all");
  const [priceMin, setPriceMin] = useState<number>(0);
  const [priceMax, setPriceMax] = useState<number>(1000);
  const [inStockOnly, setInStockOnly] = useState<boolean>(false);
  const [sortBy, setSortBy] = useState<string>("relevance");

  useAngiComponent({
    description:
      "Product filter panel that lets users narrow product listings by category, " +
      "price range (min and max), in-stock availability, and sort order. " +
      "Category options include: all, electronics, clothing, home, sports. " +
      "Sort options include: relevance, price_asc, price_desc, rating, newest.",

    getState: () => ({
      category,
      priceMin,
      priceMax,
      inStockOnly,
      sortBy,
    }),

    actions: {
      setCategory: {
        description:
          "Set the product category filter. Valid values: 'all', 'electronics', " +
          "'clothing', 'home', 'sports'.",
        schema: { category: "string" },
        execute: ({ category }) => setCategory(category as string),
      },

      setPriceMin: {
        description:
          "Set the minimum price filter in dollars. Use 0 to remove the lower bound.",
        schema: { priceMin: "number" },
        execute: ({ priceMin }) => setPriceMin(Number(priceMin)),
      },

      setPriceMax: {
        description:
          "Set the maximum price filter in dollars. For example, 200 means 'under $200'.",
        schema: { priceMax: "number" },
        execute: ({ priceMax }) => setPriceMax(Number(priceMax)),
      },

      setPriceRange: {
        description:
          "Set both the minimum and maximum price filter at the same time. " +
          "Use this when the user specifies a price range rather than a single bound.",
        schema: { priceMin: "number", priceMax: "number" },
        execute: ({ priceMin, priceMax }) => {
          setPriceMin(Number(priceMin));
          setPriceMax(Number(priceMax));
        },
      },

      setInStockOnly: {
        description:
          "Enable or disable the 'in stock only' filter. " +
          "Pass true to show only in-stock items, false to show all items.",
        schema: { inStockOnly: "boolean" },
        execute: ({ inStockOnly }) => setInStockOnly(inStockOnly as boolean),
      },

      setSortBy: {
        description:
          "Set the sort order for product results. " +
          "Valid values: 'relevance', 'price_asc', 'price_desc', 'rating', 'newest'.",
        schema: { sortBy: "string" },
        execute: ({ sortBy }) => setSortBy(sortBy as string),
      },

      clearAllFilters: {
        description:
          "Reset all filters to their default values: category='all', priceMin=0, " +
          "priceMax=1000, inStockOnly=false, sortBy='relevance'.",
        schema: {},
        execute: () => {
          setCategory("all");
          setPriceMin(0);
          setPriceMax(1000);
          setInStockOnly(false);
          setSortBy("relevance");
        },
      },
    },
  });

  return (
    <aside className="filter-panel">
      <h2>Filters</h2>
      <label>
        Category
        <select value={category} onChange={(e) => setCategory(e.target.value)}>
          <option value="all">All</option>
          <option value="electronics">Electronics</option>
          <option value="clothing">Clothing</option>
          <option value="home">Home</option>
          <option value="sports">Sports</option>
        </select>
      </label>
      <label>
        Min Price ($)
        <input type="number" value={priceMin} onChange={(e) => setPriceMin(Number(e.target.value))} min={0} />
      </label>
      <label>
        Max Price ($)
        <input type="number" value={priceMax} onChange={(e) => setPriceMax(Number(e.target.value))} min={0} />
      </label>
      <label>
        <input type="checkbox" checked={inStockOnly} onChange={(e) => setInStockOnly(e.target.checked)} />
        In Stock Only
      </label>
      <label>
        Sort By
        <select value={sortBy} onChange={(e) => setSortBy(e.target.value)}>
          <option value="relevance">Relevance</option>
          <option value="price_asc">Price: Low to High</option>
          <option value="price_desc">Price: High to Low</option>
          <option value="rating">Rating</option>
          <option value="newest">Newest</option>
        </select>
      </label>
    </aside>
  );
}
```

---

## Why the actions are designed this way

Angi's design principle is to keep actions granular and composable. Claude picks and combines actions based on what the user said:

- "Only show in-stock items" triggers exactly one action
- "Clear everything" triggers exactly one action (`clearAllFilters`)
- "Electronics under $200 in stock sorted by rating" triggers all four

The `setPriceRange` action is a convenience action for when the user specifies both bounds at once ("between $50 and $200"), avoiding two separate calls.

## Descriptions carry the most weight

Claude decides which action to call — and with what parameter values — entirely from the `description` strings. Listing valid option values directly (e.g., `"Valid values: 'relevance', 'price_asc'..."`) ensures Claude passes the exact string your state setter expects.
