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

Applying this distinction to the contracts below:

* A _bug_ is the violation of a C++ API Contract or the ISO C++ Standard.
* An _error_, is when an interface couldn’t do what it advertised.

Note: a third category, _abstract machine corruption_ is identified by P0709R4,
and includes heap exhaustion and stack overflow.
But it does not relate to contracts involving C++ program developers.)

There is one time when the bug versus error distinction breaks down.
That is when considering the Test User Contract.
When testing for contract violations,
the user wishes for bugs to behave far more like errors.

## About Contracts

For our purposes, a contract is an _agreement_ between a _user_, and
a _provider_ about the run-time behaviour of some or all of a program.
_User contract violation_ of a contract is what typically leads to disappointment.

### Salient Parameters of a Contract

The following set of parameters -- expressed as questions -- are helpful in
understanding the correct error-handling strategy for a contract.

* Where, and in what form, is the agreement expressed?
* Who is the provider of the contract?
* Who is the user of the contract?
* What are the consequences of user contract violation?

### What About Provider Contract Violation?

Both the user and the provider have contractual obligations.
However, ***this document concentrates on user contract violation***.

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

### What About Contract Author?

Authors are often also providers, sometimes users and occasionally neither.
The quality of their contracts affects the chances of avoid violations.
Beyond this, they are mentioned very little.

## What Contracts Are There?

In a C++ program, some contracts that matter are as follows.

### End User Contract

The ultimate contract that a program must fulfil is to its end user.
All other contracts below are in support of this fulfilment.

* agreement: program documentation, possibly including a `--help` option
  or a 'man' page
* provider: the program developer(s)
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
Bugs written by the program developer(s) which surface during testing
are not a violation of the Test User Contract.

* agreement: varied, but may be enshrined in a project README, a wiki or project
  process document(s)
* provider:
  * analysis tool providers whose tools flag violations -- especially of the
    ISO C++ Standard and Toolchain Contract, or
  * C++ API Contract providers who are encouraged to assert that their APIs are used
    correctly.
* user: a test engineer who may be
  * a program developer testing their work,
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

* API contract expressions (e.g. assertions, pre-conditions and post-conditions),
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
* user: the program developer(s)
* violation by user: undefined behaviour

Note: while this document focuses on run-time disappointment, developers are encouraged
to use static typing to surface defects earlier in the development process.

### Toolchain Contract

* agreement: toolchain documentation, including portions of the Language
  Specification identified as
  [implementation-defined behavior](https://eel.is/c++draft/defns.impl.defined)
* provider: implementer of the toolchain used to build the program
* user: the program developer(s)
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

The provider should communicate correct API usage to the user.

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
* provider: C++ API implementer; often the program developer(s)
* user: program developer(s)
* violation by user: undefined behaviour

### Dynamically-Enforceable Contract

This category is the constituency of violations from

* ISO C++ Standard, and
* Dynamically-Enforceable API Contracts

that can be identified programmatically and unambiguously at run-time.

* agreement: constituent contract agreements
* provider: constituent contract providers
* user: program developer(s)
* violation by user: undefined behaviour; trappable

Examples of violations include

* behaviour not defined by the ISO C++ Standard such as
  * integer divide-by-zero,
  * out-of-bounds array lookup,
  * out-of-lifetime object access,
  * calling `std::vector::front()` on an empty object,
  * calling `upper_bound` on an unsorted sequence, and
  * passing iterators to separate sequences to `std::for_each`, and
* C++ API Contract violations for which a test could be expressed in-code.

The cost involved in testing for these violations varies.
But in theory, all of them can be automatically tested for
without changing the behaviour of the program or violating other contracts,
aside from the time and space taken to execute.

Further, a subset of them can be identified through static analysis,

```c++
int main()
{
  return 1/0;
}
```

and in a constant expressions, they become compiler errors:

```c++
constexpr auto a{1/0};
```

## Dynamically-Enforceable Contract Strategies

As of C++20, Dynamically-Enforceable ISO C++ Standard violations require
system or toolchain support to identify. For example,
null pointer dereferences may be trapped by systems with virtual memory, and
out-of-bounds array lookup may be trapped by dynamic analysis tools.

Dynamically-Enforceable C++ API Contract violations must be called out using assertions.

Broadly, there are three possible strategies that can be adopted
at the point where an assertion would evaluate to `false`.

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

***I strongly encourages the use of assumptions in the implementation of
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

The program's developer(s) cannot assume that its input is error-free.
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
program developer(s) did not anticipate. Out-of-bounds errors, invalid pointers and
past-the-end iterators may all result if the possible violations by the user of the
End User Contract are not tested.

See End User Provider Strategies for further discussion.

### Type Safety is King

As if it needed repeating, the best time to prevent problems is early.
To that end, program developer(s) must use the type system to their advantage.
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

## End User Provider Strategies

The End User provider is charged with enforcement of the End User Contract,
e.g. by emitting diagnostics and returning non-zero status codes.
There are several approaches that can be taken.

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
They offer determinism and fast failure in the non-happy path.

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

## C++ API Author Strategies

### The Pit Of The False Sense of Success

When the advertised goal of a C++ API cannot be achieved,
it should be difficult for the user of the API to overlook the disappointment.

Example 1:

```c++
auto load_file(string filename)
{
  ifstream file{filename};
  if (!file) {
    // bad! missing should not be the same empty
    return vector<byte>{};
  }
  ...
}
```

Example 2:

```c++
auto pull_away(color traffic_light)
{
  switch (traffic_light) {
    case color::red:
      return false;
    case color::amber:
      return false;
    case color::green:
      return true;
    default:
      // bad! suppresses compiler warnings; silences sanitizers;
      // illogical: do we wait on a blue traffic light??
      return false;
  }
}
```

These two functions return incorrect results, which is bad enough.
But by returning anything at all,
the error is allowed to propagate further into the program.
By the time the value is observed to be incorrect,
control may have passed to a distant part of the program
where the cause of -- and fix to -- the problem are lost from view.

> A trusted friend will tell you when you have spinach in your teeth.

There are two main solutions to this kind of anti-pattern.

### Disappointment By Type

The best documentation is the API itself, starting with good naming.
In a strongly-typed language, the types of a declaration are a powerful form of communication.

Example 1 revisited:

```c++
auto load_file(string filename) -> optional<vector<byte>>
{
  ifstream file{filename};
  if (!file) {
    // better; user is made aware that the result isn't always some bytes
    return nullopt;
  }
  ...
}
```

### Disappointment By Contract

Example 1 showed disappointment that could reasonably happen in a bug-free program.
But when the C++ API Contract user could reasonably avoid the disappointment,
it is far better to clearly put the responsibility onto the user:

* Make it clear that it's a bug for the API to be used disappointingly.
* Fulfil the obligations of a Test User Contract provider and help the user discover
  the bug.

Example 2 revisited:

```c++
/// @pre traffic_light must be red, amber or green
auto pull_away(color traffic_light)
{
  switch (traffic_light) {
    case color::red:
      return false;
    case color::amber:
      return false;
    case color::green:
      return true;
  }
  // better; program will trap if user passes (color)3;
  // it's clear from the code that fallthrough is a bug
  assert(false);
}
```

Note that the provider might be tempted to add a `default` clause to the switch statement.
This is not advised. There is no acceptable default behaviour here
so it's better not to express the intent that there is.

Example 2 (Slight Return):

```c++
/// @pre traffic_light must be red, amber or green
auto pull_away(color traffic_light)
{
  switch (traffic_light) {
    case color::red:
      return false;
    case color::amber:
      return false;
    case color::green:
      return true;
    default:
      // worse; there is no fourth alternative so why imply that there is?
      // What if someone adds enumerator, color::blue?
      // The compiler would not complain that there is no "case color::blue" 
      assert(false);
  }
}
```

## Discussion

### Provider Contract Violation

We can now return to the question of _provider contract violation_.

The first thing to observe is the distinction between

* preconditions: assertions expressing a user's contractual obligations; typically
  found on entry to the phase of a program where the contract applies, and
* postconditions: assertions expressing a provider's contractual obligations;
  typically found on exit from the phase of a program where the contract applies.

### Choices for Contract Users

There should be as few as possible. The user is the greatest point of failure.
Where possible, they should not be distracted by the dilemmas of API design.
Whether API users, End users or any other kind of user, their life should be made
as easy as possible. This does not mean defensive APIs which hide their mistakes.
Their mistakes should be made obvious to them as quickly and clearly as possible.

### API Contract Violation is Undefined Behaviour

It is a common misconception that the term _undefined behaviour_
refers only to violations of some subset of the ISO C++ Standard.

The standard [describes](https://eel.is/c++draft/defns.undefined) UB as:

> behavior for which this document imposes no requirements

In other words, the standard does *not* exclude violations of contracts
outside of the ISO C++ Standard.

A more accurate and helpful view of UB is that it is one possible results of
a contract violation that occurs at run-time, for which consequences are not described.

Part of the confusion is the (false) assumption that because a C++ API has a definition,
that this somehow means that its behaviour is also defined.
Let's look again at the function `number_to_letter`.

```c++
/// precondition: number is in range [1..26]
constexpr auto number_to_letter(int number)
{
  return char(number - min_number + 'A');
}
```

It so happens that we can see the definition of `number_to_letter`. Even if we couldn't,
we might feel confident guessing roughly how it was implemented.
And (assertions aside) we might feel confident to say that the function does not
exhibit undefined behaviour, even when the precondition is violated.

We would be committing a grave error.

Firstly, the author has effectively limited the warranty of the contract.
They are declining to make a guarantee about what the result of the call will be
in the fact of contract violation. Put another way, they decline to define the behaviour.

Secondly, the provider is under no obligation
to have tested the behaviour outside of contract.
So any and all assurances gathered through automated testing
do not apply to out-of-contract use.
And in the absence of testing, it's easy to miss a failure case, e.g.:

```c++
// signed integer overflow resulting in ISO C++ Standard UB
number_to_letter(0x7fffffff);
```

Finally, the provider reserves the right to change the implementation as they choose.
Assumptions drawn from the original definition are already false.
But they become yet more dangerous if applied to a new definition, for example:

```c++
constexpr auto number_to_letter(int number)
{
  constexpr auto lookup_table = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
  return lookup_table[number - min_number];
}
```

It is easy to become distracted by the wonders of modern compilers.
They are able to generate highly optimised code, by assuming contract fulfilment.
But at the end of the day, they are not magic and nor is undefined behaviour.
UB is just a way for authors to limit contracts so that providers can deliver more.

### Libraries Versus Programs

### Sanitizer Before You Optimise

## Conclusion

* You can choose bug handling strategy easily, change your mind or use different
  strategies in different builds. 
* But you cannot so easily change error-handling strategies.
* Sometimes, a bug in a program isn't a bug, it's an error.
* UB is all around.
*

## Acknowledgements

Thanks to Alicja Przybyś and Andrzej Krzemieński for much feedback, correction
and guidance.
