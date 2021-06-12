# Contractual Disappointment in C++

## Introduction

[Handling Disappointment in C++](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0157r0.html)
observes that errors are just one kind of disappointment
that [a C++ program](https://eel.is/c++draft/basic.link#def:program) might elicit.
I argue that all disappointment involves the violation of a contract
and that the correct strategy for handling the violation depends on
the parameters of the contract.

## What Is a Contract?

For our purposes, a contract is
an _agreement_ between
a _user_, and
a _provider_.

## What Are the Parameters of a Contract?

The following set of parameters -- expressed as questions -- are helpful in
understanding the correct error-handling strategy for a contract.

* Where, and in what form, is the _agreement_ expressed?
* Who is the _provider_ of the contract?
* Who is the _user_ of the contract?
* What is the _enforcement_ of the contract?
* What are the consequences of user contract _violation_?

Note: we do not ask, "what are the consequences of provider contract violation?"
***The purpose of this document is to minimise provider contract violation.***
In doing so, the provider is best able to help the user to effectively satisfy the
contract.

Note: in most circumstances, the provider of a contract is also the author.
The most prominent exception is the ISO C++ Standard which is written by
[a committee](https://isocpp.org/std/the-committee).

## What Contracts Are There?

In a C++ program, some contracts that matter are...

### End User Contract

The contract between the user of a program and the implementer of the program is
an obvious one.
However, it is important that a program is built in accordance with other contracts
such as that between the program implementer and the toolchain implementer.

* _agreement_: documentation for the program, possibly including a `--help` option
  or a 'man' page
* _provider_: the program author(s)
* _user_: the program user
* _enforcement_: the implementer should sanitize input to the program
* _violation_: the program should emit helpful diagnostics if input lies outside
  agreed parameters. (On common hosted systems, the program should emit
  diagnostics on the error stream and exit with a non-zero status.)

### Test User Contract

During development, the program -- or portions of it --
may be built for testing purposes.

* _agreement_: varied, but may be enshrined in a project README, wiki or process
  document
* _provider_: the program author(s)
* _user_: a tester who may be
  * the implementer of the program testing their code locally,
  * a dev-ops engineer testing the program remotely as part of a CI pipeline, or
  * a test engineer testing their code locally
* _enforcement_: the program should be instrumented with checks of contracts including
  * API contract expressions (e.g. asserts, pre-conditions and post-conditions),
  * API contracts (i.e. unit tests), and
  * language contracts (e.g. by enabling sanitizers).
* _violation_: ***if the program fails to trap contract violations,
  the program is more likely to exhibit undefined behaviour and
  there is increased risk that the provider will violate the End User Contract***.

### ISO C++ Standard

A typical language specification for a C++ program is a revision of
[the ISO C++ Standard](https://isocpp.org/std/the-standard).

* _agreement_: a recognised set of standing documents including a C++ standard
  such as International Standard ISO/IEC 14882:2020(E), technical specifications,
  etc..
* _provider_: implementer of the toolchain used to build the program
* _user_: the program author(s)
* _enforcement_: optional, via dynamic analysers
* _violation_: undefined behaviour

Note: we are not interested in contract violations resulting in compiler diagnostics.
These are the best kind of violation: caught early and at no run-time cost.
The user is encouraged to work in such a way that static typing catches as many errors
as possible. But otherwise, this document has nothing to say on the matter.

Note that enforcement is optional. There are user violations of the language
standard on which the provider is not required to act.

Certain bugs may not be reliably detected without impeding portability or violating
the zero-overhead principle. When these occur, the behaviour of the program is undefined
and defects including security vulnerabilities may arise.

A program may be ill-formed in ways which are prohibitively difficult to diagnose.

Nevertheless, a toolchain is allowed to enforce such violations and often does.
***It is highly recommended that the user exploits the capabilities of the toolchain
to their maximum effect to enhance enforcement in the Test User Contract.***

### Toolchain Contract

* _agreement_: toolchain documentation, including portions of the Language
  Specification identified as
  [implementation-defined behavior](https://eel.is/c++draft/defns.impl.defined)
* _provider_: implementer of the toolchain used to build the program
* _user_: the program author(s)
* _enforcement_: the toolchain may be certified, but is often maintained through
  good SDE practices alone
* _violation_: emit a helpful diagnostic and not output a program

Observe that the toolchain has an End User Contract.

### C++ API Contract

Good API design is a broad subject beyond the scope of this document. But contracts
are an essential aspect of good API design.

Most C++ APIs center around functions: expectations around their invocation, details
of the types required by those invocations and their necessary definitions and declarations.

It is helpful to subcategorise API contracts in descending order of desirability:

1. statically-enforceable;
1. semantically-enforceable; and
1. dynamically-enforceable.

#### Statically-Enforceable API Contracts

Thankfully, static typing provides early identification of bugs caused by incorrect
uses of APIs. This should be considered the provider's earlier line of defence.

Example violation:

```c++
double sqrt(double a);

void f()
{
  sqrt("4");
}
```

* _agreement_: function declaration
* _provider_: program author(s)
* _user_: other program author(s)
* _enforcement_: use of function checked by tool-chain via static typing
* _violation_: diagnostic message (ill-formed), no program output

#### Semantically-Enforceable API Contracts

Some contract violations can only be detected by program authors.
Good API design can help to avoid such violations.

Example violation:

```c++
double sqrt(double a);

// postconditions: returns x*x
auto square(double x)
{
  return sqrt(x);
}
```

* _agreement_: documentation
* _provider_: program author(s)
* _user_: other program author(s)
* _enforcement_: careful naming, code review, automated testing
* _violation_: none

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

* _agreement_: documentation, C++ Contracts (TBD)
* _provider_: program author(s)
* _user_: other program author(s)
* _enforcement_: assertions, e.g. [`assert`](https://en.cppreference.com/w/cpp/error/assert)
* _violation_: undefined behaviour (see discussion)

## Enforcement of Dynamically-Enforceable Contracts

Between Dynamically-Enforceable API Contracts and the ISO C++ Standard,
there are a set of bugs which *could* be detected at run-time at a cost.

TODO: Bridge the gap between `unreachable` and `assert(false)`.

Broadly, there are three possible strategies that can be adopted
at the point where an assertion expression would evaluate to `false`:

### Log-And-Continue Enforcement Strategy

The assert statement could emit a run-time diagnostic regarding the contract violation
and then continue past the assertion.

This is unwise: the program is known to exhibit undefined behaviour at this
point. If the _provider_ were able to identify a reason why a violation might have
occurred they would have identified a bug. If the _provider_ had identified a bug,
they should have fixed the bug. If the _provider_ had fixed the bug,
they would no longer be able to identify a reason why a violation might have occurred.
Thus any violation is beyond the comprehension of the _provider_ and "all bets are
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

## Enforcement Profiles

The situation of the end user greatly affect preferred enforcement strategy.

### Tester

Somebody who is testing the program, e.g.

* a developer investigating a bug,
* a QA/test engineer, or
* a CI machine performing automated tests

will all prefer Trap Enforcement Strategy because bugs are a likelihood
and performance and stability are secondary concerns.

### Safety-Critical System With Redundancy

A safety-critical system *with* backup/redundancy will also prefer Trap Enforcement
Strategy because knowingly allowing the program to continue in an incorrect state
is an unacceptable risk. A life-support system or the autopilot
in an autonomous vehicle provide good examples.

### Safety-Critical System Without Redundancy

A safety-critical system *without* backup/redundancy
may prefer Log-And-Continue Enforcement Strategy

### Performance-Critical/Resource-Constrained

Applications which are performance-critical or resource-constrained
may favour Prevention Enforcement Strategy.
There must be a high degree of confidence in the correctness of their program.

### Business-Critical Systems

Where money is at stake but lives are not, either the Trap Enforcement Strategy or
Log-And-Continue Enforcement Strategy should be preferable.
A RESTful server with persistent state may eventually identify bad state
that has lain unnoticed for a while.
The developer can choose what happens on discovery:

* Do we kill the process and restart it in the hope that the problem goes away
  for a while, or
* do we 'soldier on' and hope the problem doesn't cause some catastrophe or other?

This choice will be affected by the scale of the fleet,
exposure to malicious agents and the cost of incorrect behaviour.

## Discussion

### Bugs versus Errors

Section 4.2 of [P0709R4, Zero-overhead Deterministic Exceptions: Throwing values](https://wg21.link/p0709r4),
"Proposed cleanup: Don’t report logic errors using exceptions",
makes clear the distinction between errors and bugs.

Applying this distinction to the contracts above:

* A bug is the violation of a C++ API Contract or the ISO C++ Standard.
* An error is the violation of a End User Contract.

***Special case***: what would be a bug in an End User Contract
is treated an error in a Test User Contract.

### Dynamically-Enforceable API Contract Violation is Undefined Behaviour

The popular idea that undefined behaviour is confined to violations of some subset
of the ISO C++ Standard is wrong and harmful.

The standard [describes](https://eel.is/c++draft/defns.undefined) UB as:

> behavior for which this document imposes no requirements

In other words, the standard does *not* exclude from the definition of UB
violations of contracts outside of the ISO C++ Standard.

A more accurate and helpful view of UB is that it is one possible consequence of
a contract violation that occurs at run-time for which _violation_ is not described.
