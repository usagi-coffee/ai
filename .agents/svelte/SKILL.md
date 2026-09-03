---
name: svelte
description: Create or edit Svelte/SvelteKit files (`*.svelte`, `*.svelte.js`, `+page.svelte`, `+page.js`, `+layout.js`)
---

# Svelte Edit

Apply these rules when creating or modifying Svelte files. Preserve existing project conventions when they are stricter, but do not introduce legacy Svelte patterns.

## Svelte

- Use Svelte 5 runes. Do not introduce legacy APIs such as `export let`, `$:`, `on:click`, or slots.
- Use event properties, snippets, and `{@render ...}`.
- Replace `{@const ...}` with let-declarations. Let declarations are not reactive by default. Use `{const value = $derived(expression)}` when the value must update and `{let value = $state(initial)}` for mutable reactive state. Use bare `{const value = expression}` only when the value does not need to update, and avoid bare `{let ...}` for state.
- Use `{@attach ...}` over `bind:this`, `onMount`, and `onDestroy`. Reactive reads inside an attachment cause reattachment.
- Destructure component props from `$props()`.
- Replace `{#await ...}` with `<svelte:boundary>`. Use top-level `<script>` await for component-level work and `{await expression}` inside `<svelte:boundary>` for a local subtree. The latter is a markup await, not top-level await. Add a `pending` snippet only when the expected delay justifies replacing the current UI; omit it for short work to avoid loader flashes.
- Never start a free-floating promise in a component script (for example, `initialize();`). Give every initialization promise an explicit lifecycle owner: await it at the top level, directly or as part of an awaited promise chain, or start it from an attachment that owns the related behavior and cleanup.

## SvelteKit

- Use `$app/state`, not `$app/stores`.
- Use `$lib` for imports from `src/lib`.
- Use `goto` from `$app/navigation` for application navigation instead of assigning `window.location.href`.
- Treat `const { data } = $props()` and route load functions used for anything other than navigation guards or redirects as legacy patterns. Initialize and fetch in components under `<svelte:boundary>`; initialize the application in `+layout.svelte` and gate its children with the boundary.

## State and reactivity

- Use `$state` for mutable state, `$derived` for expressions, and `$derived.by` for multi-step calculations. Default to `$state([])` for arrays. When an array is only reassigned, `$state.raw([])` avoids unnecessary proxy overhead, especially for large arrays. Its elements can still be independently reactive through their own `$state` fields.
- Declare `$derived` with `const` unless it is reassigned; then use `let`, `$derived.by()` cannot be reassigned/mutated.
- Prefer many small, composable `$derived` values over one large computation. They stay lazy, track narrower dependency sets, and make broad or expensive invalidations easy to locate and fix.
- Use classes for domain entities and domain workflows that own invariants, derived facts, and mutations. Do not create page or component wrapper classes such as `ConnectPage` or `PageState`; keep view-only state and derived values as top-level runes in the component. Put reusable domain classes in `.svelte.js` and page-local domain classes in the component.
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

### Derived initial values

Prefer `$derived` even when a computed value is described as an initial or default value. In most cases, allowing it to update when its dependencies change is harmless and simpler than deliberately breaking reactivity:

```js
// Avoid by default
let selected = $state(untrack(() => lines[0]));

// Prefer, even when lines[0] is only the initial selection
let selected = $derived(lines[0]);
```

Declaring the derived value with `let` still permits an explicit override. Use `$state(untrack(() => expression))` only when a later dependency change updating the value would be incorrect, not merely because the value is considered initial.

### Split derived calculations

Keep each dependency step narrow and lazy. This exposes where work happens and lets an unchanged intermediate value stop invalidation from reaching later calculations:

```js
const search = $derived(query.toUpperCase());
const visible = $derived(records.filter((record) => record.name.includes(search)));
const groups = $derived(group_by(visible, (record) => record.group));
```

A boolean derived is a useful gate:

```js
const overweight = $derived(weight > 100);
const warning = $derived(overweight ? x : y);
```

Changing `weight` from `110` to `120` keeps `overweight` `true`, so `warning` is not recalculated. Changing it from `120` to `90` changes `overweight` to `false` and recalculates `warning`; further changes below `100` are skipped again.

### Async dependencies

Values read inside asynchronous callbacks are not tracked automatically. Pass reactive dependencies as function arguments so they are read synchronously by the reactive expression and become stable snapshots inside the callback:

```js
let filter = $state("active");
let limit = $state(20);

function createRequest(filter, limit) {
  return observe(async () => {
    // Values read inside this callback are not tracked automatically
    return await loadItems({ filter, limit });
  });
}

const results = $derived(createRequest(filter, limit));
```

### Local markup declarations

Declaration tags only provide local variables and block scope; `$derived` and `$state` provide reactivity. Keep small local values next to the markup that uses them instead of hoisting them into the component script:

```svelte
<section>
  {const template = $derived(compact ? "1fr 5rem" : "1fr 8rem")}

  <header style:grid-template-columns={template}>
    ...
  </header>

  <article style:grid-template-columns={template}>
    ...
  </article>
</section>
```

`{const template = compact ? a : b}` would not update when `compact` changes; `$derived` is what makes it reactive. Likewise, `{let expanded = false}` is not reactive; use `{let expanded = $state(false)}` when assignments must update the markup. Their scope and lifetime follow the surrounding block.

### Reactive collections

Use svelte's built-in reactive collections when mutations such as `.add()`, `.set()`, or `.delete()` must update derived values or markup:

```svelte
<script>
  import { SvelteSet } from "svelte/reactivity";

  const selected = new SvelteSet();
</script>

<button onclick={() => selected.add(record.id)}>
  Select
</button>

{selected.size} selected
```

Use `SvelteMap`, `SvelteSet`, or `SvelteURLSearchParams` instead of their native counterparts when the collection itself participates in reactivity.

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

### Domain classes

Classes represent concepts in the application domain, not the component containing them. Keep view-only state at the component level and use classes for domain models that own behavior:

```svelte
<script>
  class Order {
    name;
    lines = $state([]);
    total = $derived(this.lines.reduce((sum, line) => sum + line.quantity, 0));

    constructor(name) {
      this.name = name;
    }

    add(line) {
      this.lines.push(line);
    }
  }

  let search = $state("");

  const orders = $state([new Order("First order")]);
  const visible = $derived(orders.filter((order) => order.name.includes(search)));
</script>

{#each visible as order}
  <p>{order.name}: {order.total}</p>
{/each}
```

`search` and `visible` only shape this view, while `Order` represents a domain entity and owns its lines and total.

### Search parameters

Use `SvelteURLSearchParams` when search parameters participate in reactivity. Keeping filters and selections in the URL makes the current view shareable: someone opening the link gets the same values already applied.

```svelte
<script>
  import { replaceState } from "$app/navigation";
  import { page } from "$app/state";
  import { SvelteURLSearchParams } from "svelte/reactivity";

  const params = $derived(new SvelteURLSearchParams(page.url.search));
  const query = $derived(params.get("query") ?? "");
</script>

<input bind:value={() => query, (value) => {
  params.set('query', value);
  replaceState(`?${params}`, {});
}}/>
```

### Layout initialization

Run application initialization in `+layout.svelte`. Model startup as an explicit promise chain and give each meaningful stage its own boundary so its pending and failure UI accurately describes the work being performed:

```svelte
<script>
  import { invalidate } from "$app/navigation";

  import { App } from "$lib/app.svelte";
  import { ensure_service_worker } from "$lib/service-worker";

  /** @type {{ children?: import("svelte").Snippet<[void]> }} */
  const { children } = $props();
</script>

<svelte:boundary>
  {const service_worker = ensure_service_worker()}
  {void (await service_worker)}

  <svelte:boundary>
    {const application = service_worker.then(() => App.initialize())}
    {void (await application)}

    <svelte:boundary>
      {const routes = application.then(() => invalidate("app:initialized"))}
      {@render children?.(await routes)}

      {#snippet failed(error, reset)}
        {@render Failure({ text: "Could not refresh routes", error, reset })}
      {/snippet}
    </svelte:boundary>

    {#snippet pending()}
      <p>Loading application…</p>
    {/snippet}

    {#snippet failed(error, reset)}
      {@render Failure({ text: "Could not initialize application", error, reset })}
    {/snippet}
  </svelte:boundary>

  {#snippet pending()}
    <p>Starting service worker…</p>
  {/snippet}

  {#snippet failed(error, reset)}
    {@render Failure({ text: "Could not start service worker", error, reset })}
  {/snippet}
</svelte:boundary>

{#snippet Failure({ text, error, reset })}
  <section>
    <p>{text}</p>
    <pre>{error instanceof Error ? error.message : String(error)}</pre>
    <button onclick={() => reset()}>Try again</button>
  </section>
{/snippet}
```

Create each promise once with a local declaration, await it before entering the dependent subtree, and derive the next stage with `.then(...)`. Add or remove nested stages to match the application; do not collapse independent startup failures into one generic boundary.

After initialization, invalidate the dependency used by route loaders so they run against initialized application state. If needed, pass the awaited `void` result to `children` and type it as `Snippet<[void]>`; this makes child snippet invocation depend on the final promise. Calling `children()` without that argument can allow child route code to run before globals populated during initialization are available.

Keep startup ordering, loading UI, retry behavior, and initialization errors in the component tree instead of hiding them in a route loader. Attachments for service workers, navigation, update checks, or other DOM-lifetime integrations belong on the layout's stable outer element and can wrap this boundary tree.
