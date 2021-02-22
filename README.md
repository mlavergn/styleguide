# Style Guides

## Foreword

These guides is the culmination of best practices gleaned over decades experience at technology companies including Oracle, Apple, and Disney. Having participated on projects in C / C++ / SQL / PL/SQL / Java / Python / JavaScript / Go / TypeScript / Objective-C and Swift, for user bases ranging from dozens to over a billion, running on hardware ranging from mobiles to mainframes. We have first hand experience with an endless variety of coding styles, development strategies, and organizational requirements.

These documents will not be a one-size-fits-all; legacy code, language paradigms, organizational constraints, development team makeup, all preclude the existence of a single strategy. Rather, these guides serve as a blueprint for what wd have seen work best in most scenarios.

## GRUNT Principals

*Readable means reliable - Rob Pike*

Following the recommendations presented should result in code that is:

* [Grokable](https://en.wikipedia.org/wiki/Grok)
* [Refactorable](https://en.wikipedia.org/wiki/Code_refactoring)
* [Unit Testable](https://en.wikipedia.org/wiki/Software_testability)
* [Normalized](https://en.wikipedia.org/wiki/Database_normalization)
* [Tunable](https://en.wikipedia.org/wiki/Performance_tuning)

These are the GRUNT principals and are the characteristics of future-proof code. No code exists forever, it will change over time, but code exhibiting these qualities has an innate capacity to evolve and adapt, while code which lacking these will attract technical debt and eventually become obsolete. The overarching reason for this future-proof-ness is simplicity. However, simplicity is not equivalent to simple. Simplicity means programming an unwavering focus on architectural coherence. 

### Grokable

Grokable is measure of understandability or cognitive complexity. Under GRUNT, complexity is considered to be use of opaque constructs. Opaque constructs hinder readability by hiding implementation details. The GRUNT principal is that it should be immediately apparent what a code block does and how it does it.

### Refactorable

In software engineering, minimal object graphs with minimal state are inherently stable and malleable, while broad object graphs with broad dependencies are inherently unstable and rigid. This is a universal truth that we have encountered this pattern countless times. GRUNT eschews inheritance in favor of encapsulation. Object trees should be shallow to minimize the levels of indirection. 

### Unitized Logic

Under GRUNT, unit-logic is an expression of the functional units should be logical units. In other words, functions are to be designed as input-output or event-reaction units. State modification units are highly discouraged. It is impossible to design applications without state modification units, the main function is a canonical example of a state modification unit.

### Normalized

Normalization is a single source of truth (SSOT) construct that applies to the object graph. GRUNT aims to achieve object normalization with the goal of each piece of information belonging to a single unit. 

### Tunable

As we approach the limits of the nm scale, we are witnessing the inevitable end of Moore's Law. Cloud computing delayed the need for focus on performance tuning, but horizontal scale is reaching it's limits as the cost of distributing increasingly expensive tasks runs into diminishing returns. Performance tuning will become an ever increasing factor in software development. GRUNT enables tunability via it's granular approach to segmenting logic. One of the primary challenges in performance tuning is isolating bottlenecks and relieving bottlenecks without requiring architectural changes to applications. GRUNT leverages encapsulation, SSOT, unitized logic, and shallow object graphs to enable targeted optimization efforts.

## MSCV Architecture

GRUNT should be paired with the 4-tier Model-Service-Coordinator-View (MSCV) architecture. 

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

MSCV is simply a localized abstraction of the universal real world application use pattern. A more familiar representation would be:

Disk <- Server <- Device -> User 

This is a need-to-know (N2K) based architecture. Models and views do not know about anything other than models and views respectively. Services only know about models and other services. Coordinators know about all layers. Views only know about other views while exposing an I/O protocol. This pattern is designed to impede the growth of the object graph and disambiguate scopes of responsibility. The clear delineations on which objects can talk to which objects make attempts to embed service, model, or view functionality in the coordinator is highly discoverable since it must be access with xModel::x, xService::x, or xView::x prefixing.

## Terminology

We reference some  terminology in my arguments, here's a legend of what they represent:

- bug surface:
  - code in which bugs could potentially exist

- fixtures
  - data that mimics device I/O (eg. network, disc, memory)

- mocks
  - fake object instances that mimic real object instances

- [bit rot](https://en.wikipedia.org/wiki/Software_rot)

- [technical debt](https://en.wikipedia.org/wiki/Technical_debt))

- cognitive complexity
  - size of the call stack and branch permutations when manually reading source code

- object graph
  - the combined depth of inheritance chains (ie. class, subclasses) and parameterized inheritance (ie. function taking a class as a parameter)

## Recommendations

The following are broad generic guidelines:

* Simplicity bests complexity
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
