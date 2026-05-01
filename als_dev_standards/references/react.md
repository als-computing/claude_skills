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

## Component API Design

### Name all prop combinations explicitly

When a component has multiple boolean style props (e.g. `isSecondary`, `active`), define all combinations as named variants in an object rather than using nested ternaries. This makes every visual state explicit and prevents combinations from being silently dropped.

```tsx
// ✅ Named variants — all combinations are visible and intentional
const buttonVariants = {
    primary:         'bg-sky-500 text-white hover:bg-sky-600',
    primaryActive:   'bg-sky-700 text-white hover:bg-sky-800',
    secondary:       'bg-transparent text-black border hover:bg-slate-100',
    secondaryActive: 'bg-slate-200 text-black border hover:bg-slate-300',
};

const getButtonClasses = (active?: boolean, isSecondary?: boolean) => {
    if (active && isSecondary) return buttonVariants.secondaryActive;
    if (active)                return buttonVariants.primaryActive;
    if (isSecondary)           return buttonVariants.secondary;
    return buttonVariants.primary;
};

// ❌ Nested ternaries — secondaryActive combination is silently missing
const classes = active
    ? 'bg-sky-700 text-white'
    : isSecondary
    ? 'bg-transparent text-black border'
    : 'bg-sky-500 text-white';
```

### Apply patterns consistently across sibling components

If a pattern (variant logic, ARIA attributes, disabled styling) is applied to one component in a family, apply it to all siblings with the same props. Inconsistency across `Button`, `ButtonIconOnly`, `ButtonWithIcon` etc. creates unpredictable behavior for consumers.

### Prefer `variant` over multiple booleans at major version boundaries

`isSecondary?: boolean` works fine for small prop sets. If a component grows beyond 2–3 style booleans, migrate to `variant: 'primary' | 'secondary' | 'ghost'` at the next major version bump — this prevents conflicting prop combinations and scales cleanly. Never make this change in a minor/patch release as it breaks the public API.

---

## Accessibility

### Toggle buttons must have `aria-pressed`

Any button that can be in an active/pressed/selected state must include `aria-pressed`. Pass the prop value directly — when the prop is `undefined` the attribute is absent entirely, which is correct (don't label a non-toggle button as a toggle).

```tsx
// ✅ aria-pressed is set when active=true, absent when active=undefined
<button aria-pressed={active} ...>

// ❌ Hard-coding false adds an incorrect implicit toggle role
<button aria-pressed={active ?? false} ...>
```

### Disabled elements must have visual styling

The HTML `disabled` attribute prevents interaction but provides no visual feedback. Always pair it with `opacity-50 cursor-not-allowed` so users can see the element is not interactive.

```tsx
// ✅
<button
    disabled={disabled}
    className={`... ${disabled ? 'opacity-50 cursor-not-allowed' : ''}`}
>

// ❌ Functionally disabled but visually identical to enabled
<button disabled={disabled} className="...">
```

---

## Testing with Vitest

Every new component must have a Vitest test file. Test each prop and each combination of props that produces a distinct visual or behavioral state.

### Test each variant combination independently

Don't test "A overrides B" — test each state on its own with both positive (`toHaveClass`) and negative (`not.toHaveClass`) assertions. This catches regressions in any individual state without relying on override order.

```tsx
// ✅ One test per combination, positive + negative assertions
it('applies primary styling when not active and not secondary', () => {
    const { container } = render(<Button text="Primary" />);
    expect(container.firstChild).toHaveClass('bg-sky-500', 'text-white');
    expect(container.firstChild).not.toHaveClass('bg-sky-700', 'bg-transparent');
});

it('applies primary active styling when active is true', () => {
    const { container } = render(<Button text="Primary Active" active={true} />);
    expect(container.firstChild).toHaveClass('bg-sky-700', 'text-white');
    expect(container.firstChild).not.toHaveClass('bg-sky-500', 'bg-transparent');
});

it('applies secondary active styling when both active and isSecondary are true', () => {
    const { container } = render(<Button text="Secondary Active" active={true} isSecondary={true} />);
    expect(container.firstChild).toHaveClass('bg-slate-200', 'text-black', 'border');
    expect(container.firstChild).not.toHaveClass('bg-sky-500', 'bg-sky-700', 'bg-transparent');
});

// ❌ Tests override behavior — will pass even if secondaryActive has wrong styles
it('active overrides isSecondary styling', () => {
    const { container } = render(<Button active={true} isSecondary={true} />);
    expect(container.firstChild).toHaveClass('bg-sky-700');
    expect(container.firstChild).not.toHaveClass('bg-transparent');
});
```

### Test ARIA attributes explicitly

```tsx
it('sets aria-pressed when active is true', () => {
    render(<Button text="Active" active={true} />);
    expect(screen.getByRole('button')).toHaveAttribute('aria-pressed', 'true');
});

it('does not set aria-pressed when active is not provided', () => {
    render(<Button text="Default" />);
    expect(screen.getByRole('button')).not.toHaveAttribute('aria-pressed');
});
```

### General Vitest patterns

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

## Storybook

Every named visual variant must have a corresponding Storybook story. If a component has 4 visual states, there must be 4 stories — one per state. Stories serve as living documentation and catch visual regressions that unit tests miss.

```tsx
// ✅ One story per variant — all 4 combinations covered
export const Primary: Story = { args: { text: 'primary' } };
export const Active: Story = { args: { text: 'active', active: true } };
export const Secondary: Story = { args: { text: 'secondary', isSecondary: true } };
export const ActiveSecondary: Story = { args: { text: 'active secondary', active: true, isSecondary: true } };
export const Disabled: Story = { args: { text: 'disabled', disabled: true } };

// ❌ Missing ActiveSecondary — that visual state is undocumented and unreviewed
export const Primary: Story = { args: { text: 'primary' } };
export const Active: Story = { args: { text: 'active', active: true } };
export const Secondary: Story = { args: { text: 'secondary', isSecondary: true } };
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