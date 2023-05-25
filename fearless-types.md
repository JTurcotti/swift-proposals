# An Intro to Fearless Types

* Authors: [Joshua Turcotti](https://github.com/JTurcotti)

## Introduction

Swift allows values to be sent between concurrency domains only if their type ensures the resulting sharing is safe (i.e. cannot lead to data races).
Such types are called [`Sendable`](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md),
and only deeply immutable types can be statically assured to be `Sendable`. This prevents sharing of mutable state - a restriction that may be unacceptable for the
long-term evolution of Swift. 

Linear (technically, affine) type systems offer a promising solution called *ownership*; various concurrency domains can *own* values over the course of execution, and static
analyses can prove that no distinct domains can simultaneously own the same value. This ensures race-freedom while allowing any type to be "Sendable". 
Unfortunately, linear type systems can constrain natural programming patterns. In particular, any programming patterns that cause aliasing to arise
will often turn out to be un-typeable. [Recent work](https://www.cs.cornell.edu/andru/papers/gallifrey-types/) presented a new approach for tempering linear
types to achieve safe concurrency (i.e. race freedom) without constraining natural programming patterns. In the tradition of Rust, this goal is called 
*Fearless Concurrency*, and we will call the type system presented in that paper *fearless types*.

This document outlines what fearless types could look like if implemented in Swift.

## Linear Ownership Types

In short, the addition of linear ownership types to Swift would allow mutable state to be safely shared between actors. 

Consider the following example, in which a `Client` creates a mutable `IntBox`, and asks a `Server` to process the contents of the box. `IntBox` cannot be made `Sendable`, 
so Swift currently cannot allow this natural programming pattern.

```swift
class IntBox {
  var x = 0
}

protocol Server {
  func process(box : IntBox) async
}

actor NonRacyServer : Server {
  func process(box : IntBox) {
    // for this actor to modify the contents of a passed box - it must *take ownership* of the box
    box.x = box.x + 1
    // after the call, ownership is returned to the calling actor
  }
}

actor Client {
  let box : IntBox = IntBox()
  let server : Server = NonRacyServer()

  func main() async {
    // the `box` value is owned by this actor, so access is safe
    print("pre: the box contains \(box.x)")
  
    // this call temporarily transfers ownership to the Server actor
    await server.process(box: box)
      
    // after the call, ownership of `box` is returned to this actor, so access is again safe
    print("post: the box contains \(box.x)")
}
```

This example works out of the box with any reasonable linear ownership type system. Race-causing bugs such are easily caught as well. Consider the following `RacyServer` that keeps a reference to any `IntBox` it is passed, using that reference the next time its `process` function is called. This code is race-prone because a client that uses `RacyServer` for processing could continue to update its state after `process` returns, not realizing the server is accessing that state while serving another client. From an ownership perspective, the assignment `oldBox = box` transfers ownership of `box` from the local context to the field `oldBox`, so ownership is not free to be returned to the caller. Thus this example would not typecheck in a linear ownership system.

```swift
actor RacyServer : Server {
  var oldBox : IntBox = IntBox()

  func process(box : IntBox) {
    // takes ownership of `box`
    box.x = box.x + oldBox.x
    // transfers ownership to the field `oldBox`
    oldBox = box
    // Error: cannot return ownership of `box` to the caller because the field `oldBox` has ownership instead
  }
}
```

`RacyServer` demonstrates a common class of ownership bugs: inability to transfer ownership due to escaping aliases. Another common class of bugs involves attempting to use values when ownership is not present. The following example demonstrates an alternative client method that could attempt to access its `IntBox` while it is still being processed by thre server. Since ownership has not been returned to the client yet at that point, an ownership type system would prevent the access.

```swift
extension Client {
  func unsafeMain() async {
    async let handle = server.process(box: box)
    
    print(box.x) // Error: ownership has been transered to the server, so this access is unsafe
      
    await handle
      
    print(box.x) // ownership has been returned to the client, so this access is safe
  }
}



