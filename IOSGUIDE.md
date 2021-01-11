# Style Guide

The style guide establishes conventions intended to keep the code base uniform and achieve world-class reliability and performance

## Mission

Our mission is to provide a crash and bug free guest experience, in a well tuned, public SDK level quality, highly adaptable  code base

## Preface

The long term goal is to migrate our projects to SwiftUI / Combine pub/sub pattern

New code should be written in MVC-style or SwiftUI-style Swift. There remains legacy code in Objective-C. There remains legacy code in CleanSwift. There remains legacy code that violates the core principals of the project. Legacy code should be rewritten when a significant change (eg. large sections of 20+ lines) to that code is needed.

Follow the optimization guidelines from the [Apple Swift Project](https://github.com/apple/swift/blob/main/docs/OptimizationTips.rst).

Follow recommendations from the [Google Swift Style Guide](https://google.github.io/swift/) with precedence to the guidelines in this document

## Principals

We follow a function programming inspired approach to logic where state is minimized and input-output is preferred. We seek to avoid any side-effects which can lead to unpredictable behavior. Specifically, we follow some tenets of functional programming, but do not the pure functional style. These tenets are represented by the Targets (TGTS) paradigm. All code for Studio Progress should adhere to TGTS.

Targets (TGTS) paradigm:

* Terse - Do one thing well
  * Short, simple code blocks decrease bug surface (amount of code in which bugs can occur) and increase developer productivity by being easier to read and decompose (low cognitive overhead).

* Grokable - Context needed to walk though logic should be self-contained
  * Language features that reduce transparency by introducing opaque mechanisms should be avoided. (eg. inheritance, polymorphism, extensions on 3rd party classes). We strive to create logic that is simple and easily identifiable. It is better to duplicate some code than to rely on abstractions.

* Testable - Each function should have static input and static output
  * Functions should be single-purpose and atomically testable. Tests should not require pulling state from additional context or relying on mock implementations to mimic state.

* Static Types - Input and output types should be static
  * Generic and Any types should be avoided to minimize bug surface and unintentional behaviors. Generics are seldom required and exponentially increase bug surface and test requirements.

Following this paradigm will result in a code base with the following qualities:

* small, simple, logic routines
* low bug surface
* high testability
* minimal side-effects
* minimal cognitive overhead
* high adaptability

## Formatting

* use explicit "self" to disambiguate ownership calling methods/properties within an object
* use "Self" (camel-case) when referring to static methods within an object
* write comments for any method that is not trivially simple (eg. setter / getter)
* use the 'triple-slash ///' pattern for single line comments
* @objc decorators should be placed on the line above the method decorators
* avoid marking APIs public unless intended to be consumed externally
* in general one class or struct per file, but enums and nested structs are exceptions

## Patterns

### Stateless

Prefer static members

Stateless implementations are simpler to test and generate predictable results. Avoid global state in stateless methods, unless necessary

Where possible, member variables should be `static let`. This is a thread safe lazy inits that the compiler can optimize. Beware of `static var` variables since these lack thread safety on init

Generally, stateless code is marked by `struct` or `enum` objects rather than `class`. Preference should always be given to struct / enum unless there is a specific need to use a **mutable object with multiple references**, such as a cache or datastore.

In cases where Swift code must be exposed to Objective-C, classes must be used, those class should be declared final and all methods made static. These classes should be convertible to a Swift struct by little beyond changing the base type.

### Views

All new view code must be explicitly coded, no new Storyboards will be accepted to the project.

Transitions should be accomplished using UIViewController show / present and UINavigationController push APIs

Minor modifications to existing storyboards are permitted

### Exceptions

Never throw an exception or pass through unhandled exceptions

Exceptions require immediate processing which is fundamentally incompatible with reactive programming's observer pattern

There is no defensible use case for declaring exceptions in Swift

### Inheritance

Avoid inheritance

Inheritance is an opaque API contract when looking at the subclass code and makes refactoring subclasses brittle. Inheritance is rarely required and Swift protocols should be preferred

New classes should be marked as final to enforce this style, or defined as structs if they carry no state

An exception is UIKit constructs which are (currently) reliant on subclassing

### Extensions

Avoid extensions

Extensions introduce opaque forced API contracts and create refactoring barriers. Inline code or encapsulation should be preferred in most use cases (eg. MyColor.default vs UIColor.myDefaultColor)

Extensions are not opt-in and reserve namespace, leading to scenarios where unintended code gets executed or callers are forced to execute a piece of unwanted code

Extensions may be acceptable where a 3rd party implementation does not expose a public initializer or is final

Extensions against Foundation or UIKit constructs are only defensible to provide API compatibility, not for utility methods

### Generics

Avoid generics

Beyond Foundation types, generic implementations are rarely required. Generics increase bug surface and exponentially complicate testing

Generics are an acceptable means of avoiding inheritance based conformance

### Protocols

Use protocols when multiple types need to satisfy an API contract

Concrete types are safer due to explicit binding, but protocols are the preferred solution when multiple implementations need to be supported

Protocols are simply contracts, they do not add bug surface and are not antithetical with stateless-ness

## MVC

Follow MVC paradigms, fully separate UI code from non-UI code. There is legacy Clean Swift code that is slated to be removed. Do not use Clean Swift or VIPER type patterns. These approaches are incompatible with the observer pattern and result in redundant bridge patterns. Use basic MVC keeping views and controller logic minimal.

Specifically, strive for a UNIX paradigm where all the logic operates as a command line application, with a UI bolted onto the CLI. No model or business logic should exist in the UI. Ideally, this pattern will result in 4 file types:

1) MyView.swift - View objects (View == rendering)
2) MyCoordinator.swift - View IO layer (View <-> Service bridge) [aka ViewController pre-SwiftUI]
3) MyService.swift - Data IO layer (Service == data logic)
4) MyModel.swift - Data objects (Codable == serialization)

### UI

* View files (*View.swift)
  * layout logic (ie. constraints)
  * run on main thread (mandatory)
  * view attribute setters
  * accessibility setters

* Coordinator files (*Coordinator.swift)
  * minimal IO proxy between view and service
  * bridge main and background threads
  * UI delegate target
  * UI event handling (ie. animation)

### Non-UI

* Service files (*Service.swift)
  * avoid references to UIKit classes (exceptions are non-UI objects such images and colors)
  * most likely be the lengthiest of the 3 components
  * run on background threads
  * CRUD layer for model I/O
  * enforces business logic

* Model files (*Model.swift)
  * avoid references to UIKit classes (exceptions are non-UI objects such images and colors)
  * exists to expose abstract data as a Codable object

## Tests

Avoid mock classes

Mock classes do not sufficiently exercise the target API. Objects should be designed with mock "gates" where necessary or simply make use of the existing mock gate on the network handler class to prevent wire hits.

### Coverage

The following guidelines should be observed

* unit tests
  * defines test coverage for the project
  * *only* tests model files (XCTest)
  * given input A, output B is expected

* functional tests
  * provides testing of the application UX flows
  * controller files (XCUITest)
  * given user activity set A, app state B is expected

* wireframe tests
  * tests rendering fidelity for a given screen
  * functional test active screenshot compared to a reference screenshot
  * given app state A, screen rendering B is expected

## Project Structure

### Code

New code should be placed into one of the following groups

* Standalone
  * objects that emulate 3rd party functionality
  * enables for isolated horizontal testing of components
  * never used outside of example / test purposes

* API
  * objects that expose APIs used by integrators
  * should have full documentation

* Core
  * all non-UI production code
  * "Foundation" type code

* UI
  * all view controllers
  * all views
  * "UIKIt" type code

* Storyboards
  * *.storyboard files exclusively

* FunctionalTests
  * XCUI objects exclusively

### Unit Tests

Test files should be in the same folder as the the code they test using a Test-suffix naming convention

* MyFolder
  * MyWork.swift
  * MyWorkTest.swift
  * MyTask.swift
  * MyTaskTest.swift
  * ...

Tests should go "all the way to the wire". Network access has a logic gate Self.isMock that mocks network I/O. Unit tests should test the full stack used by the function they are testing, but stop short of triggering state changes or performing actual network requests. Several classes that wrap access to resources have Self.isMock gates. Currently, these gates are static members, so testing concurrency requires care when muxing responses.
