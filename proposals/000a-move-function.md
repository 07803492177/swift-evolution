# Move Function + "Use After Move" Diagnostic

* Proposal: [SE-NNNN](NNNN-move-function.md)
* Authors: [Michael Gottesman](https://github.com/gottesmm), [Andrew Trick](https://github.com/atrick)
* Review Manager: TBD
* Status: Implemented on main as stdlib SPI (_move instead of move)
* Pitch v1: https://github.com/gottesmm/swift-evolution/blob/move-function-pitch-v1/proposals/000a-move-function.md

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
local var, or consuming parameter. In order to enforce this, the compiler will
emit a flow sensitive diagnostic upon any uses that are after the move
function. As an example:

```
// Ends lifetime of x, y's lifetime begins.
let y = move(x) // [1]
// Ends lifetime of y, since _ is no-op, we perform an actual release here.
let _ = move(y) // [2]
useX(x) // error, x's lifetime was ended at [1]
useY(y) // error, y's lifetime was ended at [2]
```

this allows the user to influence uniqueness and where the compiler inserts
retain releases manually that is future-proof against new changes due to the
diagnostic. Consider the following array/uniqueness example:

```
// Array/Uniqueness Example

// Get an array
func test() {
  var x: [Int] = getArray()
  
  // x is appended to. After this point, we know that x is unique. We want to
  // preserve that property.
  x.append(5)
  
  // We create a new variable y so we can write an algorithm where we may
  // change the value of x (causing a copy), but we might not.
  var y = x
  // ... long algorithm using y ...
  let _ = move(y) // end the lifetime of y. It is illegal to use y later and
                  // people can not add a new reference by mistake. This ensures that
                  // a release occurs at this point in the code as well since move
                  // consumes its parameter.

  // x is again unique so we know we can append without copying.
  x.append(7)
}
```

in the example above without the `move`, `y`'s lifetime would go to end of scope
and there is a possibility that we or some later programmer may copy x again
later in the function. But luckily since we also have the diagnostic guarantees
that `y`'s lifetime will end at the move meaning after the move, `x` can be
known to always be unique again. Importantly if someone later modifies the code
and tries to use y later, the diagnostic will always be emitted so one can
program with confidence against subsequent source code changes.

As a final dimension to this, once Swift has move only values (values that are
not copyable), `move` as we have defined it above automatically generalizes to a
utility that runs end of the lifetime code associated with the value (e.x.:
destructors for unique classes). This is a basic facility if one wants to be
able to express things like having a move only type that represents a file
descriptor and one wants to guarantee that the descriptor closes. That being
said, the authors think this facility is useful enough to bring forward and do
early in preparation for the addition of move semantics to the language. Since
this aspect is not directly related to this proposal, the author will not
mention it further.

Swift-evolution pitch thread: [https://forums.swift.org/t/pitch-move-function-use-after-move-diagnostic](https://forums.swift.org/t/pitch-move-function-use-after-move-diagnostic)

## Motivation: Allow binding lifetimes to be ended by use of the move function

In Swift today there is not a language guaranteed method to end the lifetime of
a specific variable binding preventing users from easily controlling properties
like uniqueness and ARC traffic. As an example, consider the following code:

```
func useX(_ x: SomeClassType) -> () {}
// __owned causes x to be released before consumeX returns rather than in f.
func consumeX(_ x: __owned SomeClassType) -> () {}

func f(_ x: SomeClassType) -> () {
  useX(x)
  consumeX(x)
  useX(x)
  consumeX(x)
}
```

notice how even though we are going to release `x` within `consumeX` (due to the
`__owned` attribute on its parameter), we are still allowed to pass `x` again to
`useX`, `consumeX`. This is because `x` is a copyable type and thus the compiler
will just insert an extra copy of `x` implicitly before calling the first
`consumeX` to lifetime extend x over `consumeX`, in pseudo-code:

```
func useX(_ x: SomeClassType) -> () {}
func consumeX(_ x: __owned SomeClassType) -> () {}

func f(_ x: SomeClassType) -> () {
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
guarantee that a value will truly never be used again (e.x.: restoring
uniqueness to an Array or controlling where the compiler inserts ARC). In such a
case, we are up a creek and the compiler will not help us.

NOTE: One could write code using scopes but it is bug prone and one has no
guarantee that the property that all future programmers will respect the
invariant that one is attempting to maintain.

## Proposed solution: Move Function + "Use After Move" Diagnostic

That is where the `move` function comes into play. The `move` function is a new
generic stdlib function that when given a local let, local var, or parameter
argument provides a compiler guarantee to the programmer that the binding will
be unable to be used again locally. If such a use occurs, the compiler will emit
an error diagnostic. Lets look at this in practice using the following code:

```
func useX(_ x: SomeClassType) -> () {}
func consumeX(_ x: __owned SomeClassType) -> () {}

func f() -> () {
  let x = ...     // Creation of x binding
  useX(x)
  let _ = move(x) // Lifetime of x is ended here.
  useX(x)         // !! But we use it here afterwards. Error?!
}
```

In this case, we get the following output from the compiler as expected:

```
test.swift:7:15: error: 'x' used after being moved
  let x = ...
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
func useX(_ x: SomeClassType) -> () {}
func consumeX(_ x: __owned SomeClassType) -> () {}

func f() -> () {
  let x = ...
  useX(x)
  let other = x   // other is a new binding used to extend the lifetime of x
  let _ = move(x) // x's lifetime ends
  useX(other)     // other is used here... no problem.
  consumeX(other) // other is used here... no problem.
}
```

which then successfully compiles. What is important to notice is that move ends
the lifetime of a specific local let binding. It is not tied to the lifetime of
the underlying class being references, just to the binding. That is why we could
just assign to other to get a value that we could successfully use after the
move of x. Of course, since other is a local let, we can also apply move to that
and would also get diagnostics, e.x.:

```
func useX(_ x: SomeClassType) -> () {}
func consumeX(_ x: __owned SomeClassType) -> () {}

func f() -> () {
  let x = ...
  useX(x)
  let other = x
  let _ = move(x)
  useX(move(other)) // other's lifetime ended here.
  consumeX(other)   // !! Error! other's lifetime ended on previous line!
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
func useX(_ x: SomeClassType) -> () {}
func consumeX(_ x: __owned SomeClassType) -> () {}

func f() -> () {
  let x = ...
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
  let x = ...
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

If one applies move to a var, one gets the same semantics as let except that one
can begin using the var again after one re-assigns to the var, e.x.:

```
func f() {
  var x = getValue()
  let _ = move(x)
  // Can't use x here.
  x = getValue()
  // But I can use x here, since x now has a new value
  // within it.
}
```

This follows from move being applied to the binding (`x`), not the value in the
binding (the value returned from `getValue()`).

NOTE: In the future, we may add support for globals/ivars, but for now we have
restricted where you can use this to only the places where we have taught the
compiler how to emit diagnostics. If one attempts to use move on something we
don’t support, one will get an error diagnostic, e.x.:

```
var global = SomeClassType()
func f() {
  let _ = move(global)
}
```

yielding,

```
test.swift:9:11: error: move applied to value that the compiler does not support checking
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
diagnostic passes that prove that the underlying binding does not have any uses
that are reachable from the move using a flow sensitive dataflow. Since it is
flow sensitive, one is able to end the lifetime of a value conditionally:

```
if (...) {
  let y = move(x)
  // I can't use x anymore here!
} else {
  // I can still use x here!
}
// But I can't use x here.
```

This works because the diagnostic passes are able to take advantage of
control-flow information already tracked by the optimizer to identify all places
where a variable use could possible following passing the variable to as an
argument to `move()`.

In practice, the way to think about this dataflow is to think about paths
through the program. Consider our previous example with some annotations:

```
let x = ...
// [PATH1][PATH2]
if (...) {
  // [PATH1] (if true)
  let _ = move(x)
  // I can't use x anymore here!
} else {
  // [PATH2] (else)
  // I can still use x here!
}
// [PATH1][PATH2] (continuation)
// But I can't use x here.
```

in this example, there are only 2 program paths, the `[PATH1]` that goes through
the if true scope and into the continuation and `[PATH2]` through the else into
the continuation. Notice how the move only occurs along `[PATH1]` but that since
`[PATH1]` goes through the continuation that one can not use x again in the
continuation despite `[PATH2]` being safe.

If one works with vars, the analysis is exactly the same except that one can
conditionally re-initialize the var and thus be able to use it in the
continuation path. Consider the following example:

```
var x = ...
// [PATH1][PATH2]
if (...) {
  // [PATH1] (if true)
  let _ = move(x)
  // I can't use x anymore here!
  useX(x) // !! ERROR! Use after move.
  x = newValue
  // But now that I have re-assigned into x a new value, I can use the var
  // again.
} else {
  // [PATH2] (else)
  // I can still use x here!
}
// [PATH1][PATH2] (continuation)
// Since I reinitialized x along [PATH1] I can reuse the var here.
```

Notice how in the above, we are able to use `x` both in the true block AND the
continuation block since over all paths, x now has a valid value.

The value based analysis uses Ownership SSA to determine if values are used
after the move and handles non-address only lets. The address based analysis is
an SSA based analysis that determines if any uses of an address are reachable
from a move. All of these are already in tree and can be used today by invoking
the stdlib non-API function "_move" on a local let or move. *NOTE* This function
is always emit into client and transparent so there isn't an ABI impact so it is
safe to have it in front of a flag.

## Source compatibility

This is additive. If a user already in their module has a function called
"move", they can call the Stdlib specific move by calling Swift.move.

## Effect on ABI stability

None, move will always be a transparent always emit into client function.

## Effect on API resilience

None, this is additive.

## Alternatives considered

We could also introduce a separate `drop` function like languages like Rust does
that doesn't have a result like `move` does. We decided not to go with this
since in Swift the idiomatic way to throw away a value is to assign to `_`
implying that the idiomatic way to write `drop` would be:

```
let _ = move(x)
```

suggesting adding an additional API would not be idiomatic.

## Acknowledgments

Thanks to Nate Chandler, Tim Kientzle, Joe Groff for their help with this!
