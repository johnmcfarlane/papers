# Contractual Disappointment in C++

This document provides guidance on how to write robust C++ programs.

For much of the document, I explore the factors related to how error-handling choices
are made:

* What are the types of problem that a running program might encounter?
* What contracts are in play?
* Of these contracts, which ones relate to bugs?
* What are the strategies for reacting to bugs in a running program?
* Who writes programs and what strategies might they prefer?

Then the different elements are brought together in a simple program which
illustrates how errors and bugs are treated very differently and how this
isn't affected by bug-handling strategies.

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
_User contract violation_ of a contract is what typically leads to disappointment.

### What About Provider Contract Violation?

Both the user and the provider can have contractual obligations.
However, ***this document concentrates on violation by user***.

Why?
Firstly, it's simpler to focus on one side of the contract and apply the findings
to the other side later on.
Secondly, the user is typically the less experienced and more error-prone of
the two parties. Therefore, violation by user is more common. It seems highly likely
that more software defects are caused by the user, rather than the provider
of a contract.

Nowhere is this clearer than in the case of the Toolchain Contract (below)
where the toolchain provider is far less likely to be the cause of a contract
violation by virtue of the significant rigour, effort and feedback poured into such
tools.

It is also important to note that providing timely corrective feedback to the _user_
is an effective way to help them learn their craft with less supervision.

## What Are the Parameters of a Contract?

The following set of parameters -- expressed as questions -- are helpful in
understanding the correct error-handling strategy for a contract.

* Where, and in what form, is the agreement expressed?
* Who is the provider of the contract?
* Who is the user of the contract?
* What are the consequences of violation by user?

## What Contracts Are There?

In a C++ program, some contracts that matter are as follows.

### End User Contract

The ultimate contract that a program must fulfil is to its end user.
All other contracts below are in support of this fulfilment.

* agreement: program documentation, possibly including a `--help` option
  or a 'man' page
* provider: the program author(s)
* user: the program user
* violation by user: user input is sanitized and ill-formed input is handled
  early through normal control flow (including exception handling)

User input might include command-line parameters, UI interaction, data files.
Input identified as ill-formed must be rejected early by the program.

Where possible, rejection should be accompanied by meaningful feedback
which helps the user correct the input so as to increase the chance of future success.
For example, on a typical terminal-based system,
meaningful feedback might take the form of a diagnostic emitted on the error stream,
followed by exit with non-zero status.

Input that is not rejected must be incapable of causing violations
of any contracts mentioned below.
In particular, contract violations could result in security vulnerabilities.
For programs with exposure to malicious agents, this is critical.
For safety-critical applications, this is important.
To identify flaws in input sanitization use fuzz testing.

### Test User Contract

During development, the program -- or portions of it --
may be built for testing purposes.

The program should be instrumented to test for violations of the below contracts.
As many violations as is practical should be checked.
Trapped contract violations should be treated like user errors
as described in the End User Contract above,
i.e. terminate with helpful diagnostic and non-zero status.

***Important:*** this fundamentally changes the status of some violations from bugs
to errors.
It is important to understand that what may be considered a bug outside of testing
(e.g. signed integer overflow or passing an unsorted sequence to `lower_bound`)
is instead considered an error and treated differently.
Bugs written by the program author(s) which surface during testing
are not a violation of the Test User Contract.

* agreement: varied, but may be enshrined in a project README, a wiki or project
  process document(s)
* provider:
  * analysis tool providers whose tools flag violations -- especially of the
    ISO C++ Standard and Toolchain Contract, or
  * C++ API Contract providers who are encouraged to assert that their APIs are used
    correctly.
* user: a test engineer who may be
  * an author of the program testing their work,
  * a test engineer testing the program for correct operation, or
  * a dev-ops engineer testing the program as part of a CI pipeline
* violation by user:
  * If the analysis tool advertises that it will trap a particular
    contract violation and fails to do so, that is a violation of this contract
    and considered a defect of the tool.
  * If a C++ API Contract violation could reasonably be identified (e.g. through
    an assertion) but does not, that is a violation of the Test User Contract.
  * The contract is restored when the program successfully fails
    as a result of a violation of contracts below.

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
* violation by user: undefined behaviour

Note: while this document focuses on run-time disappointment, authors are encouraged
to use static typing to surface defects earlier in the development process.

### Toolchain Contract

* agreement: toolchain documentation, including portions of the Language
  Specification identified as
  [implementation-defined behavior](https://eel.is/c++draft/defns.impl.defined)
* provider: implementer of the toolchain used to build the program
* user: the program author(s)
* violation by user: implementation-specific(?)

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
* violation by user: undefined behaviour (see discussion)

## Strategies For Handling Dynamically-Enforceable Contracts

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
and then continue past the assertion. This is not ideal: the program is known to
exhibit undefined behaviour at this point.

There is a strong desire to distinguish between contract violations that
exhibit UB and ones which are somehow safe. But this distinction is false.
An assert expression needs to be something that is inconceivable. As such, its cause
can only be explained by some unforeseen error earlier in the control flow.
This in turn leaves the door open to UB caused by the error elsewhere.
The horse has already bolted, so to speak.

Nevertheless, under certain constraints this strategy is appealing. If the cost of
terminating the program is greater than the risks of undefined behaviour, then
a programmer might choose this path. It would be sensible to do everything possible
to mitigate the effects, e.g. by disabling optimisations.

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

The domain of the program greatly affects preferred enforcement strategy
for Handling Dynamically-Enforceable Contracts.

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

## Example Program

The following excerpts from the program hosted
[here](https://github.com/johnmcfarlane/eg-error-handling/blob/6bee393c245debf4ef921f8518b1b6a89a477b25/src/main.cpp),
illustrate the recommended approach.

### Assertion Logic

Here is an example assert function which illustrates the above Strategies For Handling
Dynamically-Enforceable Contracts. It is compatible with some modern tool-chains.

```c++
constexpr void eg_assert(bool condition)
{
  if (condition) {
    return;
  }

#if defined(LOG_AND_CONTINUE_STRATEGY)
  fmt::print(stderr, "a C++ API violation occurred\n");
#elif defined(TRAP_STRATEGY)
  std::terminate();
#elif defined(PREVENTION_STRATEGY)
  __builtin_unreachable();
#endif
}
```

### C++ API

The following function, `number_to_letter`, illustrates a C++ API which might make
use of the `eg_assert` facility:

```c++
constexpr auto min_number{1};
constexpr auto max_number{26};

/// precondition: number is in range [1..26]
constexpr auto number_to_letter(int number)
{
  eg_assert(number >= min_number);
  eg_assert(number <= max_number);
  return char(number - min_number + 'A');
}
```

Calls to this API *must* observe the contract. In production code, this isn't
open to negotiation. In a C++ program, the contract is typically documented
in comments. Other language allow some of the preconditions and postconditions to
be expressed directly in the code using predicates of the kind passed into `eg_assert`.

### Calling a C++ API

APIs within a program should never try to deal with Dynamically-Enforceable Contract
violations resulting from incorrect invocation by users of the API.
The set of possible violations is often significant and can quickly clutter the implementation.

```c++
/// @pre Requires sanitized data, i.e. number in the range 1<=number<=26.
void sanitized_run(int number)
{
  fmt::print("{}", number_to_letter(number));
}
```

### Sanitizing Input

Code which digests input into the program is a very different matter.

The program's author(s) cannot assume that its input is error-free.
In order to be confident that no Dynamically-Enforceable Contracts are violated,
input must first be sanitized.

In this example, the command line parameter is converted to an `int`.

```c++
auto unsanitized_run(std::span<char*> args)
{
  // Verify correct number of arguments.
  constexpr auto expected_num_params{1};
  auto const actual_num_params{args.size()};
  if (actual_num_params != expected_num_params) {
    // End User Contract violation; emit diagnostic and exit with non-zero exit code
    fmt::print(
        stderr, "Wrong number of arguments provided. Expected={}; Actual={}\n", actual_num_params, expected_num_params);
    return false;
  }

  // Print help text if requested.
  auto const argument{std::string_view{args[0]}};
  if (argument == "--help"sv) {
    // **Not** an End User Contract violation;
    // print to stdout and exit with zero status code
    fmt::print("This program prints the letter of the alphabet at the given position.\n");
    fmt::print("Usage: letter N\n");
    fmt::print("N: number between {} and {}\n", min_number, max_number);
    return true;
  }

  // Convert the argument to a number.
  // Note: this further enhances type safety.
  int number;
  auto [ptr, ec] = std::from_chars(std::begin(argument), std::end(argument), number);
  if (ec == std::errc::invalid_argument || ptr != std::end(argument)) {
    // End User Contract violation; emit diagnostic and exit with non-zero exit code
    fmt::print(stderr, "Unrecognized number, '{}'\n", argument);
    return false;
  }

  // Verify the range of number.
  if (number < min_number || number > max_number) {
    // End User Contract violation; emit diagnostic and exit with non-zero exit code
    fmt::print(stderr, "Out-of-range number, {}\n", number);
    return false;
  }

  // The input is now successfully sanitized. If the program gets this far,
  // the End User Contract was not violated by the user.
  sanitized_run(number);

  return true;
}
```

In contrast with `sanitized_run`, `unsanitized_run` does a lot of validating of data.
Even if the users of the program are 'friendly', they may make mistakes which the
program author(s) did not anticipate. Out-of-bounds errors, invalid pointers and
past-the-end iterators may all result if the possible violations by the user of the
End User Contract are not tested.

See the section on Control Flow Alternatives for further discussion.

### Type Safety is King

As if it needed repeating, the best time to prevent problems is early.
To that end, program author(s) must use the type system to their advantage.
Well written APIs mean that most bugs do not survive compilation.

Here we see `std::span` used to encapsulate the program's implicitly-bound input
parameters.

```c++
auto main(int argc, char* argv[]) -> int
{
  if (!unsanitized_run(std::span{argv + 1, std::size_t(argc) - 1U})) {
    fmt::print("Try --help\n");
    return EXIT_FAILURE;
  }

  return EXIT_SUCCESS;
}
```

## Control Flow Alternatives

### Return Values

The Example Program (above) is simple enough that Boolean return values suffice as
a method for sending news of failure back to the calling process.

However, things get complicated quickly and several alternatives should be considered.
Unfortunately, this choice cannot be changed easily once code is written.

### Vocabulary Return Types

One disadvantage of reporting the success of the function as the return value is
that C++ functions only return one object.
APIs are more readable if they return the expected results of a successful invocation.
So returning errors gets in the way of readability.

To one degree or another, types such as `expected`, `outcome` and `optional` are
capable of packing together both the desired result and the successfulness.
They offer determinism and 

### Exceptions

Exceptions are well-suited to error handling. They keep the 'happy path' free of
error propagation logic which should make them more efficient in the case that nothing
goes wrong. Indeed, [they are often the most efficient solution](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1947r0.pdf)
when where there is high confidence that throwing is rare.

Exceptions are also a flexible solution which allow error handling code to be localised
within the call graph: imagine a program which needs to report defects differently
in separate sections of the code. In one section, errors are logged to a file. In
another section, errors are reported to a bug-tracking server. And in both sections,
the program is not allowed to access any IO. Such constraints can be overcome trivially
using just two `catch` blocks.

However, such versatility is rarely necessary in real applications. And exceptions
bring other costs.

* Exceptions require additional [space](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1640r1.html),
  which can render them impractical in small systems such as microcontrollers,
* When thrown, they are costly in [time and determinism](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1886r0.html),
  which can break real-time system constraints, violating the End User Contract.
* Even when not thrown, they can make it difficult to reason about control flow,
  which has implications for both developers and optimisers.

### Abnormal Program Termination

A program can be stopped quickly via APIs such as `std::terminate` and `std::abort`.

A simple error-handling function can be used much like the assert routine:

```c++
// error handler function
template <typename... args>
[[noreturn]] void eg_fatal(args&&... parameters)
{
  fmt::print(stderr, std::forward<args>(parameters)...);
  fmt::print("Try --help\n");
  std::abort();
}
```

Usage couldn't be much simpler:

```c++
  if (actual_num_params != expected_num_params) {
    eg_fatal("Wrong number of arguments provided. Expected={}; Actual={}\n", actual_num_params, expected_num_params);
  }
```

This approach is often discouraged in C++ programs. The main reason is that it typically
bypasses destructors. That's not such a worry if the program is already in a bad
state, so terminating in reaction to contract violations is relatively palatable.
And destructors in a modern, well designed system are only important to a running
process: memory, file descriptors and peripherals should all be freed up by
the system once the owning process is ended.

So with caveats, this can be the best approach for reacting to violations of the
End User Contract -- as well as contracts which cause UB. In profile,
Safety-Critical System With Redundancy, the requirement to 'fail fast' can be well
served by this approach. And in profile, Business-Critical Systems, the problems
associated with bypassing destructors may not be significant.

### Push UB Onto The End User

The program might be part of a larger system which is internal to a group of developers.
For example, essential data might be stored in a read-only filesystem which is essential
to the system. Loss of this filesystem might be unrecoverable. In such a circumstance,
why not simply assume that the filesystem is there? Why not treat the End User Contract
in the same way as a C++ API contract?

This is entirely possible but the implications must be fully understood.
There is reduced freedom to invoke the program with ill-formed input.
There is heightened risk of compromise, corruption or critical failure if the
program is ever invoked outside the parameters of the End User Contract.
Even when it is possible, it is rarely worthwhile taking this risk.

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

### Contract Violation By Provider

Finally we can return to the matter of _violation by provider_.

The first thing to observe is the distinction between

* preconditions: assertions expressing a user's contractual obligations; typically
  found on entry to the phase of a program where the contract applies, and
* postconditions: assertions expressing a provider's contractual obligations;
  typically found on exit from the phase of a program where the contract applies.

### Kings Are Costly, Type Safety Is Overrated

Slavish observance to any rule always ends badly.
This is increasingly true in C++ where harmful misuse of the type system confuses:

* constraining the space of possible programs so as to eliminate the bad programs,
  with
* constraining the space of possible programs so as to camouflage the bad programs.

```c++
// good example here
```

### Libraries Versus Programs

## Conclusion

* You can choose bug handling strategy easily, change your mind or use different
  strategies in different builds. 
* But you cannot so easily change error-handling strategies.
* Sometimes, a bug in a program isn't a bug, it's an error.
* UB is all around.
*
