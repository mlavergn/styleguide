# Style Guides

## Foreword

These guides represent best practices refined over decades experience at top technology companies including Oracle, Apple, and Disney. Having participated on projects in C / C++ / SQL / PL/SQL / Java / Python / JavaScript / Go / TypeScript / Objective-C and Swift, for user bases ranging from dozens to over a billion, running on hardware ranging from mobiles to mainframes. We have professionally encountered an endless variety of coding styles, development strategies, and organizational requirements.

These documents will not be a one-size-fits-all; legacy code, language paradigms, organizational constraints, development team makeup, all preclude the existence of a single strategy. Rather, these guides serve as a blueprint for what wd have seen work best in most scenarios.

## GRUNT Principal

Following the recommendations presented should result in code that is:

* [Grokable](https://en.wikipedia.org/wiki/Grok)
* [Refactorable](https://en.wikipedia.org/wiki/Code_refactoring)
* [Unit Testable](https://en.wikipedia.org/wiki/Software_testability)
* [Normalized](https://en.wikipedia.org/wiki/Database_normalization)
* [Tunable](https://en.wikipedia.org/wiki/Performance_tuning)

These as the GRUNT principals and are the characteristics of future-proof code. No code exists forever, it will change over time, but code exhibiting these qualities has an innate capacity to evolve and adapt, while code which lacks any of these will attract technical debt and eventually become obsolete. 

The golden rule of software development is minimal object graphs with minimal state are inherently stable and malleable, while broad object graphs with broad dependencies are inherently unstable and rigid. We have encountered this pattern countless times; complexity hastens obsolescence.

## Other Principals

### SOLID

We agree with single responsibility and interface segregation, this is the "normalized" aspect of GRUNT, but the remaining [SOLID](https://en.wikipedia.org/wiki/SOLID) recommendations are magnets for technical debt.

### YAGNI

We agree with the simplicity focus of [YAGNI](https://en.wikipedia.org/wiki/YAGNI), but the "continuous refactoring" assumption never holds in practice. GRUNT places focuses on single responsibility which would typically not happen under YAGNI approaches leading to duplicated code.

## MSCV Architecture

GRUNT recommends using the 4-tier MSCV architecture:

* Model: I/O unit representation including serialization logic
  - dependencies: other models
* Service: Manages model layer I/O to a service provider (ie. network, disk)
  - dependencies: models + other services
* Coordinator: Manages view layer I/O between the service layer and scenes (ie. screen, glasses, printer)
  - dependencies: views + services + other coordinators
* View: UI definition including layout logic
  - dependencies: other views

Visually, this can be represented as:

```text
MODEL <- SERVICE <- COORDINATOR -> VIEW
```

This is a "need-to-know" based architecture, model and views do not know about anything other than models and views respectively. Services only know about models and other services. Only coordinators know about all layers. This pattern is designed to impede inheritance and limit creep of the object graph. The scopes of responsibility are clearly delineated and against-pattern attempts to embed service, model, or view functionality in the coordinator is highly discoverable since it must be access with xModel::x, xService::x, or xView::x prefixing.

## Other Architecture

### MVVM

MVVM and it's related approaches attempt to create a unidirectional flow of logic. The concept has merit since it attempts to limit stateful changes, but the implementation explodes the object graph and vastly increases cognitive complexity. This violates all of GRUNT with the possible exception of testing, as such we discourage using MVVM.

## Terminology

We reference some  terminology in my arguments, here's a legend of what they represent:

- bug surface:
  - code in which bugs could potentially exist

- fixtures
  - data that mimics device I/O (eg. network, disc, memory)

- mocks
  - fake object instances that mimic real object instances

- grokability
  - how well does a block of code express it's purpose at a glance

- future-proof
  - code that is unlikely to require changes when modifying application behavior or adding features

- normalized
  - the breaking down of logical units to the atomic, single purpose level

- [bit rot](https://en.wikipedia.org/wiki/Software_rot)

- [technical debt](https://en.wikipedia.org/wiki/Technical_debt))

- cognitive complexity
  - size of the call stack and branch permutations when manually reading source code

- object graph
  - the combined depth of inheritance chains (ie. class, subclasses) and parameterized inheritance (ie. function taking a class as a parameter)

## Recommendations

The following are broad generic guidelines:

* Simplicity beats complexity
    If you cannot explain what a code block does in 1-2 sentences, continue to simplify it.
* Self-documenting code is a myth, after a year, even the authors have mentally unravel their code 
    Developers are not compilers, and rarely have the time or inclination to parse your code to derive your intentions. Comments, even wrong / old / incomplete comments, convey information that code cannot. Your code documents the "how", your comments document the "why, when, and what"
* Write non-UI logic as though it was destined for a command line program
    Assume you are writing server side code. Request in, response out. Isolated logic is easier to test and adapt since it is not tied to a UI state chart
* Write UI as though you are wrapping a command line program
    Assume that the UI cannot manipulate data directly. The UI must talk to a "closed box" command line program.
* Static functions have less bug surface than non-static functions
    Static functions restrict accesses to (most) non-local state which highlights state externalities. They are also predictable such that tests can readily identify regressions
* Functions should be considered atomic, assume nothing is thread safe.
    Keep functions small, get in and out quick, assume a thread will enter your function and another is locked waiting for you to return
* Encapsulation and abstraction help maintainability, inheritance and polymorphism hinder maintainability
    Constructs that encapsulate functionality grow code paths linearly, constructs inherit functionality grow code paths exponentially.
* Avoid unspecialized object types
    Untyped data, base object types, inheritance, casting, any language construct which hides implementation details.
* Follow the patterns recommended by the platform owners
    Do not fight the recommendations of the platform maintainers (ie. Apple, Google, Microsoft). In the long term, the platform maintainer is *always* going to be right. If a pattern leads to a poor outcome, you are almost certainly the cause, not the pattern. The choices are to do it as recommended now, or invest effort in a "different" approach that will eventually have to be replaced with the recommended approach
* Test deep, not shallow
    Unit tests should test the call stack, not just the first call. If a function relies on a deep call stack, test all the logic layers to the point prior to an externality occurring. A real world example would be testing that an electric socket has 3 prongs, and is flush with the wall, but not testing if the socket is connected to then electrical panel.

## Summary

My GitHub account contains many projects that show me not always following these guidelines. It takes significant thought and planning to achieve simplicity and maintainability. If you are writing long term code that will be developed in a team setting, then it is worth expending additional effort to follow GRUNT and MSCV. Code bases that follow these guidelines will be easier to onboard developers on, have fewer bugs, be more flexible, have lower levels of bit rot, and resist the accumulation of technical debt.

Keep things simple, do one thing at a time, do not hide implementations, leverage the compiler, and keep cognitive overhead low so developers can focus on the task they're working on.
