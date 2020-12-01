# Rethrowing protocol conformances

Swift's `rethrows` feature allows a function to specify that it will throw only in narrow circumstances where it is calling a user-provided function argument. For example, the `Sequence.map` operation is `rethrows`:

```swift
extension Sequence {
  func map<Transformed>(transform: (Element) throws -> Transformed) rethrows -> [Transformed] {
    var result = [Transformed]()
    var iterator = self.makeIterator()
    while let element = iterator.next() {
      result.append(try transform(element))   // note: this is the only `try`!
    }
    return result
  }
}
```

When calling `Sequence.map`, the `map` is only considered to throw when the function argument passed to the `transform` parameter can throw:

```swift
_ = [1, 2, 3].map { String($0) }  // okay: map does not throw because the closure does not throw
_ = try ["1", "2", "3"].map { (string: String) -> Int in
  guard let result = Int(string) else { throw IntParseError(string) }
  return result
} // okay: map can throw because the closure can throw
```

Swift's `rethrows` is effective so long as the user-provided operations that potentially throw errors are passed via function parameters. However, many user-provided operations are provided indirectly through protocol conformances. This proposal extends the notion of `rethrows` to protocol conformances.

## Motivation

Let's consider variant of `map` where getting the next element from an iterator could fail. This would allow us to (for example) better model input streams as sequences, because reading from an input stream can fail. The existing `Sequence` protocol doesn't support this, so we'll invent a new `FailableSequence` protocol and its corresponding iterator protocol:

```swift
protocol FailableIterator {
  associatedtype Element

  mutating func next() throws -> Element?
}

protocol FailableSequence {
  associatedtype Iterator: FailableIterator
  typealias Element = Iterator.Element

  func makeIterator() -> Iterator
}
```

Now, a sequence type that (say) reads lines from a terminal can conform to `FailableSequence`:

```swift
/// Read a line from standard input
func readline() throws -> String? { ... }

struct ReadLine: FailableSequence {
  struct Iterator: FailableIterator {
    typealias Element = String

    mutating func next() throws -> String? {
      return try readline()
    }
  }
  
  func makeIterator() -> Iterator { ... }
}
```

Note that types that conform to `IteratorProtocol` also satisfy the requirements of `FailableIterator`, and types that conform to `Sequence` also satisfy the requirements of `FailableSequence`, because a non-throwing method can satisfy a corresponding requirement that `throws`. For example, we can make `Array` conform to `FailableSequence`:

```swift
extension IndexingIterator: FailableIterator { } // Array.Iterator is an IndexingIterator
extension Array: FailableSequence { }
```

This makes `FailableSequence` more general than the existing `Sequence`. However, using arrays via `FailableSequence` is likely to be frustrating, because one has to assume that every operation that traverses a failable sequence can `throw`, even though traversing an array *never* throws. Historically, this is one of the reasons why `Sequence` doesn't support failable sequences.

Let's try to implement a form of `map` (call it `map2`) on `FailableSequence`:

```swift
extension FailableSequence {
  func map2<Transformed>(transform: (Element) throws -> Transformed) rethrows -> [Transformed] {
    var result = [Transformed]()
    var iterator = self.makeIterator()
    while let element = try iterator.next() { // error: call can throw, but the error is not handled; a function declared 'rethrows' may only throw if its parameter does
      result.append(try transform(element))
    }
    return result
  }
}
```

The error produced is correct: in a `rethrows` function, the only permissible throwing operations are calls to function parameters that are potentially-throwing functions. That guarantees, statically, that when the argument for `transform` is non-throwing, `map2` will never throw. By having the call to `next()` be potentially throwing, we violate that guarantee because an error could be thrown from `next()` (and out through `map2`) even in cases where the `transform` argument is non-throwing.

This proposal seeks to make the above definition of `map2` well-formed, and have a call to `map2` be consider throwing when either the `transform` argument or the iterator's `next` operation is throwing. For example:

```swift
_ = try ReadLine().map2 { $0 + "!" } // okay: map2 can throw because ReadLine.Iterator's next() throws
_ = try ReadLine().map2 { (string: String) -> Int in
  guard let result = Int(string) else { throw IntParseError(string) }
  return result
} // okay: map2 can throw because the closure can throw and ReadLine.Iterator's next() throws

_ = [1, 2, 3].map2 { String($0) }  // okay: map2 does not throw because the closure does not throw and Array.Iterator's next() does not throw
_ = try ["1", "2", "3"].map2 { (string: String) -> Int in
  guard let result = Int(string) else { throw IntParseError(string) }
  return result
} // okay: map2 can throw because the closure can throw
```

## Proposed solution

The proposed solution is to consider protocol conformances to be a source of throwing behavior for `rethrows`, allowing `rethrows` to reason about the throwing behavior of user operations provided via protocol conformances.

## Rethrows protocols

Rethrowing behavior for protocol conformances begins within the protocols themselves. A protocol requirement will be able to be marked as `rethrows` as follows:

```swift
protocol FailableIterator {
  associatedtype Element

  mutating func next() rethrows -> Element?
}
```

Such a protocol is called a *rethrows protocol*. When a type conforms to a rethrows protocol, Swift records whether the method used to satisfy the `next()` requirement was throwing or not. For example, the conformance of `ReadLine.Iterator` to `FailableIterator` throws (because `ReadLine.Iterator.next()` is marked `throws`), but the conformance of `IndexingIterator` to `Iterator` does not (because `IndexingIterator.next()` is not marked `throws`). 

### Rethrows checking with protocol conformances

Any generic requirement that requires conformance to a rethrows protocol becomes part of `rethrows` checking. For example, consider a simple wrapper over `FailableIterator`'s `next()`:

```swift
func getNext<Iterator: FailableIterator>(_ iterator: inout Iterator) rethrows -> Iterator.Element? {
  return try iterator.next()
}
```

`getNext(_:)` is only using `rethrows` operations from `FailableIterator`, so it is well-formed: it only throws when the call to `next()` throws. Therefore, calls to `getNext(_:)` will throw only when the conformance provided for the `Iterator: FailableIterator` requirement throws. For example:

```swift
func testGetNext<C: Collection>(
    indexing: inout IndexingIterator<C>, readline: inout ReadLine.Iterator
) throws {
  getNext(&indexing).    // okay: conformance of IndexingIterator: FailableIterator does not throw, so call does not throw
  try getNext(&readline) // okay: conformance of ReadLine.Iterator: FailableIterator does throw, so call throws
}
```

The definition of a rethrows protocol is transitive: if any generic requirement within the definition of the protocol (e.g., via an inherited protocol or a requirement on an associated type) is a rethrows protocol, then at protocol is also a rethrows protocol. Therefore, `FailableSequence` is a rethrows protocol because its `Iterator` type must conform to the rethrows protocol `FailableIterator`:

```swift
protocol FailableSequence {  // implicitly a rethrows protocol
  associatedtype Iterator: FailableIterator   // generic requirement on the rethrows protocol FailableIterator
  typealias Element = Iterator.Element

  func makeIterator() -> Iterator
}
```

This definition is what permits our `map2` example to become well-formed: 

```swift
extension FailableSequence {
  func map2<Transformed>(transform: (Element) throws -> Transformed) rethrows -> [Transformed] {
    var result = [Transformed]()
    var iterator = self.makeIterator()
    while let element = try iterator.next() { // okay: FailableIterator.next() is rethrows
      result.append(try transform(element))   // okay: transform is throws
    }
    return result
  }
}
```

Defining a method in an extension of the `FailableSequence` protocol implies a requirement `Self: FailableSequence`, and `FailableSequence` is a rethrows protocol. In the body of `map2`, both calls to potentially-throwing operations are covered by `rethrows` checking: the call to `iterator.next()` throws when `Self.Iterator`'s conformance to `FailableIterator` throws, and the call to `transform` throws when the `transform` argument throws.

### Conditionally-rethrowing conformances

Both `IndexingIterator` and `ReadLine.Iterator` are simple in the sense that the `next()` either can't throw or can throw, and they don't depend on anything else. However, consider an adapter over another `FailableIterator` that does nothing but pass-through its `next()` to the underlying iterator:

```swift
struct FailableAdapter<Wrapped: FailableIterator>: FailableIterator {
  typealias Element = Wrapper.Element
  
  var wrapped: Wrapped
  
  mutating func next() rethrows -> Element? {
    return try wrapped.next()
  }
}
```

The `FailableAdapter.next()` function is permitted to be `rethrows` because it only calls into `rethrows` protocol requirements. Hence, calling `next()` on `FailableAdapter<IndexingIterator<[Int]>>` will not throw, but calling `next()` on `FailableAdapter<ReadLine.Iterator>` can throw.

This rethrows logic extends to the conformance of `FailableAdapter` to `FailableIterator`: because the `next()` method satisfying the `rethrows` requirement is itself `rethrows`, the conformance is throwing when the conformance of `Wrapper: FailableIterator` is throwing. This allows rethrowing-ness to compose, e.g.,

```swift
func adapted(arrayIterator: IndexingIterator<[Int]>, readLineIterator: ReadLine.Iterator) {
  var adaptedArrayIterator = FailableAdapter(wrapped: arrayIterator)
  getNext(&adaptedArrayIterator) // okay: FailableAdapter<IndexingIterator<[Int]>>: FailableIterator does not throw

  var adaptedReadLineIterator = FailableAdapter(wrapped: readLineIterator)
  try getNext(&adaptedReadLineIterator) // okay: FailableAdapter<ReadLine.Iterator>: FailableIterator can throw
}
```

## Open questions

* Q: This is so cool. Can we fix `Sequence`?
    * A: Probably not directly, because of ABI. In theory we might be able to retroactively add `FailableSequence` as a protocol that `Sequence` inherits (ditto for `FailableIterator` and `IteratorProtocol`), with an additional rule that allows `Sequence` and `IteratorProtocol` to not be rethrowing protocols because they've restated non-throwing versions of the protocols they inherit. This needs more thought, but would solve a longstanding weakness in the `Sequence` protocol.
* Q: Do we need to mark protocol requirements as `rethrows`? Why won't `throws` work?
    * A: It's possible that we could use `throws` as the indicator on protocol requirements, similar to what we for function parameters. That makes this potentially a source-breaking change, since there are existing protocol conformances that would become non-throwing and that could change the behavior of some existing `rethrows` operations.
