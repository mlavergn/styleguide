# STYLE: Svelte

Authoritative Svelte code style and component layout. Shared and global: one ruleset per language, referenced by every project that uses it, never copied into one.

Where code does not conform, change the code. Where the rule is wrong, raise an amendment — it changes every project using this language. A project that must deviate records it under Style exceptions in its PROJECT.md.

Svelte 5 with runes.

## Critical Constraints

### Forbidden

- Never use a legacy idiom: `export let`, `createEventDispatcher`, slots, or stores inside a shared component kit. Runes only.
- Never reach out of a component for an adopter's store, API, or router. A component takes data as props and reports back through callbacks; one that needs a data shape rather than a callback does not belong in a shared kit.
- Never hardcode a color. Use the design tokens. A raw hex or a stock Tailwind color bypasses the design system.

```svelte
<p class="text-[#9aa0a6]">     <!-- WRONG — raw hex -->
<p class="text-gray-400">      <!-- WRONG — off-system -->
<p class="text-ink-muted">     <!-- CORRECT — token -->
```

- Never define the design tokens in more than one place. A second `@theme` block makes two screens in the same window disagree.

## Architecture & File Organization

### Code Layout

- Shared components live together in one directory, display helpers in a `format.js`, and calls to endpoints the kit itself serves in an `api.js`. An adopter's own `api.js` and `store.js` stay in the adopter.

### File Layout

- One component per file. The filename is the import name.
- File order, top to bottom: the `<!-- @component -->` block, then `<script>`, then markup. A `<style>` block only when a utility class genuinely cannot express it.
- Script order: imports — framework, then third-party, then local — then the typed `$props()` destructure, then `$state`, then `$derived`, then `$effect`, then functions.

### Dependencies & Standard Library

- Reach for the component library for structure — modals, buttons, spinners — and replace it where its light-ground assumptions break on a dark palette. Say why in the `@component` block when you do.

## Code Construction & Logic

### Logic Layout

- Runes only: `$props`, `$state`, `$derived`, `$effect`.
- Destructure props with defaults where a sensible one exists. A required prop has none, so its absence is visible at the call site.
- Data accessors are props with defaults — `idOf`, `titleOf`, `subtitleOf`, `contentOf` — so a caller whose items already match the default shape passes none of them. This is what keeps one component usable by two consoles with unrelated data layers.
- Caller-supplied markup is a snippet, rendered with `{@render row(item)}`.

### Naming

- `[component]` — PascalCase for its role, e.g. `ListDetail.svelte`, `DirectoryPicker.svelte`.
- `[callback prop]` — `on<Thing>`, e.g. `onSave`, `onCancel`, `onDirtyChange`.
- `[test hook]` — `camelCaseComponent.element`, e.g. `rawEditor.save`, `confirm.cancel`.

### Error Handling

- A callback that can fail rejects. `onSave` is awaited and a rejection is what renders the error; a component never invents its own error channel.

## Quality & Tooling

### Documentation & Comments

- Every component opens with a `<!-- @component -->` block, before `<script>`. Svelte's language tooling lifts it onto the component's type, so it becomes the hover documentation at every usage site. Prose first — what it does and why a caller reaches for it, never how it works — then a fenced usage example.
- Props are typed with one JSDoc `@type` object on the `$props()` destructure. That is the machine-readable half; do not restate every prop in prose, where it would drift.

```svelte
<!--
@component
A small capsule label, for a scope tag beside a title.

The library's own badge assumes a light ground and disappears on this palette.
-->
<script>
  /**
   * @type {{
   *   tone?: 'neutral' | 'accent',
   *   children: import('svelte').Snippet,
   * }}
   */
  let { tone = 'neutral', children } = $props();
</script>
```

- Implementation notes are `//` comments inside `<script>`, beside the code they explain — never in the `@component` block, which is caller-facing.
- Every export in `format.js` and `api.js` gets full JSDoc: prose, then `@param` per parameter, then `@returns`, and `@throws` where it rejects. A non-trivial local function gets a one-line `/** */`; a one-liner needs nothing.

### UI Patterns

- Tailwind utilities inline in the markup; tokens carry the palette, typography, and spacing.

### Build & Tooling

- A new component directory needs an `@source` entry in the stylesheet. Tailwind v4 does not scan `node_modules`, so without one the component renders unstyled — silently, with no build error.

### Testing

- Every element a test drives carries `data-testid`. Never key a test on a class or on visible text.
- The demo renders every component the kit ships. A component only an adopter exercises has no coverage, and validation would pass while it was broken. Adding a component to the kit means adding it to the demo in the same change.
