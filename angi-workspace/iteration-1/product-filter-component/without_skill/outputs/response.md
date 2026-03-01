# Making a ProductFilterPanel Component AI-Addressable with Angi

## Overview

To make your `ProductFilterPanel` component AI-addressable using `@angi-ai/angi`, you register it with the Angi system so that natural language commands like "show me electronics under $200 in stock, sorted by rating" automatically update your React state. Angi lets you declare component capabilities, their parameter schemas, and handlers that execute state updates.

---

## Prerequisites

Ensure Angi is installed and your app root has the provider:

```tsx
import { AngiProvider } from '@angi-ai/angi';

function App() {
  return (
    <AngiProvider>
      <YourAppContent />
    </AngiProvider>
  );
}
```

---

## Full Implementation

```tsx
import React, { useState, useEffect } from 'react';
import { useAngi } from '@angi-ai/angi';

interface FilterState {
  category: string;
  priceMin: number;
  priceMax: number;
  inStockOnly: boolean;
  sortBy: string;
}

const COMPONENT_ID = 'product-filter-panel';

export function ProductFilterPanel() {
  const [filters, setFilters] = useState<FilterState>({
    category: '',
    priceMin: 0,
    priceMax: 10000,
    inStockOnly: false,
    sortBy: 'relevance',
  });

  const { register, unregister } = useAngi();

  useEffect(() => {
    register({
      id: COMPONENT_ID,
      description:
        'A product filter panel that allows users to filter products by category, price range, stock availability, and sort order.',
      actions: [
        {
          name: 'setCategory',
          description: 'Set the product category to filter by.',
          parameters: {
            category: {
              type: 'string',
              description:
                'The product category, e.g. "electronics", "clothing", "books". Use an empty string to clear the category filter.',
            },
          },
          handler: ({ category }: { category: string }) => {
            setFilters((prev) => ({ ...prev, category }));
          },
        },
        {
          name: 'setPriceRange',
          description: 'Set the minimum and/or maximum price filter.',
          parameters: {
            priceMin: {
              type: 'number',
              description: 'Minimum price in dollars.',
              optional: true,
            },
            priceMax: {
              type: 'number',
              description: 'Maximum price in dollars.',
              optional: true,
            },
          },
          handler: ({ priceMin, priceMax }: { priceMin?: number; priceMax?: number }) => {
            setFilters((prev) => ({
              ...prev,
              priceMin: priceMin ?? prev.priceMin,
              priceMax: priceMax ?? prev.priceMax,
            }));
          },
        },
        {
          name: 'setInStockOnly',
          description: 'Toggle whether to show only in-stock products.',
          parameters: {
            inStockOnly: {
              type: 'boolean',
              description:
                'Set to true to show only in-stock products. Set to false to show all products.',
            },
          },
          handler: ({ inStockOnly }: { inStockOnly: boolean }) => {
            setFilters((prev) => ({ ...prev, inStockOnly }));
          },
        },
        {
          name: 'setSortBy',
          description: 'Set the sort order for the product list.',
          parameters: {
            sortBy: {
              type: 'string',
              description:
                'Sort option. Valid values: "relevance", "price_asc", "price_desc", "rating", "newest".',
            },
          },
          handler: ({ sortBy }: { sortBy: string }) => {
            setFilters((prev) => ({ ...prev, sortBy }));
          },
        },
        {
          name: 'applyFilters',
          description:
            'Apply multiple filters at once. Use this when the user specifies several filter criteria in a single request.',
          parameters: {
            category: { type: 'string', description: 'The product category.', optional: true },
            priceMin: { type: 'number', description: 'Minimum price in dollars.', optional: true },
            priceMax: { type: 'number', description: 'Maximum price in dollars.', optional: true },
            inStockOnly: { type: 'boolean', description: 'Whether to show only in-stock products.', optional: true },
            sortBy: { type: 'string', description: 'Sort order: "relevance", "price_asc", "price_desc", "rating", "newest".', optional: true },
          },
          handler: (params: Partial<FilterState>) => {
            setFilters((prev) => ({ ...prev, ...params }));
          },
        },
        {
          name: 'resetFilters',
          description: 'Reset all filters to their default values.',
          parameters: {},
          handler: () => {
            setFilters({ category: '', priceMin: 0, priceMax: 10000, inStockOnly: false, sortBy: 'relevance' });
          },
        },
      ],
    });

    return () => {
      unregister(COMPONENT_ID);
    };
  }, [register, unregister]);

  return (
    <div className="product-filter-panel">
      {/* your JSX here */}
    </div>
  );
}
```

---

## Key Design Decisions

**1. Include a compound `applyFilters` action.**
When a user phrases a request as a single sentence with multiple criteria, Angi needs a single action that accepts all of them together.

**2. Write rich parameter descriptions.**
The `description` field on each parameter is what Angi's AI layer uses to understand valid values and semantics.

**3. Mark optional parameters.**
`optional: true` lets Angi call `applyFilters` with only the fields the user mentioned, leaving others unchanged.

**4. Clean up on unmount.**
The `return () => unregister(COMPONENT_ID)` in your `useEffect` cleanup ensures no stale registrations remain.
