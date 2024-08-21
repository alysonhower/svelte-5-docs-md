Svelte 5 introduces _runes_, a powerful set of primitives for controlling reactivity inside your Svelte components and — for the first time — inside `.svelte.js` and `.svelte.ts` modules.

Runes are function-like symbols that provide instructions to the Svelte compiler. You don't need to import them from anywhere — when you use Svelte, they're part of the language.

When you opt in to runes mode, the non-runes features listed in the 'What this replaces' sections are no longer available.

> Check out the Introducing runes blog post before diving into the docs!

## $state

Reactive state is declared with the `$state` rune:

```
<script>
  let count = $state(0);
</script>

<button on:click={() => count++}>
  clicks: {count}
</button>
```

You can also use `$state` in class fields (whether public or private):

```
class Todo {
  done = $state(false);
  text = $state();

  constructor(text) {
    this.text = text;
  }
}
```

> In this example, the compiler transforms `done` and `text` into `get`/`set` methods on the class prototype referencing private fields

Only plain objects and arrays are made deeply reactive by wrapping them with `Proxies`:

```
<script>
  let numbers = $state([1, 2, 3]);
</script>

<button on:click={() => numbers.push(numbers.length + 1)}>
  push
</button>

<button on:click={() => numbers.pop()}> pop </button>

<p>
  {numbers.join(' + ') || 0}
  =
  {numbers.reduce((a, b) => a + b, 0)}
</p>
```

### What this replaces

In non-runes mode, a `let` declaration is treated as reactive state if it is updated at some point. Unlike `$state(...)`, which works anywhere in your app, `let` only behaves this way at the top level of a component.

## $state.link

Linked state stays up to date with its input:

```
let a = $state(0);
let b = $state.link(a);
a = 1;
console.log(a, b); // 1, 1
```

You can temporarily _unlink_ state. It will be _relinked_ when the input value changes:

```
b = 2; // unlink
console.log(a, b); // 1, 2
a = 3; // relink
console.log(a, b); // 3, 3
```

As with `$state`, if `$state.link` is passed a plain object or array it will be made deeply reactive. If passed an existing state proxy it will be reused, meaning that mutating the linked state will mutate the original. To clone a state proxy, you can use `$state.snapshot`.

If you pass a callback to `$state.link`, changes to the input value will invoke the callback rather than updating the linked state, allowing you to choose whether to (for example) preserve or discard local changes, or merge incoming changes with local ones:

```
let { stuff } = $props();
let incoming = $state();
let hasUnsavedChanges = $state(false);
let current = $state.link({ ...stuff }, (stuff) => {
  if (hasUnsavedChanges) {
    incoming = stuff;
  } else {
    incoming = null;
    current = stuff;
  }
});
```

## $state.raw

State declared with `$state.raw` cannot be mutated; it can only be _reassigned_. In other words, rather than assigning to a property of an object, or using an array method like `push`, replace the object or array altogether if you'd like to update it:

```
<script>
let numbers = $state.raw([1, 2, 3]);
</script>

<button onclick={() => numbers = [...numbers, numbers.length + 1]}>
push
</button>

<button onclick={() => numbers = numbers.slice(0, -1)}> pop </button>

<p>
{numbers.join(' + ') || 0}
=
{numbers.reduce((a, b) => a + b, 0)}
</p>
```

This can improve performance with large arrays and objects that you weren't planning to mutate anyway, since it avoids the cost of making them reactive. Note that raw state can _contain_ reactive state (for example, a raw array of reactive objects).

## $state.snapshot

To take a static snapshot of a deeply reactive `$state` proxy, use `$state.snapshot`:

```
<script>
let counter = $state({ count: 0 });

function onclick() {
  // Will log `{ count: ... }` rather than `Proxy { ... }`
  console.log($state.snapshot(counter));
}
</script>
```

This is handy when you want to pass some state to an external library or API that doesn't expect a proxy, such as `structuredClone`.

## $derived

Derived state is declared with the `$derived` rune:

```
<script>
let count = $state(0);
let doubled = $derived(count * 2);
</script>

<button onclick={() => count++}>
{doubled}
</button>

<p>{count} doubled is {doubled}</p>
```

The expression inside `$derived(...)` should be free of side-effects. Svelte will disallow state changes (e.g. `count++`) inside derived expressions.

As with `$state`, you can mark class fields as `$derived`.

### What this replaces

If the value of a reactive variable is being computed it should be replaced with `$derived` whether it previously took the form of `$: double = count * 2` or `$: { double = count * 2; }` There are some important differences to be aware of:

-   With the `$derived` rune, the value of `double` is always current (for example if you update `count` then immediately `console.log(double)`). With `$:` declarations, values are not updated until right before Svelte updates the DOM
-   In non-runes mode, Svelte determines the dependencies of `double` by statically analysing the `count * 2` expression. If you refactor it...
    
    ```
    const doubleCount = () => count * 2;
    $: double = doubleCount();
    ```
    
    ...that dependency information is lost, and `double` will no longer update when `count` changes. With runes, dependencies are instead tracked at runtime.
-   In non-runes mode, reactive statements are ordered _topologically_, meaning that in a case like this...
    
    ```
    $: triple = double + count;
    $: double = count * 2;
    ```
    
    ...`double` will be calculated first despite the source order. In runes mode, `triple` cannot reference `double` before it has been declared.

## $derived.by

Sometimes you need to create complex derivations that don't fit inside a short expression. In these cases, you can use `$derived.by` which accepts a function as its argument.

```
<script>
let numbers = $state([1, 2, 3]);
let total = $derived.by(() => {
  let total = 0;
  for (const n of numbers) {
    total += n;
  }
  return total;
});
</script>

<button onclick={() => numbers.push(numbers.length + 1)}>
{numbers.join(' + ')} = {total}
</button>
```

In essence, `$derived(expression)` is equivalent to `$derived.by(() => expression)`.

## $effect

To run _side-effects_ when the component is mounted to the DOM, and when values change, we can use the `$effect` rune (demo):

```
<script>
let size = $state(50);
let color = $state('#ff3e00');

let canvas;

$effect(() => {
  const context = canvas.getContext('2d');
  context.clearRect(0, 0, canvas.width, canvas.height);

  // this will re-run whenever `color` or `size` change
  context.fillStyle = color;
  context.fillRect(0, 0, size, size);
});
</script>

<canvas bind:this={canvas} width="100" height="100" />
```

The function passed to `$effect` will run when the component mounts, and will re-run after any changes to the values it reads that were declared with `$state` or `$derived` (including those passed in with `$props`). Re-runs are batched (i.e. changing `color` and `size` in the same moment won't cause two separate runs), and happen after any DOM updates have been applied.

Values that are read asynchronously — after an `await` or inside a `setTimeout`, for example — will _not_ be tracked. Here, the canvas will be repainted when `color` changes, but not when `size` changes (demo):

```
$effect(() => {
  const context = canvas.getContext('2d');
  context.clearRect(0, 0, canvas.width, canvas.height);

  // this will re-run whenever `color` changes...
  context.fillStyle = color;
  setTimeout(() => {
    // ...but not when `size` changes
    context.fillRect(0, 0, size, size);
  }, 0);
});
```

An effect only reruns when the object it reads changes, not when a property inside it changes. (If you want to observe changes _inside_ an object at dev time, you can use `$inspect`.)

```
<script>
let state = $state({ value: 0 });
let derived = $derived({ value: state.value * 2 });

// this will run once, because `state` is never reassigned (only mutated)
$effect(() => {
  state;
});

// this will run whenever `state.value` changes...
$effect(() => {
  state.value;
});

// ...and so will this, because `derived` is a new object each time
$effect(() => {
  derived;
});
</script>

<button onclick={() => (state.value += 1)}>
{state.value}
</button>

<p>{state.value} doubled is {derived.value}</p>
```

You can return a function from `$effect`, which will run immediately before the effect re-runs, and before it is destroyed (demo).

```
<script>
let count = $state(0);
let milliseconds = $state(1000);

$effect(() => {
  // This will be recreated whenever `milliseconds` changes
  const interval = setInterval(() => {
    count += 1;
  }, milliseconds);

  return () => {
    // if a callback is provided, it will run
    // a) immediately before the effect re-runs
    // b) when the component is destroyed
    clearInterval(interval);
  };
});
</script>

<h1>{count}</h1>

<button onclick={() => (milliseconds *= 2)}>slower</button>
<button onclick={() => (milliseconds /= 2)}>faster</button>
```

### When not to use $effect

In general, `$effect` is best considered something of an escape hatch — useful for things like analytics and direct DOM manipulation — rather than a tool you should use frequently. In particular, avoid using it to synchronise state. Instead of this...

```
<script>
let count = $state(0);
let doubled = $state();

// don't do this!
$effect(() => {
  doubled = count * 2;
});
</script>
```

...do this:

```
<script>
let count = $state(0);
let doubled = $derived(count * 2);
</script>
```

> For things that are more complicated than a simple expression like `count * 2`, you can also use `$derived.by`.

You might be tempted to do something convoluted with effects to link one value to another. The following example shows two inputs for "money spent" and "money left" that are connected to each other. If you update one, the other should update accordingly. Don't use effects for this (demo):

```
<script>
let total = 100;
let spent = $state(0);
let left = $state(total);

$effect(() => {
  left = total - spent;
});

$effect(() => {
  spent = total - left;
});
</script>

<label>
<input type="range" bind:value={spent} max={total} />
{spent}/{total} spent
</label>

<label>
<input type="range" bind:value={left} max={total} />
{left}/{total} left
</label>
```

Instead, use callbacks where possible (demo):

```
<script>
let total = 100;
let spent = $state(0);
let left = $state(total);

function updateSpent(e) {
  spent = +e.target.value;
  left = total - spent;
}

function updateLeft(e) {
  left = +e.target.value;
  spent = total - left;
}
</script>

<label>
<input
  type="range"
  value={spent}
  oninput={updateSpent}
  max={total}
/>
{spent}/{total} spent
</label>

<label>
<input
  type="range"
  value={left}
  oninput={updateLeft}
  max={total}
/>
{left}/{total} left
</label>
```

If you need to use bindings, for whatever reason (for example when you want some kind of "writable `$derived`"), consider using getters and setters to synchronise state (demo):

```
<script>
let total = 100;
let spent = $state(0);

let left = {
  get value() {
    return total - spent;
  },
  set value(v) {
    spent = total - v;
  }
};
</script>

<label>
<input type="range" bind:value={spent} max={total} />
{spent}/{total} spent
</label>

<label>
<input type="range" bind:value={left.value} max={total} />
{left.value}/{total} left
</label>
```

If you absolutely have to update `$state` within an effect and run into an infinite loop because you read and write to the same `$state`, use untrack.

### What this replaces

The portions of `$: {}` that are triggering side-effects can be replaced with `$effect` while being careful to migrate updates of reactive variables to use `$derived`. There are some important differences:

-   Effects only run in the browser, not during server-side rendering
-   They run after the DOM has been updated, whereas `$:` statements run immediately _before_
-   You can return a cleanup function that will be called whenever the effect refires

Additionally, you may prefer to use effects in some places where you previously used `onMount` and `afterUpdate` (the latter of which will be deprecated in Svelte 5). There are some differences between these APIs as `$effect` should not be used to compute reactive values and will be triggered each time a referenced reactive variable changes (unless using `untrack`).

## $effect.pre

In rare cases, you may need to run code _before_ the DOM updates. For this we can use the `$effect.pre` rune:

```
<script>
  import { tick } from 'svelte';

  let div = $state();
  let messages = $state([]);

  // ...

  $effect.pre(() => {
    if (!div) return; // not yet mounted

    // reference `messages` array length so that this code re-runs whenever it changes
    messages.length;

    // autoscroll when new messages are added
    if (
      div.offsetHeight + div.scrollTop >
      div.scrollHeight - 20
    ) {
      tick().then(() => {
        div.scrollTo(0, div.scrollHeight);
      });
    }
  });
</script>

<div bind:this={div}>
  {#each messages as message}
    <p>{message}</p>
  {/each}
</div>
```

Apart from the timing, `$effect.pre` works exactly like `$effect` — refer to its documentation for more info.

### What this replaces

Previously, you would have used `beforeUpdate`, which — like `afterUpdate` — is deprecated in Svelte 5.

## $effect.tracking

The `$effect.tracking` rune is an advanced feature that tells you whether or not the code is running inside a tracking context, such as an effect or inside your template (demo):

```
<script>
console.log('in component setup:', $effect.tracking()); // false

$effect(() => {
console.log('in effect:', $effect.tracking()); // true
});
</script>

<p>in template: {$effect.tracking()}</p> <!-- true -->
```

This allows you to (for example) add things like subscriptions without causing memory leaks, by putting them in child effects.

## $effect.root

The `$effect.root` rune is an advanced feature that creates a non-tracked scope that doesn't auto-cleanup. This is useful for nested effects that you want to manually control. This rune also allows for creation of effects outside of the component initialisation phase.

```
<script>
let count = $state(0);

const cleanup = $effect.root(() => {
$effect(() => {
console.log(count);
});

return () => {
console.log('effect root cleanup');
};
});
</script>
```

## $props

To declare component props, use the `$props` rune:

```
let { optionalProp = 42, requiredProp } = $props();
```

You can use familiar destructuring syntax to rename props, in cases where you need to (for example) use a reserved word like `catch` in `<MyComponent catch={22} />`:

```
let { catch: theCatch } = $props();
```

To get all properties, use rest syntax:

```
let { a, b, c, ...everythingElse } = $props();
```

You can also use an identifier:

```
let props = $props();
```

If you're using TypeScript, you can declare the prop types:

```
interface MyProps {
required: string;
optional?: number;
partOfEverythingElse?: boolean;
};
let { required, optional, ...everythingElse }: MyProps = $props();
```

> In an earlier preview, `$props()` took a type argument. This caused bugs, since in a case like this...
> 
> ```
> let { x = 42 } = $props<{ x?: string }>();
> ```
> 
> ...TypeScript widens the type of `x` to be `string | number`, instead of erroring.

If you're using JavaScript, you can declare the prop types using JSDoc:

```
/** @type {{ x: string }} */
let { x } = $props();
// or use @typedef if you want to document the properties:
/**
 * @typedef {Object} MyProps
 * @property {string} y Some documentation
 */
/** @type {MyProps} */
let { y } = $props();
```

By default props are treated as readonly, meaning reassignments will not propagate upwards and mutations will result in a warning at runtime in development mode. You will also get a runtime error when trying to `bind:` to a readonly prop in a parent component. To declare props as bindable, use `$bindable()`.

### What this replaces

`$props` replaces the `export let` and `export { x as y }` syntax for declaring props. It also replaces `$$props` and `$$restProps`, and the little-known `interface $$Props {...}` construct.

Note that you can still use `export const` and `export function` to expose things to users of your component (if they're using `bind:this`, for example).

## $bindable

To declare props as bindable, use `$bindable()`. Besides using them as regular props, the parent can (_can_, not _must_) then also `bind:` to them.

```
<script>
let { bindableProp = $bindable() } = $props();
</script>
```

You can pass an argument to `$bindable()`. This argument is used as a fallback value when the property is `undefined`.

```
<script>
let { bindableProp = $bindable('fallback') } = $props();
</script>
```

Note that the parent is not allowed to pass `undefined` to a property with a fallback if it `bind:`s to that property.

## $inspect

The `$inspect` rune is roughly equivalent to `console.log`, with the exception that it will re-run whenever its argument changes. `$inspect` tracks reactive state deeply, meaning that updating something inside an object or array using fine-grained reactivity will cause it to re-fire. (Demo:)

```
<script>
let count = $state(0);
let message = $state('hello');

$inspect(count, message); // will console.log when `count` or `message` change
</script>

<button onclick={() => count++}>Increment</button>
<input bind:value={message} />
```

`$inspect` returns a property `with`, which you can invoke with a callback, which will then be invoked instead of `console.log`. The first argument to the callback is either `"init"` or `"update"`, all following arguments are the values passed to `$inspect`. Demo:

```
<script>
let count = $state(0);

$inspect(count).with((type, count) => {
if (type === 'update') {
debugger; // or `console.trace`, or whatever you want
}
});
</script>

<button onclick={() => count++}>Increment</button>
```

A convenient way to find the origin of some change is to pass `console.trace` to `with`:

```
$inspect(stuff).with(console.trace);
```

> `$inspect` only works during development.

## $host

Retrieves the `this` reference of the custom element that contains this component. Example:

```
<svelte:options customElement="my-element" />

<script>
function greet(greeting) {
$host().dispatchEvent(
new CustomEvent('greeting', { detail: greeting })
);
}
</script>

<button onclick={() => greet('hello')}>say hello</button>
```

> Only available inside custom element components, and only on the client-side

## How to opt in

Current Svelte code will continue to work without any adjustments. Components using the Svelte 4 syntax can use components using runes and vice versa.

The easiest way to opt in to runes mode is to just start using them in your code. Alternatively, you can force the compiler into runes or non-runes mode either on a per-component basis...

YourComponent.svelte

```
<!-- this can be `true` or `false` -->
<svelte:options runes={true} />
```

...or for your entire app:

svelte.config.js

```
export default {
compilerOptions: {
runes: true
}
};
```
# Snippets

Snippets, and _render tags_, are a way to create reusable chunks of markup inside your components. Instead of writing duplicative code like this...

```
{#each images as image}
  {#if image.href}
    <a href={image.href}>
      <figure>
        <img
          src={image.src}
          alt={image.caption}
          width={image.width}
          height={image.height}
        />
        <figcaption>{image.caption}</figcaption>
      </figure>
    </a>
  {:else}
    <figure>
      <img
        src={image.src}
        alt={image.caption}
        width={image.width}
        height={image.height}
      />
      <figcaption>{image.caption}</figcaption>
    </figure>
  {/if}
{/each}
```

...you can write this:

```
{#snippet figure(image)}
  <figure>
    <img
      src={image.src}
      alt={image.caption}
      width={image.width}
      height={image.height}
    />
    <figcaption>{image.caption}</figcaption>
  </figure>
{/snippet}

{#each images as image}
  {#if image.href}
    <a href={image.href}>
      {@render figure(image)}
    </a>
  {:else}
    {@render figure(image)}
  {/if}
{/each}
```

Snippet parameters can be destructured (demo):

```
{#snippet figure({ src, caption, width, height })}
  <figure>
    <img alt={caption} {src} {width} {height} />
    <figcaption>{caption}</figcaption>
  </figure>
{/snippet}
```

Like function declarations, snippets can have an arbitrary number of parameters, which can have default values. You cannot use rest parameters however.

## Snippet scope

Snippets can be declared anywhere inside your component. They can reference values declared outside themselves, for example in the `<script>` tag or in `{#each ...}` blocks (demo)...

```
<script>
  let { message = `it's great to see you!` } = $props();
</script>

{#snippet hello(name)}
  <p>hello {name}! {message}!</p>
{/snippet}

{@render hello('alice')}
{@render hello('bob')}
```

...and they are 'visible' to everything in the same lexical scope (i.e. siblings, and children of those siblings):

```
<div>
  {#snippet x()}
    {#snippet y()}...{/snippet}

    <!-- this is fine -->
    {@render y()}
  {/snippet}

  <!-- this will error, as `y` is not in scope -->
  {@render y()}
</div>

<!-- this will also error, as `x` is not in scope -->
{@render x()}
```

Snippets can reference themselves and each other (demo):

```
{#snippet blastoff()}
  <span>🚀</span>
{/snippet}

{#snippet countdown(n)}
  {#if n > 0}
    <span>{n}...</span>
    {@render countdown(n - 1)}
  {:else}
    {@render blastoff()}
  {/if}
{/snippet}

{@render countdown(10)}
```

## Passing snippets to components

Within the template, snippets are values just like any other. As such, they can be passed to components as props (demo):

```
<script>
  import Table from './Table.svelte';

  const fruits = [
    { name: 'apples', qty: 5, price: 2 },
    { name: 'bananas', qty: 10, price: 1 },
    { name: 'cherries', qty: 20, price: 0.5 }
  ];
</script>

{#snippet header()}
  <th>fruit</th>
  <th>qty</th>
  <th>price</th>
  <th>total</th>
{/snippet}

{#snippet row(d)}
  <td>{d.name}</td>
  <td>{d.qty}</td>
  <td>{d.price}</td>
  <td>{d.qty * d.price}</td>
{/snippet}

<Table data={fruits} {header} {row} />
```

As an authoring convenience, snippets declared directly _inside_ a component implicitly become props _on_ the component (demo):

```
<!-- this is semantically the same as the above -->
<Table data={fruits}>
  {#snippet header()}
    <th>fruit</th>
    <th>qty</th>
    <th>price</th>
    <th>total</th>
  {/snippet}

  {#snippet row(d)}
    <td>{d.name}</td>
    <td>{d.qty}</td>
    <td>{d.price}</td>
    <td>{d.qty * d.price}</td>
  {/snippet}
</Table>
```

Any content inside the component tags that is _not_ a snippet declaration implicitly becomes part of the `children` snippet (demo):

```
<Table data={fruits}>
  {#snippet header()}
    <th>fruit</th>
    <th>qty</th>
    <th>price</th>
    <th>total</th>
  {/snippet}
  <th>fruit</th>
  <th>qty</th>
  <th>price</th>
  <th>total</th>

  <!-- ... -->
</Table>
```

```
<script>
  let { data, children, row } = $props();
</script>

<table>
  {#if children}
    <thead>
      <tr>{@render children()}</tr>
    </thead>
  {/if}

  <!-- ... -->
</table>
```

> Note that you cannot have a prop called `children` if you also have content inside the component — for this reason, you should avoid having props with that name

## Typing snippets

Snippets implement the `Snippet` interface imported from `'svelte'`:

```
<script lang="ts">
  import type { Snippet } from 'svelte';

  let { data, children, row }: {
    data: any[];
    children: Snippet;
    row: Snippet<[any]>;
  } = $props();
</script>
```

With this change, red squigglies will appear if you try and use the component without providing a `data` prop and a `row` snippet. Notice that the type argument provided to `Snippet` is a tuple, since snippets can have multiple parameters.

We can tighten things up further by declaring a generic, so that `data` and `row` refer to the same type:

```
<script lang="ts" generics="T">
  import type { Snippet } from 'svelte';

  let { data, children, row }: {
    data: T[];
    children: Snippet;
    row: Snippet<[T]>;
  } = $props();
</script>
```

## Creating snippets programmatically

In advanced scenarios, you may need to create a snippet programmatically. For this, you can use `createRawSnippet`

## Snippets and slots

In Svelte 4, content can be passed to components using slots. Snippets are more powerful and flexible, and as such slots are deprecated in Svelte 5.

They continue to work, however, and you can mix and match snippets and slots in your components.

When using custom elements, you should still use `<slot />` like before. In a future version, when Svelte removes its internal version of slots, it will leave those slots as-is, i.e. output a regular DOM tag instead of transforming it.

# Event handlers

Event handlers have been given a facelift in Svelte 5. Whereas in Svelte 4 we use the `on:` directive to attach an event listener to an element, in Svelte 5 they are properties like any other:

```
<script>
let count = $state(0);
</script>

<button onclick={() => count++}>
clicks: {count}
</button>
```

Since they're just properties, you can use the normal shorthand syntax...

```
<script>
let count = $state(0);

function onclick() {
  count++;
}
</script>

<button {onclick}>
clicks: {count}
</button>
```

...though when using a named event handler function it's usually better to use a more descriptive name.

Traditional `on:` event handlers will continue to work, but are deprecated in Svelte 5.

## Component events

In Svelte 4, components could emit events by creating a dispatcher with `createEventDispatcher`.

This function is deprecated in Svelte 5. Instead, components should accept _callback props_ - which means you then pass functions as properties to these components (demo):

```
<script>
import Pump from './Pump.svelte';

let size = $state(15);
let burst = $state(false);

function reset() {
  size = 15;
  burst = false;
}
</script>

<Pump
  inflate={(power) => {
    size += power;
    if (size > 75) burst = true;
  }}
  deflate={(power) => {
    if (size > 0) size -= power;
  }}
/>

{#if burst}
  <button onclick={reset}>new balloon</button>
  <span class="boom">💥</span>
{:else}
  <span class="balloon" style="scale: {0.01 * size}">
    🎈
  </span>
{/if}
```

```
<script>
let { inflate, deflate } = $props();
let power = $state(5);
</script>

<button onclick={() => inflate(power)}>inflate</button>
<button onclick={() => deflate(power)}>deflate</button>
<button onclick={() => power--}>-</button>
Pump power: {power}
<button onclick={() => power++}>+</button>
```

## Bubbling events

Instead of doing `<button on:click>` to 'forward' the event from the element to the component, the component should accept an `onclick` callback prop:

```
<script>
let { onclick, children } = $props();
</script>

<button {onclick}>
{@render children()}
</button>
```

Note that this also means you can 'spread' event handlers onto the element along with other props:

```
<script>
let { children, ...props } = $props();
</script>

<button {...props}>
{@render children()}
</button>
```

## Event modifiers

In Svelte 4, you can add event modifiers to handlers:

```
<button on:click|once|preventDefault={handler}>...</button>
```

Modifiers are specific to `on:` and as such do not work with modern event handlers. Adding things like `event.preventDefault()` inside the handler itself is preferable, since all the logic lives in one place rather than being split between handler and modifiers.

Since event handlers are just functions, you can create your own wrappers as necessary:

```
<script>
function once(fn) {
  return function (event) {
    if (fn) fn.call(this, event);
    fn = null;
  };
}

function preventDefault(fn) {
  return function (event) {
    event.preventDefault();
    fn.call(this, event);
  };
}
</script>

<button onclick={once(preventDefault(handler))}>...</button>
```

There are three modifiers — `capture`, `passive` and `nonpassive` — that can't be expressed as wrapper functions, since they need to be applied when the event handler is bound rather than when it runs.

For `capture`, we add the modifier to the event name:

```
<button onclickcapture={...}>...</button>
```

Changing the `passive` option of an event handler, meanwhile, is not something to be done lightly. If you have a use case for it — and you probably don't! - then you will need to use an action to apply the event handler yourself.

## Multiple event handlers

In Svelte 4, this is possible:

```
<button on:click={one} on:click={two}>...</button>
```

This is something of an anti-pattern, since it impedes readability (if there are many attributes, it becomes harder to spot that there are two handlers unless they are right next to each other) and implies that the two handlers are independent, when in fact something like `event.stopImmediatePropagation()` inside `one` would prevent `two` from being called.

Duplicate attributes/properties on elements — which now includes event handlers — are not allowed. Instead, do this:

```
<button
  onclick={(e) => {
    one(e);
    two(e);
  }}
>
  ...
</button>
```

## Why the change?

By deprecating `createEventDispatcher` and the `on:` directive in favour of callback props and normal element properties, we:

-   reduce Svelte's learning curve
-   remove boilerplate, particularly around `createEventDispatcher`
-   remove the overhead of creating `CustomEvent` objects for events that may not even have listeners
-   add the ability to spread event handlers
-   add the ability to know which event handlers were provided to a component
-   add the ability to express whether a given event handler is required or optional
-   increase type safety (previously, it was effectively impossible for Svelte to guarantee that a component didn't emit a particular event)

# Imports

As well as runes, Svelte 5 introduces a handful of new things you can import, alongside existing ones like `getContext`, `setContext` and `tick`.

## svelte

### flushSync

Forces any pending effects (including DOM updates) to be applied immediately, rather than in the future. This is mainly useful in a testing context — you'll rarely need it in application code.

```
<script>
import { flushSync } from 'svelte';

let count = $state(0);
let element;

function onclick() {
  flushSync(() => (count += 1));

  // without `flushSync`, the DOM would be updated in the future
  console.log(element.textContent === String(count));
}
</script>

<span bind:this={element}>{count}</span>
<button {onclick}>update</button>
```

### mount

Instantiates a component and mounts it to the given target:

```
import { mount } from 'svelte';
import App from './App.svelte';

const app = mount(App, {
  target: document.querySelector('#app'),
  props: { some: 'property' }
});
```

Note that unlike calling `new App(...)` in Svelte 4, things like effects (including `onMount` callbacks, and action functions) will not run during `mount`. If you need to force pending effects to run (in the context of a test, for example) you can do so with `flushSync()`.

### hydrate

Like `mount`, but will reuse up any HTML rendered by Svelte's SSR output (from the `render` function) inside the target and make it interactive:

```
import { hydrate } from 'svelte';
import App from './App.svelte';

const app = hydrate(App, {
  target: document.querySelector('#app'),
  props: { some: 'property' }
});
```

As with `mount`, effects will not run during `hydrate` — use `flushSync()` immediately afterwards if you need them to.

### unmount

Unmounts a component created with `mount` or `hydrate`:

```
import { mount, unmount } from 'svelte';
import App from './App.svelte';

const app = mount(App, {...});
// later
unmount(app);
```

### untrack

To prevent something from being treated as an `$effect`/`$derived` dependency, use `untrack`:

```
<script>
import { untrack } from 'svelte';

let { a, b } = $props();

$effect(() => {
  // this will run when `a` changes,
  // but not when `b` changes
  console.log(a);
  console.log(untrack(() => b));
});
</script>
```

### createRawSnippet

An advanced API designed for people building frameworks that integrate with Svelte, `createRawSnippet` allows you to create snippets programmatically for use with `{@render ...}` tags:

```
import { createRawSnippet } from 'svelte';

const greet = createRawSnippet((name) => {
  return {
    render: () => `
      <h1>Hello ${name()}!</h1>
    `,
    setup: (node) => {
      $effect(() => {
        node.textContent = `Hello ${name()}!`;
      });
    }
  };
});
```

The `render` function is called during server-side rendering, or during `mount` (but not during `hydrate`, because it already ran on the server), and must return HTML representing a single element.

The `setup` function is called during `mount` or `hydrate` with that same element as its sole argument. It is responsible for ensuring that the DOM is updated when the arguments change their value — in this example, when `name` changes:

```
{@render greet(name)}
```

If `setup` returns a function, it will be called when the snippet is unmounted. If the snippet is fully static, you can omit the `setup` function altogether.

## svelte/reactivity

Svelte provides reactive `SvelteMap`, `SvelteSet`, `SvelteDate` and `SvelteURL` classes. These can be imported from `svelte/reactivity` and used just like their native counterparts. Demo:

```
<script>
import { SvelteURL } from 'svelte/reactivity';

const url = new SvelteURL('https://example.com/path');
</script>

<!-- changes to these... -->
<input bind:value={url.protocol} />
<input bind:value={url.hostname} />
<input bind:value={url.pathname} />

<hr />

<!-- will update `href` and vice versa -->
<input bind:value={url.href} />
```

## svelte/events

Where possible, event handlers added with attributes like `onclick` use a technique called _event delegation_. It works by creating a single handler for each event type on the root DOM element, rather than creating a handler for each element, resulting in better performance and memory usage.

Delegated event handlers run after other event handlers. In other words, a handler added programmatically with `addEventListener` will run _before_ a handler added declaratively with `onclick`, regardless of their relative position in the DOM (demo). It also means that calling `event.stopPropagation()` inside a declarative handler _won't_ prevent the programmatic handler (created inside an action, for example) from running.

To preserve the relative order, use `on` rather than `addEventListener` (demo):

```
import { on } from 'svelte/events';

const off = on(element, 'click', () => {
  console.log('element was clicked');
});
// later, if we need to remove the event listener:
off();
```

`on` also accepts an optional fourth argument which matches the options argument for `addEventListener`.

## svelte/server

### render

Only available on the server and when compiling with the `server` option. Takes a component and returns an object with `body` and `head` properties on it, which you can use to populate the HTML when server-rendering your app:

```
import { render } from 'svelte/server';
import App from './App.svelte';

const result = render(App, {
  props: { some: 'property' }
});
```

If the `css` compiler option was set to `'injected'`, `<style>` elements will be included in the `head`.

## svelte/store

In addition to the existing store-related imports such as `writable`, `svelte/store` gains two new functions: `fromStore` and `toStore`. These allow you to easily use stores and rune-based state interchangeably, even outside `.svelte` files.

### fromStore

Takes a store and turns it into an object with a reactive (and readonly, if the store is not writable) `current` property.

```
import { fromStore, get, writable } from 'svelte/store';

const store = writable(0);
const count = fromStore(store);
count.current; // 0;
store.set(1);
count.current; // 1
count.current += 1;
get(store); // 2
```

### toStore

Creates a store from a function that returns reactive state (and, optionally, a second function that sets the state):

```
import { toStore } from 'svelte/store';

let count = $state(0);
const store = toStore(
  () => count,
  (v) => (count = v)
);
store.set(1);
count; // 1
```

## svelte/elements

Svelte provides built-in DOM types. A common use case for DOM types is forwarding props to an HTML element. To properly type your props and get full intellisense, your props interface should extend the attributes type for your HTML element:

```
<script lang="ts">
import { HTMLAttributes } from 'svelte/elements';

interface Props extends HTMLAttributes<HTMLDivElement> {
  username: string;
}

let { username, ...rest }: Props = $props();
</script>

<div {...rest}>
  Hi, {username}!
</div>
```

> You can use `ComponentProps<ImportedComponent>`, if you wish to forward props to a Svelte component.

Svelte provides a best-effort of all the HTML DOM types that exist. If an attribute is missing from our type definitions, you are welcome to open an issue and/or a PR fixing it. For experimental attributes, you can augment the existing types locally by creating a `.d.ts` file:

```
import { HTMLButtonAttributes } from 'svelte/elements';

declare module 'svelte/elements' {
  export interface SvelteHTMLElements {
    'custom-button': HTMLButtonAttributes;
  }
  // allows for more granular control over what element to add the typings to
  export interface HTMLButtonAttributes {
    veryexperimentalattribute?: string;
  }
}
export {}; // ensure this is not an ambient module, else types will be overridden instead of augmented
```

The `.d.ts` file must be included in your `tsconfig.json` file. If you are using the standard `"include": ["src/**/*"]`, this just means the file should be placed in the `src` directory.

# Universal reactivity

In Svelte 5, you can create reactive state anywhere in your app — not just at the top level of your components.

Suppose we have a component like this:

```
<script>
    let count = $state(0);

    function increment() {
        count += 1;
    }
</script>

<button on:click={increment}>
    clicks: {count}
</button>
```

We can encapsulate this logic in a function, so that it can be used in multiple places:

```
<script>
    function createCounter() {
        let count = $state(0);

        function increment() {
            count += 1;
        }

        return {
            get count() { return count },
            increment
        };
    }

    const counter = createCounter();
</script>

<button on:click={counter.increment}>
    clicks: {counter.count}
</button>
```

> Note that we're using a `get` property in the returned object, so that `counter.count` always refers to the current value rather than the value at the time the `createCounter` function was called.
> 
> As a corollary, `const { count, increment } = createCounter()` won't work. That's because in JavaScript, destructured declarations are evaluated at the time of destructuring — in other words, `count` will never update.

We can also extract that function out into a separate `.svelte.js` or `.svelte.ts` module...

```
export function createCounter() {
    let count = $state(0);

    function increment() {
        count += 1;
    }

    return {
        get count() {
            return count;
        },
        increment
    };
}
```

...and import it into our component:

```
<script>
    import { createCounter } from './counter.svelte.js';

    const counter = createCounter();
</script>

<button on:click={counter.increment}>
    clicks: {counter.count}
</button>
```

See this example in the playground.

## Stores equivalent

In Svelte 4, the way you'd do this is by creating a custom store, perhaps like this:

```
import { writable } from 'svelte/store';

export function createCounter() {
    const { subscribe, update } = writable(0);

    function increment() {
        update((count) => count + 1);
    }

    return {
        subscribe,
        increment
    };
}
```

Back in the component, we retrieve the store value by prefixing its name with `$`:

```
<script>
    import { createCounter } from './counter.js';

    const counter = createCounter();
</script>

<button on:click={counter.increment}>
    clicks: {$counter}
</button>
```

The store approach has some significant drawbacks. A counter is just about the simplest custom store we could create, and yet we have to completely change how the code is written — importing `writable`, understanding its API, grabbing references to `subscribe` and `update`, changing the implementation of `increment` from `count += 1` to something far more cryptic, and prefixing the store name with a `$` to retrieve its value. That's a lot of stuff you need to understand.

With runes, we just copy the existing code into a new function.

## Gotchas

Reactivity doesn't magically cross function boundaries. In other words, replacing the `get` property with a regular property wouldn't work...

```
export function createCounter() {
    let count = $state(0);

    function increment() {
        count += 1;
    }

    return {
        count,
        increment
    };
}
```

...because the value of `count` in the returned object would always be `0`. Using the `$state` rune doesn't change that fact — it simply means that when you _do_ read `count` (whether via a `get` property or a normal function) inside your template or inside an effect, Svelte knows what to update when `count` changes.

# Fine-grained reactivity

In Svelte 4, reactivity centres on the _component_ and the top-level state declared therein. What this means is that in a situation like this...

```
<script>
    let todos = [];

    function remaining(todos) {
        console.log('recalculating');
        return todos.filter((todo) => !todo.done).length;
    }

    function addTodo(event) {
        if (event.key !== 'Enter') return;

        todos = [
            ...todos,
            {
                done: false,
                text: event.target.value
            }
        ];

        event.target.value = '';
    }
</script>

<input onkeydown={addTodo} />

{#each todos as todo}
<div>
<input bind:value={todo.text} />
<input type="checkbox" bind:checked={todo.done} />
</div>
{/each}

<p>{remaining(todos)} remaining</p>
```

...editing any individual `todo` will invalidate the entire list. You can see this for yourself by opening the playground, adding some todos, and watching the console in the bottom right. `remaining(todos)` is recalculated every time we edit the `text` of a todo, even though it can't possibly affect the result.

Worse, everything inside the `each` block needs to be checked for updates. When a list gets large enough, this behaviour has the potential to cause performance headaches.

With runes, it's easy to make reactivity _fine-grained_, meaning that things will only update when they need to:

```
<script>
let todos = $state([]);

function remaining(todos) {
    console.log('recalculating');
    return todos.filter(todo => !todo.done).length;
}

function addTodo(event) {
    if (event.key !== 'Enter') return;

    todos.push({
        done: false,
        text: event.target.value
    });

    event.target.value = '';
}
</script>
```

In this version of the app, editing the `text` of a todo won't cause unrelated things to be updated.

# Old vs new

This page intends to give a broad overview of how code written using the new APIs looks compared to code not using them. You will see that for most simple tasks that only involve a single component it will actually not look much different. For more complex logic, they simplify things.

## Counter

The `$state`, `$derived` and `$effect` runes replace magic `let` declarations, `$: x = ...` and `$: { ... }`. Event handlers can be written as event attributes now, which in practise means just removing the colon.

-   Before
-   After

```
<script>
    let count = $state(0);
    let double = $derived(count * 2);
    $effect(() => {
        if (count > 10) {
            alert('Too high!');
        }
    });
</script>

<button onclick={() => count++}>
    {count} / {double}
</button>
```

## Tracking dependencies

In non-runes mode, dependencies of `$:` statements are tracked at _compile time_. `a` and `b` changing will only cause `sum` to be recalculated because the expression — `add(a, b)` — refers to those values.

In runes mode, dependencies are tracked at _run time_. `sum` will be recalculated whenever `a` or `b` change, whether they are passed in to the `add` function or accessed via closure. This results in more maintainable, refactorable code.

-   Before
-   After

```
<script>
    let a = $state(0);
    let b = $state(0);
    let sum = $derived(add());

    function add() {
        return a + b;
    }
</script>

<button onclick={() => a++}>a++</button>
<button onclick={() => b++}>b++</button>
<p>{a} + {b} = {sum}</p>
```

## Untracking dependencies

Conversely, suppose you — for some reason — wanted to recalculate `sum` when `a` changes, but _not_ when `b` changes.

In non-runes mode, we 'hide' the dependency from the compiler by excluding it from the `$:` statement. In runes mode, we have a better and more explicit solution: `untrack`.

-   Before
-   After

```
<script>
    import { untrack } from 'svelte';

    let a = $state(0);
    let b = $state(0);
    let sum = $derived(add());

    function add() {
        return a + untrack(() => b);
    }
</script>

<button onclick={() => a++}>a++</button>
<button onclick={() => b++}>b++</button>
<p>{a} + {b} = {sum}</p>
```

## Simple component props

```
<script>
    let { count = 0 } = $props();
</script>

{count}
```

## Advanced component props

```
<script>
    let { class: classname, ...others } = $props();
</script>

<pre class={classname}>
    {JSON.stringify(others)}
</pre>
```

To implement a chat window that autoscrolls to the bottom when new messages appear (but only if you were _already_ scrolled to the bottom), we need to measure the DOM before we update it.

In Svelte 4, we do this with `beforeUpdate`, but this is a flawed approach — it fires before _every_ update, whether it's relevant or not. In the example below, we need to introduce checks like `updatingMessages` to make sure we don't mess with the scroll position when someone toggles dark mode.

With runes, we can use `$effect.pre`, which behaves the same as `$effect` but runs before the DOM is updated. As long as we explicitly reference `messages` inside the effect body, it will run whenever `messages` changes, but _not_ when `theme` changes.

`beforeUpdate`, and its equally troublesome counterpart `afterUpdate`, will be deprecated in Svelte 5.

-   Before
-   After

```
<script>
    import { tick } from 'svelte';

    let theme = $state('dark');
    let messages = $state([]);
    let viewport;

    $effect.pre(() => {
        messages;
        const autoscroll = viewport && viewport.offsetHeight + viewport.scrollTop > viewport.scrollHeight - 50;

        if (autoscroll) {
            tick().then(() => {
                viewport.scrollTo(0, viewport.scrollHeight);
            });
        }
    });

    function handleKeydown(event) {
        if (event.key === 'Enter') {
            const text = event.target.value;
            if (!text) return;

            messages = [...messages, text];
            event.target.value = '';
        }
    }

    function toggle() {
        theme = theme === 'dark' ? 'light' : 'dark';
    }
</script>

<div class:dark={theme === 'dark'}>
    <div bind:this={viewport}>
        {#each messages as message}
            <p>{message}</p>
        {/each}
    </div>

    <input onkeydown={handleKeydown} />

    <button onclick={toggle}>
        Toggle dark mode
    </button>
</div>
```

## Forwarding events

Because event handlers are just regular attributes now, the "forwarding events" concept is replaced with just passing callback props. Before, you would have to mark every event that you want to forward separately. You can still do this with event attributes...

```
<script>
    let { onclick, onkeydown, ...attributes } = $props();
</script>

<button
    {...attributes}
    {onclick}
    {onkeydown}
>a button</button>
```

...but in practise what you probably _really_ want to do in these situations is forward _all_ events. This wasn't possible before, but it is now:

```
<script>
    let { ...props } = $props();
</script>

<button
    {...props}
>a button</button>
```

-   Before
-   After

## Passing UI content to a component

Previously, you would pass UI content into components using slots. Svelte 5 provides a better mechanism for this, snippets. In the simple case of passing something to the default slot, nothing has changed for the consumer:

```
<!-- same with both slots and snippets -->
<script>
    import Button from './Button.svelte';
</script>

<Button>click me</Button>
```

Inside `Button.svelte`, use `@render` instead of the `<slot>` tag. The default content is passed as the `children` prop:

```
<script>
    let { children } = $props();
</script>

<button>
    {@render children()}
</button>
```

When passing props back up to the consumer, snippets make things easier to reason about, removing the need to deal with the confusing semantics of the `let:`-directive:

```
<!-- provider -->
<script>
    let { children } = $props();
</script>

<button>
    {@render children("some value")}
</button>
```

```
<!-- consumer -->
<script>
    import Button from './Button.svelte';
</script>

<Button>
    {#snippet children(prop)}
        click {prop}
    {/snippet}
</Button>
```

Combined with event attributes, this reduces the number of concepts to learn — everything related to the component boundary can now be expressed through props.

-   Before
-   After

> When using custom elements, you should still use `<slot />` like before. In a future version, when Svelte removes its internal version of slots, it will leave those slots as-is, i.e. output a regular DOM tag instead of transforming it.

# Frequently asked questions

## Background and motivations

### What is this?

You're on the Svelte 5 preview site! If you don't know what Svelte is but somehow ended up here anyway, we suggest visiting svelte.dev first to get familiar.

### What's special about Svelte 5?

Svelte 5 is a ground-up rewrite of the framework, designed to make your apps faster, smaller, and more robust.

It introduces _runes_, a powerful set of primitives for controlling reactivity inside your Svelte components and — for the first time — inside `.svelte.js` and `.svelte.ts` modules. You can learn about runes by reading the Introducing runes blog post and watching the accompanying video, and by reading the preliminary docs on this site.

### Doesn't this make Svelte harder to learn?

Au contraire! Svelte today involves certain mental gymnastics:

-   `let x` declares reactive state, but only at the top level of a component
-   `export let x` declares a prop, but only inside a component. `export const y = ...`, meanwhile, means something totally different
-   In addition to `export let`, you have to learn `$$props` and `$$restProps`
-   `$:` might be declaring a reactive binding, or running side-effects; like `let x`, it only works at the top level of a component. When these statements re-run is dependent on rules that are hard to understand
-   In general, code behaves differently inside and outside components, making refactoring difficult and requiring frequent context-switching

Runes, by contrast, are explicit, predictable and refactorable.

### Why can't we keep the old syntax?

Beyond the complexities listed above, the current design imposes some unfortunate limitations:

-   There's no way to indicate which variables should _not_ be considered reactive. This becomes problematic when applying Svelte rules outside the top level of a component (for example in `.js` files)
-   The `$:` syntax doesn't play well with TypeScript. For example, you can't declare the type of `theme` in a statement like this — it's a syntax error:
    
    ```
    $: theme: 'light' | 'dark' = dark ? 'dark' : 'light';
    ```
    
    But with runes, it works just fine:
    
    ```
    let theme: 'light' | 'dark' = $derived(
      dark ? 'dark' : 'light'
    );
    ```
    
-   Updating values inside `$:` statements can cause confusing behaviour and impossible to resolve bugs and the statements may run in an unexpected order
-   `$: {...}` doesn't let you return a cleanup function the way that `$effect` does
-   Typing props is unduly cumbersome when you want to share interfaces between multiple components
-   Prefixing store names with `$` to access their values works in a `.svelte` file, but cannot work in `.js` and `.ts` without causing linting and typechecking errors. Having a unified approach to reactive state solves this problem

## Breaking changes and migration

### Is it a breaking change?

We're striving to make Svelte 5 a drop-in replacement for Svelte 4, and to that end we've ported over the entire test suite. The new features are opt-in, and you can mix-and-match the new stuff with the old stuff within an app (though not within a component — in 'runes mode', certain features are deliberately disabled).

Having said that, the underlying mechanisms are totally different. It's inevitable that some of you will hit edge cases, which is why this is a major version (5.0) rather than a minor (4.x).

### No but really, am I going to have to rewrite everything?

Eventually, you'll have to make some changes — most of which we hope to automate. We don't want to end up in a situation where people feel like they have to juggle knowledge of a bunch of different ways of doing things.

Our current plan is that some or all of the features that runes make unnecessary like `let`\-style reactivity, `$:`, `$$props` and `$$restProps` will be deprecated in Svelte 6 and removed in Svelte 7. But don't worry — that won't happen for some time, and we'll provide automatic migration tooling to do as much of the change as possible. There are no plans to deprecate `onMount` or stores at the current time.

### Which things are disabled in runes mode?

When you opt into runes mode, you can no longer use the features that runes replace:

-   `$state` replaces top-level `let` declarations implicitly creating reactive state
-   `$derived` replaces `$: x = ...`
-   `$effect` replaces `$: {'{ ... }'}`
-   `$props` replaces `export let`, `$$props` and `$$restProps`

All other features, including stores, are still fully supported in runes mode.

### Which things will be deprecated in Svelte 5?

`beforeUpdate` and `afterUpdate` are deprecated — use `$effect.pre` and `$effect` instead, as these are more conservative about when they run code. Everything else will remain.

### Prettier formatting is broken, what gives?

`svelte-lsp` ships with a stable version of Svelte to support the largest number of people out of the box. To make the language server compatible with Svelte 5 you will need to install the latest versions of `prettier` and `prettier-plugin-svelte` in your project and create (or update) a `.prettierrc` file:

```
npm i --save-dev prettier-plugin-svelte prettier
```

.prettierrc

```
{
  "plugins": ["prettier-plugin-svelte"],
  "overrides": [
    {
      "files": "*.svelte",
      "options": { "parser": "svelte" }
    }
  ]
}
```

## Schedule and future plans

### When is it coming out?

When it's done. The goal is sometime in 2024.

### Should I prepare my code for Svelte 5?

No. You can do the migration towards runes incrementally when Svelte 5 comes out.

### When can I npm install the Svelte 5 preview?

Right now!

```
npm install svelte@next
```

You can also opt into Svelte 5 when creating a new SvelteKit project:

```
npm create svelte@latest
```

### What's left to do?

A great many things. Transitions, for example, are not fully implemented. We also haven't fully solved all aspects of things like server-side rendering. We're getting there!

### Will feature X be part of 5.0?

If you have to ask, then probably not. Aside from runes, 5.0 is mostly about taking everything we've learned over the last few years (including from other frameworks — thanks friends!) and making Svelte the leanest and most powerful framework out there.

We know that some of you are very keen on certain feature ideas, and we are too. We have some big ideas for 5.1 and beyond.

## Discussion, contributing, and help

### I want to help. How do I contribute?

We appreciate your enthusiasm! We welcome issues on the sveltejs/svelte repo. Pull requests are a little dicier right now since many things are in flux, so we recommended starting with an issue.

You can use the `#svelte-5-rc` channel on the Discord server or the tag `#svelte-5` on social media.

### My question wasn't answered. What gives?

It must not have been asked frequently enough. To fix that, stop by the `#svelte-5-rc` channel of the Discord server.

# Breaking changes

While Svelte 5 is a complete rewrite, we have done our best to ensure that most codebases can upgrade with a minimum of hassle. That said, there are a few small breaking changes which may require action on your part. They are listed here.

## Components are no longer classes

In Svelte 3 and 4, components are classes. In Svelte 5 they are functions and should be instantiated differently. If you need to manually instantiate components, you should use `mount` or `hydrate` (imported from `svelte`) instead. If you see this error using SvelteKit, try updating to the latest version of SvelteKit first, which adds support for Svelte 5. If you're using Svelte without SvelteKit, you'll likely have a `main.js` file (or similar) which you need to adjust:

```
import { mount } from 'svelte';
import App from './App.svelte'

const app = mount(App, { target: document.getElementById("app") });

export default app;
```

`mount` and `hydrate` have the exact same API. The difference is that `hydrate` will pick up the Svelte's server-rendered HTML inside its target and hydrate it. Both return an object with the exports of the component and potentially property accessors (if compiled with `accessors: true`). They do not come with the `$on`, `$set` and `$destroy` methods you may know from the class component API. These are its replacements:

For `$on`, instead of listening to events, pass them via the `events` property on the options argument.

```
import { mount } from 'svelte';
import App from './App.svelte'

const app = mount(App, { target: document.getElementById("app"), events: { event: callback } });
```

> Note that using `events` is discouraged — instead, use callbacks

For `$set`, use `$state` instead to create a reactive property object and manipulate it. If you're doing this inside a `.js` or `.ts` file, adjust the ending to include `.svelte`, i.e. `.svelte.js` or `.svelte.ts`.

```
import { mount } from 'svelte';
import App from './App.svelte'

const props = $state({ foo: 'bar' });
const app = mount(App, { target: document.getElementById("app"), props });
props.foo = 'baz';
```

For `$destroy`, use `unmount` instead.

```
import { mount, unmount } from 'svelte';
import App from './App.svelte'

const app = mount(App, { target: document.getElementById("app") });
unmount(app);
```

As a stop-gap-solution, you can also use `createClassComponent` or `asClassComponent` (imported from `svelte/legacy`) instead to keep the same API known from Svelte 4 after instantiating.

```
import { createClassComponent } from 'svelte/legacy';
import App from './App.svelte'

const app = createClassComponent({ component: App, target: document.getElementById("app") });
export default app;
```

If this component is not under your control, you can use the `compatibility.componentApi` compiler option for auto-applied backwards compatibility, which means code using `new Component(...)` keeps working without adjustments (note that this adds a bit of overhead to each component). This will also add `$set` and `$on` methods for all component instances you get through `bind:this`.

```
/// svelte.config.js
export default {
    compilerOptions: {
        compatibility: {
            componentApi: 4
        }
    }
};
```

Note that `mount` and `hydrate` are _not_ synchronous, so things like `onMount` won't have been called by the time the function returns and the pending block of promises will not have been rendered yet (because `#await` waits a microtask to wait for a potentially immediately-resolved promise). If you need that guarantee, call `flushSync` (import from `'svelte'`) after calling `mount/hydrate`.

### Server API changes

Similarly, components no longer have a `render` method when compiled for server side rendering. Instead, pass the function to `render` from `svelte/server`:

```
import { render } from 'svelte/server';
import App from './App.svelte';

const { html, head } = render(App, { props: { message: 'hello' } });
```

In Svelte 4, rendering a component to a string also returned the CSS of all components. In Svelte 5, this is no longer the case by default because most of the time you're using a tooling chain that takes care of it in other ways (like SvelteKit). If you need CSS to be returned from `render`, you can set the `css` compiler option to `'injected'` and it will add `<style>` elements to the `head`.

### Component typing changes

The change from classes towards functions is also reflected in the typings: `SvelteComponent`, the base class from Svelte 4, is deprecated in favour of the new `Component` type which defines the function shape of a Svelte component. To manually define a component shape in a `d.ts` file:

```
import type { Component } from 'svelte';
export declare const MyComponent: Component<{
    foo: string;
}>;
```

To declare that a component of a certain type is required:

```
<script lang="ts">
import type { Component } from 'svelte';
import {
ComponentA,
ComponentB
} from 'component-library';

let component: Component<{ foo: string }> = $state(
Math.random() ? ComponentA : ComponentB
);
</script>

<svelte:component this={component} foo="bar" />
```

The two utility types `ComponentEvents` and `ComponentType` are also deprecated. `ComponentEvents` is obsolete because events are defined as callback props now, and `ComponentType` is obsolete because the new `Component` type is the component type already (e.g. `ComponentType<SvelteComponent<{ prop: string }>>` == `Component<{ prop: string }>`).

### bind:this changes

Because components are no longer classes, using `bind:this` no longer returns a class instance with `$set`, `$on` and `$destroy` methods on it. It only returns the instance exports (`export function/const`) and, if you're using the `accessors` option, a getter/setter-pair for each property.

## Whitespace handling changed

Previously, Svelte employed a very complicated algorithm to determine if whitespace should be kept or not. Svelte 5 simplifies this which makes it easier to reason about as a developer. The rules are:

-   Whitespace between nodes is collapsed to one whitespace
-   Whitespace at the start and end of a tag is removed completely
-   Certain exceptions apply such as keeping whitespace inside `pre` tags

As before, you can disable whitespace trimming by setting the `preserveWhitespace` option in your compiler settings or on a per-component basis in `<svelte:options>`.

## Modern browser required

Svelte 5 requires a modern browser (in other words, not Internet Explorer) for various reasons:

-   it uses `Proxies`
-   elements with `clientWidth`/`clientHeight`/`offsetWidth`/`offsetHeight` bindings use a `ResizeObserver` rather than a convoluted `<iframe>` hack
-   `<input type="range" bind:value={...} />` only uses an `input` event listener, rather than also listening for `change` events as a fallback

The `legacy` compiler option, which generated bulkier but IE-friendly code, no longer exists.

## Changes to compiler options

-   The `false`/`true` (already deprecated previously) and the `"none"` values were removed as valid values from the `css` option
-   The `legacy` option was repurposed
-   The `hydratable` option has been removed. Svelte components are always hydratable now
-   The `enableSourcemap` option has been removed. Source maps are always generated now, tooling can choose to ignore it
-   The `tag` option was removed. Use `<svelte:options customElement="tag-name" />` inside the component instead
-   The `loopGuardTimeout`, `format`, `sveltePath`, `errorMode` and `varsReport` options were removed

## The children prop is reserved

Content inside component tags becomes a snippet prop called `children`. You cannot have a separate prop by that name.

## Dot notation indicates a component

In Svelte 4, `<foo.bar>` would create an element with a tag name of `"foo.bar"`. In Svelte 5, `foo.bar` is treated as a component instead. This is particularly useful inside `each` blocks:

```
{#each items as item}
<item.component {...item.props} />
{/each}
```

## Breaking changes in runes mode

Some breaking changes only apply once your component is in runes mode.

### Bindings to component exports are not allowed

Exports from runes mode components cannot be bound to directly. For example, having `export const foo = ...` in component `A` and then doing `<A bind:foo />` causes an error. Use `bind:this` instead — `<A bind:this={a} />` — and access the export as `a.foo`. This change makes things easier to reason about, as it enforces a clear separation between props and exports.

### Bindings need to be explicitly defined using $bindable()

In Svelte 4 syntax, every property (declared via `export let`) is bindable, meaning you can `bind:` to it. In runes mode, properties are not bindable by default: you need to denote bindable props with the `$bindable` rune.

If a bindable property has a default value (e.g. `let { foo = $bindable('bar') } = $props();`), you need to pass a non-`undefined` value to that property if you're binding to it. This prevents ambiguous behavior — the parent and child must have the same value — and results in better performance (in Svelte 4, the default value was reflected back to the parent, resulting in wasteful additional render cycles).

### accessors option is ignored

Setting the `accessors` option to `true` makes properties of a component directly accessible on the component instance. In runes mode, properties are never accessible on the component instance. You can use component exports instead if you need to expose them.

### immutable option is ignored

Setting the `immutable` option has no effect in runes mode. This concept is replaced by how `$state` and its variations work.

### Classes are no longer "auto-reactive"

In Svelte 4, doing the following triggered reactivity:

```
<script>
let foo = new Foo();
</script>

<button on:click={() => (foo.value = 1)}>{foo.value}</button>
```

This is because the Svelte compiler treated the assignment to `foo.value` as an instruction to update anything that referenced `foo`. In Svelte 5, reactivity is determined at runtime rather than compile time, so you should define `value` as a reactive `$state` field on the `Foo` class. Wrapping `new Foo()` with `$state(...)` will have no effect — only vanilla objects and arrays are made deeply reactive.

### <svelte:component> is no longer necessary

In Svelte 4, components are _static_ — if you render `<Thing>`, and the value of `Thing` changes, nothing happens. To make it dynamic you must use `<svelte:component>`.

This is no longer true in Svelte 5:

```
<script>
import A from './A.svelte';
import B from './B.svelte';

let Thing = $state();
</script>

<select bind:value={Thing}>
<option value={A}>A</option>
<option value={B}>B</option>
</select>

<!-- these are equivalent -->
<Thing />
<svelte:component this={Thing} />
```

### Touch and wheel events are passive

When using `onwheel`, `onmousewheel`, `ontouchstart` and `ontouchmove` event attributes, the handlers are passive to align with browser defaults. This greatly improves responsiveness by allowing the browser to scroll the document immediately, rather than waiting to see if the event handler calls `event.preventDefault()`.

In the very rare cases that you need to prevent these event defaults, you should use `on` instead (for example inside an action).

## Other breaking changes

### Stricter @const assignment validation

Assignments to destructured parts of a `@const` declaration are no longer allowed. It was an oversight that this was ever allowed.

### :is(...) and :where(...) are scoped

Previously, Svelte did not analyse selectors inside `:is(...)` and `:where(...)`, effectively treating them as global. Svelte 5 analyses them in the context of the current component. As such, some selectors may now be treated as unused if they were relying on this treatment. To fix this, use `:global(...)` inside the `:is(...)/:where(...)` selectors.

When using Tailwind's `@apply` directive, add a `:global` selector to preserve rules that use Tailwind-generated `:is(...)` selectors:

```
main :global {
@apply bg-blue-100 dark:bg-blue-900
}
```

### CSS hash position no longer deterministic

Previously Svelte would always insert the CSS hash last. This is no longer guaranteed in Svelte 5. This is only breaking if you have very weird css selectors.

### Scoped CSS uses :where(...)

To avoid issues caused by unpredictable specificity changes, scoped CSS selectors now use `:where(.svelte-xyz123)` selector modifiers alongside `.svelte-xyz123` (where `xyz123` is, as previously, a hash of the `<style>` contents). You can read more detail here.

In the event that you need to support ancient browsers that don't implement `:where`, you can manually alter the emitted CSS, at the cost of unpredictable specificity changes:

```
css = css.replace(/:where\((.+?)\)/, '$1');
```

### Error/warning codes have been renamed

Error and warning codes have been renamed. Previously they used dashes to separate the words, they now use underscores (e.g. foo-bar becomes foo\_bar). Additionally, a handful of codes have been reworded slightly.

### Reduced number of namespaces

The number of valid namespaces you can pass to the compiler option `namespace` has been reduced to `html` (the default), `mathml` and `svg`.

The `foreign` namespace was only useful for Svelte Native, which we're planning to support differently in a 5.x minor.

### beforeUpdate/afterUpdate changes

`beforeUpdate` no longer runs twice on initial render if it modifies a variable referenced in the template.

`afterUpdate` callbacks in a parent component will now run after `afterUpdate` callbacks in any child components.

Both functions are disallowed in runes mode — use `$effect.pre(...)` and `$effect(...)` instead.

### contenteditable behavior change

If you have a `contenteditable` node with a corresponding binding _and_ a reactive value inside it (example: `<div contenteditable=true bind:textContent>count is {count}</div>`), then the value inside the contenteditable will not be updated by updates to `count` because the binding takes full control over the content immediately and it should only be updated through it.

### oneventname attributes no longer accept string values

In Svelte 4, it was possible to specify event attributes on HTML elements as a string:

```
<button onclick="alert('hello')">...</button>
```

This is not recommended, and is no longer possible in Svelte 5, where properties like `onclick` replace `on:click` as the mechanism for adding event handlers.

### null and undefined become the empty string

In Svelte 4, `null` and `undefined` were printed as the corresponding string. In 99 out of 100 cases you want this to become the empty string instead, which is also what most other frameworks out there do. Therefore, in Svelte 5, `null` and `undefined` become the empty string.

### bind:files values can only be null, undefined or FileList

`bind:files` is now a two-way binding. As such, when setting a value, it needs to be either falsy (`null` or `undefined`) or of type `FileList`.

### Bindings now react to form resets

Previously, bindings did not take into account `reset` event of forms, and therefore values could get out of sync with the DOM. Svelte 5 fixes this by placing a `reset` listener on the document and invoking bindings where necessary.

### walk not longer exported

`svelte/compiler` reexported `walk` from `estree-walker` for convenience. This is no longer true in Svelte 5, import it directly from that package instead in case you need it.

### Content inside svelte:options is forbidden

In Svelte 4 you could have content inside a `<svelte:options />` tag. It was ignored, but you could write something in there. In Svelte 5, content inside that tag is a compiler error.

### <slot> elements in declarative shadow roots are preserved

Svelte 4 replaced the `<slot />` tag in all places with its own version of slots. Svelte 5 preserves them in the case they are a child of a `<template shadowrootmode="...">` element.

### <svelte:element> tag must be an expression

In Svelte 4, `<svelte:element this="div">` is valid code. This makes little sense — you should just do `<div>`. In the vanishingly rare case that you _do_ need to use a literal value for some reason, you can do this:

```
<svelte:element this={"div"}>
```

Note that whereas Svelte 4 would treat `<svelte:element this="input">` (for example) identically to `<input>` for the purposes of determining which `bind:` directives could be applied, Svelte 5 does not.

### mount plays transitions by default

The `mount` function used to render a component tree plays transitions by default unless the `intro` option is set to `false`. This is different from legacy class components which, when manually instantiated, didn't play transitions by default.

### <img src={...}> and {@html ...} hydration mismatches are not repaired

In Svelte 4, if the value of a `src` attribute or `{@html ...}` tag differ between server and client (a.k.a. a hydration mismatch), the mismatch is repaired. This is very costly: setting a `src` attribute (even if it evaluates to the same thing) causes images and iframes to be reloaded, and reinserting a large blob of HTML is slow.

Since these mismatches are extremely rare, Svelte 5 assumes that the values are unchanged, but in development will warn you if they are not. To force an update you can do something like this:

```
<script>
let { markup, src } = $props();

if (typeof window !== 'undefined') {
// stash the values...
const initial = { markup, src };

// unset them...
markup = src = undefined;

$effect(() => {
// ...and reset after we've mounted
markup = initial.markup;
src = initial.src;
});
}
</script>

{@html markup}
<img {src} />
```

### Hydration works differently

Svelte 5 makes use of comments during server side rendering which are used for more robust and efficient hydration on the client. As such, you shouldn't remove comments from your HTML output if you intend to hydrate it, and if you manually authored HTML to be hydrated by a Svelte component, you need to adjust that HTML to include said comments at the correct positions.

# Deprecations

Aside from the breaking changes listed on the previous page, Svelte 5 should be a drop-in replacement for Svelte 4. That said, there are some features that we will remove in a future major version of Svelte, and we encourage you to update your apps now to avoid breaking changes in future.

## beforeUpdate and afterUpdate

`beforeUpdate(fn)` schedules the `fn` callback to run immediately before any changes happen inside the current component. `afterUpdate(fn)` schedules it to run after any changes have taken effect.

These functions run indiscriminately when _anything_ changes. By using `$effect.pre` and `$effect` instead, we can ensure that work only happens when the things we care about have changed. The difference is visible in this example — with `afterUpdate`, the callback runs on every `mousemove` event, whereas with `$effect`, the function only runs when `temperature` changes:

```
<script>
    import { afterUpdate } from 'svelte';

    let coords = $state({ x: 0, y: 0 });
    let temperature = $state(50);
    let trend = $state('...');

    let prev = temperature;

    afterUpdate(() => {
        console.log('running afterUpdate');
    });

    $effect(() => {
        console.log('running $effect');
        if (temperature !== prev) {
            trend = temperature > prev ? 'getting warmer' : 'getting cooler';
            prev = temperature;
        }
    });
</script>

<svelte:window on:mousemove={(e) => coords = { x: e.clientX, y: e.clientY }} />

<input type="range" bind:value={temperature}>
<p>{trend}</p>
```

Note that using `$effect` and `$effect.pre` will put you in runes mode — be sure to update your props and state accordingly.

## createEventDispatcher

`createEventDispatcher` returns a function from which you can dispatch custom events. The usage is somewhat boilerplate-y, but it was encouraged in Svelte 4 due to consistency with how you listen to dom events (via `on:click` for example).

Svelte 5 introduces event attributes which deprecate event directives (`onclick` instead of `on:click`), and as such we also encourage you to use callback properties for events instead:

```
<script>
    import { createEventDispatcher } from 'svelte';
    const dispatch = createEventDispatcher();
    let { greet } = $props();

    function greet() {
        dispatch('greet');
    }
</script>

<button onclick={greet}>greet</button>
```

When authoring custom elements, use the new host rune to dispatch events (among other things):

```
<script>
    import { createEventDispatcher } from 'svelte';
    const dispatch = createEventDispatcher();

    function greet() {
        $host().dispatchEvent(new CustomEvent('greet'));
    }
</script>

<button onclick={greet}>greet</button>
```

Note that using `$props` and `$host` will put you in runes mode — be sure to update your props and state accordingly.

## <svelte:component> in runes mode

In previous versions of Svelte, the component constructor was fixed when the component was rendered. In other words, if you wanted `<X>` to re-render when `X` changed, you would either have to use `<svelte:component this={X}>` or put the component inside a `{#key X}...{/key}` block.

In Svelte 5 this is no longer true — if `X` changes, `<X>` re-renders.

In some cases `<object.property>` syntax can be used as a replacement; a lowercased variable with property access is recognized as a component in Svelte 5.

For complex component resolution logic, an intermediary, capitalized variable may be necessary. E.g. in places where `@const` can be used:

```
{#each items as item}
    {@const Component = item.condition ? Y : Z}
    <Component />
{/each}
```

A derived value may be used in other contexts:

```
<script>
    ...
    let condition = $state(false);
    const Component = $derived(condition ? Y : Z);
</script>

<Component />
```

## immutable

The `immutable` compiler option is deprecated. Use runes mode instead, where all state is immutable (which means that assigning to `object.property` won't cause updates for anything that is observing `object` itself, or a different property of it).