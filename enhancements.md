# Enhancements

While the [API](API.md) is a relatively modest extension of the Promise semantics
that can be polyfilled in plain ES5-7, it does already lay the foundation
for more advanced use cases (which are not specced by this proposal).
They are expected to be implemented in userland libraries
or with the help of transpilers before maybe becoming standardised one day.

## Tasks

Tasks greatly simplify working with cancellation, as they eliminate the need
to pass cancellation tokens around everywhere. Unlike promises, every task is
implicitly cancellable by anyone holding a reference to it, as long as it has
no uncancelled descendants that depend on it.
This is often referred to as reference counting.

It is incompatible with the requirement that a promise may only be cancelled
by its creator and that by default any consumers should not be able to request cancellation.
It has however proven so useful for developers that almost all [existing
implementations](prior_work.md#existing-implementations) employ these semantics.

By passing tokens to basically every `then` call, especially even when assimilating
thenables, this proposal leads to interoperability between such `Task` implementations.
If the cancellation of a <s>promise</s>task depends on its descendants, it is possible
to propagate cancellation requests upwards on a <s>promise</s>task chain.

### Demo Implementation

This proposal also enables us to implement `Task`s as a `Promise` subclass with
pretty simple code in less than 50 lines:
```javascript
import { Promise } from '..'

export default class Task extends Promise {
	constructor (executor) {
		const pool = new TokenPool()
		super(executor, pool.token);
		this.tokenPool = pool;
	}
	getToken () {
		return this.tokenPool.token;
	}
	then (onFulfilled, onRejected, ...opts) {
		return this.tokenPool.addChild(super.then(onFulfilled, onRejected, ...opts));
	}
	cancel (reason) {
		this.getToken().attemptCancel(reason);
	}
}

class PoolCancelToken extends CancelToken {
	constructor (pool) {
		var cancel;
		super(c => { cancel = c })
		this.attemptCancel = reason => {
			if (super.requested) return
			if (this.pool.check())
				cancel(reason);
		};
		this.pool = pool;
	}
	get requested () {
		return super.requested || this.pool.check()
	}
}

class TokenPool {
	constructor () {
		this.dependencies = [];
		this.token = new PoolCancelToken(this);
	}
	addChild (task) {
		this.dependencies.push(task.getToken());
		task.trifurcate(null, null, this.token.attemptCancel);
		return task;
	}
	check () {
		return this.dependencies.every(t => t.requested);
	}
}
```
[`race`](API.md#promise-race) and [`all`](API.md#promise-all) should work
out of the box when inherited by `Task`. It is of course possible to overwrite
them in the `Task` class, but that is cumbersome, so the builtin `Promise`
methods do deal with tokens already.

### Example usage

* Cancelling parts of a branched task chain:

		const ajax = taskFetch(…);
		const some = ajax.then(doSomething);
		const json = ajax.then(JSON.parse);
		// later:
		json.cancel(); // `ajax` can't be cancelled (because `doSomething` is still interested in it),
		               // but the `JSON.parse` won't need to be executed

* A race against a timeout:

		const p = taskDelay(5000);
		const q = p.then(() => {
			console.log("p fulfilled"); // never logged
			return "q"
		});
		const r = taskDelay(2000, "r");
		Task.race([r, q]) // the slower input is implicitly cancelled
		.then(x => console.log(x + " was faster")); // logs "r was faster"

## `async`/`await`

[`async function`s](https://tc39.github.io/ecmascript-asyncawait/) will be expected
to support cancellation as well. They will be accepting cancellation tokens
in their parameters, and the promises that they return when called are supposed
to get cancelled when the cancellation is requested.

While control flow can trivially branch on the `token.requested` property,
there are a few challenges:

* How to associate a token with the returned promise (that is nowhere accessible
  in the function body)?
* How to immediately interrupt the `await`ing of a promise when cancellation
  is requested, and complete the function with only triggering `finally` clauses?
  Even when the awaited promise is not cooperative (doesn't get cancelled)?

For clarity take this snippet:
```javascript
async function example(token) {
	… M_A_G_I_C ( token ) … // uh, yeah. See below.
	try {
		const p = longRunningOperation(); // possibly but not necessarily pass the token
		await p;
		console.log("A");
	} catche(e) {
		console.log("B");
	} finally {
		console.log("C");
	}
}
example(new CancelToken(cancel => setTimeout(cancel, 1000)));
```
With `longRunningOperation` taking multiple seconds, we will expect `C` to be logged
exactly after 1000 milliseconds, and the `A`/`B` log statements to be never executed.

Basically the `await` is expected to do
```
p.then(onFulfilled, onRejected, token);
token.subscribe(onCancelled);
```
(or more precisely, <code>[trifurcate](trifurcation.md)(p, token, onFulfilled, onRejected, onCancelled)</code>)  
where `onFulfilled` is [*AsyncFunction Awaited Fulfilled*](https://tc39.github.io/ecmascript-asyncawait/#abstract-ops-awaited-fulfilled),
`onRejected` is [*AsyncFunction Awaited Rejected*](https://tc39.github.io/ecmascript-asyncawait/#abstract-ops-awaited-rejected),
and `onCancelled` would be a new *AsyncFunction Awaited Cancelled*
that does resume the suspended evaluation with with a *Completion* { [[type]]: *return* }
(like `.return()` on a generator would do).

### Syntax proposal A

The `await` keyword is supplemented by a new meta property `await.unlessCancel`
that has to be followed by a *PrimaryExpression* and otherwise works like `await`.

Examples:
```javascript
await promise
await.unlessCancel token promise
await.unlessCancel (new CanelToken(…)) promiseCreatingOperation()
```

Every time a promise is awaited, the cancellation token that should be monitored
needs to be passed explicitly.

While being explicit is usually a good thing, this approach is too verbose
and grammatically just horrible (two expressions without a real separator).
This could be better with something like `await promise orReturnOnCancel token`
but I'm not confident that the grammar can support this either.

Also it's not really clear what should happen to the promise returned by the
async function call, or how a token would be associated to it.

Suggestions for improvement are welcome.

### Syntax proposal B

A new meta property `await.associateCancel` behaves like a unary operator
that will associate a token with the promise returned by the async function.

This meta property is only available prior to the first usage of `await`,
and a syntax error otherwise. This would allow to call the promise constructor
only after [*AsyncFunctionStart*](https://tc39.github.io/ecmascript-asyncawait/#abstract-ops-async-function-start)
had run, and pass the acquired token into the *NewPromiseCapability* call afterwards.

Example:
```javascript
async function(token) {
	await.associateCancel token;
	…
	await promise
	…
}
```

On every `await`, the associated token would be monitored for cancellation requests.

Since it would be expected that `await.associateCancel` is usually used in the
first line of the function with one of the arguments, an alternative would be
to introduce sugar for that use case:
```javascript
async function(await.cancelToken) {
	…
	await promise
	…
}
```

This is a fine approach, and works well especially for large async functions
with many `await`s that should be cancelled through the same token every time.
It might however not be flexible enough.

### Syntax proposal C

A new meta property `await.cancelOn` can be used as an operator of arity zero or one.

If an async function body *Contains* the meta property, a call will not only
create a `Promise` instance but also a `CancelToken` that is associated to it.
Every function evaluation will have a *current token reference* that can either point
to `null` or a cancellation token. If it is not null, it will be constantly monitored,
and when a cancellation is requested then the result promise will get cancelled
through its associated token.

When the `await.cancelOn` operator is used without an operand, it will yield the associated token.  
When the `await.cancelOn` operator has an operand, it will set the reference to the new value.
If the currently referenced token was already cancelled, it will throw;
if the newly referenced token is already cancelled it will behave like `return`.

Example:
```javascript
async function(token) {
	await.cancelOn token;
	…
	{
		await.cancelOn null;
		… // in this section, awaits are not getting interrupted
		await.cancelOn token;
	}
	…
}
```

This approach is quite powerful, but probably too complicated to specify.
A token reference might better be implemented in userland if it is necessary at all.

### async Tasks

To support `Promise` subclassing with `async function`s, one could allow
a *PrimaryExpression* that evaluates to the used promise constructor after
the `async` keyword:
```javascript
async Task function example() {
	…
}
const p = example();
p.cancel();
```
If an approach similar to the [syntax B](#syntax-proposal-b) is chosen,
the `await.associateCancel` could be omitted in such a function since `Task`s
create their associated token themselves. It still would be used when `await`ing
thenables though, so this would make it possible to write aynchronous functions
(using `async`/`await`) that are cancellable by default, without ever seeing
a single token in the code.
