---
name: vue-analyze
description: Analyzes Vue 3 component structure and suggests optimizations for performance and code reuse. Checks Composition API patterns, computed vs method usage, reactivity correctness, and cross-component duplication.
---

# Vue Component Analysis

Analyze Vue 3 component(s) in this project for performance issues and code reuse opportunities. The argument is optional: if a file path is provided, analyze that component only; otherwise analyze all `.vue` files under `client/src/`.

## Analysis Steps

### 1. Discover Scope

If `$ARGUMENTS` is non-empty, analyze only that file. Otherwise glob `client/src/**/*.vue` for the full list.

### 2. Per-Component Checks

For each component, read the full file and evaluate:

**Reactivity correctness**
- `ref()` used for mutable state, `computed()` for derived values — not the reverse
- No computed property that is mutated (`computed.value = ...`)
- Destructured props in `setup()` — flags broken reactivity (`const { x } = props` loses reactivity; must use `props.x` or `toRefs(props)`)
- `.value` used in `<script>` but NOT in `<template>` — flag mis-uses in either direction
- Date objects created without `isNaN()` guard before `.getMonth()` / `.getFullYear()` calls

**Performance issues**
- Heavy calculations placed in methods instead of `computed` (methods recalculate on every render)
- `v-if` used to toggle elements that switch frequently — suggest `v-show`
- `v-for` with `index` as `:key` — flag and suggest domain key (`.id`, `.sku`, `.month`, etc.)
- `watch` with deep:true on large arrays when a targeted `watchEffect` would suffice
- Template expressions containing non-trivial logic that should move to a `computed`
- Inline `style` bindings that recompute on every render instead of being cached in `computed`

**Composition API patterns**
- Options API mixed with Composition API in the same component
- Logic that could be extracted into a composable (same pattern appearing in setup across 2+ components)
- Async operations in `setup()` without `loading`/`error` refs following the standard pattern in `client/CLAUDE.md`
- Missing `onUnmounted` cleanup for event listeners or timers set in `onMounted`

**Code reuse opportunities**
Note function names and logic blocks that appear in multiple components. Candidates for extraction to `client/src/composables/`:
- Filter/query-param construction repeated across views
- `formatCurrency`, `getStockStatus`, `translateWarehouse`, or similar utilities duplicated
- Identical `loadData` wrappers (try/catch/finally around an API call)
- Chart data transformation patterns repeated in multiple views

**Template quality**
- Deeply nested ternaries inside `{{ }}` interpolations — suggest computed property
- Repeated sections that could be a child component (same block of markup in 3+ places)
- Missing `scoped` on `<style>` — flag global style leakage risk

### 3. Cross-Component Summary

After checking all components, produce a **cross-component report** listing:
- Duplicated logic across files (with file names and approximate line numbers)
- Suggested composable names and what they would consolidate
- Any patterns that conflict with the rules in `client/CLAUDE.md`

### 4. Output Format

Structure findings as:

```
## <ComponentName>.vue  [path]

### Performance
- [line N] <issue> → <suggestion>

### Reactivity
- [line N] <issue> → <suggestion>

### Code Reuse
- <observation>
```

Then a final section:

```
## Cross-Component Opportunities
- <composable name>: consolidates X from FileA.vue and FileB.vue
```

Only report real findings. If a component has no issues in a category, omit that category. Keep suggestions actionable: reference the exact pattern to change and the preferred replacement.

**Do not make any edits.** This skill is read-only analysis. If the user wants to apply changes, they should invoke the `vue-expert` subagent or ask you to proceed with specific fixes.
