# Contractual Disappointment in C++

## Introduction

[Handling Disappointment in C++](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0157r0.html)
observes that run-time errors are just one kind of disappointment
that [a C++ program](https://eel.is/c++draft/basic.link#def:program) might elicit.
I argue that all disappointment involves the violation of a contract
and that the correct strategy for handling a violation depends on two things:

1. the parameters of the contract, and
1. the usage profile of the program.

## Bugs and Errors

Section 4.2 of [P0709R4, Zero-overhead Deterministic Exceptions: Throwing values](https://wg21.link/p0709r4),
"Proposed cleanup: Don’t report logic errors using exceptions",
makes clear the distinction between errors and bugs.

Applying this distinction to the contracts above:

* A bug is the violation of a C++ API Contract or the ISO C++ Standard.
* An error is the violation of a End User Contract.

However, there is an exception to this rule
which applies to something called the Test User Contract.
When testing for contract violations,
the user wishes for bugs to behave far more like errors.
(I believe that this fact is important to understanding
why so much disagreement surrounds the C++ Contracts feature.)

## What Is a Contract?

For our purposes, a contract is an _agreement_ between a _user_, and
a _provider_ about the run-time behaviour of some or all of a program.
_Violation_ of a contract is what leads to disappointment.

For the sake of simplicity and focus,
I will concentrate on the obligations imposed on the user by the contract.
I believe that extending the principles laid out here
to obligations imposed on the provider is relatively straight-forward.

## What Are the Parameters of a Contract?

The following set of parameters -- expressed as questions -- are helpful in
understanding the correct error-handling strategy for a contract.

* Where, and in what form, is the agreement expressed?
* Who is the provider of the contract?
* Who is the user of the contract?
* What are the consequences of user contract violation?

## What Contracts Are There?

In a C++ program, some contracts that matter are as follows.

### End User Contract

The ultimate contract that a program must fulfil is to its end user.
All other contracts below are in support of this fulfilment.

* agreement: program documentation, possibly including a `--help` option
  or a 'man' page
* provider: the program author(s)
* user: the program user
* violation: user input is sanitized and ill-formed input is handled through
  normal control flow (including exception handling)

User input might include command-line parameters, UI interaction, data files.
Input identified as ill-formed must be rejected by the program.

Where possible rejection should be accompanied by meaningful feedback
which helps the user correct the input so as to increase the chance of future success.
For example, on a typical terminal-based system,
meaningful feedback might take the form of a diagnostic emitted on the error stream,
followed by exit with non-zero status.

Input that is not rejected must be incapable of causing violations
of any contracts mentioned below.
In particular, contract violations could result in security vulnerabilities.
For programs with exposure to malicious agents, this is critical.
For safety-critical applications, this is important.
Fuzz testing is a method which helps identify flaws in input sanitization.

### Test User Contract

During development, the program -- or portions of it --
may be built for testing purposes.

The program should be instrumented to test for violations of the contracts below.
As many violations as is practical should be checked.
Trapped violations of the contracts below should be treated like user errors
as described in the End User Contract above,
i.e. terminate with helpful diagnostic and non-zero status.

* agreement: varied, but may be enshrined in a project README, a wiki or project
  process document(s)
* provider: the program author(s)
* user: a test engineer who may be
  * an author of the program testing their work,
  * a test engineer testing the program for correct operation, or
  * a dev-ops engineer testing the program as part of a CI pipeline
* violation: If the program fails to identify a violation of one of the contracts
  below, this is considered a violation of the Test User Contract.
  A typical remedial step would be to fix an existing test or write a regression test.
  The contract is restored when the program successfully fails
  as a result of a violation of contract below.

Examples of contract trapping techniques that are commonly employed are

* API contract expressions (e.g. asserts, pre-conditions and post-conditions),
* API contract tests (e.g. unit tests), and
* ISO C++ Standard contracts (e.g. by enabling sanitizers
  and flags such as `_GLIBCXX_DEBUG` and `_ITERATOR_DEBUG_LEVEL`).

### ISO C++ Standard

A typical language specification for a C++ program is a revision of
[the ISO C++ Standard](https://isocpp.org/std/the-standard).

* agreement: a recognised set of standing documents including a C++ standard
  such as International Standard ISO/IEC 14882:2020(E), technical specifications,
  etc..
* provider: implementer of the toolchain used to build the program
* user: the program author(s)
* violation: undefined behaviour

Note: while this document focuses on run-time disappointment, authors are encouraged
to use static typing to surface defects earlier in the development process.

### Toolchain Contract

* agreement: toolchain documentation, including portions of the Language
  Specification identified as
  [implementation-defined behavior](https://eel.is/c++draft/defns.impl.defined)
* provider: implementer of the toolchain used to build the program
* user: the program author(s)
* violation: implementation-specific(?)

Note: being a collection of programs, the toolchain has its own End User Contracts.

### C++ API Contract

Contracts play an essential role in good API design.
Most C++ APIs center around functions:
expectations around their invocation,
details of the types required by those invocations, and
their supporting definitions and declarations.

It is helpful to subcategorise API contracts in descending order of desirability:

1. statically-enforceable;
1. dynamically-enforceable; and
1. manually-enforceable.

Violation of statically-enforceable API contracts, e.g.

```c++
double sqrt(double a);

void f()
{
  sqrt("4");
}
```

and violation of manually-enforceable API contracts, e.g.

```c++
double sqrt(double a);

// postconditions: returns x*x
auto square(double x)
{
  return sqrt(x);
}
```

Are outside the scope of this document.

#### Dynamically-Enforceable API Contracts

The provider should specify correct API usage.

Example violation:

```c++
// precondition: a is non-negative
double sqrt(double a);

void f()
{
  sqrt(-1);
}
```

* agreement: documentation, C++ Contracts (TBD)
* provider: program author(s)
* user: other program author(s)
* _enforcement_: assertions, e.g. [`assert`](https://en.cppreference.com/w/cpp/error/assert)
* violation: undefined behaviour (see discussion)

## Dynamically-Enforceable Contracts

Dynamically-Enforceable API Contracts and the ISO C++ Standard
form a group of contracts whose violation represents run-time bugs.
Of these, a subset can be expressed formally within the program
using assertions (including API preconditions and postconditions).

These expressions can be evaluated at run-time but typically incur a cost
which violates the zero-overhead principle in the case of a bug-free program.

Broadly, there are three possible strategies that can be adopted
at the point where an assertion would evaluate to `false`:

### Log-And-Continue Enforcement Strategy

The assert statement could emit a run-time diagnostic regarding the contract violation
and then continue past the assertion.

This is unwise: the program is known to exhibit undefined behaviour at this
point. If the provider were able to identify a reason why a violation might have
occurred they would have identified a bug. If the provider had identified a bug,
they should have fixed the bug. If the provider had fixed the bug,
they would no longer be able to identify a reason why a violation might have occurred.
Thus any violation is beyond the comprehension of the provider and "all bets are
off" regarding the behaviour of the program. This is essentially what UB means.

Nevertheless, under certain constraints

### Trap Enforcement Strategy

The assert statement can halt the program.

This has the disadvantage that the program completely fails to perform its task.
But the advantage is that no further incorrect behaviour
will be exhibited by the program.

### Prevention Enforcement Strategy

Assuming bugs are eliminated, no assert expression will ever be false.
This means that assert statements behave as if they are not there.
This in turn means that they can be empty or used as hints to the optimiser.

There is concern ([P2064](https://wg21.link/p2064r0)) that assuming no violations
is somehow incorrect. This is founded on the acceptance of bugs in production code
and the assumption that optimisers *cause* undefined behaviour. They do not. Bugs
cause undefined behaviour. If developers wish to release code with bugs to production,
they are welcome to disable such assumptions. But they are invited, instead, to follow
practices described as part of the Test User Contract which should help identify
such bugs early on.

***The author strongly encourages the use of assumptions in the implementation of
assertions in order to unify the set of Dynamically-Enforceable Contracts and to
simplify the implementation of the Test User Contract.***
In practice, this entails marking violated assertions as unreachable

```c++
#define ASSERT(cond) ((cond) ? static_cast<void>(0) : __builtin_unreachable())
```

which is semantically sound, given violations already must never occur in correctly-written
C++ programs.

This strategy provides the maximum leeway to support enforcement profiles (below)
and ensures that the existing ability of sanitizers is extended,
and that constant expressions containing bugs become ill-formed.

## Enforcement Profiles

The domain of the program greatly affects preferred enforcement strategy.

### Tester

Somebody who is testing the program, e.g.

* a developer investigating a bug,
* a QA/test engineer, or
* a DevOps engineer programming a CI runner to perform automated tests

will all prefer Trap Enforcement Strategy because bugs are a likelihood
and performance and stability are secondary concerns.

### Safety-Critical System With Redundancy

A safety-critical system *with* backup/redundancy will also prefer Trap Enforcement
Strategy because knowingly allowing the program to continue in an incorrect state
is an unacceptable risk.
A life-support system or the autopilot in an autonomous vehicle provide good examples.

### Safety-Critical System Without Redundancy

A safety-critical system *without* backup/redundancy
may prefer Log-And-Continue Enforcement Strategy.

### Performance-Critical/Resource-Constrained

Applications which are performance-critical or resource-constrained
may favour Prevention Enforcement Strategy.

There must be a high degree of confidence in the correctness of their program.
Much confidence can be attained through the Test User Contract.

Examples applications include multimedia software entertainment and embedded controllers.

### Business-Critical Systems

Where business interests are at stake but lives are not,
either the Trap Enforcement Strategy or Log-And-Continue Enforcement Strategy
may be preferable.
A RESTful server which persists between requests may eventually identify bad state
that has lain unnoticed for a while.
The developer can choose what happens on discovery:

* Do we kill the process and restart it in the hope that the problem goes away
  for a while, or
* do we 'soldier on' and hope the problem doesn't cause some catastrophe or other?

This choice will be affected by factors such as the scale of the fleet,
exposure to malicious agents and the cost of incorrect behaviour.

## Discussion

### Dynamically-Enforceable API Contract Violation is Undefined Behaviour

The popular idea that undefined behaviour is confined to violations of some subset
of the ISO C++ Standard is incorrect and harmful.

The standard [describes](https://eel.is/c++draft/defns.undefined) UB as:

> behavior for which this document imposes no requirements

In other words, the standard does *not* exclude from the definition of UB
violations of contracts outside of the ISO C++ Standard.

A more accurate and helpful view of UB is that it is one possible consequence of
a contract violation that occurs at run-time for which consequences are not described.
