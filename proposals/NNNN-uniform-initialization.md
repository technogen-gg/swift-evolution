# Uniform Initialization

* Proposal: [SE-NNNN](NNNN-uniform-initialization.md)
* Authors: [Gor Gyolchanyan](https://github.com/technogen-gg)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

This proposal seeks to unify the logic of initializers and add support for indirect initializers (a.k.a factory initializers).

Swift-evolution thread: [\[Proposal\] Uniform Initialization Syntax](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170605/037128.html)

## Motivation

Currently, there are two general categories of initializers: *failable* and *non-failable*. Initializers annotated as `convenience` or `required` or are not considered their own category because they don't affect the signature and largely serve type-safety and performance purposes. Initializers marked as `throws` are uniformly applicable to all types of initializers, so they are also not considered as their own category.

In non-failable initializers, these are all the ways to initialize an instance:

* Value types (`struct`, `enum`):
	* Assign a value to each member (exactly once for `let` members, one or more times for `var` members)(except optionals).
	* Assign a value to `self` (exactly once if at least one `let` member 	exists, otherwise one of more times).
	* Make a delegating initializer call (`self.init(...)`) exactly once.
* Reference types (`class`):
	* Assign a value to each member exactly once (except optionals) and make a superclass initializer call exactly once.
	* Mark the initializer as `convenience` and make a delegating initializer call (`self.init(...)`) exactly once.

In failable initializers, there is an additional way to finish the initialization:

* Return `nil` from the initializer at any point to cause the initializer to fail immediately.

This behavior of initializers is counter-intuitive because:

* On one hand, initializers are conceptually supposed to bring an already existing object in memory into a type-specific valid state and are not perceived as *returning* anything, so returning `nil` seems like a confusing hack aimed at solving a specific use case.
* On the other hand, returning from an initializer can be seen as a convenient way of initializing the instance and returning immediately, in which case inability to return a non-`nil` instance from an initializer seems unjustified.

Another problem is a powerful feature of Objective-C is unavailable in Swift, which allows one to return an arbitrary instance of the class from its initializer, enabling powerful design patterns, such as [abstract factory](https://en.wikipedia.org/wiki/Abstract_factory_pattern), [factory method](https://en.wikipedia.org/wiki/Factory_method_pattern) and [singleton](https://en.wikipedia.org/wiki/Singleton_pattern).

Even though such initializers in Objective-C, as well as C functions decorated with `__attribute__((swift_name("MyType.init(self:...)")))` are imported into Swift, they are imported as normal initializers, which break all expectations and rules enforced by the compiler in Swift code and make the initialization difficult to reason about, especially the latter case, where the return type of the function doesn't even have to be related to the type being initialized, which would result in odd (and potentially compiler-crashing) behavior.

## Proposed solution

1. Allow value types to return an instance of the type from initializers, in addition to returning `nil` from failable initializers.
2. Introduce a new `indirect` initializer type, which only allows returning the instance as part of initialization, also allowing any compatible type to be returned instead (conforming types for initializers in protocol extensions, subclass instances for class initializers).

### Examples

Given:

```swift
protocol MyProtocol {

	func myAlgorithm()

}

private struct MyProtocolSafeImpl: MyProtocol {

	func myAlgorithm() {
		// Safe but slow implementation of my algorithm.
	}
	
}

private struct MyProtocolFastImpl: MyProtocol {

	func myAlgorithm() {
		// Fast but unsafe implementation of my algorithm.
	}	

}
```

Before:

```swift
extension Bool {

	init?(exactly value: Int) {
		// A full-fledged switch or an if-tree is unavoidable.
		switch value {
			case 0: self = false
			case 1: self = true
			default: return nil
		}
	}

}

extension MyProtocol {
	
	// Verbose, ugly, unable satisfy `init` protocol requirements.
	static func create(optimized: Bool) -> MyProtocol {
		return optimized ? MyProtocolFastImpl() : MyProtocolSafeImpl()
	}

}

let instance = MyProtocol.create(optimized: true) // Unintuitive and wonky way of creating instances.

```

After:

```swift
extension Bool {

	init?(_ value: Int) {
		return value == 0 ? false : value == 1 ? true : nil // A compact, single-line initializer.
	}

}

extension MyProtocol {
	
	/// Compact, beautiful and able to satisfy `init` protocol requirements.
	indirect init(optimized: Bool) {
		return optimized ? MyProtocolFastImpl() : MyProtocolSafeImpl()
	}

}

let instance = MyProtocol(optimized: true) // Intuitive, transparently decoupled implementation.

```

## Detailed design

This proposal demonstrates a conceptually equivalent function to all initializer categories, specifying the type of the `self` parameter (if any), and the return type.

The following is the list of initializer categories and their conceptual function equivalents:

| Initializer | Function |
|---|---|
| `init(...)` | `static func __init(_ __self: inout Self, ...) -> Self` |
| `init?(...)` | `static func __init(_ __self: inout Self, ...) -> Self?` |
| `indirect init(...)` | `static func __init(...) -> Result` |
| `indirect init?(...)` | `static func __init(...) -> Result?` |

In all `__init` functions above, the following rules will apply to corresponding initializers:

* `Result` is the type in which the function is defined.
* If there is a `__self` parameter, the compiler:
	* Replaces all non-empty non-`nil` `return ...` statements with `__self = ...; return`.
    * Replaces all empty `return` statements (including the implicit one at the end of the function) with `return __self`.
	* Requires:
		* Either assigning to all non-static `let` constants of `__self` exactly once and assigning to all non-static `var` stored properties at least once.
		* Or assigning to `__self` exactly once if there is at least one non-static `let` variable in `__self`, or at least once if there are no non-static `let` constants in `__self`. 
		* Or calling another `__init` functions that has a `__self` parameter, by passing it `&__self`.

#### Interoperability with the C family of languages

4. Add new mutually exclusive attributes  `__attribute__((swift_direct_init_func))` and `__attribute__((swift_indirect_init_func))` to the C family of languages that is only applicable to functions decorated with `__attribute__((swift_name("MyType.init(self:...)")))` and Objective-C initializer methods.
6. Import all C functions decorated with `__attribute__((swift_direct_init_func)))` as `init`, while emitting a compile-time error during import if the `self` parameter is not a mutable pointer of the exact Swift type and return type is not `void`.
7. Import all C functions not decorated with `__attribute__((swift_direct_init_func))` or decorated with `__attribute__((swift_indirect_init_func)))` as `indirect init`, while emitting a compile-time error during import if a `self` parameter is specified and return type is not a subtype of the Swift type (see details below).
8. Import all Objective-C initializer methods not decorated with `__attribute__((swift_direct_init_func))` or decorated with `__attribute__((swift_indirect_init_func)))` as well as convenience factory methods as `indirect init`.
9. Import all Objective-C initializer methods decorated with `__attribute__((swift_direct_init_func)))` as `init`, while emitting a runtime error if the dynamic type of the returned instance does not match the static type of the class.

## Source compatibility

This proposal is mostly additive (with a single very rare exception). The most notable compatibility considerations are:

* All existing C-family initializers (lacking annotations) will be imported as `indirect init`, which is exactly how they behaved before, except the initializer will now be appropriately annotated to reflect the nature of the imported initializer and no behavioral change (except the aforementioned run-time error on newly annotated initializer methods) will take place.
* C functions with mismatching Swift type and return type will fail to be imported into Swift, however these cases are very rare and are fundamentally erroneous, so they should never have been accepted in the first place.
* All existing swift initializers will compile and behave as before.

### Swift 3 compatibility mode

In Swift 3 compatibility mode, the possible C compile-time error and the Objective-C runtime error can be replaced with a warning.

### Migration

No migration is necessary, because the new syntax and its associative 
behavior is purely additive.

## Effect on ABI stability

This proposal could be implemented without affecting the ABI, however the existence of `indirect init` can provide static typing guarantees about non-indirect `init`, opening up optimization opportunities on the ABI level.

## Effect on API resilience

This proposal introduces changes to the initialization model, which even for public APIs does not change the syntax or behavior of existing code in any way, so any changes to the public API as a result of utilizing the features of this proposal are completely voluntary, so no Swift version checking is necessary.

## Alternatives considered

Forego the additions and keep the initialization logic the way it is.
