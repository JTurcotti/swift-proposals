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

This example works out of the box with any reasonable linear ownership type system. If a race-causing bug were introduced, it would be easily caught as well. For example, consider the following `RacyServer` that keeps a reference to any `IntBox` it is passed, using that reference the next time its `process` function is called. This code is race-prone because a client that uses `RacyServer` for processing could continue to update its state after `process` returns, not realizing the server is accessing that state while serving another client. From an ownership perspective, the assignment `oldBox = box` transfers ownership of `box` from the local context to the field `oldBox`, so ownership is not free to be returned to the caller. Thus this example would not typecheck in a linear ownership system.

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

`RacyServer` demonstrates a common class of ownership bugs: inability to transfer ownership due to escaping aliases. Another common class of bugs involves attempting to use values when ownership is not present. The following example demonstrates an alternative client method that could attempt to access its `IntBox` while it is still being processed by the server. Since ownership has not been returned to the client yet at that point, an ownership type system would prevent the access.

```swift
extension Client {
  func unsafeMain() async {
    async let handle = server.process(box: box)
    
    print(box.x) // Error: ownership has been transered to the server, so this access is unsafe
      
    await handle
      
    print(box.x) // ownership has been returned to the client, so this access is safe
  }
}
```

## Fearless Types

So far ownership seems like a silver bullet for handling the sharing of mutable state between threads. Unfortunately, even slightly alterations to the surrounding code demonstrate its shortcomings. In the following `aliasingMain` method, an alias to `box` is introduced. Traditional linear ownership does not allow this example, because as soon as `boxAlias` is created the variable `box` becomes inaccessible - ownership has already been transfered - and the call `server.process(box : box)` is ill-typed. 

```swift
extension Client {
  func aliasingMain() async {
    let boxAlias = box
    
    async let handle = server.process(box: box)
    
    //ideally, both `box` and `boxAlias` would be inaccessible here
    
    await handle
    
    //ideally both `box` and `boxAlias` would be once again accessible here
  }
}
```

The issue here is that *each value is individually linear* (resembling the [move-only types proposal](https://github.com/apple/swift-evolution/blob/main/proposals/0390-noncopyable-structs-and-enums.md)). This provides safe concurrency, but is very cumbersome as a programming model. [Fearless types](https://www.cs.cornell.edu/andru/papers/gallifrey-types/) alleviates this burden by introducing *regions*. Regions are static groupings of values that could potentially alias or be reachable from each other. The fearless type system does not treat individual values linearly, but does treat regions linearly; when sharing occurs between concurrency domains, entire regions are gained and lost at a time. The above `aliasingMain` method would thus typecheck exactly as is with fearless types. Aliasing is undecidable in general, so, as the following example demonstrates, regions are a conservative approximation.

```swift
extension Client {
    func possiblyAliasingMain() async {
        let definiteAlias = box //`box` is in a region
        let possibleAlias = if ... ? IntBox() : box //same region as `box` due to possible aliasing
        let notAlias = IntBox() //`notAlias` gets a fresh region
        
        async let handle = server.process(box: box)
        
        print(notAlias) //safe - was not in the region that was sent
        print(possibleAlias) //Error: unsafe - in the sent region
        print(box) //Error: unsafe - in the sent region
        
        await handle
        
        //all three variable safe to access again
    }
}
```

### Implementation

To realize this idea of linear ownership regions - we need to make two additions to our standard typing context `Γ ::= x : τ, Γ | ε` that associates variables `x` with types `τ`. First, we must extend `Γ` to include region labels as well: `x : r τ` instead of `x : τ`. Second, we must add a context `ℋ ::= r, ℋ | ε` that lists the regions currently owned. `Γ` now has *strong update* semantics - to write `x = y` it must be the case that `x` and `y` have the same type, but their region may differ (i.e. `Γ ⊢ x : r τ` and `Γ ⊢ y : r' τ`). After the update, `Γ ⊢ x : r' τ`. This yields the desired behavior for aliases obtaining the same region labels. Value-semantic types can always be safely given fresh region labels. To unify the branches of conditionals that yield differing region labels for the same variable, it may be necessary to *attach* regions; merging their labels. Fresh regions get added to `ℋ`, and communicating across concurrency domains causes the appropriate reasons to be added/dropped from `ℋ` without altering `Γ`. Full rules for this system can be found in the [appendix](https://www.cs.cornell.edu/andru/papers/gallifrey-types/appendix.pdf) of the fearless types paper.

## Regions and Pointers

```swift
class Pair<L, R> {
    var left : L
    var right : R
}

func main() {
    let fst = IntBox()
    let snd = IntBox()
    let pair = Pair(fst, snd)
    
    async let handle = server.process(pair)
    
    //unsafe to use `fst` or `snd` as well as `pair`
    
    await handle
}
```

What happens if we throw a pointer structure into the mix? In the above example, we send a `pair` (by reference) to the server, which would allow it to access both the `left` and `right` fields as well. Thus if the client retained access to those fields, we'd have a data race. Thus regions must grow to include not just potential aliases, but potential reachability in the object graph as well. This immediately presents a bit of an issue:

```swift
let fst = IntBox()
let snd = IntBox()
let pair = Pair(fst, snd)
    
async let handle = server.process(fst)
    
//unsafe to use `snd` :(

await handle
```

When regions are defined as the minimal partition that groups potential aliases and potential reachability, we lose the ability to partially share data structures. Like pure linearity, this is too cumbersome to work with. 

We could try the opposite route - placing values read from fields into fresh regions. The immediate caveat is that we must remember these relationships between regions, or risk the following "double discovery" problem:

```swift
var left1 = pair.left //left1 lives in a fresh region
var left2 = pair.left //left2 lives in a fresh region as well
```

Since `left1` and `left2` here live in distinct regions, one could be used while the other was in use by a different concurrency domain - yielding a data race. The solution is to record in our typing context that `pair.left` points to a known region - using the same region label when `pair.left` is read a second time. To implement this, the `ℋ` context discussed above must be augmented to include these relationships: `ℋ = r⟨pair[left ↣ r']⟩, r'⟨⟩`. This way any time `pair.left` is read, the result will be known to live in region `r'`. 

These pointer-bounded regions allow data structures to be broken up and shared across concurrency domains at any desired granularity, but come at a cost: now that we've reified the pointer structure in our typing contexts, we must solve pointer analysis problems to perform typechecking. In particular, it can be very hard to come up with a "most general form" of a typing context that indicates the join of two branches that yield different pointer structures. 

### A Happy Medium

There are 
 
 
