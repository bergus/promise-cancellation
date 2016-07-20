## Problem

> See also https://github.com/zenparsing/es-cancel-token/issues/7 for discussion

Any cancellation proposal should provide a way to race a cancellation against a promise resolution,
and distinguish the outcome.
Put in different terms, we are looking for a function
```
trifurcate(promise, token, onFulfilled, onRejected, onCancelled)
```
that should

* call `onFulfilled` when the `promise` was fulfilled before a cancellation request
* call `onRejected` when the `promise` was rejected before a cancellation request
* call `onCancelled` when the cancellation was requested through the token before the promise was settled
* guarantee that at most one these three callbacks is executed.

One could implement this with flags
```
function trifurcate(promise, token, onFulfilled, onRejected, onCancelled) {
	let isCancelled = false;
	let isSettled = false;
	token.subscribe(reason => {
		isCancelled = true;
		if (!isSettled) onCancelled(reason);
	});
	promise.then(result => {
		isSettled = true;
		if (!isCancelled) onFulfilled(result);
	}, reason => {
		isSettled = true;
		if (!isCancelled) onRejected(reason);
	});
}
```
or cancellation tokens (to allow cleanup)
```
function trifurcate(promise, token, onFulfilled, onRejected, onCancelled) {
	let cancelled = CancelToken.source();
	let settled = CancelToken.source();
	token.subscribe(reason => {
		cancelled.cancel("already cancelled");
		onCancelled(reason);
	}, settled.token);
	promise.then(result => {
		settled.cancel("already fulfilled");
		onFulfilled(result);
	}, reason => {
		settled.cancel("already rejected");
		onRejected(reason);
	}, cancelled.token);
}
```
but both rely on `then` and `subscribe` using the same async queuing mechanism to detect the earlier event,
and that a cancellation request queues the token subscriptions before the promise rejection handlers.

Also they are cumbersome to write and easy to get wrong.
If `token.requested` is used instead of `isCancelled` or `token` instead of `cancelled.token`
they are prone to race conditions if the cancellation is requested from an (earlier) promise handler
because of the synchronous inspection.
In the better case, the wrong callback will be executed (`onCancelled` despite the promise having settled normally),
in the worse case, neither of the three callbacks is ever executed. An example:
```
const {token, cancel} = CancelToken.source();
function x() {
	…
	cancel(); // Oops.
	…
}
p.then(x);
…

let isSettled = false;
p.then(result => {
	isSettled = true;
	if (!token.requested) onFulfilled(result);
}, reason => {
	isSettled = true;
	if (!token.requested) onRejected(reason);
});
token.subscribe(reason => {
	if (!isSettled) onCancelled(reason);
});
```
Here `isSettled = true` is set, despite `token.requested` also being `true`.

## Solution

I therefore propose to provide such trifurcation functionality as a builtin.
The exact API is not entirely clear, a function with up to 5 parameters
(some of the callbacks might be optional) is not ideal.

The most common case is when the token is associated with the promise.
The cancellation presents itself [like a third state](third_state.md) for the promise then,
the API would be that of a promise method:
```
promise.trifurcate(onFulfilled, onRejected, onCancelled)
```

If the token that is supposed to be raced with the promise is not the one associated to it,
it is still possible to do
```
promise.then(null, null, token).trifurcate(onFulfilled, onRejected, onCancelled)
```
(although the exact timing of the callbacks is a bit different
in the presence of other callbacks on the `promise` here)