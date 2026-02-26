# UI Compiler Plugin

## Overview

This document specifies the QDS UI Plugin: a Vite/Rolldown build plugin that detects when consumers reference QDS component namespace state properties (e.g., `select.selectedLabels`) inside JSX and automatically generates `component$()` boundaries so those references have proper Qwik context access.

File location: `libs/tools/rolldown/ui.ts`
Plugin name: `vite-plugin-qds-ui`

---

## Section 1: Problem Statement

### The Qwik Context Boundary Constraint

Qwik's context system requires that `useContext()` is called inside a `component$()` that is a descendant of the context provider in the component tree. This is a fundamental constraint of Qwik's resumable architecture — context lookup walks up the virtual DOM tree at runtime, and that walk only works within component boundaries.

For consumers of QDS components, this creates a friction point. Consider the `select` component: internally, `SelectRoot` calls `useContextProvider(selectContextId, context)` and all child components like `SelectValueLabel` call `useContext(selectContextId)` inside their own `component$()`. This works because `SelectValueLabel` is rendered as a descendant of `SelectRoot`.

But what if a consumer wants to read the selected labels to build a custom trigger display? They cannot call `useContext` at the parent scope level:

```tsx
// THIS DOES NOT WORK
// useContext cannot be called here — this is the parent component's scope,
// not a descendant of the select provider.
import { select } from "@qds.dev/ui";
import { component$, useContext } from "@qwik.dev/core";
import { selectContextId } from "@qds.dev/ui/select-root"; // not even public

const MySelect = component$(() => {
  const ctx = useContext(selectContextId); // Error: context not found
  return (
    <select.root>
      <select.trigger>{ctx.selectedValues.value}</select.trigger>
      ...
    </select.root>
  );
});
```

The context provider lives inside `<select.root>`. Any `useContext` call must be inside a `component$()` that is rendered _inside_ that root, not alongside or above it.

### This Problem Is Universal Across Frameworks

This is not unique to Qwik. Every headless UI library across every framework has this same fundamental constraint: the Root component owns the state, but consumers want to access that state from the parent scope — above Root, not inside it.

In React, the workaround is controlled state — lift everything up to the parent:

```tsx
// React workaround: lift state to parent
const [value, setValue] = useState([]);

return (
  <div>
    <span>{value.length} selected</span>
    <Select.Root value={value} onValueChange={setValue}>
      ...
    </Select.Root>
  </div>
);
```

This defeats the purpose of the component managing its own state. The consumer is now responsible for state management, event wiring, and keeping things in sync. Every headless library's GitHub issues are full of "how do I access X from outside Root" questions.

In Qwik, the controlled state workaround exists too, but it's even more constrained: render props and callback-as-child patterns don't work because closures can't be serialized across the resumability boundary. There is no clean runtime-only escape hatch.

No headless UI library — React, Solid, Svelte, or Qwik — has solved this at the build tool level. This plugin would be a genuinely novel approach that eliminates the problem entirely.

### The Current Workaround

Consumers who need to access select state today must create a manual wrapper component that lives inside the tree:

```tsx
// CURRENT WORKAROUND: manually create a child component$ inside the tree
import { select } from "@qds.dev/ui";
import { component$, useContext } from "@qwik.dev/core";
import { selectContextId } from "..."; // not a public API

const SelectedDisplay = component$(() => {
  const ctx = useContext(selectContextId);
  const labels = ctx.navigation.value?.getSelectedLabels({
    selectedValues: ctx.selectedValues.value,
    isMultiple: ctx.multiple
  });
  return <span>{labels?.join(", ")}</span>;
});

const MySelect = component$(() => {
  return (
    <select.root>
      <select.trigger>
        <SelectedDisplay /> {/* must be inside root to access context */}
      </select.trigger>
      <select.content>...</select.content>
    </select.root>
  );
});
```

This is poor DX for three reasons:

1. Consumers must know the internal `selectContextId` export — a private implementation detail.
2. Consumers must understand Qwik's context boundary rules.
3. Consumers must write a separate component for what should be a simple inline expression.

### How QDS Solves This Internally

`SelectValueLabel` (`libs/components/src/select/select-value-label.tsx`) is QDS's internal solution: it wraps the context access in its own `component$()` so it can call `useContext(selectContextId)`. Consumers use `<select.valuelabel />` to render the current selection.

But `SelectValueLabel` only covers a single rendering pattern. Consumers who want different formatting — chip-style multi-select, custom placeholders, conditional rendering based on count — cannot extend it without creating their own wrapper.

### The Solution

The Component State Compiler Plugin transforms consumer code at build time. Consumers write state access expressions that look like top-level calls, and the plugin generates the necessary `component$()` wrappers automatically.

The consumer writes code that looks like:

```tsx
const { selectedLabels } = select;

<select.trigger>{selectedLabels.value.join(", ")}</select.trigger>;
```

The plugin transforms it to effectively be:

```tsx
<select.trigger>
  <_QdsSelectSelectedLabels0 />
</select.trigger>
```

Where `_QdsSelectSelectedLabels0` is a generated `component$()` that uses `useContext` to read the select context and renders the appropriate value. The consumer never sees this transformation — they write natural-looking JSX.

---

## Section 2: Consumer DX (Before/After)

### Import Pattern

Consumers import from the QDS namespace as they do today:

```tsx
import { select } from "@qds.dev/ui";
```

No additional imports are required. The plugin injects any needed context imports into the generated output.

### Before: Manual Wrapper Required

```tsx
import { select } from "@qds.dev/ui";
import { component$, useContext } from "@qwik.dev/core";
import { selectContextId } from "@qds.dev/ui/internals"; // hypothetical, not public

// Consumer must write this boilerplate
const ChipDisplay = component$(() => {
  const ctx = useContext(selectContextId);
  const labels = ctx.selectedLabels.value; // doesn't exist yet on context
  return (
    <>
      {labels.length > 3 ? (
        <span>{labels.length} selected</span>
      ) : (
        labels.map((l) => <span class="chip">{l}</span>)
      )}
    </>
  );
});

export const MySelect = component$(() => {
  return (
    <select.root multiple>
      <select.trigger>
        <ChipDisplay />
      </select.trigger>
      <select.content>
        <select.item value="a">
          <select.itemlabel>Alice</select.itemlabel>
        </select.item>
        <select.item value="b">
          <select.itemlabel>Bob</select.itemlabel>
        </select.item>
      </select.content>
    </select.root>
  );
});
```

### After: Inline State Access

```tsx
import { select } from "@qds.dev/ui";
import { component$ } from "@qwik.dev/core";

export const MySelect = component$(() => {
  return (
    <select.root multiple>
      <select.trigger>
        {select.selectedLabels.value.length > 3 ? (
          <span>{select.selectedLabels.value.length} selected</span>
        ) : (
          select.selectedLabels.value.map((l) => <span class="chip">{l}</span>)
        )}
      </select.trigger>
      <select.content>
        <select.item value="a">
          <select.itemlabel>Alice</select.itemlabel>
        </select.item>
        <select.item value="b">
          <select.itemlabel>Bob</select.itemlabel>
        </select.item>
      </select.content>
    </select.root>
  );
});
```

The compiler detects `select.selectedLabels` (a state property per the manifest), finds the JSX expression container that uses it, and wraps the entire expression in a generated `component$()`.

### Simple State Access

```tsx
// Consumer writes:
<p>You selected: {select.selectedLabels.value}</p>

// Compiler generates (conceptual):
<p>You selected: <_QdsSelectSelectedLabels0 /></p>
```

### isOpen State Access

```tsx
// Consumer writes:
<div>{select.isOpen.value ? "Open" : "Closed"}</div>

// Compiler generates:
<div><_QdsSelectIsOpen0 /></div>
```

### Aliasing

```tsx
// Consumer writes:
const labels = select.selectedLabels;
// ...later in JSX:
<span>{labels.value.join(", ")}</span>;

// Compiler tracks that `labels` aliases `select.selectedLabels`,
// detects the JSX expression, and generates the component$ wrapper.
```

### Destructuring

```tsx
// Consumer writes:
const { selectedLabels } = select;

// ...later in JSX:
<span>{selectedLabels.value.join(", ")}</span>;

// Compiler tracks that `selectedLabels` was destructured from `select`,
// detects the JSX expression, and generates the component$ wrapper.
```

Destructuring is a variant of alias tracking. The plugin walks `ObjectPattern` nodes in `VariableDeclarator`, checks if the initializer is a tracked namespace identifier, and records each destructured property as an alias binding — the same mechanism used for `const labels = select.selectedLabels`.

Only nested or renamed destructuring emits a build error:

```tsx
// ERROR: nested destructuring — cannot track
const { selectedLabels: { value: labels } } = select;
```

### Component Exports Are Unchanged

`select.root`, `select.trigger`, `select.content`, `select.item`, `select.itemlabel` are all component exports in the manifest. The compiler does not transform them. They continue to work exactly as JSX elements:

```tsx
// These are component exports — no transformation:
<select.root multiple>
<select.trigger>
<select.content>
<select.item value="x">
<select.itemlabel>Option</select.itemlabel>
```

---

## Section 3: Manifest Format

### Purpose

The manifest tells the plugin which properties of each QDS namespace are component exports versus state properties. Without the manifest, the plugin cannot distinguish `select.root` (a component) from `select.selectedLabels` (state).

### Format Definition

Each QDS component namespace has a manifest object. The manifest is a flat map of property name to descriptor:

```typescript
type ComponentDescriptor = "component";

type StateDescriptor = {
  type: "state";
  contextId: string; // The string ID passed to createContextId — e.g. "qds-select"
  contextIdExport: string; // The exported name of the context ID — e.g. "selectContextId"
  contextIdSource: string; // The package path to import contextId from
  field: string; // The field name on the context object
  signalType: string; // TypeScript type of the signal value for the generated component
  derived?: true; // True when the value requires computation beyond a direct field read
};

type ManifestEntry = ComponentDescriptor | StateDescriptor;

type ComponentManifest = Record<string, ManifestEntry>;
```

### Select Manifest (Reference Implementation)

```typescript
// libs/tools/rolldown/manifests/select.ts

import type { ComponentManifest } from "../ui-types";

export const selectManifest: ComponentManifest = {
  root: "component",
  trigger: "component",
  content: "component",
  item: "component",
  itemlabel: "component",
  label: "component",
  valuelabel: "component",
  itemindicator: "component",

  // State properties — these trigger the compiler transform
  isOpen: {
    type: "state",
    contextId: "qds-select",
    contextIdExport: "selectContextId",
    contextIdSource: "@qds.dev/ui",
    field: "isOpen",
    signalType: "boolean"
  },
  selectedLabels: {
    type: "state",
    contextId: "qds-select",
    contextIdExport: "selectContextId",
    contextIdSource: "@qds.dev/ui",
    field: "selectedLabels",
    signalType: "string[]",
    derived: true // computed from selectedValues + itemLabelText
  },
  selectedValues: {
    type: "state",
    contextId: "qds-select",
    contextIdExport: "selectContextId",
    contextIdSource: "@qds.dev/ui",
    field: "selectedValues",
    signalType: "string | string[]"
  },
  highlightedIndex: {
    type: "state",
    contextId: "qds-select",
    contextIdExport: "selectContextId",
    contextIdSource: "@qds.dev/ui",
    field: "highlightedIndex",
    signalType: "number | null"
  }
};
```

### Master Registry

All namespace manifests are collected into a master registry keyed by the import namespace name:

```typescript
// libs/tools/rolldown/manifests/index.ts

import { selectManifest } from "./select";
// import { checkboxManifest } from "./checkbox";
// ... other components

export const qdsManifests: Record<string, ComponentManifest> = {
  select: selectManifest
  // checkbox: checkboxManifest,
};
```

### Where Manifests Live

Manifests are static TypeScript files checked into `libs/tools/rolldown/manifests/`. They are authored by QDS component maintainers when adding state properties to the public API surface.

A future build step (not part of this spec) could generate manifests automatically from component context type definitions. For now, they are written by hand. The authoring burden is low: a new state property requires one entry in the manifest file plus a corresponding context field (see Section 5).

---

## Section 4: Compiler Transform (Detailed)

### Overview

The plugin transform runs on each `.tsx` or `.jsx` file. It:

1. Parses the file with oxc-parser.
2. Detects QDS namespace imports and maps them to manifests.
3. Walks the AST to find state property references.
4. Tracks aliases (variable assignments to state properties).
5. Finds JSX expression containers containing state references.
6. Generates replacement `component$()` declarations.
7. Rewrites the JSX expression containers to use the generated components.
8. Injects required imports (context ID, `component$`, `useContext`).

### Phase 1: Detection

Parse the file and find import declarations from `@qds.dev/ui`:

```typescript
// Input file contains:
import { select } from "@qds.dev/ui";

// Detection result:
// { localName: "select", manifest: selectManifest }
```

Walk `ImportDeclaration` nodes. For each specifier where `source.value === "@qds.dev/ui"`, check if the imported name exists in `qdsManifests`. If so, record:

```typescript
type BoundNamespace = {
  localName: string; // "select" (or alias like "mySelect")
  manifestKey: string; // "select" (the key in qdsManifests)
  manifest: ComponentManifest;
};
```

If no QDS state namespaces are found, return `null` (no transform needed).

### Phase 2: Reference Tracking

Walk the entire AST. For each `MemberExpression` where the object is a tracked namespace identifier and the property is a state descriptor in the manifest:

```typescript
// select.selectedLabels
// ^^^^^^ object: tracked namespace identifier
//        ^^^^^^^^^^^^^^ property: state descriptor in manifest

type StateReference = {
  node: MemberExpression;
  namespace: BoundNamespace;
  propertyName: string; // "selectedLabels"
  descriptor: StateDescriptor;
  start: number; // source position
  end: number; // source position
};
```

Collect all `StateReference` instances across the file.

If no state references found, return `null`.

### Phase 3: Alias and Destructuring Tracking

Walk `VariableDeclarator` nodes. Two patterns produce alias bindings:

**Direct assignment:**

```typescript
// const labels = select.selectedLabels;
//       ^^^^^^   ^^^^^^^^^^^^^^^^^^^^^ initializer: state member expression
//       |
//       alias identifier
```

**Destructuring:**

```typescript
// const { selectedLabels, isOpen } = select;
//        ^^^^^^^^^^^^^^  ^^^^^^     ^^^^^^ initializer: tracked namespace identifier
//        |               |
//        alias for select.selectedLabels
//                        alias for select.isOpen
```

For destructuring, walk `ObjectPattern` nodes inside `VariableDeclarator`. If the `init` is a tracked namespace identifier, check each destructured property name against the manifest. For each property that is a state descriptor, record an `AliasBinding`. If any property has a nested `ObjectPattern` (deep destructuring), emit a build error — deep destructuring cannot be tracked.

Both patterns produce the same record:

```typescript
type AliasBinding = {
  localName: string; // "labels" or "selectedLabels"
  namespace: BoundNamespace;
  propertyName: string; // "selectedLabels"
  descriptor: StateDescriptor;
};
```

After tracking aliases, walk again to find uses of alias identifiers in JSX. Treat alias references identically to direct member expression references for the purpose of JSX wrapping.

If an alias is later reassigned (detected via another `VariableDeclarator` or `AssignmentExpression` targeting the same name), emit a build warning and skip wrapping for that alias — the tracking is broken by reassignment.

### Phase 4: JSX Expression Detection

Walk `JSXExpressionContainer` nodes. For each container, check if its expression subtree contains any `StateReference` node (by position overlap or identity). Also check if it contains any alias identifier that maps to a `StateReference`.

When a JSX expression container contains a state reference, mark it as a transform target:

```typescript
type TransformTarget = {
  container: JSXExpressionContainer;
  stateRefs: StateReference[]; // all state refs within this container
  start: number;
  end: number;
};
```

A single JSX expression container may reference multiple state properties (e.g., `{select.isOpen.value && select.selectedLabels.value.join(", ")}`). In that case, one generated component handles all state access within that container.

### Phase 5: Code Generation

For each `TransformTarget`, generate a unique component name:

```typescript
function generateComponentName(
  namespace: string,
  propertyName: string,
  index: number
): string {
  // "_QdsSelectSelectedLabels0"
  const ns = namespace[0].toUpperCase() + namespace.slice(1);
  const prop = propertyName[0].toUpperCase() + propertyName.slice(1);
  return `_Qds${ns}${prop}${index}`;
}
```

When a single container references multiple state properties, use the first property name in the generated name.

Generate the component body. For each state reference within the container, replace the member expression with a context field access:

```typescript
// State reference: select.selectedLabels
// Generated replacement: ctx.selectedLabels
```

The full expression from the original JSX container is preserved, but with `select.selectedLabels` replaced by `ctx.selectedLabels`:

```typescript
// Input container expression:
select.selectedLabels.value.length > 3
  ? <span>{select.selectedLabels.value.length} selected</span>
  : select.selectedLabels.value.map(l => <span class="chip">{l}</span>)

// Generated component body:
const _QdsSelectSelectedLabels0 = component$(() => {
  const ctx = useContext(selectContextId);
  return (
    <>
      {ctx.selectedLabels.value.length > 3
        ? <span>{ctx.selectedLabels.value.length} selected</span>
        : ctx.selectedLabels.value.map(l => <span class="chip">{l}</span>)
      }
    </>
  );
});
```

When a container references state from multiple namespaces (e.g., `select` and `checkbox`), generate one `useContext` call per unique namespace:

```typescript
const _QdsMultiState0 = component$(() => {
  const selectCtx = useContext(selectContextId);
  const checkboxCtx = useContext(checkboxContextId);
  return <>{selectCtx.selectedLabels.value.length + checkboxCtx.checked.value}</>;
});
```

### Phase 6: Source Rewriting

Use `MagicString` to rewrite the original source:

1. For each `TransformTarget`, overwrite the JSX expression container with a self-closing JSX element:

   ```typescript
   s.overwrite(target.start, target.end, `<${generatedName} />`);
   ```

2. Append all generated component declarations after the last existing import declaration in the file (before the first non-import statement), so the Qwik optimizer sees them as module-scope `component$` calls.

3. Inject missing imports at the top of the file. Track which imports are already present in the AST before injecting:
   - `component$` from `@qwik.dev/core` (if not already imported)
   - `useContext` from `@qwik.dev/core` (if not already imported)
   - Each `contextIdExport` from its `contextIdSource` (if not already imported)

### Complete Transform Example

Input:

```tsx
import { select } from "@qds.dev/ui";
import { component$ } from "@qwik.dev/core";

export const MySelect = component$(() => {
  return (
    <select.root multiple>
      <select.trigger>
        {select.selectedLabels.value.length > 3 ? (
          <span>{select.selectedLabels.value.length} selected</span>
        ) : (
          select.selectedLabels.value.map((l) => <span class="chip">{l}</span>)
        )}
      </select.trigger>
      <select.content>
        <select.item value="a">
          <select.itemlabel>Alice</select.itemlabel>
        </select.item>
      </select.content>
    </select.root>
  );
});
```

Output:

```tsx
import { select } from "@qds.dev/ui";
import { component$, useContext } from "@qwik.dev/core";
import { selectContextId } from "@qds.dev/ui";

const _QdsSelectSelectedLabels0 = component$(() => {
  const ctx = useContext(selectContextId);
  return (
    <>
      {ctx.selectedLabels.value.length > 3 ? (
        <span>{ctx.selectedLabels.value.length} selected</span>
      ) : (
        ctx.selectedLabels.value.map((l) => <span class="chip">{l}</span>)
      )}
    </>
  );
});

export const MySelect = component$(() => {
  return (
    <select.root multiple>
      <select.trigger>
        <_QdsSelectSelectedLabels0 />
      </select.trigger>
      <select.content>
        <select.item value="a">
          <select.itemlabel>Alice</select.itemlabel>
        </select.item>
      </select.content>
    </select.root>
  );
});
```

### Why Standalone component$ at Module Scope

The Qwik optimizer requires `component$()` calls to appear at module scope so it can extract them into separate lazy-loaded chunks. Immediately invoked patterns or inline component expressions would confuse the optimizer. Generated components must be declared as top-level `const` declarations in the module, not nested inside other components or expressions.

The plugin appends generated components after the import block but before the first exported declaration. This placement ensures:

1. The Qwik optimizer sees them as module-scope components.
2. Source maps remain accurate (MagicString preserves original positions).
3. No circular reference — generated components reference context IDs, not other consumer code.

### Deduplication

If the same JSX expression container appears in multiple transform passes (should not happen normally, but guard against it), the plugin must deduplicate by checking if a container's start/end range already has a generated component.

If the same state property is referenced in multiple JSX expression containers in the same file, generate a separate component for each container (each gets a unique index suffix).

---

## Section 5: Runtime Requirements

### What the Context Must Expose

For the generated component to be minimal and correct, each state property referenced in the manifest must correspond to a direct field on the context object that the generated component can read.

For fields that already exist on the context as signals (like `isOpen`, `selectedValues`, `highlightedIndex`), no changes are needed. The generated component reads them directly.

For derived state (like `selectedLabels`, which must be computed from `selectedValues` + `itemLabelText` + navigation helpers), the context must expose a pre-computed signal rather than requiring the generated component to duplicate derivation logic.

### Required Changes to SelectContext

Add a `selectedLabels` computed signal to `SelectContext` in `libs/components/src/select/select-root.tsx`:

```typescript
export type SelectContext = {
  localId: string;
  isOpen: Signal<boolean>;
  itemRefs: Signal<Array<Signal<HTMLElement | undefined>>>;
  itemLabelText: Signal<string[]>;
  itemValues: Signal<string[]>;
  disabledItems: Signal<boolean[]>;
  selectedValues: Signal<string | string[]>;
  highlightedIndex: Signal<number | null>;
  currentIndex: Signal<number | null>;
  multiple: boolean;
  isDisabled: Signal<boolean>;
  currItemIndex: number;
  totalItems: Signal<number>;
  isDistinctValue: Signal<boolean>;
  navigation: Signal<SelectNavigation | null>;
  typeahead: Signal<SelectTypeahead | null>;
  // New: pre-computed derived state for compiler-generated components
  selectedLabels: Signal<string[]>;
};
```

In `SelectRoot`, compute this signal:

```typescript
const selectedLabels = useComputed$(() => {
  if (!navigation.value) return [];
  return navigation.value.getSelectedLabels({
    selectedValues: selectedValues.value,
    isMultiple: multiple
  });
});
```

And include it in the context object:

```typescript
const context: SelectContext = {
  // ...existing fields
  selectedLabels
};
```

### Why Pre-Computed Signals on Context

The alternative — putting derivation logic inside each generated component — creates several problems:

1. Generated code becomes larger and harder to reason about.
2. Derivation logic must be duplicated into the plugin (or templated), coupling the plugin to component internals.
3. Multiple generated components in the same file would each run the derivation independently, wasting work.

With pre-computed signals, the generated component is always minimal:

```typescript
const ctx = useContext(selectContextId);
// ctx.selectedLabels is already a Signal<string[]>
// No additional computation required
```

### Existing State Properties That Already Work

The following `SelectContext` fields are already signals and need no changes:

- `isOpen: Signal<boolean>` — maps to `select.isOpen`
- `selectedValues: Signal<string | string[]>` — maps to `select.selectedValues`
- `highlightedIndex: Signal<number | null>` — maps to `select.highlightedIndex`

### Pattern for New Components

When adding state properties to other components (checkbox, menu, etc.), component authors must:

1. Ensure the field is a `Signal<T>` on the context type.
2. For derived state, add a `useComputed$()` in the root component and expose it on the context.
3. Add the corresponding entry to the manifest file.
4. Export the `contextId` from the component's `index.ts` so the plugin can inject the import.

---

## Section 6: Edge Cases

### Aliasing

```tsx
const labels = select.selectedLabels;

// Later in JSX:
<p>{labels.value.join(", ")}</p>;
```

The plugin tracks `labels` as an alias for `select.selectedLabels`. When it encounters `labels.value` inside a JSX expression container, it generates the same wrapping as for a direct `select.selectedLabels.value` reference.

The generated component replaces `labels.value` with `ctx.selectedLabels.value`:

```tsx
const _QdsSelectSelectedLabels0 = component$(() => {
  const ctx = useContext(selectContextId);
  return <>{ctx.selectedLabels.value.join(", ")}</>;
});

// Rewritten JSX:
<p>
  <_QdsSelectSelectedLabels0 />
</p>;
```

The original `const labels = select.selectedLabels;` declaration is left in place. It will be dead code after the transform, but that is acceptable — tree-shaking or minification removes it in production.

### Destructuring

```tsx
const { selectedLabels } = select;
```

Destructuring is supported via the same mechanism as alias tracking. The plugin walks `ObjectPattern` nodes in `VariableDeclarator`. When the initializer (`init`) is a tracked namespace identifier, each `BindingProperty` in the `ObjectPattern.properties` array provides a `key` (the original property name) and `value` (the local binding). The plugin records each destructured property as an `AliasBinding` — the same record type used for `const labels = select.selectedLabels`.

For the example above, `selectedLabels` is recorded as an alias for `select.selectedLabels`. When used in JSX:

```tsx
<span>{selectedLabels.value.join(", ")}</span>
```

The generated component replaces it with the context reference:

```tsx
const _QdsSelectSelectedLabels0 = component$(() => {
  const ctx = useContext(selectContextId);
  return <>{ctx.selectedLabels.value.join(", ")}</>;
});
```

Renamed destructuring also works:

```tsx
const { selectedLabels: labels } = select;
// `labels` is tracked as an alias for `select.selectedLabels`
```

Only **nested destructuring** emits a build error — deep patterns cannot be reliably tracked:

```tsx
// ERROR: nested destructuring
const { selectedLabels: { value: labels } } = select;
```

Detection: Walk `ObjectPattern` nodes inside `VariableDeclarator`. If the `init` is a tracked namespace identifier, check each destructured property name against the manifest. For properties with a nested `ObjectPattern` value (deep destructuring), emit the build error. For simple and renamed destructuring, record an `AliasBinding`.

This is consistent with Phase 3 (Section 4) which describes alias and destructuring tracking as a unified mechanism, and with Section 2 which shows the consumer-facing DX.

### Prop Passing

```tsx
<MyComponent labels={select.selectedLabels} />
```

This case passes a state property as a prop to another component. The consumer likely intends to pass the `Signal<string[]>` to `MyComponent`, which will read `.value` inside its own `component$()`.

This case is handled by the same JSX expression container detection: the prop value `select.selectedLabels` is a `JSXExpressionContainer`. The plugin wraps it:

```tsx
<MyComponent labels={<_QdsSelectSelectedLabels0 />} />
```

But this changes the type of the prop from `Signal<string[]>` to a JSX element, which is almost certainly wrong.

The correct behavior for this case is to emit a build-time error or warning:

```
[vite-plugin-qds-ui] QDS state property 'select.selectedLabels'
is used as a JSX prop value for '<MyComponent labels={...}>'.

State properties cannot be passed as props to other components because they
require a Qwik context boundary. Options:
  1. Access the state inside <MyComponent> using the compiler transform.
  2. Use bind: binding if MyComponent supports it.
  3. Pass the value (not the signal): {select.selectedLabels.value}
     Note: passing .value passes a snapshot, not a reactive signal.
```

Detection: Check if the `JSXExpressionContainer` is a `JSXAttribute` value (i.e., its parent is a `JSXAttribute` node rather than a `JSXElement` child). If so, emit the error instead of transforming.

### Multiple Instances

When two `<select.root>` elements appear on the same page:

```tsx
<select.root id="select-a">
  <select.trigger>
    {select.selectedLabels.value}  {/* refers to which select? */}
  </select.trigger>
</select.root>

<select.root id="select-b">
  <select.trigger>
    {select.selectedLabels.value}  {/* refers to which select? */}
  </select.trigger>
</select.root>
```

The generated `component$()` calls `useContext(selectContextId)`, which resolves to the nearest ancestor context provider in the render tree. Since each generated component is rendered inside a different `<select.root>`, each reads from its own context. This is the correct behavior by default.

No special handling is required for multiple instances.

### SSR

Generated components use `useContext()`, which works correctly during SSR in Qwik. The context provider (`SelectRoot`) renders server-side and provides the context, and generated child components read from it during the same SSR pass.

One caveat: the `SelectValueLabel` component uses a `useTask$` that checks `isServer` before performing DOM operations on `containerRef`. Generated components that reference `navigation.value` (which uses `useSerializer$` and may not be fully initialized during SSR) must account for this.

For the `selectedLabels` pre-computed signal approach (recommended in Section 5), the `useComputed$` in `SelectRoot` uses `navigation.value.getSelectedLabels()`. During SSR, `navigation` is initialized via `useSerializer$` with `initial: null`. This means `selectedLabels` computes to `[]` during SSR, which is correct initial state.

If a consumer writes:

```tsx
{
  select.selectedLabels.value.join(", ");
}
```

On SSR the generated component renders `""` (empty string from `[].join(", ")`). On the client, after hydration, the context updates and the component re-renders with actual labels. This matches the existing `SelectValueLabel` behavior.

### Conditional Rendering

```tsx
{
  isActive && select.selectedLabels.value;
}
```

The entire JSX expression container is the transform target. The generated component includes the full condition:

```typescript
const _QdsSelectSelectedLabels0 = component$(() => {
  const ctx = useContext(selectContextId);
  return <>{isActive && ctx.selectedLabels.value}</>;
});
```

`isActive` is captured from the parent component scope. This is valid because Qwik can capture signals and values from parent scope through the optimizer's `$`-boundary serialization.

If `isActive` is a signal from the parent scope (`isActive.value`), the generated component captures the signal itself (not the value snapshot), and reactivity is preserved.

### Template Literals

```tsx
{
  `You chose: ${select.selectedLabels.value.join(", ")}`;
}
```

The JSX expression container wraps the template literal. The generated component includes the template literal with the state reference replaced:

```typescript
const _QdsSelectSelectedLabels0 = component$(() => {
  const ctx = useContext(selectContextId);
  return <>{`You chose: ${ctx.selectedLabels.value.join(", ")}`}</>;
});
```

### Nested State Access (Method Calls)

```tsx
{
  select.selectedLabels.value.join(", ");
}
{
  select.selectedLabels.value.length;
}
{
  select.selectedLabels.value.map((l) => <span>{l}</span>);
}
```

All of these are member accesses or method calls on `.value`. The state reference is `select.selectedLabels` (the `MemberExpression` at the namespace property level). The plugin replaces `select.selectedLabels` with `ctx.selectedLabels` and leaves the rest of the access chain unchanged:

- `select.selectedLabels.value.join(", ")` → `ctx.selectedLabels.value.join(", ")`
- `select.selectedLabels.value.length` → `ctx.selectedLabels.value.length`
- `select.selectedLabels.value.map(...)` → `ctx.selectedLabels.value.map(...)`

The replacement is purely textual at the `select.selectedLabels` node boundaries (start/end position), not at deeper access levels.

### State Access Outside JSX

```tsx
const handleClick$ = $(() => {
  console.log(select.selectedLabels.value); // inside event handler
});
```

Event handlers run inside the consumer's `component$()` scope, but `useContext()` cannot be called inside `$()` callbacks. However, the handler can capture a value from a signal that was read inside the component.

The plugin must NOT transform state access inside `$()` boundaries or event handlers. This is out of scope for the compiler plugin. Detection rule: if a state reference is inside a `CallExpression` whose callee ends with `$`, skip it.

For this case, the consumer should either:

1. Use the state through a JSX expression (the compiler handles it), then pass the value via event data.
2. Create their own `component$()` wrapper if they need context in handlers.

Emit a build warning (not error) when state access is found inside `$()` boundaries, directing the consumer to the correct pattern.

---

## Section 7: Plugin Architecture

### File Location

```
libs/tools/rolldown/ui.ts
libs/tools/rolldown/manifests/
libs/tools/rolldown/manifests/index.ts
libs/tools/rolldown/manifests/select.ts
```

### Export from Index

Add to `libs/tools/rolldown/index.ts`:

```typescript
export type { UiPluginOptions } from "./ui";
export { ui } from "./ui";
```

### Plugin Signature

Following the same pattern as `icons.ts` and `as-child.ts`:

```typescript
export type UiPluginOptions = {
  debug?: boolean;
  importSources?: string[]; // defaults to ["@qds.dev/ui"]
};

export const ui = (
  options: UiPluginOptions = {}
): Plugin & { enforce?: "pre" | "post" } => {
  const importSources = options.importSources ?? ["@qds.dev/ui"];

  const isTSXOrJSX: RegExp = createRegExp(
    exactly(".").and(anyOf("tsx", "jsx")).at.lineEnd()
  );

  return {
    name: "vite-plugin-qds-ui",
    enforce: "pre",
    transform: {
      filter: {
        id: isTSXOrJSX
      },
      handler(code: string, id: string): TransformResult | null {
        // Implementation follows Section 4 phases
      }
    }
  };
};
```

### File Filter

The plugin filters `.tsx` and `.jsx` files only. `.mdx` files are excluded (see Section 8 for future consideration). `.ts` files (no JSX) are excluded because there are no JSX expression containers to transform.

### Plugin Ordering

The plugin must run with `enforce: "pre"`, matching the existing plugins. Within `enforce: "pre"` plugins, ordering follows the order they appear in the Vite/Rolldown config.

Required ordering:

1. **Icons plugin** (`vite-plugin-qds-icons`) — transforms icon JSX elements (e.g. `<Lucide.Check />` → `<svg />`) before other plugins see them.
2. **UI plugin** (`vite-plugin-qds-ui`) — transforms JSX expression containers that reference QDS state.
3. **AsChild plugin** (`vite-plugin-as-child`) — hoists child element props; runs after state transforms so generated components are in their final positions.
4. **Qwik optimizer** — must run after all `enforce: "pre"` plugins; sees the fully transformed source with generated `component$()` declarations at module scope.

In a consumer's `vite.config.ts`:

```typescript
import { icons, asChild, ui } from "@qds.dev/tools/rolldown";
import { qwikVite } from "@qwik.dev/core/optimizer";

export default defineConfig({
  plugins: [icons(), ui(), asChild(), qwikVite()]
});
```

The Vite plugin execution order for `enforce: "pre"` plugins follows their array index. `ui()` must appear after `icons()` and before `asChild()`.

### Dependencies

The plugin uses the same dependencies already present in `libs/tools/package.json`:

- `oxc-parser` — AST parsing
- `oxc-walker` — AST traversal
- `magic-string` — source rewriting with sourcemap support
- `magic-regexp` — file filter regex construction
- `@oxc-project/types` — TypeScript types for AST nodes

No new dependencies are required.

### AST Traversal Options

The plugin can use two approaches for AST traversal, both from the oxc ecosystem:

1. **oxc-walker `walk()` function** — Generic traversal that visits every node. Simple and sufficient for most phases. oxc-walker also provides a `ScopeTracker` class that can resolve identifier declarations — useful for alias and destructuring tracking in Phase 3.

2. **oxc-parser `Visitor` class** — Built-in visitor with strongly-typed per-node callbacks (e.g., `visitMemberExpression`, `visitVariableDeclarator`, `visitJSXExpressionContainer`). More ergonomic than generic walk when only specific node types are relevant. Alternative to oxc-walker for targeted traversal.

Both approaches work with the same AST. All AST nodes have a `parent?: Node` field for backtracking up the tree — useful for detecting whether a state reference is inside a `JSXAttribute` (prop passing case) vs a `JSXElement` child.

The recommended approach: use `walk()` with `ScopeTracker` for Phase 3 (alias/destructuring tracking needs scope resolution), and either `walk()` or `Visitor` for other phases based on implementation preference.

### Internal Module Structure

```typescript
// ui.ts — main plugin entry
// ui-types.ts — shared type definitions
// manifests/index.ts — master registry
// manifests/select.ts — select component manifest
// manifests/checkbox.ts — (future)
// manifests/menu.ts — (future)
```

The types module (`ui-types.ts`) exports `ComponentManifest`, `StateDescriptor`, `BoundNamespace`, `StateReference`, `AliasBinding`, and `TransformTarget`. These are consumed by both the plugin and the manifest files.

---

## Section 8: Non-Goals and Future Considerations

### Non-Goals

**Not replacing `bind:` props.** The `bind:value`, `bind:open`, and `bind:disabled` props provide two-way binding for controlled components. This plugin handles read-only state access in JSX. These are complementary, not competing mechanisms.

**Not transforming state access in event handlers.** Consumers who need state in `$()` callbacks should design their components to pass that state through props or use a different pattern. The plugin does not transform non-JSX contexts.

**Not replacing `<select.valuelabel />`**. The `SelectValueLabel` component remains a valid, simpler option for displaying the current selection text. The compiler plugin is for consumers who need custom rendering that `valuelabel` does not support.

**Not supporting `.mdx` files.** MDX files with JSX are processed by a separate remark pipeline. Integrating the state compiler into MDX processing requires additional remark plugins and is deferred.

### Future: MDX Support

MDX files in documentation often include live component examples. Adding support requires:

1. A remark plugin (parallel to `transformMDXFile` in the icons plugin) that processes JSX expressions in MDX.
2. The same detection and transform logic as the `.tsx` handler.
3. Testing with the existing MDX processing pipeline.

### Future: TypeScript Plugin for Autocompletion

IDE autocompletion for `select.selectedLabels` would require a TypeScript language service plugin that:

1. Intercepts member expression completions on QDS namespace imports.
2. Reads the manifest to provide state property names as completions.
3. Provides hover documentation explaining the compiler transform.

This is independent of the build plugin and would be developed as a separate package or extension to the existing TypeScript configuration.

### Future: Manifest Generation from Source

Currently, manifests are hand-authored. A future build step could generate them automatically by:

1. Parsing each component's context type definition (`SelectContext`, etc.) using oxc-parser.
2. Extracting field names and signal types.
3. Distinguishing context fields from non-signal config fields (e.g., `multiple: boolean` vs `isOpen: Signal<boolean>`).
4. Generating manifest entries for all `Signal<T>` fields.

This would eliminate the manual step of keeping manifests synchronized with context type changes.

### Future: Warning for Unused State Properties in Manifest

After the transform runs, the plugin could emit development-mode warnings when a state property is in the manifest but never referenced in any consumer file. This would help QDS maintainers identify state properties that should be removed from the manifest.

---

## Appendix: Implementation Checklist

For the implementer building this plugin, the following steps cover the full scope:

**Runtime changes (libs/components/src/select/):**

- [ ] Add `selectedLabels: Signal<string[]>` to `SelectContext` type in `select-root.tsx`
- [ ] Add `useComputed$` for `selectedLabels` in `SelectRoot` component body
- [ ] Include `selectedLabels` in the context object passed to `useContextProvider`
- [ ] Export `selectContextId` from `libs/components/src/select/index.ts` (if not already)
- [ ] Verify `selectContextId` is accessible via `@qds.dev/ui` barrel export

**Manifest files (libs/tools/rolldown/manifests/):**

- [ ] Create `manifests/` directory
- [ ] Create `ui-types.ts` with type definitions
- [ ] Create `manifests/select.ts` with `selectManifest`
- [ ] Create `manifests/index.ts` with `qdsManifests` registry

**Plugin implementation (libs/tools/rolldown/ui.ts):**

- [ ] Phase 1: Import detection from `@qds.dev/ui`
- [ ] Phase 2: State reference tracking via AST walk
- [ ] Phase 3: Alias tracking via `VariableDeclarator` walk
- [ ] Phase 4: JSX expression container detection
- [ ] Phase 4: Destructuring detection and error emission
- [ ] Phase 4: Prop-passing detection and error emission
- [ ] Phase 4: `$()` boundary detection and warning emission
- [ ] Phase 5: Component name generation
- [ ] Phase 5: Generated component body construction
- [ ] Phase 6: MagicString rewrites for JSX containers
- [ ] Phase 6: Generated component declarations appended to module
- [ ] Phase 6: Import injection (`useContext`, `component$`, context IDs)

**Plugin export:**

- [ ] Add `ui` export to `libs/tools/rolldown/index.ts`
- [ ] Add to `/vite` export if a Vite-only version is needed

**Tests (libs/tools/rolldown/ui.unit.ts):**

- [ ] Test: no QDS import — returns null (no transform)
- [ ] Test: component-only namespace usage — returns null (no state refs)
- [ ] Test: simple state access `select.isOpen.value`
- [ ] Test: complex expression `select.selectedLabels.value.length > 3 ? ... : ...`
- [ ] Test: aliased state property
- [ ] Test: simple destructuring tracked as alias
- [ ] Test: nested destructuring emits build error
- [ ] Test: prop passing emits warning
- [ ] Test: multiple state refs in one container
- [ ] Test: state refs in multiple separate containers
- [ ] Test: import injection does not duplicate existing imports
- [ ] Test: generated component names are unique per file

**Documentation:**

- [ ] Add `ui` to `libs/tools/ARCHITECTURE.md` plugin list
- [ ] Update `/vite` and `/rolldown` sections of `libs/tools/README.md`
