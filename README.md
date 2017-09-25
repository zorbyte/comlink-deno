# Comlink
A tiny RPC library that works on windows, iframes, WebWorkers and
ServiceWorkers.

**TL;DR: With Comlink you can work on objects from another JavaScript realm
(like a Worker or an iframe) as if it was a local object. Just use `await`
whenever using the remote value.**

```
$ npm install --save comlinkjs
```

Comlink allows you to expose an arbitrary JavaScript value (objects, classes,
functions, etc) to the endpoint of an communications channel. Anything that
works with `postMessage` can be used as a communication channel. On the other
end of that channel you can use Comlink to synthesize an ES6 proxy. Every action
performed on that proxy object will be serialized using a simple (and naïve) RPC
protocol and be applied to the exposed value on the other side.

**Size**: ~3.1k, ~1.3k gzip’d.

## Example

```html
<-- index.html -->
<!doctype html>
<script src="../../dist/comlink.global.js"></script>
<script>
  const worker = new Worker('worker.js');
  // WebWorkers use `postMessage` and therefore work with Comlink.
  const api = Comlink.proxy(worker);

  async function init() {
    // Note the usage of `await`:
    const app = await new api.App();

    alert(`Counter: ${await app.count}`);
    await app.inc();
    alert(`Counter: ${await app.count}`);
  };

  init();
</script>
```

```js
// worker.js
importScripts('../dist/comlink.global.js');

class App {
  constructor() {
    this._counter = 0;
  }

  get count() {
    return this._counter;
  }

  inc() {
    this._counter++;
  }
}

Comlink.expose({App}, self);
```

## API

The Comlink module exports 3 functions:

### `Comlink.proxy(endpoint)`

`proxy` creates an ES6 proxy and sends all operations performed on that proxy
through the channel behind `endpoint`. `endpoint` can be a `Window`, a `Worker`
or a `MessagePort`.* The other endpoint of the channel should be passed to
`expose`.

Note that as of now all parameters for function or method invocations will be
structurally cloned or transferred if they are [transferable]. As a result,
callback functions are currently not supported as parameter values.

*) Technically it can be any object with `postMessage`, `addEventListener` and
`removeEventListener`.

### `expose(rootObj, endpoint)`

`expose` is the counter-part to `proxy`. It listens for RPC messages on
`endpoint` and applies the operations to `rootObj`. Return values of functions
will be structurally cloned or transfered if they are [transferable]. The same
restrictions as for `proxy` apply.

### `proxyValue(value)`

If structurally cloning a value is undesired (either for a function parameter or
its return value), wrapping the value in a `proxyValue` call proxy that value
instead. This is necessary for callback functions being passed around or for the
Singleton pattern:

```js
// main.js
const worker = new Worker('worker.js');
const doStuff = Comlink.proxy(worker);
await doStuff(result => console.log(result));
```

```js
// worker.js
Comlink.expose(async function (f) {
  const result = /* omg so expensive */;
  /* f is a proxy, as if created by proxy(). So we need to use `await. */
  await f(result);
}, self);
```

## Module formats

The Comlink module is provided in 3 different formats:

* **“es6”**: This package uses the native ES6 module format. Due to some
  necessary hackery, the module exports a `Comlink` object. Import it as
  follows:

  ```js
  import {Comlink} from '../dist/comlink.es6.js';

  // ...
  ```

* **“global”**: This package adds a `Comlink` namespace on `self`. Useful for
  workers or projects without a module loader.
* **“umd”**: This package uses [UMD] so it is compatible with AMD, CommonJS
  and requireJS.

These packages can be mixed and matched. A worker using `global` can be
connected to a window using `es6`. For the sake of network conservation, I do
recommend sticking to one format, though.

[UMD]: https://github.com/umdjs/umd
[transferable]: https://developer.mozilla.org/en-US/docs/Web/API/Transferable
[MessagePort]: https://developer.mozilla.org/en-US/docs/Web/API/MessagePort

---
License Apache-2.0
