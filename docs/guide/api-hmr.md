# HMR API

:::tip Note
This is the client HMR API. For handling HMR update in plugins, see [handleHotUpdate](./api-plugin#handlehotupdate).

The manual HMR API is primarily intended for framework and tooling authors. As an end user, HMR is likely already handled for you in the framework specific starter templates.
:::

Vite exposes its manual HMR API via the special `import.meta.hot` object:

```ts
interface ImportMeta {
  readonly hot?: ViteHotContext
}

type ModuleNamespace = Record<string, any> & {
  [Symbol.toStringTag]: 'Module'
}

interface ViteHotContext {
  readonly data: any

  accept(): void
  accept(cb: (mod: ModuleNamespace | undefined) => void): void
  accept(dep: string, cb: (mod: ModuleNamespace | undefined) => void): void
  accept(
    deps: readonly string[],
    cb: (mods: Array<ModuleNamespace | undefined>) => void
  ): void

  dispose(cb: (data: any) => void): void
  decline(): void
  invalidate(): void

  // `InferCustomEventPayload` provides types for built-in Vite events
  on<T extends string>(
    event: T,
    cb: (payload: InferCustomEventPayload<T>) => void
  ): void
  send<T extends string>(event: T, data?: InferCustomEventPayload<T>): void
}
```

## Required Conditional Guard

First of all, make sure to guard all HMR API usage with a conditional block so that the code can be tree-shaken in production:

```js
if (import.meta.hot) {
  // HMR code
}
```

## `hot.accept(cb)`

For a module to self-accept, use `import.meta.hot.accept` with a callback which receives the updated module:

```js
export const count = 1

if (import.meta.hot) {
  import.meta.hot.accept((newModule) => {
    if (newModule) {
      // newModule is undefined when SyntaxError happened
      console.log('updated: count is now ', newModule.count)
    }
  })
}
```

A module that "accepts" hot updates is considered an **HMR boundary**.

Note that Vite's HMR does not actually swap the originally imported module: if an HMR boundary module re-exports imports from a dep, then it is responsible for updating those re-exports (and these exports must be using `let`). In addition, importers up the chain from the boundary module will not be notified of the change.

This simplified HMR implementation is sufficient for most dev use cases, while allowing us to skip the expensive work of generating proxy modules.

## `hot.accept(deps, cb)`

A module can also accept updates from direct dependencies without reloading itself:

```js
import { foo } from './foo.js'

foo()

if (import.meta.hot) {
  import.meta.hot.accept('./foo.js', (newFoo) => {
    // the callback receives the updated './foo.js' module
    newFoo?.foo()
  })

  // Can also accept an array of dep modules:
  import.meta.hot.accept(
    ['./foo.js', './bar.js'],
    ([newFooModule, newBarModule]) => {
      // the callback receives the updated modules in an Array
    }
  )
}
```

## `hot.dispose(cb)`

A self-accepting module or a module that expects to be accepted by others can use `hot.dispose` to clean-up any persistent side effects created by its updated copy:

```js
function setupSideEffect() {}

setupSideEffect()

if (import.meta.hot) {
  import.meta.hot.dispose((data) => {
    // cleanup side effect
  })
}
```

## `hot.data`

The `import.meta.hot.data` object is persisted across different instances of the same updated module. It can be used to pass on information from a previous version of the module to the next one.

## `hot.decline()`

Calling `import.meta.hot.decline()` indicates this module is not hot-updatable, and the browser should perform a full reload if this module is encountered while propagating HMR updates.

## `hot.invalidate()`

A self-accepting module may realize during runtime that it can't handle a HMR update, and so the update needs to be forcefully propagated to importers. By calling `import.meta.hot.invalidate()`, the HMR server will invalidate the importers of the caller, as if the caller wasn't self-accepting.

Note that you should always call `import.meta.hot.accept` even if you plan to call `invalidate` immediately afterwards, or else the HMR client won't listen for future changes to the self-accepting module. To communicate your intent clearly, we recommend calling `invalidate` within the `accept` callback like so:

    ```ts
    import.meta.hot.accept(module => {
      // You may use the new module instance to decide whether to invalidate.
      if (cannotHandleUpdate(module)) {
        import.meta.hot.invalidate()
      }
    })
    ```

## `hot.on(event, cb)`

Listen to an HMR event.

The following HMR events are dispatched by Vite automatically:

- `'vite:beforeUpdate'` when an update is about to be applied (e.g. a module will be replaced)
- `'vite:beforeFullReload'` when a full reload is about to occur
- `'vite:beforePrune'` when modules that are no longer needed are about to be pruned
- `'vite:invalidate'` when a module is invalidated with `import.meta.hot.invalidate()`
- `'vite:error'` when an error occurs (e.g. syntax error)

Custom HMR events can also be sent from plugins. See [handleHotUpdate](./api-plugin#handlehotupdate) for more details.

## `hot.send(event, data)`

Send custom events back to Vite's dev server.

If called before connected, the data will be buffered and sent once the connection is established.

See [Client-server Communication](/guide/api-plugin.html#client-server-communication) for more details.
