# Prior work

There is nothing new under the sun.

> All wise thoughts have been thought before,
> one only has to try to think them again  
> &nbsp;— [Goethe, *Wilhelm Meister's Wanderjahre*](https://en.wikiquote.org/wiki/Johann_Wolfgang_von_Goethe#Wilhelm_Meister.27s_Wanderjahre_.28Journeyman_Years.29_.281821.E2.80.931829.29)

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
* https://github.com/promises-aplus/cancellation-spec/issues/11, https://gist.github.com/bergus/c160b1c0c9e755108d1b
* https://github.com/domenic/cancelable-promise
* https://github.com/zenparsing/es-cancel-token
* https://github.com/promises-aplus/cancellation-spec/issues/2, https://github.com/promises-aplus/cancellation-spec/issues/5, https://gist.github.com/novemberborn/4407774
* https://github.com/promises-aplus/cancellation-spec/issues/7
* https://github.com/promises-aplus/cancellation-spec/issues/8
* https://github.com/SaschaNaz/cancelable

## Discussions
* https://github.com/promises-aplus/cancellation-spec
  - [Background](https://github.com/promises-aplus/cancellation-spec/issues/1)
  - [Communication channel](https://github.com/promises-aplus/cancellation-spec/issues/3)
  - [Canceled vs. Cancelled](https://github.com/promises-aplus/cancellation-spec/issues/4)
  - [Cancel as a message](https://github.com/promises-aplus/cancellation-spec/issues/15)
* https://github.com/kriskowal/gtor/blob/master/cancelation.md
* https://github.com/rwaldron/tc39-notes/blob/master/es7/2016-05/may-25.md#cancelable-promises-dd
* https://github.com/petkaantonov/bluebird/issues/415
* https://github.com/whatwg/fetch/issues/27: Cancelling fetch
  - https://github.com/whatwg/fetch/issues/20: timeouts
  - https://github.com/slightlyoff/ServiceWorker/issues/592
  - https://github.com/slightlyoff/ServiceWorker/issues/625
  - https://github.com/whatwg/streams/issues/297: Aborting streams
* http://www.bennadel.com/blog/2731-canceling-a-promise-in-angularjs.htm

### On ES-Discuss
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