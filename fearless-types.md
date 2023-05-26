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

We could try the opposite route - placing values read from fields into fresh regions. This allows the above example that sends away `fst` to concurrently access `snd` as desired. The immediate caveat is that we must remember these relationships between regions, or risk the following "double discovery" problem:

```swift
var left1 = pair.left //left1 lives in a fresh region
var left2 = pair.left //left2 lives in a fresh region as well
```

Since `left1` and `left2` here live in distinct regions, one could be used while the other was in use by a different concurrency domain - yielding a data race. The solution is to record in our typing context that `pair.left` points to a known region - using the same region label when `pair.left` is read a second time. To implement this, the `ℋ` context discussed above must be augmented to include these relationships: `ℋ = r⟨pair[left ↣ r']⟩, r'⟨⟩`. This way any time `pair.left` is read, the result will be known to live in region `r'`. 

These pointer-bounded regions allow data structures to be broken up and shared across concurrency domains at any desired granularity, but come at a cost: now that we've reified the pointer structure in our typing contexts, we must solve pointer analysis problems to perform typechecking. In particular, it can be very hard to come up with a "most general form" of a typing context that indicates the join of two branches that yield different pointer structures. Concretely, imagine a loop that iterates along a linked list. The typing context associated with each step of that iteration would specify the the depth attained thusfar, so no general fixpoint would exist and the loop would be untypeable. Alternatively, imagine wanting to be able to return from a function either the `left` *or* the `right` side of a `Pair` data structure - if regions were fully pointer-bounded, no single type for that function would allow both.

## Isolated Fields

To summarize the above section, defining regions so that pointers cannot escape them yields overly large regions that prevent fine-grained sharing of data structures, and defining regions so that pointers must escape them yields overly precise typing contexts that prohibit general purpose programming. The natural solution is a language with *both* types of pointers - region-escaping and region-bounded. By default, pointers will be region-bounded. By placing the `iso` "isolated" annotation on the declaration of a field, the pointers accessed through that field will instead be region-escaping. Good programming patterns will define data structures with a mix of isolated and non-isolated fields. 

The pair data structure could now be written either of these ways:

```swift
class Pair<T> {
	var left : T
	var right : T
	
	// this initializer is valid for both Pair and IsoPair
	func init(l : T, r : T) {
		left = l
		right = r
	}

	// this initializer is NOT valid for IsoPair
	func init(val : T) {
		left = val
		right = val
	}
		
	// unfortunately, only this sequential `process` is valid for Pair
	func process(l_server : Server<T>, r_server : Server<T>) async {
		await l_server.process(left)
		await r_server.process(right)
	}
}

class IsoPair<T> {
	iso var left : T
	iso var right : T
	
	func init(l : T, r : T) {
		left = l
		right = r
	}
	
	// IsoPair can implement `process` either sequentially or in parallel
	func process(l_server : Server<T>, r_server : Server<T>) async {
		async let l_handle = l_server.process(left)
		async let r_handle = r_server.process(right)
		await l_handle
		await r_handle
	}
}
```

With this simple data structure, the advantages and disadvantages of `iso` fields are fairly straighforward. `Pair` does not treat its two fields linearly, so initializers have the freedom to use a single value to initialize both. Unfortunately, this means that `Pair` does not have sufficient information to allow its two fields to be processed in parallel. Because some contexts will benefit from increased flexibility, and some will benefit from increased power to parallelize computation, the ability to define `Pair` and `IsoPair` could be advantageous. 

For more complex, possibly recursive, data structures that use isolated fields, the tradeoffs become a bit a more complex. Building up recursive data structures is not possible without the possibility of pointer structures becoming arbitrarily large, so the typing context must allow for isolated fields to be accessed without having to be permantly tracked in `ℋ`. This is done by an invariant stating that the untracked isolated pointers form a tree structure. This means they can be safely dropped from the typing context, while still being able to be re-explored at any time with a fresh region name without risking double discovery.

The nature of this tree-of-untracked-regions invariant directly motivates the design principles around recursive isolated fields: fields should be made isolated when, except for temporary violations, they will form a tree structure. For example, this is true for the spine pointers of a singly linked list, but not of a doubly linked list. That temporary violations are allowed is an important relaxation; even for a singly linked list, operations such as swapping nodes will cause the structure to temporarily pass through a non-tree state. For example, we could write the following `swap` function for our `IsoPair`:

```swift
extension IsoPair<T> {
	func swap() {
		let temp = left
		left = right //after this assignment, `left` and `right` alias each other
		right = temp
	}
}
```

This `swap` function violates not just linearity of values, but even linearity of regions; `left` and `right` are isolated fields, so they should point to fresh regions, but they do not - they point to the same region. `swap` typechecks because our system determines that this is just a temporary violation. This increased expressivity makes programming much easier. Here's the same example with typing contexts added:

```swift
extension IsoPair<T> {
	func swap() {
		// ℋ: r₀⟨⟩; Γ: self: r₀ IsoPair<T>
		let temp = left
		// ℋ: r₀⟨self[left ↣ r₁]⟩, r₁⟨⟩; Γ: self: r₀ IsoPair<T>, temp : r₁ T
		// ↑ can be implicitly elaborated to ↓ - this is called "focus and explore"
		// ℋ: r₀⟨self[left ↣ r₂, right ↣ r₁]⟩, r₁⟨⟩, r₂⟨⟩; Γ: self: r₀ IsoPair<T>, temp : r₁ T //tree invariant holds
		left = right //after this assignment, `left` and `right` alias each other
		// ℋ: r₀⟨self[left ↣ r₂, right ↣ r₂]⟩, r₁⟨⟩, r₂⟨⟩; Γ: self: r₀ IsoPair<T>, temp : r₁ T //tree invariant broken
		right = temp
		// ℋ: r₀⟨self[left ↣ r₂, right ↣ r₁]⟩, r₁⟨⟩, r₂⟨⟩; Γ: self: r₀ IsoPair<T>, temp : r₁ T //tree invariant holds
		// ↑ can be implicitly simplified to ↓ - this is called "retract and unfocus"
		// ℋ: r₀⟨⟩; Γ: self: r₀ IsoPair<T>, temp : r₁ T //note `temp` is still in scope as a variable, but inaccessible - its region has been dropped
	}
}
```

Although it may appear intimidating, all the typing contexts are doing in the above example is encoding the minimal graph structure involved in this example: two fields are explored in a tree-like state, temporarily deviate from that state, then are returned to it. Note that some typing rules, such as to read an isolated field, require `ℋ` to have a specific form, such as the presence of that isolated field. Similarly, the typing rule for functions requires the starting and finishing typing contexts to exactly match those expected by the signature of the function. Here, that means `self` is in a region with no tracked state. The implicit elaborations and simplifications mentioned above are necessary to help these typing rules apply, and come from a language of *virtual transformations*. Virtual transformations allow the typing context to be subsituted for an alternative that captures the same, or strictly less, information. To successfully typecheck programs, these must be inferred by the compiler. 


For symmetry, here's what typechecking `swap` for the non-isolated `Pair` would look like:

```swift
extension Pair<T> {
	func swap() {
		// ℋ: r₀⟨⟩; Γ: self: r₀ Pair<T>
		let temp = left
		// ℋ: r₀⟨⟩; Γ: self: r₀ Pair<T>, temp : r₀ T
		left = right
		// ℋ: r₀⟨⟩; Γ: self: r₀ Pair<T>, temp : r₀ T
		right = temp
		// ℋ: r₀⟨⟩; Γ: self: r₀ Pair<T>, temp : r₀ T
	}
}
```

The interesting part of typechecking `Pair.swap` is that there is no interesting part; no fields need appear in the typing context because only isolated fields need to be tracked that way. A second (or third) region is never introduced, because fresh regions only get introduced when reading isolated fields, not non-isolated fields. `temp` remains accessible throughout the entire function. Finally, the function would've typechecked even if the final assignment `right = temp` were removed - this is in contrast to `IsoPair.swap` in which the function would have been rejected by the typechecker if that final assignment were not in place to return the function to a tree-like state.

### More Expressiveness Differences between Isolated and Non-Isolated fields

Why stop at typechecking `swap`? Let's look at the typing contexts behind the other claims made above about the differences between `Pair` and `IsoPair`.

#### 2-Arg Initializer

The 2-arg initializer typechecks in both `Pair` and `IsoPair`, but with slightly different mechanisms:

```swift
class Pair<T> {
	...
	func init(l : T, r : T) {
		// ℋ: r₀⟨⟩, r₁⟨⟩, r₂⟨⟩; Γ: self: r₀ Pair<T>, l: r₁ T, r: r₂ T //each arg comes from a distinct region
		// ↑ can be implicitly coerced to ↓ - this is called "attach"
		// ℋ: r₀⟨⟩, r₂⟨⟩; Γ: self: r₀ Pair<T>, l: r₀ T, r: r₂ T //can only assign non-isolated fields of a value to another value in the same region
		left = l
		// ℋ: r₀⟨⟩, r₂⟨⟩; Γ: self: r₀ Pair<T>, l: r₀ T, r: r₂ T //assignment has no effect on typing context
		// ↑ can be implicitly coerced to ↓ - this is another "attach"
		// ℋ: r₀⟨⟩; Γ: self: r₀ Pair<T>, l: r₀ T, r: r₀ T
		right = r
		// ℋ: r₀⟨⟩; Γ: self: r₀ Pair<T>, l: r₀ T, r: r₀ T
	}
}
```

2-arg `Pair.init` highlights an important invariant mentioned above: non-isolated cannot cross regions. Because arguments to a function by default come from distinct regions, the *attach* virtual transformation is necessary to 'forget' the fact that `self`, `l`, and `r` come from distinct regions. This is a strictly information losing operation, unlike *explore*, *retract*, and *focus* above that are information-preserving. Other than the application of the attach transformation, typechecking `Pair.init` is straightforward, as no isolated fields are involved. 

```swift
class IsoPair<T> {
	...
	func init(l : T, r: T) {
		// ℋ: r₀⟨self[left ↣ ⊥, right ↣ ⊥]⟩, r₁⟨⟩, r₂⟨⟩; Γ: self: r₀ IsoPair<T>, l: r₁ T, r: r₂ T
		left = l
		// ℋ: r₀⟨self[left ↣ r₁, right ↣ ⊥]⟩, r₁⟨⟩, r₂⟨⟩; Γ: self: r₀ IsoPair<T>, l: r₁ T, r: r₂ T
		right = r
		// ℋ: r₀⟨self[left ↣ r₁, right ↣ r₂]⟩, r₁⟨⟩, r₂⟨⟩; Γ: self: r₀ IsoPair<T>, l: r₁ T, r: r₂ T
		// ↑ can be implicitly coerced to ↓ - two "retract"s and an "unfocus"
		// ℋ: r₀⟨⟩; Γ: self: r₀ Pair<T>, l: r₁ T, r: r₂ T //note `l` and `r` are inaccessible
	}
}
```

2-arg `IsoPair.init` is also simple, and highlights properties of `ℋ` we've already seen above: assignments of isolated fields must be exactly reflected in `ℋ`, and to return from a function the regions of results such as `self` must be in a simple state. Performing the *retract*s mentioned in the example render the arguments `l` and `r` inacccessible - this is exactly what we want. If `l` and `r` were still accessible but not known to be reachable from `self`, they could be sent to another thread before `init` returns, yielding dobule-ownership and a potential race. Also of slight note is the way that initialization of isolated fields can be handled. Here, we choose to typecheck initializers by using a starting typing context of `r₀⟨self[left ↣ ⊥, right ↣ ⊥]⟩`. The `⊥` indicates that the isolated fields `left` and `right` point to inaccessible regions - to access the fields or to return from the function, they must first be assigned to an accessible region. In a language such as Swift that already enforces those two rules, using the simple starting type of `r₀⟨⟩` for `self`'s region would also be sound, but would yield typing contexts that contain "phantom" regions that appear accessible but are not. This is a design decision to potentially be made down the road.

#### 1-Arg Initializer.

The 1-arg initializer typechecks only for `Pair`:

```swift
class Pair<T> {
	...
	func init(val : T) {
		// ℋ: r₀⟨⟩, r₁⟨⟩; Γ: self: r₀ Pair<T>, val : r₁ T
		// ↑ attach ↓
		// ℋ: r₀⟨⟩; Γ: self: r₀ Pair<T>, val : r₀ T
		left = val
		// same context as above
		right = val
		// same context as above
	}
}
```

Typechecking the 1-arg `Pair.init` is nearly identical to its 2-arg counterpart, all that changes is we need to perform only 1 attach instead of 2. This is expected: non-isolated fields "don't care" about aliasing/uniqueness of their values so re-using the same value for two non-isolated fields should be just as easy as using distinct values. Now let's try to do this for `IsoPair`:

```swift
class IsoPair<T> {
	...
	func init(val : T) {
		// ℋ: r₀⟨self[left ↣ ⊥, right ↣ ⊥]⟩, r₁⟨⟩; Γ: self: r₀ IsoPair<T>, val : r₁ T
		left = val
		// ℋ: r₀⟨self[left ↣ r₁, right ↣ ⊥]⟩, r₁⟨⟩; Γ: self: r₀ IsoPair<T>, val : r₁ T
		right = val
		// ℋ: r₀⟨self[left ↣ r₁, right ↣ r₁]⟩, r₁⟨⟩; Γ: self: r₀ IsoPair<T>, val : r₁ T
		// Error: region `r₀` of `self` cannot be reduced to simple state at conclusion of function `init`
	}
}
```

We can't typecheck a 1-arg `IsoPair.init`. The two assignments both typecheck, but we're left with the context `ℋ: r₀⟨self[left ↣ r₁, right ↣ r₁]⟩, r₁⟨⟩`. The signature of `init` specifies `r₀⟨⟩` as the desired output type for the region of `self`, which clearly doesn't match exactly what we've got but it looks like maybe some inferred *retract*s could do the job. WLOG, we could try *retract `self.left`*, which would give us `ℋ: r₀⟨self[right ↣ r₁]⟩` - retract removes the isolated field `self.left` from tracking and drops the region `r₁` that it pointed to. This latter part, dropping the region it pointed to, is vital to prevent the "double discovery" problem mentioned earlier; if we let `r₁` stick around, leaving `val` accessible as well, we could re-explore `self.left` with an assignment like `badVar = self.left`, and get a new region label to access `badVar` with while still being able to access `val` through `r₁`. This opens us up to data races, because now we could keep `val` while giving another thread access to `badVar`. Thus the fact that *retract* has the signature `r⟨x[f ↣ r', F]⟩, r'⟨⟩ ↦ r⟨x[F]⟩` instead of `r⟨x[f ↣ r', F]⟩ ↦ r⟨x[F]⟩` is vital. So though we can apply *retract* once to remove `self.left` from `ℋ`, there will be no way to also also remove `self.right`. Morally, it functions as a tombstone.

Of course, we don't want to allow this `init` function anyways - so this ill-typedness is a desired result.

#### Parallel process 
	
> Note: this section uses a preliminary formalization of futures into the fearless types context involving a separate context Φ, this is meant to be a placeholder for whatever alternate formalizations (such as rolling that information directly into Γ or ℋ) eventually fit most nicely with the desired behavior of futures.

We've alluded to, but haven't actually seen instances of any communcation between concurrency domains in the above examples. To provide the first one, let's say we have a function `server.foo` with signature `(ℋ: r₀⟨⟩; Γ: x: r₀ T) → (ℋ: r₀⟨⟩, r₁⟨⟩; Γ: x: r₀ T, result: r₁ S)`. This could be declared with syntax such as `func foo(preserves x : T) → (fresh S)` - indicating that it takes an argument of type `T`, and leaves that argument accessible to the caller while also providing a result of type `S` in a fresh region. Calling `server.foo` could appear as follows:

```swift
let x : T = ...
// ℋ: r₀⟨⟩; Γ: x: r₀ T; Φ: ∅
async let y̅ = server.foo(x)
// ℋ: ∅; Γ: x: r₀ T; Φ: y̅: (r₀⟨⟩, r₁⟨⟩, r₁, S)
... 
//note for whatever computation happens here, `x` is still in scope but inaccessible
...
let y = await y̅
// ℋ: r₀⟨⟩, r₁⟨⟩; Γ: x: r₀ T, y: t₁ S; Φ: ∅
```

Here, creation of the future (`async let`) moves the region of `x` out of `ℋ` and adds a new 3-tuple to Φ recording 1) the region types that will be returned to `ℋ` when the future completes 2) the region of the resulting value 3) the type of the resulting value. Redeeming the future (`await`) removes the binding from `Φ`, and returns the appropriate region types to `ℋ`. 

Now let's see why these rules prevent a parallel `Pair.process` from typechecking:

```swift
class Pair<T> {
	...
	func process(l_server : Server<T>, r_server : Server<T>) async {
		// `l_sever` and `r_server` omitted from contexts for simplicity
		// ℋ: r₀⟨⟩; Γ: self: r₀ Pair<T>; Φ: ∅
		async let l_handle = l_server.process(left)
		// ℋ: ∅; Γ: self: r₀ Pair<T>; Φ: l_handle: (r₀⟨⟩, ∅, ∅)
		async let r_handle = r_server.process(right) //Error: `self` not accessible - region is held by future handle `l_handle`
		await l_handle
		await r_handle
	}
```	

Now let's see why parallel `IsoPair.process` does typecheck:

```swift
class IsoPair<T> {
	...
	func process(l_server : Server<T>, r_server : Server<T>) async {
		// `l_sever` and `r_server` omitted from contexts for simplicity
		// ℋ: r₀⟨⟩; Γ: self: r₀ IsoPair<T>; Φ: ∅
		// ↑ explore self.left ↓
		// ℋ: r₀⟨self[left ↣ r₁]⟩, r₁⟨⟩; Γ: self: r₀ IsoPair<T>; Φ: ∅
		async let l_handle = l_server.process(left)
		// ℋ: r₀⟨self[left ↣ r₁]⟩; Γ: self: r₀ IsoPair<T>; Φ: l_handle: (r₁⟨⟩, ∅, ∅)
		// ↑ explore self.right ↓
		// ℋ: r₀⟨self[left ↣ r₁, right ↣ r₂]⟩, r₂⟨⟩; Γ: self: r₀ IsoPair<T>; Φ: l_handle: (r₁⟨⟩, ∅, ∅)
		async let r_handle = r_server.process(right)
		// ℋ: r₀⟨self[left ↣ r₁, right ↣ r₂]⟩; Γ: self: r₀ IsoPair<T>; Φ: l_handle: (r₁⟨⟩, ∅, ∅), r_handle: (r₂⟨⟩, ∅, ∅)
		...
		//perform other work... note neither `self.left` or `self.right` is accessible here, but `self` and its other fields are
		//also... the function can't return here!
		...
		await l_handle
		// ℋ: r₀⟨self[left ↣ r₁, right ↣ r₂]⟩, r₁⟨⟩; Γ: self: r₀ IsoPair<T>; Φ: r_handle: (r₂⟨⟩, ∅, ∅)
		await r_handle
		// ℋ: r₀⟨self[left ↣ r₁, right ↣ r₂]⟩, r₁⟨⟩, r₂⟨⟩; Γ: self: r₀ IsoPair<T>; Φ: ∅
		// ↑ retract self.right, retract self.left ↓
		// ℋ: r₀⟨⟩; Γ: self: r₀ IsoPair<T>; Φ: ∅
		//function can return!
	}
```	

## Function Signatures

> incomplete section

In this section, discuss the simple function signatures likely to be used in the implementation of the fearless types system. Namely, parameters' regions can be preserved or consumed, and the results can come from fresh or existing regions. Also discuss the possibility of functions that take pinned regions for increased expressiveness at the call site and decresed expressiveness within the function body.


## Function Calls

> incomplete section

In this section, talk about how `ℋ` has to be massaged to match the function signatures in order to perform calls. Ideally, we relax the extent to which this massaging is necessary for standard classes of signatures.

## Branch Unification

> incomplete section

In this section, talk about "the hard part" of typechecking - branch unification. Basically, all other inference tasks are syntax-directed, but unifying branches correctly involved nonlocal reasoning and possibly backtracking. Discuss the fact that heuristics work to prevent backtracking on all example code written so far, but at large scale alternatives might need to be discussed.


## Programming with Concurrency and Futures

> incomplete section 

In this section, explain that just starting to use `async` functions with direct `await` calls is no problem - it's typed the same as a regular function invocation - but with futures we need to "move parts of ℋ" into a separate context to indicate that they're gone but they'll be back. Investigate ways to make this ergonomic.


## Design Decisions to be Made

### Should fancier function signatures be exposed?

### Exposing assertions about `ℋ`

This can be good for documenting isolation reasoning, and possibly hinting to the typechecker if for whatever reason inference hits a pathological case.

### Alternative names for "isolated"

Swift already uses the word "isolated" in the context of "actor-isolated" storage. Perhaps a distinct name would be easier. Here are some ideas:

* Region
* Boundary
* Bounding
* Separable
* Escaping
* Tracked
* Domain
