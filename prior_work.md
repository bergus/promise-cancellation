# Prior work

There is nothing new under the sun.

> All wise thoughts have been thought before,
> one only has to try to think them again  
> &nbsp;— [Goethe, *Wilhelm Meister's Wanderjahre*](https://en.wikiquote.org/wiki/Johann_Wolfgang_von_Goethe#Wilhelm_Meister.27s_Wanderjahre_.28Journeyman_Years.29_.281821.E2.80.931829.29)

Another overview can be found at [boopathi/awesome-promise-cancellation](//github.com/boopathi/awesome-promise-cancellation).

## Existing implementations
* http://bluebirdjs.com/docs/api/cancellation.html
* http://dojotoolkit.org/reference-guide/1.10/dojo/promise/Promise.html
* http://google.github.io/closure-library/api/goog.Promise.html#cancel
* https://msdn.microsoft.com/de-de/library/windows/apps/br211667.aspx
* https://github.com/bergus/creed/blob/cancellation/cancellation.md
* http://taskjs.org/
* https://docs.angularjs.org/api/ng/service/$http timeout parameter
* https://github.com/rbuckton/asyncjs/wiki/cancellation-module
* https://github.com/bergus/F-promise
* https://github.com/guzzle/promises#cancellation
* …

## Proposals

and their <s>deficiencies</s> differences to this one:

* https://github.com/promises-aplus/cancellation-spec/issues/11, https://gist.github.com/bergus/c160b1c0c9e755108d1b
  - did "Tasks" with (token) reference counting on `then` invocations
  - implicitly created tokens when none were passed
  - `.cancel()` method on promises, effective on chain ends
* [https://github.com/domenic/cancelable-promise](https://github.com/domenic/cancelable-promise/tree/2f601acb3d4d438be0ebc9e4e54e6febde9d7a3a) (old)
  - needed a new abrupt completion value in the language
  - impossible to polyfill and rather hard to transpile
  - did propagate cancellation as a result downwards
  - burden of explicitly implementing cancellation (and returning `Promise.cancel`) on the callee
  - burden of explicitly checking for cancellation (if not supported by the called) on the caller
  - same for `async` functions (as both callee and caller), though [under discussion](https://github.com/domenic/cancelable-promise/issues/23)
  - A+ assimilation just stays forever pending
  - compatibility issues with third `then` callback argument
  - `.cancelIfRequested` on tokens
  - no composability of tokens
* https://github.com/domenic/cancelable-promise (revised)
  - special `Cancel` exceptions that are treated differently by a new `catch`-like clause
  - impossible to polyfill, transpilation necessary
  - did propagate cancellation as a result downwards
  - burden of explicitly implementing cancellation (and throwing a `Cancel()`) on the callee
  - burden of explicitly checking for cancellation (if not supported by the called) on the caller
  - `.throwIfRequested` on tokens
  - non-generic solution of the [garbage collection problem](https://github.com/tc39/proposal-cancelable-promises/issues/52)
  - asynchronous propagation of cancellation signal to composed tokens
* https://github.com/zenparsing/es-cancel-token
  - subscription via a `.promise` that never rejected
  - `.throwIfRequested`
* https://github.com/promises-aplus/cancellation-spec/issues/2, https://github.com/promises-aplus/cancellation-spec/issues/5, https://gist.github.com/novemberborn/4407774
* https://github.com/promises-aplus/cancellation-spec/issues/7
* https://github.com/promises-aplus/cancellation-spec/issues/8, https://github.com/ForbesLindesay/cancellation
* https://github.com/SaschaNaz/cancelable

## Discussions
* https://github.com/promises-aplus/cancellation-spec
  - [Background](https://github.com/promises-aplus/cancellation-spec/issues/1)
  - [Communication channel](https://github.com/promises-aplus/cancellation-spec/issues/3)
  - [Canceled vs. Cancelled](https://github.com/promises-aplus/cancellation-spec/issues/4)
  - [Cancel as a message](https://github.com/promises-aplus/cancellation-spec/issues/15)
* https://github.com/stefanpenner/random/tree/master/cancellation
* https://github.com/kriskowal/gtor/blob/master/cancelation.md
* https://github.com/tc39/tc39-notes/blob/master/es7/2016-05/may-25.md#cancelable-promises-dd
* https://github.com/tc39/tc39-notes/blob/master/es7/2016-07/jul-28.md#10ivb-cancelable-promises-update
* https://github.com/petkaantonov/bluebird/issues/415
* https://github.com/whatwg/fetch/issues/27: Cancelling fetch
  - https://github.com/whatwg/fetch/issues/20: timeouts
  - https://github.com/slightlyoff/ServiceWorker/issues/592
  - https://github.com/slightlyoff/ServiceWorker/issues/625
  - https://github.com/whatwg/streams/issues/297: Aborting streams
* http://www.bennadel.com/blog/2731-canceling-a-promise-in-angularjs.htm

### On ES-Discuss
* https://esdiscuss.org/topic/cancellation-architectural-observations
* https://esdiscuss.org/topic/es7-promise-cancellation-was-re-promise-returning-delay-function
  - https://gist.github.com/rbuckton/256c4e929f4a097e2c16
* https://esdiscuss.org/topic/promises-as-cancelation-tokens
* https://esdiscuss.org/topic/promise-returning-delay-function
* https://esdiscuss.org/topic/cancelable-promises-proposal
* https://esdiscuss.org/topic/future-cancellation
  - [Cancellation using Future](https://gist.github.com/rbuckton/5486149)
  - [Cancellation using Future.cancelable](https://gist.github.com/rbuckton/5484591)
  - [Cancellation using Future#cancel](https://gist.github.com/rbuckton/5484478)
* https://esdiscuss.org/topic/cancelable-promises
