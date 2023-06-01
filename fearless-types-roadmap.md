# Roadmap for the Introduction of Fearless Types to Swift

## Ordered tasks

* **1**: **Add AST node** indicating sites at which values of non-*sendable* types are sent between concurrency domains. **Emit diagnostics** for each node suring SILGen.
* **2**: **Add SIL instruction** at code sites that encounter above AST node. **Emit diagnostics** in a **new mandatory SIL pass** that removes the instructions.
* **3**: **Perform typechecking** with fearless types to identify safe concurrency. **Add a mandatory SIL pass** before the diagnostic pass that removes above instructions corresponding to safe concurrency.
  * **a**: Perform typechecking in the "big regions" style: no `iso` keyword, all functions use a monoregion (single region for all non-*sendable* arguments and results).
  * **b**: Perform typechecking in the "small regions" style: *introduce `iso` keyword*, implement `ℋ`, add minimal annotations (grouping, preserves/consumes) on function signatures. Very lightweight unification - likely tight timeout. Possibly add an unsafe `if-disconnected`. 
  * **c**: Implement better unification.
  * **c'**: Implement fancier function signatures (pinnedness, pre-focus/pre-explore)
  * **c''**: Implement and integrate `if-disconnected` runtime component

## Motivating examples

### For task **3a**

The following two examples are currently disallowed by sendable checking, but are trivially safe, and would be allowed by an implementation of task **3a** above.

```swift
var a = 10
let t = Task {
	print(a)
}
```

```swift
var a = 10
async let b = a
await b
print(a)
```
