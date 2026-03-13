# React / TypeScript Reference

## Design System

> **Primary reference:** Always consult the live ALS Design System at
> **https://als-computing.github.io/design-system/** and its Storybook at
> **https://als-computing.github.io/design-system/docs/** first. These are the
> authoritative sources for colors, icons, components, and patterns.

### ALS Color Palette

Colors come from the `@als-computing/colors` npm package. Never hardcode hex values.

```bash
npm i @als-computing/colors
```

```tsx
import { colors } from "@als-computing/colors"

// Primary Sky (dark → light blue)
colors.primary.sky   // ['#073048', '#004C70', '#095B87', '#0886CA']

// Primary Slate (dark → light grey-blue)
colors.primary.slate // ['#464B53', '#334256', '#64768D', '#C1D3E3']
```

#### Primary palette — Sky

| Hex | Use |
|---|---|
| `#073048` | Darkest — deep navy backgrounds, sidebar |
| `#004C70` | Dark blue — primary actions, headers |
| `#095B87` | Mid blue |
| `#0886CA` | Light blue — highlights, hover states |

#### Primary palette — Slate

| Hex | Use |
|---|---|
| `#464B53` | Dark — text, borders |
| `#334256` | Dark grey-blue |
| `#64768D` | Mid grey-blue |
| `#C1D3E3` | Light — subtle backgrounds, muted borders |

#### Finch / secondary colors

Used for interactive borders, buttons, and focus states. Two colors per interaction
(default + selected/hover):

| Hex | Notes |
|---|---|
| `#004B71` | Default border / selected state |
| `#F0F5F9` | Light background surface |
| `#52BEE2` | Bright accent blue |
| `#7DA9CD` | Mid accent |
| `#9ACFFC` | Light accent |
| `#47484A` | Dark neutral |
| `#CED7DC` | Light neutral border |
| `#0E3B55` | Deep dark blue |

### Icons

Icons come from the `@als-computing/icons` npm package:

```bash
npm i @als-computing/icons
```

```tsx
import { Scan } from "@als-computing/icons"

function Example() {
  return <Scan aria-label="Scan" />
}
```

Browse all available icons in Storybook under **Foundations → Icons**.

### Token system

Map palette values to semantic Tailwind tokens in `tailwind.config.js`:

```js
// tailwind.config.js
export default {
  darkMode: 'class',
  theme: {
    extend: {
      colors: {
        // Pull from @als-computing/colors
        'sky-darkest':  '#073048',
        'sky-dark':     '#004C70',
        'sky-mid':      '#095B87',
        'sky-light':    '#0886CA',
        'slate-dark':   '#464B53',
        'slate-mid':    '#64768D',
        'slate-light':  '#C1D3E3',
        // Finch interactive colors
        'finch-border':         '#004B71',
        'finch-border-hover':   '#52BEE2',
        'finch-surface':        '#F0F5F9',
      }
    }
  }
}
```

Then use token names in components, never raw hex:

```tsx
// ✅ Semantic token
<div className="bg-sky-dark text-white">
<button className="border border-finch-border hover:border-finch-border-hover">

// ❌ Hardcoded hex
<div style={{ backgroundColor: '#004C70' }}>
```

### Tailwind CSS

Use Tailwind utility classes for all styling. The project uses `darkMode: 'class'`
— always add `dark:` variants for dark mode support.

```tsx
// ✅ Light + dark mode
<div className="bg-white dark:bg-[#2d3748] text-[#1f1f1f] dark:text-white">

// ✅ Using token classes
<div className="bg-sky-darkest dark:bg-slate-dark">
```

### shadcn/ui

Use shadcn components for complex base UI (date pickers, dialogs, dropdowns, etc.).
The project uses CSS variables (`--primary`, `--secondary`, etc.) defined in
`tailwind.css` for shadcn theming. Extend rather than replace.

---

## Finch (`@blueskyproject/finch`)

Finch is the ALS React component library for Bluesky beamlines, built on Tailwind + shadcn.

- Check Storybook before creating new components — reuse what exists
- New pages must follow Finch's page layout patterns
- New components must integrate visually with existing Finch pages

---

## Tiled API Calls (`@blueskyproject/tiled`)

The `@blueskyproject/tiled` package exports API functions for interacting with
a Tiled server. These functions:

- Accept a **`config` argument** containing base URL and auth credentials
- Use a **shared axios instance** with interceptors for automatic auth token injection
- Are the **only** approved way to make requests to a Tiled server

### Never do this:
```tsx
// ❌ Raw fetch or axios — bypasses auth token injection
const data = await fetch(`${tiledBaseUrl}/api/v1/node/${id}`);
import axios from 'axios';
const data = await axios.get(`${tiledBaseUrl}/api/v1/node/${id}`);
```

### Always do this:
```tsx
// ✅ Use exported API functions
import { getNode, fetchMetadata } from '@blueskyproject/tiled';

const config = { baseUrl: tiledBaseUrl, apiKey };
const node = await getNode(nodeId, config);
```

### Config shape
```ts
interface TiledConfig {
  baseUrl: string;       // e.g. 'http://tiled-server:8000/api/v1'
  apiKey?: string;       // static API key (dev/testing)
  bearerToken?: string;  // OIDC bearer token (production)
}
```

### Current exported API functions

> This list reflects `@blueskyproject/tiled` as of early 2026 (PR #10).
> Always check the package exports for the latest functions.

| Function | Description |
|---|---|
| `getNode(id, config)` | Fetch a node by ID |
| `fetchMetadata(id, config)` | Fetch metadata for a node |
| `fetchTableData(id, config)` | Fetch table/dataframe contents |
| `searchNodes(query, config)` | Search nodes by metadata or spec |

### Wrapping with TanStack Query

All Tiled API calls in components must be wrapped with TanStack Query:

```tsx
import { useQuery } from '@tanstack/react-query';
import { getNode } from '@blueskyproject/tiled';

function ScanNode({ nodeId, config }) {
  const { data, isLoading, error } = useQuery({
    queryKey: ['tiled-node', nodeId],
    queryFn: () => getNode(nodeId, config),
  });

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorMessage error={error} />;
  return <NodeView data={data} />;
}
```

---

## JSDoc

Add JSDoc to all components, hooks, and utility functions:

```tsx
/**
 * Displays a scan metadata summary card.
 *
 * @param nodeId - The Tiled node ID to display metadata for.
 * @param config - Tiled server connection configuration.
 * @param onSelect - Optional callback fired when the user selects the node.
 */
export function ScanCard({ nodeId, config, onSelect }: ScanCardProps) { ... }
```

---

## Testing with Vitest

Every new component must have a Vitest test file with at least one test per prop:

```tsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import { ScanCard } from './ScanCard';

describe('ScanCard', () => {
  it('renders the node ID', () => {
    render(<ScanCard nodeId="abc123" config={mockConfig} />);
    expect(screen.getByText('abc123')).toBeInTheDocument();
  });

  it('calls onSelect when clicked', async () => {
    const onSelect = vi.fn();
    render(<ScanCard nodeId="abc123" config={mockConfig} onSelect={onSelect} />);
    await userEvent.click(screen.getByRole('button'));
    expect(onSelect).toHaveBeenCalledWith('abc123');
  });
});
```

---

## Routing & TanStack Query Setup

React Router v7 for routing; TanStack Query for all server state:

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { createBrowserRouter, RouterProvider } from 'react-router';

const queryClient = new QueryClient({
  defaultOptions: { queries: { staleTime: 1000 * 60 * 5 } },
});

const router = createBrowserRouter([
  { path: '/', element: <HomePage /> },
  { path: '/scan/:nodeId', element: <ScanPage /> },
]);

export function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <RouterProvider router={router} />
    </QueryClientProvider>
  );
}
```