# Defining consuming and nonConsuming function attributes and consuming method attribute

* Proposal: [SE-NNNN](NNNN-consuming-nonconsuming.md)
* Authors: [Michael Gottesman](https://github.com/gottesmm) [Andrew Trick](https://github.com/atrick)
* Review Manager: TBD
* Status: **Awaiting implementation**
* Pitch v1: [https://github.com/gottesmm/swift-evolution/blob/consuming-nonconsuming-pitch-v1/proposals/000b-consuming-nonconsuming.md](https://github.com/gottesmm/swift-evolution/blob/consuming-nonconsuming-pitch-v1/proposals/000b-consuming-nonconsuming.md)

<!--
*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN) or [apple/swift-evolution-staging#NNNNN](https://github.com/apple/swift-evolution-staging/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)
-->

## Introduction

Currently in Swift there isn't any way to override the default ownership passing
semantics that the language uses for function arguments. Sometimes when writing
certain APIs one needs to be able to control this convention. In this proposal,
we formalize the semantics of the consuming and nonConsuming function attributes
to enable this to be expressed in the language.

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

## Motivation

In Swift, all non-trivial function arguments possess a function argument
convention that defines whether or not management of the lifetime of the
argument is managed by the caller or the callee. These two convention types are:

* NonConsuming. A non consuming function argument is an argument where the
  callee does not own the function argument and the caller provides a guarantee
  to the callee that the lifetime of the argument lasts beyond the end of the
  callee. Often times people will refer to this as "passing an argument at
  +0". We use +0 since as a result of the function call, the argument will not
  be copied.

* Consuming. A consumed function argument is an argument where the caller is
  passing to the callee ownership. As a result of this, the callee is
  responsible for ensuring that the lifetime of the argument is managed and may
  have to destroy the argument. 

As mentioned above, one can not customize these conventions currently and must
use the default conventions. These conventions are that the following arguments
are passed as consuming:

1. All arguments passed to initializers.
2. All arguments except for self passed to a setter.

and that all other arguments of functions are passed as nonConsuming.

Sometimes an API designer needs to be able to customize these since the default
does not fit their specific situation. Some examples of this are:

1. Passing a non-consuming parameter to an initializer or setter if one is going
   to actually internally to the function consume a derived value from that
   parameter. Example: String initializer for Substring.

2. Passing a consuming parameter to a normal function or method that isn't a
   setter. Example: bridging APIs, mutating functions like insertNew
   Dictionary.insertNew and Array.append.

These are especially important in the face of us wanting to support move only
values in the future in standard library APIs.

## Proposed solution

The compiler already internally supports these semantics in the guise of the
underscored keywords: `__owned` and `__shared`. We propose that we rename these
keywords to `consuming` and `nonConsuming` and remove the underscores.

## Detailed design

Since `__owned` and `__shared` have existed for many years already and been in
used in the stdlib for many years as well (thus have a solid implementation
already), the only real work here is to:

1. Refactor the compiler to accept the new names and the old names.
2. Change the internals of the compiler to refer to consuming and non consuming
   instead of owned/shared. The frontend in either case of using the old/new
   names would use the refactored code path. None of this will be user visible.

## Source compatibility

In order to ensure source compatibility, as mentioned above, we will still
accept `__owned` and `__shared` initially and will begin a deprecation cycle
starting in the swift version after this proposal is accepted. In that release,
we would begin to emit deprecation warnings with a fixit to tell people to
perform the conversion. Then in the N+2 swift version, we will then stop
accepting `__owned` and `__shared`. The reason why we are doing this is that
unfortunately certain projects outside of the stdlib have started to use
`__owned` and `__shared` without them being accepted into the language
itself. We could be difficult and just break them, but that is relatively
unfriendly to our users thus the authors would prefer to avoid doing that if
possible.

## Effect on ABI stability

This should not effect the ABI of any existing language features since all uses
that already use `__owned`, `__shared` will be unchanged semanitcally and any
other potential uses of the new keywords directly are additive.

## Effect on API resilience

If a user marks a parameter as consuming or nonconsuming and the parameter would
by default have the opposite convention, an ABI break would occur. Thus for
instance, if one marked an initializer argument with consuming or a normal
function parameter with nonconsuming, one would not break ABI. Example:

```
struct SortedArray {
    // consuming matches default convention => no ABI difference.
    init(_ array: consuming [SomeClass]) { ... }

    // ABI is changed from default to be nonconsuming => ABI difference
    init(_ array: nonconsuming [SomeClass]) { ... }

    // nonconsuming matches the default convention => no ABI difference.
    func useSomeClass(_ elt: nonconsuming SomeClass) { ... }

    // consuming does not match the default convention => ABI difference.
    func append(_ elt: consuming SomeClass) { ... }
}
```

## Alternatives considered

We could reuse `owned` and `shared` and just remove the underscore. This was
viewed as confusing since `shared` is used in other contexts since `shared` can
mean a "shared borrow" to steal Rust terminology a much stronger condition than
`nonconsuming` is. Once one realizes that shared's semantics should be called
`nonconsuming`, it is natural to also rename `owned` to `consuming` as well to
simplify what the author must remember.

## Acknowledgments

Thanks to Robert Widmann for the original underscored implementation of
`__owned` and `__shared`.
