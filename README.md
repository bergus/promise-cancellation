# Promise Cancellation

Developing a spec level proposal for cancellation in promise-based code.

For discussion of open questions see the Github issues.

## Motivation

Asynchronous operations should offer more control about their execution.
This is already possible through sharing state and choosing different paths based on it,
but there has not yet emerged a standard.
To actively abort an operation, some way of passing messages to the operation is necessary,
which further raises the implementation complexity.
On top of that, many asynchronous primitives do not yet offer a means to terminate or interrupt them,
though it is desired by the community for many of them.

Examples include

* timers
* network requests
* file system operations
* animations
* long-running background-thread computations (cryptography, graphics rendering etc.)
* waiting for user input

The majority of these use cases can (or already does) utilise promises to represent their results
and to enable monadic chaining of multiple actions.

The most simple way of impacting an asynchronous operation is to terminate it.
The need for such typically arises from either a competition between operations to compute a result,
a timeout, or from interactivity in the application where a user wants to stop or drop the operation.

The topic is threefold:

* How to initiate the cancellation?
* How to affect further scheduled work in the promise chain?
* How to represent the cancellation result in the promises?

There has been [a lot of prior work](prior_work.md) with many different approaches
and lots of discussion, but despite manifold opinions the desire of the community
for such a feature is clear;
it even is a blocking issue for some new promise-based (web) APIs such as `fetch`.

## Challenges

Any solution should offer

* [Promises/A+](http://promisesaplus.com/) compatibility
* integration with `async`/`await`
* non-verbose syntax with low overhead
* powerful composition
* separation of cancellation capability from result promise
* interoperability with potential (userland) `Task` implementations
* easy to polyfill

## Proposed Solution

### How to initiate the cancellation? ###

We use cancellation tokens, objects that represent the cancellation state and allow for subscription and synchronous queries.
They are implemented in the `CancelToken` class, can be instantiated and passed around.
Only their creator (issuer) can request the cancellation, they do not have a public `.cancel()` method.

They clearly separate the capability to cancel an action from the promise itself,
and can be composed through various utility methods, e.g. to allow cancellation requests from multiple sources.

By testing whether cancellation has already been requested, asynchronous operations can choose
to do whatever they think is reasonable with intermediate results.

With the ability to register a callback on a cancellation token it is possible to actively terminate ongoing work.
Ideally all native asynchronous primitives do support this through a token parameter.

### How to affect further scheduled work in the promise chain? ###

The `then` and `catch` promise methods are given an optional `cancelToken` parameter
which will prevent the callbacks from executed when the cancellation was requested.
This essentially is a way to unsubscribe promise callbacks.

The approach makes clear that a cancellation request leads to **ignoring** the results of the promise
regardless what happens to the promise. This is an often-desired flavour of cancellation.

A function is free to ignore cancellation requests of the token passed to it, and might still resolve the returned promise normally.
This is especially the case when the function does not support any means of cancellation at all.
It is however guaranteed that the callback will never run after the cancellation has been requested:
```javascript
anyPromise.then(result => { assert(!token.requested) }, error => assert(!token.requested), token);
```

### How to represent the cancellation result in the promises? ###

A promise can be associated to a cancellation token during its creation.
This happens via an optional second parameter of the `Promise` constructor,
and for the promises returned by `then`/`catch` uses the token passed along the callbacks.
Promises are immutable, their fate can only be set by their creator.

When a promise is settled "normally" through the resolving functions, the associated cancellation token is removed.
When the cancellation is requested, all promises associated with the respective token are immediately rejected.

Cancellation is usually not an error, but not a fulfillment either.
When cancelling, the result that was promised will not become available, which is naturally a reason for rejection.
We do not want cancelled promises to be forever pending,
to guarantee propagation and eventual settling when being assimilated using the Promises/A+ mechanism.
The builtin rejection tracking can ignore promises that were rejected through cancellation.

All promise handlers that were considering the possibility of cancellation will not be called,
since they would have registered the same token that is now cancelled together with their reaction, which is not run.
In contrast, reactions that did not expect the cancellation will have their `onRejected` handlers called
with the cancellation reason stating that there is no result because of cancellation.
This is especially interesting for handlers that are subscribed to cached promises after the cancellation.

### Documents

For details, see also

* [technical description](API.md)
* [the same approach in other words](third_state.md)
* [usage examples](examples.md)
* [integration with Tasks and `async`/`await`](enhancements.md)

If you are missing any insights, find sections hard to understand, or are looking for something not covered anywhere,
feel free to [open an issue](//github.com/bergus/promise-cancellation/issues/new)!