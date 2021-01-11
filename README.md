# Style Guides

## Foreword

The style guides in this project represent best practices acquired over 25 years of development experience at some of the most innovative technology companies in the world, including [Oracle](https://en.wikipedia.org/wiki/Oracle_Corporation) (SQL databases), [Openwave](https://en.wikipedia.org/wiki/Openwave) (Mobile Internet gateways), [Apple](https://en.wikipedia.org/wiki/Apple_Inc.) (Mac / iPhone), and [Disney](https://en.wikipedia.org/wiki/The_Walt_Disney_Company) (Streaming media / Theme parks).

Having participated in projects written in C / C++ / SQL / PL/SQL / Java / Python / JavaScript / Go / TypeScript / Objective-C and Swift, for clients ranging from 50 employees, to multinationals, stock exchanges, and governmental agencies, I have professionally encountered a wide variety of coding styles, development strategies, and organizational requirements.

These documents are in no way a one-size-fits-all, legacy code and organizational constraints preclude the existence of such a strategy. Those claiming to have universal strategies have not accumulated enough experience to fully grasp the scale and variety of development across the spectrum organizations. I continue to discover new constraints after spending a virtual lifetime in the field.

## Overview

These documents serve as a guideline for what works best in most scenarios. Following these guidelines should result in code that is:

* [Grokable](https://en.wikipedia.org/wiki/Grok)
* [Refactorable](https://en.wikipedia.org/wiki/Code_refactoring)
* [Normalized](https://en.wikipedia.org/wiki/Database_normalization)
* [Tunable](https://en.wikipedia.org/wiki/Performance_tuning)
* [Testable](https://en.wikipedia.org/wiki/Software_testability)

These characteristics are the trademarks of future-proof code. No code exists forever, it will change over time, but code exhibiting these 5 qualities is imbued with the capacity to evolve and adapt, while code which does not will have to be discarded as technology or requirements evolve.

## Recommendations

There are many inspirations for these recommendations, I did not invent anything here, much of the inspiration draws from UNIX system programming. The following is a summary of the high level recommendations:

* Simplicity is better than complexity
    If you cannot explain what a code block does does in 1-2 sentences, simplify it.
* Self-documenting code is only self-documenting to the author and only for a short while after writing it
    Comments, even wrong / old / incomplete comments, convey more information than self-documenting code. Self-documenting code only explains how, it does not describe why, when, or what. Humans are not compilers, developers rarely going to have the time or inclination to parse your code to understand what your code does, why it does it, and when it's supposed to do it.
* Write logic as though you are writing a command line program
    Assume that the logic is a server API. Isolate logic is easier to test and adapt since there is not UI impact.
* Write UI as though you are wrapping a command line program
    Assume that the UI cannot manipulate data.
* Static functions have less bug surface than non-static functions
    Static functions add friction when accessing non-local state which highlights state externalities
* Functions should be considered atomic, assume nothing is thread safe.
    Keep functions small, get in and out quick, assume a thread will enter your function and another is locked waiting for you to return
* Encapsulation and abstraction help maintainability, inheritance and polymorphism hurt maintainability
    Constructs that isolate reduce lock-in, constructs that inherit increase lock-in.
* Avoid opaque programming constructs
    Untyped data, base object types, inheritance, casting, any language construct which hides implementation details.
* Follow the recommended patterns by the platform maintainers.
    Do not fight the recommendations of the platform maintainers (ie. Apple, Google, Microsoft). In the long term, the platform maintainer is *always* going to be right. If a pattern leads to a poor outcome, you are almost certainly the cause, not the pattern.
* Test call chains, not functions
    Unit tests should test to the apex of the stack (aka. "to the wire"), not just the first logic layer. If a function relies on a deep call stack, test all the logic layers in that stack to the point just prior to an externality occurring.

    > Bug Surface: I'm not sure when I acquired this term, I don't believe I coined it, but bug surface describes code in which bugs could potentially exist. In low-level terms, it is a code section which contains branching or access to non-local state.

## Summary

There is no silver bullet in programming, but there are approaches that consistently lead to favorable outcomes. I certainly haven't always follow these guidelines, my GitHub account contains many projects that are a testament to that. However, I can say with a high degree of certainty is that when these guidelines are followed, the code bases that result are easier to onboard developers on, have fewer bugs, exhibit flexible performance tuning opportunities, have lower levels of [bit rot](https://en.wikipedia.org/wiki/Software_rot), and resist the accumulation of [https://en.wikipedia.org/wiki/Technical_debt](technical debt).

Keep things simple, do one thing at a time, do not hide implementation, leverage everything the compiler has to offer, and keep cognitive overhead low so developers can focus on the task they're working on.
