# Promise Cancellation

Developing a spec level proposal for cancellation in promise-based code.

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

## Proposed Solution

…