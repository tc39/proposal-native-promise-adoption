# Native promise adoption

ECMAScript proposal for adopting the state of native promises without using their `.then`.

## Status

[The TC39 Process](https://tc39.es/process-document/)

**Stage**: 1

**Champions**:
- Mathieu Hofman ([@mhofman](https://github.com/mhofman)) (Agoric)

## Motivation

Currently, native promises are already adopted in the [`Await`](https://tc39.es/ecma262/multipage/control-abstraction-objects.html#await) case (introduced in https://github.com/tc39/ecma262/pull/1250) and [`AsyncFromSyncIteratorContinuation`](https://tc39.es/ecma262/#sec-asyncfromsynciteratorcontinuation), ignoring any own `.then` or modified `Promise.prototype.then`. The `Await` AO is internally used not only for the `await` syntax, but also in the [`Array.fromAsync`](https://github.com/tc39/proposal-array-from-async) API, and some of the [async iterator helpers](https://github.com/tc39/proposal-async-iterator-helpers).

On the other hand, the [promise resolve functions](https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-promise-resolve-functions) do not attempt to recognize whether the resolution value is a native promise and always take the `.then` path assimilating the value. This behavior is observable not only when using APIs obviously related to the resolver functions (e.g. `new Promise()` / `Promise.withResolvers()`), but also anywhere the spec relies on [PromiseCapability records](https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-promisecapability-records), including `return` in async functions.

This results in inconsistent behavior in the spec where some syntax and APIs adopt the native promise internal state without triggering `.then` machinery, and other syntax and APIs assimilate through `.then`.

### Note on adoption bailout and fallback to `.then`

`Await` and similar operations performing native promise adoption first obtain an intrinsic `%Promise%` from the value, and adopt that promise. For that they rely on the [`PromiseResolve` AO](https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-promise-resolve), which passes through the value if it is a native promise (satisfying the [`IsPromise`](https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-ispromise) check), and the `.constructor` of the native promise value matches `%Promise%`. That latter check is actually prone to interference by user code, which can currently force the creation of a new promise by polluting `%Promise.prototype%.constructor`, triggering the assimilation of the value instead. https://github.com/tc39/ecma262/pull/3689 aims to change that check to a `[[GetPrototypeOf]]` based check which is not subject to such interferences, guaranteeing that native promise values are always adopted in those operations.

## Proposal

The Promises/A+ specification allows and actually [expects promise implementations to recognize their own instances](https://promisesaplus.com/#point-49), and adopt their state without using the `.then` machinery.

This proposal implements that adoption, for native promises that are a base promise identified as having a `%Promise.prototype%` proto. This matches the intent of `Await` and `PromiseResolve` to continue using the `.then` mechanism for derived promises, and the proposed semantics of https://github.com/tc39/ecma262/pull/3689 to use a prototype based check not subject to pollution.

If wholesale adoption through resolve functions is not web compatible, the proposal would pivot to at least changing the semantics of `return` in async functions to perform an adoption (solving issue https://github.com/tc39/ecma262/issues/2770). We could also investigate opt-in signals from the application, either implicit (`async` code parsed), or explicit.

## Relation to other PRs and proposals

Combined with https://github.com/tc39/ecma262/pull/3689, this proposal guarantees that no user code is executed when adopting a base native promise (of the same realm), whether it's `await`-ing or `return`-ing  the result value of a call to another async function.

This proposal does not change the way non promise thenables are handled, including when the resolution values are "unexpected thenables" (see [thenable curtailment proposal](https://github.com/tc39/proposal-thenable-curtailment)). It does however enable some mitigations such as guaranteeing that if a thenable fulfillment is ever possible, such fulfillment value would be adopted through a chain of native promises without unwrapping.

This PR does not make any changes to the number of tick / jobs for resolving promises, leaving that potential optimization to the existing [faster promise adoption proposal](https://github.com/tc39/proposal-faster-promise-adoption). It does reduce the scope of that proposal to focus on the number of jobs observable only by counting ticks when adopting promises, and potentially to better detect promise resolution cycles. Without normative adoption, that proposal would need to disable any optimization when `Promise.prototype.then` is polluted, effectively considering native promises as mere thenables.

## Web compatibility

### Observable effects of the proposed change

Besides the `Promise` related functions (resolvers and helpers like `Promise.all`), there are 2 places in the spec that invoke the Promise resolve functions with a user controlled value which may be a native promise subject to adoption:
- [`NewPromiseReactionJob`](https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-newpromisereactionjob), for the assimilation of promise reaction results into the chained promise (aka `promise.then(() => Promise.resolve(42))`)
- [`AsyncBlockStart`](https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-asyncblockstart) for the assimilation of the result value of an async function (aka `async () => Promise.resolve(42)`)

While the spec and host can themselves add promise reactions directly through `PerformPromiseThen`, in the 262 case, it never does so with a handler that will return a native promise (or for that matter that has a chained promise result), except for the `%Promise.prototype.then%` case. A pollution of `Promise.prototype.then` would interfere before we even got to the potential adoption point.

As such, on the 262 side, there only remains the result value of `async` function where a pollution of `Promise.prototype.then` can surprisingly interfere in the adoption of a native promise value.

### Expected impact

Code today already cannot reliably rely on hijacking a promise `.then` to detect when the outcome of a native promise is used, since `await` amongst other operations perform an internal `PerformPromiseThen`. 

None-the-less, some libraries like [Angular's `zone.js`](https://github.com/angular/angular/tree/main/packages/zone.js) replace the global `Promise` and hijack `Promise.prototype.then`. Applications that use such libraries must transpile async code to avoid some of the native adoption points already existing in the spec. If a native promise is encountered (from some host or other intrinsic API), user code will simply perform a `.then` on them, which is where the `Promise.prototype.then` [hijack](https://github.com/angular/angular/blob/0a4ad9867b382a785b97f73d90b05bac0f266c89/packages/zone.js/lib/common/promise.ts#L603-L608) comes in to transform the handling of these promises into a zone aware promise. In the case of zone.js at least, they do not seem to rely on this mechanism to track when a native resolver adopts a native promise.

### Measurements

For confirmation, engines should instrument their existing implementation to detect how often a custom `.then` behavior would be ignored by this proposed promise adoption. If the resolution value has a `.then` function (Step 12/13 of the [Promise Resolve Functions](https://tc39.es/ecma262/multipage/control-abstraction-objects.html#sec-promise-resolve-functions)), implementations would need to check whether the resolution is a native promise with a `%Promise.prototype%` proto. This defines the set of "adoptable promises". Of this set the affected cases would be if the `thenAction` function does not match `%Promise.prototype.then%`, or if obtaining the `then` function (in step 9) triggered user code.

## Userland "safe" promise capability

It is possible to leverage the adoption semantics of the `await` syntax to create a `SafePromise` constructor whose resolver does not trigger the `.then` machinery for native promises.

```js
const makePromiseKit = Promise.withResolvers.bind(Promise);

async function makePromise(executor) {
  const {promise, resolve} = makePromiseKit();

  executor(
    value => resolve({__proto__: null, status: 'resolved', value}),
    reason => resolve({__proto__: null, status: 'rejected', reason}),
  );

  const resolution = await promise;

  if (resolution.status === 'resolved') {
    return await resolution.value;
  } else {
    throw resolution.reason;
  }
}

function SafePromise(executor) {
  if (new.target !== SafePromise) throw TypeError();

  return makePromise(executor);
}
Object.setPrototypeOf(SafePromise, Promise);
Object.defineProperty(SafePromise, 'prototype', {value: Promise.prototype, writable: false});
```

This however wouldn't affect promise capabilities internal to the spec or the host that are created directly from the `%Promise%` constructor, including adoption of reaction results into chained promises, or the result value of `async` functions.

Some intrinsics rely on a regular species mechanism to construct promise capabilities for their results, and would only be affected if `%Promise.prototype%.constructor` was replaced by this `SafePromise` constructor.
