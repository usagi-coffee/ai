---
name: svelte
description: Create or edit Svelte/SvelteKit files (`*.svelte`, `*.svelte.js`, `+page.svelte`, `+page.js`, `+layout.js`)
---

# Svelte Edit

Apply these rules when creating or modifying Svelte files. Preserve existing project conventions when they are stricter, but do not introduce legacy Svelte patterns.

## Svelte

- Use Svelte 5 runes. Do not introduce legacy APIs such as `export let`, `$:`, `on:click`, or slots.
- Use event properties, snippets, and `{@render ...}`.
- Use `{const ...}`, `{const ... = $derived(...)}`, and `{let ... = $state(...)}` in markup. Do not use legacy `{@const ...}`.
- Use `{@attach ...}` over `bind:this`, `onMount`, and `onDestroy`. Reactive reads inside an attachment cause reattachment.
- Destructure component props from `$props()`.
- Prefer top-level await and async `$derived` under `<svelte:boundary>` over `{#await}` blocks and manual loading state.

## SvelteKit

- Use `$app/state`, not `$app/stores`.
- Use `$lib` for imports from `src/lib`.
- Use `goto` from `$app/navigation` for application navigation instead of assigning `window.location.href`.

## State and reactivity

- Use `$state` for mutable state, `$derived` for expressions, and `$derived.by` for multi-step calculations.
- Declare `$derived` with `const` unless it is reassigned; then use `let`, `$derived.by()` cannot be reassigned/mutated.
- Prefer many small, composable `$derived` values over one large computation. They stay lazy, track narrower dependency sets, and make broad or expensive invalidations easy to locate and fix.
- Default to class-first design for stateful workflows: classes own state, derived facts, and mutations; components render them and call their methods. Put shared workflow classes in `.svelte.js` and page-only classes in the component. Prefer reactive fields over getters and use `$state.raw` when nested proxying is unnecessary.
- Prefer mutating methods such as `push` and `splice` when updating an existing reactive array. Do not replace the entire array solely to trigger reactivity.
- Avoid `$effect` for synchronization. Use derived state, function bindings, attachments, or direct mutation at the event/API/entity method that owns the change.

## Patterns

### Derived declarations

Use `const` when code only reads the binding and `let` when code also assigns to it. Both forms are valid:

```js
const total = $derived(lines.reduce((sum, line) => sum + line.quantity, 0));

let selected = $derived(lines[0]);
function select(line) {
  selected = line;
}
```

`selected` is still derived state; declaring it with `let` allows the explicit override which quite often can help with avoiding `$effect`.

### Split derived calculations

Keep each dependency step narrow and lazy. This exposes where work happens and lets an unchanged intermediate value stop invalidation from reaching later calculations:

```js
const search = $derived(query.toUpperCase());
const visible = $derived(
  records.filter((record) => record.name.includes(search)),
);
const groups = $derived(group_by(visible, (record) => record.group));
```

A boolean derived is a useful gate:

```js
const overweight = $derived(weight > 100);
const warning = $derived(overweight ? x : y);
```

Changing `weight` from `110` to `120` keeps `overweight` `true`, so `warning` is not recalculated. Changing it from `120` to `90` changes `overweight` to `false` and recalculates `warning`; further changes below `100` are skipped again.

### Attachments

Register DOM behavior and its cleanup in the same attachment. Compose independent behaviors directly on the element:

```svelte
<script>
  function autoselect(element) {
    const select = () => element.select();

    element.addEventListener("focus", select);
    element.addEventListener("click", select);

    return () => {
      element.removeEventListener("focus", select);
      element.removeEventListener("click", select);
    };
  }
</script>

<input {@attach autoselect} {@attach tooltip("Search")} />
```

### Function binding

Normalize or coordinate writes at the binding boundary:

```svelte
<input bind:value={() => search, (value) => (search = value.toUpperCase())} />
```

Use class accessors for bindings with coordinated writes or side effects. This avoids having to use `$effect` and leaking wrapper internals such as `.current` into the template and keeps the markup as a clean domain property:

```svelte
<script>
  class Person {
    #name = $state();

    get name() {
      return this.#name;
    }

    set name(value) {
      this.#name = value;
      doSomethingElse();
    }
  };

  const person = new Person();
</script>

<input bind:value={person.name} />
```

### Class-first design

Represent a stateful entity or workflow as a class. Keep its invariants and mutations out of loose component variables:

```js
class Line {
  transactions = $state([]);
  issued = $derived(
    this.transactions.reduce((total, tx) => total + tx.quantity, 0),
  );

  add(transaction) {
    this.transactions.push(transaction);
  }
}
```
