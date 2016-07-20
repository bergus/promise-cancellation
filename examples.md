# Examples

## Creation of cancellable promises

* Delay

		function CancelPromise(executor, token) {
			return new Promise(resolve => {
				const preventCancelAndResolve = token.subscribeOrCall(reason => {
					if (typeof cancel === "function") cancel(reason);
				}, resolve);
				const cancel = executor(preventCancelAndResolve);
			}, token);
		}

		function delay(t, value, token) {
			return CancelPromise(resolve => {
				const timer = setTimeout(resolve, t, value);
				return () => {
					clearTimeout(timer);
				}
			}, token);
		}

		Promise.prototype.delay = function(t, token) {
			return this.then(value => delay(t, value, token), token);
		};

* Fetch

		function fetch(opts, token) {
			if (typeof opts == 'string') {
				opts = { method: 'GET', url: opts };
			}
			return new Promise(resolve => {
				const xhr = new XMLHttpRequest();
				const nocancelAndResolve = token
				  ? token.subscribeOrCall(r => {
				    	xhr.abort(r);
				    }, resolve)
				  : resolve;
				xhr.onload = () => nocancelAndResolve(xhr.response);
				xhr.onerror = e => nocancelAndResolve(Promise.reject(e));
				xhr.open(opts.method, opts.url, true);
			}, token);
		}

## Usage of new API

* Most simple timeout

		const { token, cancel } = CancelToken.source();
		delay(3000, 'over').then(cancel);

		const promise = delay(5000, 'result', token); // do anything that takes more than 3s and respects tokens
		promise.then(x => console.log(x), e => console.log(e)); //=> 'over' after 3 seconds

* Timeout for `then` callbacks

		const { token, cancel } = CancelToken.source();
		delay(3000, 'over').then(cancel);

		const p = delay(1000).then(x => delay(4000, 'result'), token);
		// the token is associated with p, so despite p being resolved with the delay we get
		p.then(x => console.log(x), e => console.log(e)); //=> 'over' after 3 seconds

		const q = delay(4000).then(x => {
			console.log('never executed');
			return delay(1000, 'result');
		}, token);
		// the token being revoked prevents the inner delay from ever being created, and we get
		q.then(x => console.log(x), e => console.log(e)); //=> 'over' after 3 seconds

* Racing submission and closing of a user dialogue

		function getUserchoice() {
			dialogue.show();
			const { token, cancel } = CancelToken.source();
			const promise = getClick("#radio", token).then(function(button) {
				return button.value;
			}, null, token);
			getClick("#close").then(cancel);
			return promise.finally(() => {
				dialog.hide();
			});
		}
		getUserChoice() // might get cancelled "by itself"

* `finally` for always concluding an action

		startSpinner();
		const token = new CancelToken(showStopbutton);
		const p = load('http://…', token)
		p.finally(() => {
			stopSpinner();
			hideStopbutton();
		}).then(showResult, showErrormessage, token);

* `trifurcate` to distinguish between a timeout and an error

		const token = new CancelToken(cancel => {
			setTimeout(cancel, 3000)
		});
		load('http://…', token).trifurcate(showResult, showErrormessage, showTimeoutmessage);

* Subscribe to a token and cancel it:

		const { token, cancel } = CancelToken.source();
		const p = token.subscribe(r => r + ' accepted');
		const q = token.subscribe(r => { throw new Error(…); });
		console.log(cancel('reason')); //=> [ Fulfilled { value: "reason accepted" }, Rejected { value: Error {…} } ]
		p.then(x => console.log(x)); //=> reason accepted

  The canceller (who invokes `cancel`) has the chance to handle errors that occured during cancellation
  as `cancel(…)` returns an array of promises for the subscription results (i.e. `[p, q]` in the example)

* Subscribe to a token without using `subscribe`, i.e. a callback 

		p = new Promise(()=>{}).then(null, null, token);
		p = new Promise(()=>{}, token);
		p = Promise.resolve({then(){}});

		p.trifurcate(null, null, reason => {
			…
		});
		p.catch(reason => {
			…
		});

* Trifurcate with ensuring that the first two callbacks are never called with the cancellation already requested:

		p.then(function onFulfilled(result) {
			if (token.requested) onCancelled(token.reason);
			else onFulfilledAndStillNotCancelled(result);
		}, onRejected(reason) {
			if (token.requested) onCancelled(token.reason);
			else onRejectedAndStillNotCancelled(result);
		})

* "Catching" cancellation in an async function:

		async function example(token) {
			await.cancel = token;
			try {
				await …
			} finally {
				if (token.requested) {
					…
				}
			}
		}

  (though it's [not exactly the same](trifurcation.md) as `trifurcate`)

## Lost+found

* Prevent concurrent execution by cancelling the previous calls

  compare https://github.com/domenic/cancelable-promise/issues/16, https://github.com/domenic/last/blob/master/lib/last.js, https://gist.github.com/domenic/46776bb71a2f885f79013130cd5301aa

		function last(operation, allToken) {
			var s = CancelToken.source()
			return function(...args) {
				if (allToken.requested) return allToken.getRejected() // do nothing any more
				s.cancel("next operation already started") // cancel previous token
				s = CancelToken.source() // reinitialise source for next call
				const t = allToken.concat(s.token) // wait for allToken or next call 
				return operation(...args, t) // pass token into operation
				  .then(null, null, t) // (and force cancellation if not respected)
			}
		}

* Cancelling sequential actions

		function drawPieces(token) {
			return fetch('flower.png', {}, token).then(draw)
			.then(() => fetch('chair.png')).then(draw);
		}

  does abort the fetching of the flower if it is still happening,
  but once that has finished it will always draw both pieces. In contrast,

		function drawPieces(token) {
			return fetch('flower.png', {}, token).then(draw)
			.then(() => fetch('chair.png', token), token).then(draw);
		}

  will abort the fetching of the flower or the fetching of the chair
  depending on which one is currently active, but once a piece has been fetched
  it will always be drawn.