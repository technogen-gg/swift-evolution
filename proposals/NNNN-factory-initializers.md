# Factory Initializers

* Proposal: [SE-NNNN](NNNN-factory-initializers.md)
* Authors: [Gor Gyolchanyan](https://github.com/technogen-gg)
* Review Manager: TBD
* Status: **Awaiting review**

## Introduction

This proposal seeks to introduce factory initializers (`factory init`).

Swift-evolution thread: [\[Proposal\] Uniform Initialization Syntax](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170605/037128.html)

## Motivation

The ability to return arbitrary instances form initializers has been successfully used on Objective-C for implementing various design patters (most notably, [singleton](https://en.wikipedia.org/wiki/Singleton_pattern), [factory method](https://en.wikipedia.org/wiki/Factory_method_pattern) and [abstract factory](https://en.wikipedia.org/wiki/Abstract_factory_pattern)). Sadly, this feature has been missing from Swift, making these patters difficult to implement and inconvenient to use.

## Proposed solution

* Allow marking initializers inside protocol extensions, class declarations and class extensions as `factory`.
* In initializers marked as `factory`:
	* Change the implicit `self` parameter to mean *the dynamic type of the enclosing type* (just like it does in static methods).
	* Disallow delegating initialization to initializers not marked as `factory`.
	* Require terminating the initializer by either returning a compatible type (a conforming type for protocols, a derived instance for classes) or returning `nil` (if the initializer is failable).
* In initializers inside enum declarations, enum extensions, struct declarations and struct extensions:
	* Allow terminating the initializer by returning an instance of the type being initialized.

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
	
	// Verbose, ugly, unable to satisfy `init` protocol requirements.
	static func create(optimized: Bool) -> MyProtocol {
		return optimized ? MyProtocolFastImpl() : MyProtocolSafeImpl()
	}

}

// Unintuitive and wonky way of creating instances.
let instance = MyProtocol.create(optimized: true)
```

After:

```swift
extension Bool {

	init?(_ value: Int) {
		// A compact, single-line initializer.
		return value == 0 ? false : value == 1 ? true : nil
	}

}

extension MyProtocol {
	
	/// Compact, beautiful and able to satisfy `init` protocol requirements.
	factory init(optimized: Bool) {
		return optimized ? MyProtocolFastImpl() : MyProtocolSafeImpl()
	}

}

// Intuitive, transparently decoupled implementation.
let instance = MyProtocol(optimized: true)
```

## Detailed design

The `factory` keyword, when applied to an initializer would change the underlying function that contains the initialization code, resulting in the following differences:

| Regular Initializer | Factory Initializer |
| --- | --- |
| Takes a pointer to allocated memory and sets it to a valid state for an instance of the type being initialized. | Returns a fully formed instance of the type being initialized. |
| Has an implicit `self` parameter that refers to the instance being initialized. | Has an implicit `self` parameter that refers to the dynamic type, on which the initializer is called. |
| Is backed by an instance method. | Is backed by a static method. |
| Can only result in an instance of the static type being initialized. | Can result in an instance of any subtype of the type being initialized. |

#### Interoperability with Objective-C

4. Add a new mutually exclusive attributes  `__attribute__((swift_factory_init(true)))` and `__attribute__((swift_factory_init(false)))` to Objective-C that is only applicable to Objective-C initializer methods.
8. Import all Objective-C initializer methods not decorated with one of these attributes or decorated with `__attribute__((swift_factory_init(true))))` as well as convenience instance-returning class methods as `factory init`.
9. Import all Objective-C initializer methods decorated with `__attribute__((swift_factory_init(false))))` as `init`, while emitting a runtime error if the dynamic type of the returned instance does not match the static type of the class being initialized.

## Source compatibility

This proposal is purely additive. The most notable compatibility considerations are:

* All existing Objective-C initializers (lacking annotations) will be imported as `factory init`, which is exactly how they behaved before, except the initializer will now be appropriately annotated to reflect the nature of the imported initializer and no behavioral change will take place.
* All existing swift initializers will compile and behave as before.
* No existing code will contain an Objective-C initializer decorated with `__attribute__((swift_factory_init(false))))`, so the runtime error may only occur from code developed with the features provided by this proposal, alleviating the need for compatibility mode changes.

### Migration

No migration is necessary, because all changes in this proposal are purely additive. 

## Effect on ABI stability

This proposal could be implemented without affecting the ABI, however the existence of factory initializers can provide static typing guarantees about non-factory initializer, opening up optimization opportunities on the ABI level.

## Effect on API resilience

This proposal introduces changes which are purely implementational and are not part of the API, while employing existing behavior of initializers imported from Objective-C.

## Alternatives considered

`indirect init` was considered instead of `factory init` for its lack of semantical connection to any specific use cases, but was ultimately dropped due to potential confusion with `indirect enum` and `indirect case`.
