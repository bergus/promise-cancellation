# Third state vs. associated token

There are two means of specifying how a cancelled promise looks like.
They are equivalent in behaviour, but differ in the explanation.

The choice of one will influence the understanding of what cancellation means and how it works.
It might make this proposal less accessible for people coming from the opposite school of thought,
and possibly complicate implementations if their internal representation differs from the spec text.

## Third state

A third settled state is introduced for promises: They can be either *fulfilled*, *rejected*, or *cancelled*.
A transition to the cancelled state will trigger `onRejected` callbacks.

- the name of the state needs an official spelling
- triggering rejection callbacks from the cancelled state might be unintuitive
- going from resolved to cancelled without involvement of the resolution might be unintuitive
- an associated canceltoken is probably needed anyway
- the new state might be expected to propagate in the usual way on resolution
* raises the expectation that there is an `cancel()` resolving function in the `Promise` constructor
+ inspection of the state is trivial and obvious
+ clear working of the `trifurcate` method

## Associated token

There are only two settled states: *fulfilled* and *rejected*.
If a promise is fulfilled or rejected through the resolving functions, its associated cancellation token is removed.
If a promise is rejected trough its associated cancellation token, the token remains on the promise, signifying the cancellation.

+ the canceltoken is needed anyway for assimilation/adoption via `.then`
- garbage collection of the canceltoken (not sure how large of an issue)
- complicated means to denote the state, e.g. for unhandled-rejection-tracking or `trifurcate`