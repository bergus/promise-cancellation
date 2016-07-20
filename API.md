# API

## Terminology

1. A **cancellation token** (**`CancelToken`**) is an object with methods
   for determining whether and when an operation should be cancelled.
2. The cancellation can be **requested** by the issuer of the `CancelToken`,
   denoting that the result of an operation is no longer of interest
   and that the operation should be terminated if applicable.
3. A **cancelled token** is a `CancelToken` that represents
   a requested cancellation
4. A **cancellation reason** is a value used to request a cancellation
   and reject the respective promises.
5. One `CancelToken` might be **associated** with a promise.
6. A **cancelled promise** is a promise that got rejected
   because its cancellation was requested through its associated token
7. A **cancelled callback** is an `onFulfilled` or `onRejected` handler
   whose corresponding cancellation token has been cancelled.
   It might be considered an **unregistered** or **ignored** callback.

## Promises/A+

The requirements of the Promises/A+ specification should be amended.
Specifically, extensions are made to the following sections:

### [The `then` method](https://github.com/promises-aplus/promises-spec#the-then-method)

A promise's `then` method accepts *three* arguments: <code>promise.then(onFulfilled, onRejected<i>, token</i>)</code>

2.2.1. If `onFulfilled` is a function:  
2.2.1.1. it must be called *unless it is cancelled* after `promise` is fulfilled, with `promise`’s value as its first argument.  
2.2.2.1. If `onRejected` is a function,  
2.2.2.2. it must be called *unless it is cancelled* after `promise` is rejected, with `promise`’s reason as its first argument.  

Note: 2.2.1.3. and 2.2.2.3. ("must not be called more than once") stay in place, and still at most one of the two is called.

2.2.6.1. If/when `promise` is fulfilled, all respective *uncancelled* `onFulfilled` callbacks must execute in the order of their originating calls to `then`.  
2.2.6.2. If/when `promise` is rejected, all respective *uncancelled* `onRejected` callbacks must execute in the order of their originating calls to `then`.

*2.2.7.0. If `token` is a cancellation token, it is associated with `promise2`.*  
2.2.7.3. If `onFulfilled` is not a function and `promise1` is fulfilled *and `promise2` was not already cancelled*, `promise2` must be fulfilled with the same value as `promise1`.  
2.2.7.4. If `onRejected` is not a function and `promise1` is rejected *and `promise2` was not already cancelled*, `promise2` must be rejected with the same reason as `promise1`.

### [The Promise Resolution Procedure](https://github.com/promises-aplus/promises-spec#the-promise-resolution-procedure)

2.3.2 If `x` is a promise, adopt its state  
2.3.2.1 If `x` is pending, `promise` must remain pending until `x` is fulfilled or rejected,
        *unless `promise` is cancelled through its associated token.*  
2.3.2.2 If/when `x` is fulfilled, fulfill `promise` with the same value *unless `promise` had been cancelled*.  
2.3.2.3 If/when `x` is rejected, reject `promise` with the same reason *unless `promise` had been cancelled*.

2.3.3.1 If `then` is a function, call it with `x` as `this`, first argument `resolvePromise`, second argument `rejectPromise` *and third argument `token`*, where  
*2.3.3.1.5 `token` is a `CancelToken` reflecting the state of the token associated to `promise` (it can be the same object, or a proxy for it).*

## ES next

All of the `Promise` methods would be amended, and a new `CancelToken` class would be introduced.

### Promise internals

Promise instances would get an additional [[CancelToken]] internal slot for their associated cancellation token.

*PromiseReaction* records would get an additional [[CancelToken]] field for the cancellation token registered with them.

*RejectPromise* and *FulfillPromise* get an additional step that empties the [[CancelToken]] slot of the promise.

#### CancelPromise

Essentially the same as *RejectPromise*, but without removing the cancellation token and calling the *HostPromiseRejectionTracker*.

#### NewPromiseCapability

Gets a new `token` parameter which it passes through as the second argument to the constructor.

#### PromiseReactionJob

If the [[Capability]].[[Promise]] field of the reaction is undefined, returns immediately.
Otherwise, sets the [[Capability]].[[Promise]] field of the reaction to undefined.  
Note: This allows racing multiple reactions for the same promise without a cumbersome token
and is used by the [optional but useful methods below](#optional-but-useful).

If the [[CancelToken]] field of the reaction contains a cancellation token, its `.requested` property is accessed.
If it yields `true`, the handler is set to `undefined` (so that it is not called and the resolution is passed on).

#### PromiseResolveThenableJob

If the [[CancelToken]] slot of the `promiseToResolve` contains a cancellation token,
it is passed as the third argument to the thenables `then` method.

### Promise constructor

Gets a second parameter `token`. Its `.length` stays 1.

The new promise gets a [[CancelToken]] internal slot that is filled with the `token` argument.

If the `token` argument is a cancellation token, its `.requested` property is accessed.  
If it yields `false`, *RegisterPromise* is called with the token and the new Promise instance.  
If it yields `true`, the `executor` is not invoked, and instead *CancelPromise* is run immediately.

### Promise.resolve

Gets a second parameter `token`. Its `.length` stays 1.

If a `token` is passed, the input `x` is only returned if its [[CancelToken]] slot contains the same object.

The `token` argument is passed to the promise creation (*NewPromiseCapability*).

### Promise.prototype.then

Gets a third parameter `token`. Its `.length` stays 2.

The `token` argument is passed to the promise creation (*NewPromiseCapability*)
and attached to the [[CancelToken]] field of both created *PromiseReaction*s.

### Promise.prototype.catch

Gets a second parameter `token`. Its `.length` stays 1.

The `token` argument is pass through to `then`.

### Promise.race

Creates its own `CancelToken` that is passed to each `then` invocation.

The cancellation is requested when the promise is settled by any of the resolving functions.
The cancellation reason is some kind of `"result already settled"`.

Possibly takes an optional `token` as second parameter that is associated with the result promise
and also immediately cancels the token that is passed to the inputs.

### Promise.all

Creates its own `CancelToken` that is passed to each `then` invocation.

The cancellation is requested when the promise is rejected.
The cancellation reason is some kind of `"result already rejected"`.

Possibly takes an optional `token` as second parameter that is associated with the result promise
and also immediately cancels the token that is passed to the inputs.

### CancelToken internals

`CancelToken` instances have [[Result]], [[RegisteredPromises]] and [[Reactions]] slots.

#### RegisterPromise

with parameters `token` and `promise`.

Assert: the [[Result]] of the token is still empty.

Appends the `promise` to the [[RegisteredPromises]] list of the `token`.

#### CancelFunction

Takes one parameter, a `reason` value.
It is bound via an internal slot to a specific `token`.

When the `token`s [[Result]] is not empty, it returns `undefined` immediately.

It sets the [[Result]] of the `token` to the `reason`.

It creates a list of results.

For each reaction in the [[Reactions]] list of the `token`,
it runs *EnqueueJob*(*"PromiseJobs"*, *PromiseReactionJob*, reaction, reason)
and appends the reaction.[[Capability]].[[Promise]] to the results.

For each promise in the [[RegisteredPromises]] list of the `token`,
it runs *CancelPromise*(promise, `reason`).

Note: Removing all *PromiseReaction*s which were registered with the token
from the lists they are contained in respectively
would be nice for garbage collection but is cumbersome to describe.

It empties the [[Reactions]] and [[RegisteredPromises]] lists.

It returns *CreateArrayFromList*(results).

### CancelToken constructor

The `CancelToken` constructor takes one parameter, an `executor` function.

It is designed to be subclassable.

It throws when called without `new` or without a function as the argument.

It creates a new token instances with the 3 slots and initialises [[Result]] to empty,
[[RegisteredPromises]] to a new list and [[Reactions]] to a new list.

It synchronously calls the executor with a new *CancelFunction* bound to the
new instance.

### CancelToken.prototype.requested

Is a getter property that returns whether the [[Result]] of the token is not empty.

### CancelToken.source

Is a factory function that calls its `this` value with a *GetCancelExecutor* function
so that it returns a `{cancel, token}` tuple.

### CancelToken.prototype.subscribe

with two parameters `onCancelled` and `token`. Its `.length` is 1.

Throws if not invoked on a cancellation token.

Creates a new promise capability by calling *NewPromiseCapability*
with the intrinsic *Promise* constructor and the `token`.

Sets `onCancelled` to undefined if it is not callable.

Creates a new reaction *PromiseReaction* { [[Capability]]: capability, [[Type]]: *"Reject"*, [[Handler]]: `onCancelled`, [[CancelToken]]: `token` }

If the [[Result]] is empty, appends the reaction to [[Reactions]].
Otherwise, runs *EnqueueJob*(*"PromiseJobs"*, *PromiseReactionJob*, reaction, [[Result]])

Returns the promise of the capability.

## Optional but useful

### Promise.prototype.token

Is a getter that accesses the [[CancelToken]] slot.

Note: introduces synchronous inspection.

### CancelToken.prototype.reason

Is a getter that accesses the [[Result]] slot (and throws it if is empty).

### Promise.prototype.trifurcate

Note: See the [reasoning](trifurcation.md).

with 3 parameters `onFulfilled`, `onRejected` and `onCancelled`.

Throws if not invoked on a promise.

Creates a new promise capability by calling *NewPromiseCapability*
with the appropriate species constructor and no token.

Sets all non-callable arguments to undefined.

Creates three new *PromiseReaction*s with the capability, the appropriate types
(*"Fulfill"*, *"Reject"*, *"Reject"*), the respective handler, and no token.

If the value of [[PromiseState]] is *"pending"*,
appends the fulfill reaction to [[PromiseFulfillReactions]],
appends the reject reaction to [[PromiseRejectReactions]],
and if an associated [[CancelToken]] exists appends the cancel reaction to [[CancelToken]].[[Reactions]].

Note: the promise on the cancel reaction is not supposed to be appended to the result list of the *CancelFunction*
even though it is described as such above.

Else if the value of [[PromiseState]] is *"fulfilled"*, enqueues a promise job for the fulfilled reaction
Else the the value of [[PromiseState]] is *"rejected"*. If an associated [[CancelToken]] exists,
enqueues a promise job for the cancel reaction with the [[CancelToken]].[[Reason]].
Otherwise enqueues a promise job for the rejected reaction, and tracks the rejection appropriately.

Returns the promise of the capability.

### Promise.prototype.finally

with 1 parameter `onSettled`.

Throws if not invoked on a promise or if `onSettled` is not callable.

Creates a new promise capability by calling *NewPromiseCapability*
with the appropriate species constructor and no token.

Creates a new reaction *PromiseReaction* { [[Capability]]: capability, [[Type]]: *"Fulfill"*, [[Handler]]: `onSettled`, [[CancelToken]]: null }

Note: the type is irrelevant as the handler is never undefined.

If the value of [[PromiseState]] is *"pending"*,
appends the reaction to [[PromiseFulfillReactions]], [[PromiseRejectReactions]]
and if an associated [[CancelToken]] exists also to [[CancelToken]].[[Reactions]].

Otherwise runs *EnqueueJob*(*"PromiseJobs"*, *PromiseReactionJob*, reaction, undefined).

Note: no argument should be passed to `onSettled` here,
even if a *PromiseReactionJob* is described to always pass one.

Creates a *PassThrough* function that when invoked returns the promise that `finally`
was called upon.

Invokes the `then` method on the capability promise with the arguments
*PassThrough*, undefined, and [[CancelToken]].

Returns the result of that invocation.

### CancelToken.prototype.subscribeOrCall

with 2 parameters `onCancelled` and `onCalled`.

Throws if not invoked on a cancellation token.

Creates a new promise capability by calling *NewPromiseCapability*
with the intrinsic *Promise* constructor and no token.

Sets all non-callable arguments to undefined.

Creates a new reaction *PromiseReaction* { [[Capability]]: capability, [[Type]]: *"Fulfill"*, [[Handler]]: `onCancelled`, [[CancelToken]]: null }

If the [[Result]] is empty, appends the reaction to [[Reactions]].
Otherwise, runs *EnqueueJob*(*"PromiseJobs"*, *PromiseReactionJob*, reaction, [[Result]])

Creates a *Caller* function that when invoked
and if the [[Promise]] field of the capability is not undefined
sets the [[Promise]] field of the capability to undefined
and if `onCalled` is not undefined, invokes `onCalled` with all the received arguments.

Returns that *Caller* function.
