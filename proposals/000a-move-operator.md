# Move Operator

* Proposal: [SE-NNNN](NNNN-move-operator.md)
* Authors: [Michael Gottesman](https://github.com/gottesmm)
* Review Manager: TBD
* Status: **Awaiting implementation**

<!--
*During the review process, add the following fields as needed:*

* Implementation: [apple/swift#NNNNN](https://github.com/apple/swift/pull/NNNNN) or [apple/swift-evolution-staging#NNNNN](https://github.com/apple/swift-evolution-staging/pull/NNNNN)
* Decision Notes: [Rationale](https://forums.swift.org/), [Additional Commentary](https://forums.swift.org/)
* Bugs: [SR-NNNN](https://bugs.swift.org/browse/SR-NNNN), [SR-MMMM](https://bugs.swift.org/browse/SR-MMMM)
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/...commit-ID.../proposals/NNNN-filename.md)
* Previous Proposal: [SE-XXXX](XXXX-filename.md)
-->

## Introduction

In this document I proposing adding a new function called `move` to the swift
standard library that can be used to end the lifetime of a specific local let,
var, or function parameter. In order to enforce this, the compiler will emit a
flow sensitive diagnostic upon any uses that are after the move function. As an
example:

```
// Ends lifetime of x, y's lifetime begins.
let y = move(x) // [1]
// Ends lifetime of y, since _ is no-op, we perform an actual release here.
let _ = move(y) // [2]
useX(x) // error, x's lifetime was ended at [1]
useY(y) // error, y's lifetime was ended at [2]
```

Swift-evolution thread: [Discussion thread topic for that proposal](https://forums.swift.org/)

## Motivation: Allow binding lifetimes to be ended by use of the move function

In Swift today there is not a language guaranteed method to end the lifetime of
a specific variable binding. As an example, consider the following code:

```
func useX(_ x: Klass) -> () {}
func consumeX(_ x: __owned Klass) -> () {}

func f(_ x: Klass) -> () {
  useX(x)
  consumeX(x)
  useX(x)
  consumeX(x)
}
```

notice how even though we are taking `x` at +1 into `consumeX`, we are still
allowed to pass `x` again to `useX`, `consumeX`. This is because `x` is a
copyable type and thus the compiler will just insert an extra copy of `x`
implicitly before calling the first `consumeX` to lifetime extend x over
`consumeX`, in pseudo-code:

```
func useX(_ x: Klass) -> () {}
func consumeX(_ x: __owned Klass) -> () {}

func g(_ x: Klass) -> () {
  useX(x)
  let hiddenCopy = x
  consumeX(hiddenCopy)
  useX(x)
  consumeX(x)
}
```

for general programming use cases this is great since it enables the user to not
have to worry about the low level details and just to write code regardless of
the specific calling conventions of the code they are calling. But what if we
are in a context where we care about such low level details and want some way to
guarantee that a value will truly never be used again. In such a case, we are up
a creek and the compiler will not help us.

## Proposed solution: Move Function + "Use After Move" Diagnostic

That is where the `move` function comes into play. The `move` function is a new
generic stdlib function that when given a local let, local var, or parameter
argument provides a compiler guarantee to the programmer that the binding will
be unable to be used again locally. If such a use occurs, the compiler will emit
an error diagnostic. Lets look at this in practice using the following code:

```
func useX(_ x: Klass) -> () {}
func consumeX(_ x: __owned Klass) -> () {}

func h(_ x: Klass) -> () {
  useX(x)
  let _ = move(x)
  useX(x)
  consumeX(x)
}
```

In this case, we get the following output from the compiler:

```
test.swift:7:15: error: 'x' used after being moved
func h(_ x: Klass) -> () {
              ^
test.swift:9:11: note: move here
  let _ = move(x)
          ^
test.swift:10:3: note: use here
  useX(x)
  ^
test.swift:11:3: note: use here
  consumeX(x)
  ^
```

Notice how the compiler gives us all of the information that we need to resolve
this: it tells us where the move was and gives tells us the later uses that
cause the problem. We can then resolve this by introducing a new binding ‘other’
for those uses, e.x.:

```
func useX(_ x: Klass) -> () {}
func consumeX(_ x: __owned Klass) -> () {}

func k(_ x: Klass) -> () {
  useX(x)
  let other = x
  let _ = move(x)
  useX(other)
  consumeX(other)
}
```

which then successfully compiles. What is important to notice is that move ends
the lifetime of a specific local let or parameter binding. It is not tied to the
lifetime of the underlying class being references, just to the binding. That is
why we could just assign to other to get a value that we could successfully use
after the move of x. Of course, since other is a local let, we can also apply
move to that and would also get diagnostics, e.x.:

```
func useX(_ x: Klass) -> () {}
func consumeX(_ x: __owned Klass) -> () {}

func l(_ x: Klass) -> () {
  useX(x)
  let other = x
  let _ = move(x)
  useX(move(other))
  consumeX(other)
}
```

yielding as expected:

```
test.swift:9:7: error: 'other' used after being moved
  let other = x
      ^
test.swift:11:8: note: move here
  useX(move(other))
       ^
test.swift:12:3: note: use here
  consumeX(other)
  ^
```

In fact, since each variable emits separable diagnostics, if we combine our code
examples as follows,

```
func useX(_ x: Klass) -> () {}
func consumeX(_ x: __owned Klass) -> () {}

func m(_ x: Klass) -> () {
  useX(x)
  let other = x
  let _ = move(x)
  useX(move(other))
  consumeX(other)
  useX(x)
}
```

we get separable nice diagnostics:

```
test.swift:7:15: error: 'x' used after being moved
func m(_ x: Klass) -> () {
              ^
test.swift:10:11: note: move here
  let _ = move(x)
          ^
test.swift:13:3: note: use here
  useX(x)
  ^
test.swift:9:7: error: 'other' used after being moved
  let other = x
      ^
test.swift:11:8: note: move here
  useX(move(other))
       ^
test.swift:12:3: note: use here
  consumeX(other)
  ^
```

NOTE: In the future, we may add support for globals/ivars, but for now we have
restricted where you can use this to only the places where we have taught the
compiler how to emit diagnostics. If one attempts to use move on something we
don’t support, one will get an error diagnostic, e.x.:

```
var global = Klass()
func n() -> () {
  let _ = move(global)
}
```

yielding,

```
test.swift:9:11: error: move applied to value that the compiler does not know how to check. Please file a bug or an enhancement request!
  let _ = move(global)
          ^
```

## Detailed design

We define move as follows:

```
/// This function ends the lifetime of the passed in binding.
///
/// For more information on semantics please see: $INSERT_SWIFT_EVOLUTION_URL.
@_transparent
@alwaysEmitIntoClient
func move<T>(_ t: __owned T) -> T {
  Builtin.move(t)
}
```

Builtin.move is a hook in the compiler to force emission of special SIL "move"
instructions. These move instructions trigger in the SILOptimizer two special
passes that prove that the underlying binding does not have any uses that are
reachable from the move using a flow sensitive dataflow. Since it is flow
sensitive, one is able to end the lifetime of a value conditionally:

```
if (...) {
  let y = move(x)
  // I can't use x anymore here!
} else {
  // I can still use x here!
}
// But I can't use x here.
```

The value based analysis uses Ownership SSA to determine if values are used
after the move. The address based analysis is an SSA based analysis that
determines if any uses of an address are reachable from a move. All of these are
already in tree behind the `-enable-experimental-move-only` frontend flag.

## Source compatibility

This is additive and does not effect source compatibility since the stdlib has
never vended a "move" function before.

## Effect on ABI stability

None, move will always be a transparent always emit into client function.

## Effect on API resilience

None, this is additive.

## Alternatives considered

The only other place that we could implement this functionality would be in the
typechecker, but that would prevent us from using flow sensitive diagnostics. We
could also introduce a separate `drop` function like languages like Rust does
that doesn't have a result like `move` does. We decided not to go with this
since in Swift the idiomatic way to throw away a value is to assign to `_`
implying that the idiomatic way to write `drop` would be:

```
let _ = move(x)
```

suggesting adding an additional API would not be idiomatic.

## Acknowledgments

Thanks to Andrew Trick as always for the great conversations that lead to this
proposal!
