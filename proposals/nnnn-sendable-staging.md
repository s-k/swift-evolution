# Staging in Sendable checking



## Introduction

[SE-0302](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md) introduced the `Sendable` protocol, which is used to indicate which types have values that can safely be copied across actors or, more generally, into any context where a copy of the value might be used concurrently with the original. Applied uniformly to all Swift code, `Sendable` checking eliminates a large class of data races caused by shared mutable state. Swift 5.5 does not perform complete checking for `Sendable` because doing so resulted in so many compiler errors and diagnostics that it undermined the usability of the feature. This document describes an approach to staging in `Sendable` checking in the Swift ecosystem that will trend toward complete checking.

## Motivation

`Sendable` checking ensures that a value of non-`Sendable` type does not get passed across actors and does not have its copies used concurrently. This prevents data races on shared mutable state, e.g., as introduced by classes:

```swift
class Counter {
  var value = 0
  
  func increment() { ... }
}

let counter = Counter()
Task {
  counter.increment() // SE-0302 error: cannot use let 'counter' with a non-sendable type 'Counter' from concurrently-executed code
}

```

Most classes are not `Sendable` without additional work, so fixing this code probably means moving away from classes (e.g., to an actor) or using some other synchronization mechanism, such as marking the class `@MainActor` or adding unsafe internal synchronization and using `@unchecked Sendable	`. 

On the other hand,  structs and enums often have value semantics. They are `Sendable` whenever they are composed of `Sendable` instance data. For example, the following `Point` struct is `Sendable` because `Double` is `Sendable`:

```swift
struct Point {
  var x, y: Double
}
```

Not all structs (or enums) are `Sendable`, however, because they could contain non-`Sendable` types such as our `Counter` class:

```swift
struct CountedPoint { // cannot be Sendable
  var x, y: Double
  let counter: Counter = Counter() // ... because Counter is not Sendable
}
```

Therefore, `Sendable` can only be applied "bottom-up": your struct or enum type can generally be made `Sendable` only when the types it is comprised of are already `Sendable`. However, it might be the case that a developer doesn't own all of the types. Let's say that there is a `Point3D` struct in library `A`, which has not adopted `Sendable` (even if it could):

```swift
// module A
public struct Point3D {
  var x, y, z: Double
}
```

And another module builds more geometry out of that type, but wants to use concurrency:

```swift
// module B
import A

struct Sphere { // note: trying to make this Sendable will produce an error ...
  var center: Point3D // ... because Point3D is not Sendable
  var radius: Double
}

let sphere = getSphere()
Task {
  print(sphere) // SE-0302 error: cannot use let 'sphere' with a non-sendable type 'Sphere' from concurrently-executed code
}
```

This is a problem, because the author of module `B` is prevented from using Swift Concurrency until module `A` has added appropriate `Sendable` annotations. Given the size of the Swift ecosystem, this implies significant delays in the overall usability of Swift Concurrency.

### Unchecked Sendable conformances

SE-0302 partially addresses this problem of "bottom-up" `Sendable` conformances by allowing `@unchecked Sendable` conformances, e.g. module `B` could add:

```swift
extension Point3D : @unchecked Sendable { }
```

which allows the `Sendable` conformance, but disables any checking that the instance data in `Point3D` is `Sendable`. This checking has to be disabled, because we can't necessarily see all of the instance data due to access control:

```swift
// module A
public struct Point3D {
  var x, y, z: Double
  private let changeCounter = Counter() // shared mutable state!
}
```

The author of `Point3D` doesn't know that clients are incorrectly assuming that the type is `Sendable`, and clients can't see into the private implementation to check whether the type is actually `Sendable` in practice. Moreover, if module `A` supports library evolution, the `counter` property could be added later, breaking the assumptions under which `@unchecked Sendable` was added. 

Finally, when module `A` eventually gains `Sendable` annotations, it will not indicate that `Point3D` is `Sendable`. However, module `B` will continue to rely on the its own `@unchecked Sendable` , even though module `B` should stop compiling at the point when it is clear that `Point3D` was not intended to be `Sendable`. In other words, the `@unchecked Sendable` mitigation required for `B` to adopt concurrency before `A` also prevents `B` from becoming safer once `A` has adopted concurrency.

### Source compatibility for existing clients

Adopting concurrency in a module goes beyond adding `Sendable` conformances to existing types. Often, one will need to introduce `Sendable` constraints on generic types (that then become requirements on clients) or mark closure parameters as `@Sendable` because the closures themselves need to be `Sendable`. For example, we might have an API that runs a particular user-provided closure concurrently and returns a future to monitor the operation:

```swift
// module C initially
public struct Future<T> { ... }
public func runFuture<T>(body: @escaping () -> T) -> Future<T> { ... }
```

Because the closure will be executed concurrently, `body` will need to be marked  `@Sendable` when this module `C` adopts concurrency. Additionally, the value produced by the closure will need to be `Sendable`, like this:

```swift
// module C after adopting concurrency
public struct Future<T: Sendable>: Sendable { ... }
public func runFuture<T: Sendable>(body: @Sendable @escaping () -> T) -> Future<T> { ... }
```

However, this change to module `C` will break existing clients of module `C` that haven't adopted concurrency:

```swift
// module D
import C

func wrapRunFuture<T>(body: @escaping () -> T) -> Future<T> {
  return runFuture(body: body) // error after C adopts concurrency: `T` does not conform to `Sendable`
                               // error after C adopts concurrency: cannot convert `body` to a `@Sendable` closure
}
```

This presents a different kind of problem: for module `C` to adopt concurrency, it needs to break its existing clients. Therefore, we need to introduce concurrency bottom-up (because we need `Sendable` conformances from the modules we depend on), but adopting from the bottom-up will break clients.

## Proposed solution

We propose two principles to guide the design of staged `Sendable` checking:

* Incremental adoption of concurrency should introduce incrementally more `Sendable` checking.
* `Sendable` problems outside of the user's module should not block progress nor produce an undue annotation burden.

Together, these principles allow a module to adopt concurrency with `Sendable` checking incrementally, without having to wait for the modules they depend on to have done so first, and without breaking the clients of that module. This approach enables piecemeal adoption of concurrency throughout the whole Swift ecosystem, improving data-race safety as each module adopts concurrency.

There are two specific features proposed here:

* Suppressing or downgrading `Sendable`-related errors so they do not cause the build to fail when the non-`Sendable` types involved come from modules that have not been updated for concurrency, and
* Providing a mechanism to distinguish APIs that predated concurrency and need to provide more lax concurrency-related checking .

### Module-sensitive `Sendable` checking 

We propose to phase in checking for `Sendable` by distinguishing modules that have fully adopted concurrency from those that have not. When importing a module that has fully adopted concurrency, the SE-0302 rules apply to the types from that module: a non-`Sendable` type from that module is used where a `Sendable` conformance is required, the compiler will emit an error for the missing conformance.

However, when using a type from a module that has not adopted concurrency, the lack of a `Sendable` conformance does not produce an error. Instead, the compiler will produce a warning, or even suppress the diagnostic entirely. This allows a module to adopt concurrency and get the benefits of some `Sendable` checking without waiting for all of the modules it depends on to adopt concurrency first.

For example, let's consider the prior modules `A` and `B`, where `A` has not yet adopted concurrency:

```swift
// module A (has not adopted concurrency)
public struct Point3D {
  var x, y, z: Double
}

// module B
import A

struct Sphere { // okay: we allow Sphere to be inferred as Sendable
  var center: Point3D // ... because Point3D is from a module that hasn't adopted concurrency yet
  var radius: Double
}

let sphere = getSphere()
Task {
  print(sphere) // okay: Sphere is Sendable
}
```

Here, `B` assumes that `Point3D` is `Sendable`, and `A` has not indicated whether that is true, so `B` is considered correct. This is not strictly safe from data races, but then again nothing in `A` is known to be safe from data races because it has not adopted any of the concurrency features to prevent them.

When module `A` is updated to use `Sendable`, one of two things happens when building module `B`. Most likely for this case, `Point3D` is marked as `Sendable`and module `B` remains correct (but is now known to be safer from data races). However, if `Point3D` is not `Sendable` (e.g., because it holds or may eventually hold something like `Counter`), module `B` will now fail with an error: `Sphere` cannot be `Sendable` because `Point3D` is not `Sendable`. By using knowledge about whether a module was updated for `Sendable`, we can provide a smoother transition path that allows a module to adopt concurrency with `Sendable` checking before the modules it depends on, then benefit from the stricter checking as those modules are updated. The case that fails--where `B` has assumed that `Point3D` is `Sendable` but an updated `A` declares that `Point3D` is *not* `Sendable`--is precisely the case that needs to produce an error because the data-race-safety assumptions of `B` have proven false.

Note that a given type can be comprised of multiple types from different modules that may have different rules applied to them. For example, the `Task` created below requires the dictionary type to be `Sendable`:

```swift
// module S1 that has not adopted concurrency
public struct NS1 { }

// module S2 that has adopted concurrency
public struct NS2 { }

// module S3
import S1
import S2

let dict: [String : (NS1, NS2)] = [:]
Task {
  print(dict) // capture of `dict` requires [String : (NS1, NS2)] to be Sendable
}
```

A Swift `Dictionary` is `Sendable` when both its `Key` and `Value` types are `Sendable`. A tuple type is `Sendable` when all of its element types are `Sendable`. In this case, neither `NS1` and `NS2`conforms to `Sendable`, so the overall dictionary type is not `Sendable`. However, the Swift compiler will only produce an error for the lack of the conformance of `NS2` to `Sendable` because `S2` has adopted concurrency. A similar dictionary type `[String: (NS1, NS1)]` would not produce an error because `S1` has not adopted concurrency.

### Updating a module for concurrency

Swift needs some indication as to whether a particular module has been updated for concurrency. The important bit of information here isn't that there may exist some types that are `Sendable`, but that the lack of conformance of a type to `Sendable` implies that the type should not be `Sendable`.

We propose that a module can be updated for concurrency in one of two ways:

* Use the Swift 6 language mode, which will enable all of Swift's data-race safety features.
* Use the compiler flag `-warn-concurrency` to enable warnings for Swift's data-race safety within Swift 5.x mode.

Once Swift 6 is available, we expect new code to be written in Swift 6. However, there is a large body of Swift 5.x code that may take longer to update to Swift 6, which might carry additional changes not related to concurrency. The `-warn-concurrency` flag can help this Swift 5.x code address concurrency-related issues both before Swift 6 is available and as a separate step along the path to adopting Swift 6.

### Supporting clients that predate concurrency

When adopting concurrency in existing code, it is likely that some existing public declarations will need to introduce `Sendable` requirements. In our previous example, module `C` was updated from:

```swift
// module C initially
public struct Future<T> { ... }
public func runFuture<T>(body: @escaping () -> T) -> Future<T> { ... }
```

to adopt concurrency:

```swift
// module C after adopting concurrency
public struct Future<T: Sendable>: Sendable { ... }
public func runFuture<T: Sendable>(body: @Sendable @escaping () -> T) -> Future<T> { ... }
```

As noted previously, this change will break some clients of module `C` that have not yet adopted concurrency:

```swift
// module D
import C

func wrapRunFuture<T>(body: @escaping () -> T) -> Future<T> {
  return runFuture(body: body) // error after C adopts concurrency: `T` does not conform to `Sendable`
                               // error after C adopts concurrency: cannot convert `body` to a `@Sendable` closure
}
```

The idea behind module-sensitive `Sendable` checking can be extended to cover the first problem, that the generic parameter `T` does not conform to `Sendable`, by downgrading that compiler error to a warning in Swift 5.x:

```
warning: `T` does not conform to `Sendable`
```

The second problem is that a value of non-`@Sendable` function type cannot be converted to `@Sendable` function type. While we could downgrade the error to a warning, this has the unfortunate side effect that new code that *does* use concurrency would not see `Sendable`-related problems at their appropriate severity. Therefore, we instead propose a solution that indicates that the closure parameter should be `@Sendable` for any code that has adopted concurrency, yet remains compatible with existing code:

```swift
// module C after adopting concurrency
public struct Future<T: Sendable>: Sendable { ... }
public func runFuture<T: Sendable>(@unsafeSendable body: @escaping () -> T) -> Future<T> { ... }
```

 The `@unsafeSendable` attribute indicates that the corresponding parameter (of function type) is intended to be `@Sendable` going forward. For a client that has not yet adopted concurrency, it will be treated as a non-`@Sendable` function type. For example:

```swift
// module D1 that has not adopted concurrency
import C

func wrapRunFuture<T>(body: @escaping () -> T) -> Future<T> {
  return runFuture(body: body) // warning: `T` does not conform to `Sendable`
                               // runFuture's `body` is considered to have non-`@Sendable` type
}
```

Now, if a client has adopted concurrency, either locally (because we're inside code that is using concurrency features such as `async` or `actor`) or throughout the whole module (because the code is compiled in Swift 6 or with `-warn-concurrency`), the parameter will be treated as `@Sendable`:

```swift
// module D2 that has not adopted concurrency globally, but is using some concurrency features
import C

func runFutureAsync<T: Sendable>(body: @escaping () -> T) async -> T {
  let future = runFuture(body: body) // error: cannot convert `body` to a `@Sendable` closure
  return await future.get()
}
```

Here, the fact that `runFuture(body:)` is referenced inside an `async` function, so the fact that this code has adopted concurrency locally means that the `body` parameter will be considered `@Sendable`. In fact, the type of `runFuture(body:)` in this context is

```swift
<T: Sendable>(@Sendable @escaping () -> T) -> Future<T>
```

whereas in another context (the prior `wrapRunFuture` where no concurrency had been adopted) the type would remain

```swift
<T: Sendable>(@escaping () -> T) -> Future<T>
```

### Three Stages of Concurrency Adoption

There are three stages of adoption of concurrency for a given module:

* Swift 5.x with localized use of concurrency (`async`, `actor`, `Task` APIs, explicit `Sendable` conformances, etc.): the compiler will warn about `Sendable` issues only within code that directly uses one of the above concurrency features, and limit diagnostics to types that are either from that same module or from a module that has adopted concurrency. There should be no `Sendable`-related errors, so all data-race safety is targeted at new code and does not prevent progress. `@_unsafeSendable` parameters are only treated as `@Sendable` within the code that has adopted concurrency, and non-`@Sendable` elsewhere.
* Swift 5.x with`-warn-concurrency` should diagnose `Sendable` issues throughout the whole module, but only as a warning. `@_unsafeSendable` parameters will be treated as `@Sendable` consistently.
* Swift 6 will diagnose `Sendable` issues throughout the whole module, and treat `@_unsafeSendable` parameters as `@Sendable` consistently. All of the diagnostics will be errors *except* when a type from a module that has not adopted concurrency fails to conform to `Sendable`; such cases will be emitted as warnings, because they are outside of the user's direct control.

## Unresolved questions and notes

* `-warn-concurrency` seems like the wrong flag name, given the different interpretation of `@unsafeSendable` . Perhaps `-strict-concurrency` would better describe what's happening?
* `Sendable` checking isn't the only aspect of concurrency that needs to be staged in. `@MainActor` annotations need a similar treatment, and in fact the implementation contains hidden features like `@_unsafeMainActor` and `@MainActor(unsafe)` that are used to state when existing APIs expect to be on the main actor without breaking existing clients. Should those be separate proposals, or should we unify them?
* `@_unsafeSendable` is very specialized to `@Sendable`, and cannot capture other concurrency-motivated changes like the addition of the `T: Sendable` requirement. As noted above, there's a similar `@_unsafeMainActor`parameter attribute that shows the need to generalize this approach at least somewhat. We could consider introducing a declaration attribute `@unsafePreconcurrency` that indicates that the API existed before concurrency, and then specify how the type of that declaration should be transformed to work with existing Swift 5.x code that has no adopted concurrency. The transformation could (e.g.) remove `Sendable` constraints, transform `@Sendable` and global-actor-qualified function types into their pre-concurrency counterparts, and so on.
