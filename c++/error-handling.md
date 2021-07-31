# Contractual Disappointment in C++

> "Oh who can say how subtle and safe one feels" (John Betjeman, False Security)

This document offers advice on writing robust programs in C++.
It draws on personal experience in domains including interactive entertainment,
large-scale server systems and safety-critical devices.

Contracts and their consequences are the main tool used to frame run-time failure.
Through the lens of contracts I attempt to show that — while the correctness of
C++ programs is a necessity, rather than a luxury — carefully understood
contracts are the most effective way to ensure usability, efficiency *and* reliability.

## Introduction

### Disappointment

[P0157R0, Handling Disappointment in C++](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0157r0.html),
observes that run-time errors are just one kind of disappointment
that [a C++ program](https://eel.is/c++draft/basic.link#def:program) might elicit.
I argue that all disappointment involves the violation of a contract
and that the correct strategy for handling a violation depends on two things:

1. the parameters of the contract, and
1. the usage profile of the program.

### Bugs and Errors

Section 4.2 of [P0709R4, Zero-overhead Deterministic Exceptions: Throwing values](https://wg21.link/p0709r4),
"Proposed cleanup: Don’t report logic errors using exceptions",
makes clear the distinction between errors and bugs.

Applying this distinction to the contracts introduced in the following section:

* A _bug_ is the violation of a C++ API Contract or the ISO C++ Standard.
* An _error_, is when an interface couldn’t do what it advertised.

Note: a third category, _abstract machine corruption_ is identified by P0709R4,
which includes heap exhaustion and stack overflow.
But it does not relate to contracts involving C++ program developers.

There is one time when the bug versus error distinction breaks down.
That is when considering the Test User Contract.
When testing for contract violations,
the user wishes for bugs to behave like errors.

## Contracts

### Contract Attributes

For our purposes, a contract is an _agreement_ between a _user_, and
a _provider_ about the run-time behaviour of some or all of a program.
_User contract violation_ of a contract is what typically leads to disappointment.

Those four attributes: agreement; user; provider; and user contract violation,
are the salient attributes. But there are others.

#### Provider Contract Violation

Both the user and the provider have contractual obligations.
However, this document concentrates on user contract violation.

Why?

Firstly, it's simpler to focus on one side of the contract and apply the findings
to the other side later on.
Secondly, the user is typically the less experienced and more error-prone of
the two parties. Therefore, violation by the user is more common.

For example, in the case of the Toolchain Contract (below),
the toolchain provider is far less likely to be the cause of a contract
violation by virtue of the significant rigour, effort and feedback poured into such
tools.

#### Contract Author

Authors are often also providers, sometimes users and occasionally neither.

### Types of Contracts

In a C++ program, some contracts that matter are as follows.

#### End User Contract

The overarching contract that a program must fulfil is to its end user.
All following contracts exist in support of this fulfilment.

* agreement: program documentation, possibly including a `--help` option
  or a 'man' page
* provider: the program developer
* user: the program user
* violation by user: input data is validated and erroneous input is handled
  early through normal program control flow

Input might include command-line parameters, UI interaction and data files.
Input identified as erroneous must be rejected early by the program.

Where possible, rejection should be accompanied by meaningful feedback
which helps the user correct the input and increase the chance of subsequent success.
For example, on a typical terminal-based system,
meaningful feedback might take the form of a diagnostic emitted on the error stream,
followed by exit with non-zero status.

Input that is not rejected must be incapable of causing violations
of any contracts mentioned below.
Contract violations could result in security vulnerabilities
or unsafe program behaviour in safety-critical applications.
Fuzzers can help to test whether the program fails to reject erroneous input.

#### Test User Contract

During development, the program — or portions of it —
may be built for testing purposes.

The program should be instrumented to test for violations of the Dynamically-Enforceable
C++ API Contracts (below).
The program may be intended to test the End User Contract,
as is the case for unit tests.
As many violations as is practical should be checked.
Trapped contract violations should be treated like user errors
as described in the End User Contract above,
i.e. terminate with helpful diagnostic and non-zero status.

***Important:*** this changes the status of violations described elsewhere
from bugs to errors.
It is important to understand that what may be considered a bug outside of testing
(e.g. signed integer overflow or performing binary search on an unsorted sequence)
is here instead an error.
Bugs written by the program developer which are trapped during testing
are *not* a violation of the Test User Contract.

* agreement: documentation of dynamic analysis tools and/or sanitizers
* provider:
  * analysis tool providers whose tools flag violations — especially of the
    ISO C++ Standard and Toolchain Contract, or
  * C++ API Contract providers who are encouraged to assert that their APIs are used
    correctly.
* user: an engineer who may be
  * a program developer testing the End User Contract with unit tests,
    or to test other parts of the C++ API Contract,
  * a test engineer testing the program for correct operation, or
  * a dev-ops engineer testing either of the above as part of a CI pipeline
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

#### ISO C++ Standard

A typical language specification for a C++ program is a revision of
[the ISO C++ Standard](https://isocpp.org/std/the-standard).

* agreement: a recognised set of standing documents including a C++ standard
  such as International Standard ISO/IEC 14882:2020(E), technical specifications,
  etc..
* provider: implementer of the toolchain used by the program developer
* user: the program developer
* violation by user: undefined behaviour; trappable

Note: while this document focuses on run-time disappointment, developers are encouraged
to use preventative practices such as static typing
in order to surface defects earlier in the development process.

#### Toolchain Contract

* agreement: toolchain documentation including portions of the Language
  Specification identified as
  [implementation-defined behavior](https://eel.is/c++draft/defns.impl.defined)
* provider: implementer of the toolchain used to build the program
* user: the program developer
* violation by user: implementation-specific(?)

Note: being comprised of one or more programs,
the toolchain has its own End User Contract.

#### C++ API Contract

Note: For most intents and purposes, the library incorporated into
the ISO C++ Standard can be considered to be a collection of C++ APIs.

* agreement: documentation (ideally self-documentation)
* provider: C++ API implementer (often also the program developer)
* user: program developer
* violation by user: various

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

are outside the scope of this document.
The former is left to the compiler to enforce.
The latter can only be deal with by developers.
All things being equal, good APIs favours the former and avoid the latter.

### Types of Contract Violation

From the above contracts, we can identify several important patterns based on
details about how those contracts are broken. They are often more important to the
program developer than where the contract is documented or who is disappointing who.

#### Dynamically-Enforceable C++ API Contracts

This is the subset of C++ API Contracts which it is impossible to test at compile
time, but which it is possible to test at run-time.

* agreement: documentation (ideally self-documentation)
* provider: C++ API implementer (often also the program developer)
* user: program developer
* violation by user: undefined behaviour; trappable

#### Unambiguous Bugs

This contractual category is the constituency of violations from

* ISO C++ Standard, and
* Dynamically-Enforceable C++ API Contracts

that can be identified programmatically and unambiguously at run-time.

* agreement: constituent contract agreements
* provider: constituent contract providers
* user: program developer
* violation by user: undefined behaviour; trappable

Examples of violations include

* behaviour not defined by the ISO C++ Standard, e.g.
  * integer divide-by-zero,
  * out-of-bounds array lookup,
  * out-of-lifetime object access,
  * calling `std::vector::front()` on an empty object,
  * calling `upper_bound` on an unsorted sequence, and
  * passing iterators from separate sequences to `std::for_each`, and
* C++ API Contract violations for which a test could be expressed in-code.

The cost involved in testing for these violations varies.
But in theory, all of them can be automatically tested for
without changing the behaviour of the program or violating other contracts
(aside from the time and space taken to execute).

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

## Unambiguous Bugs in Detail

### Unambiguous Bug Strategies

As of C++20, Dynamically-Enforceable ISO C++ Standard violations require
system or toolchain support to identify. For example,
null pointer dereferences may be trapped by systems with virtual memory, and
out-of-bounds array lookup may be trapped by dynamic analysis tools.

Dynamically-Enforceable C++ API Contract violations must be trapped using assertions.

Broadly, there are three possible strategies that can be adopted
at the point where an assertion would evaluate to `false`.

#### Trap Enforcement Strategy

The assert statement can halt the program.

This has the disadvantage that the program completely fails to perform its task.
But the advantage is that no further incorrect behaviour
will be exhibited by the program; the user is protected from unbounded risk.
It is also of most help to the developer as they are strongly encouraged
to confront the bug early.

Because of these advantages, it makes a good default strategy.
This is the default chosen by [the `assert` macro](https://en.cppreference.com/w/cpp/error/assert).

#### Prevention Enforcement Strategy

The program can assume that defects have been eliminated.
Assuming bugs do not occur, no assert expression will ever be false.
This means that assert statements behave as if they are not there.

```c++
#define ASSERT(condition) ((void)0)
```

This in turn means that they can be empty or used as hints to the optimiser.

```c++
#define ASSERT(condition) ((cond) ? static_cast<void>(0) : __builtin_unreachable())
```

Advantages:

* Optimiser is given hints which are often necessary to fulfil
  the zero-overhead principle.
* Sanitizer seamlessly extends the range of bugs it is able to trap.
* Nervous developers who are not certain they have yet eliminated defects
  are advised to disable optimisations regardless of new hints.

Disadvantages:

* Developers who wish to optimise and release defective code will see
  further deterioration in the behaviour of their program.

Concerns about this deterioration are raised in ([P2064](https://wg21.link/p2064r0)).
It indicates than an alarming amount of defective code reaches production
without being run through a sanitizer.
Such programs should not assume that asserts hold and should be released without
optimisations enabled.

#### Log-And-Continue Enforcement Strategy

The program can emit a run-time diagnostic regarding the contract violation
and then continue past the assertion. This is not ideal:

* Code size may increase considerably.
* It involves relying — for logging purposes — on a program which exhibits UB.

Nevertheless, it is a popular choice in some domains
where the cost of terminating a program is great
and the risk of continuing with undefined behaviour is low.

### Enforcement Profiles

The domain of the program greatly affects preferred Unambiguous Bug Strategies.

#### Tester

Somebody who is testing the program, e.g.

* a developer investigating a bug,
* a QA/test engineer, or
* a DevOps engineer programming a CI runner to perform automated tests

will all prefer Trap Enforcement Strategy because bugs are a likelihood
and performance and stability are secondary concerns.

#### Safety-Critical System With Redundancy

A safety-critical system *with* backup/redundancy will also prefer Trap Enforcement
Strategy because knowingly allowing the program to continue in an incorrect state
is an unacceptable risk.
A life-support system or the autopilot in an autonomous vehicle provide good examples.

#### Safety-Critical System Without Redundancy

A safety-critical system *without* backup/redundancy
may prefer Log-And-Continue Enforcement Strategy.

#### Performance-Critical/Resource-Constrained

Applications which are performance-critical or resource-constrained
may favour Prevention Enforcement Strategy.

Examples applications include interactive entertainment software, scientific simulations
and embedded controllers.

#### Business-Critical Systems

Where business interests are at stake but lives are not,
either the Trap Enforcement Strategy or Log-And-Continue Enforcement Strategy
may be preferable.
A RESTful server which persists between requests may eventually identify bad state
that has lain unnoticed for a while.
The developer can choose what happens on discovery:

* Kill the process and restart it in the hope that the problem goes away for a while.
* 'Soldier on' and hope the problem doesn't cause some catastrophe or other.

This choice will be affected by factors such as the scale of the fleet,
exposure to malicious agents and the cost of incorrect behaviour.

## End User Provider Strategies

The End User provider is charged with enforcement of the End User Contract,
e.g. by emitting diagnostics and returning non-zero status codes.

Several approaches that can be taken are detailed below.

Note: these choices are tempered by the distinction between libraries and programs.
The author of a program can choose a strategy as they see fit.
But the author of a library must anticipate what choices
their library user might make.
This is a general problem for C++.

### Return Values

Some programs are simple enough that Boolean return values suffice as
a method for sending news of failure back to the calling process.

However, things get complicated quickly and unfortunately,
this choice cannot be changed easily once code is written.

### Vocabulary Return Types

One disadvantage of reporting the success of the function as the return value is
that C++ functions can only return one object.
APIs are more readable if they return the results of a successful invocation.
So returning errors gets in the way of readability.

To some degree or another, types such as `expected`, `outcome` and `optional` are
capable of packing together both the desired result and the successfulness.
They offer determinism and fast failure in the disappointing case.

### Exceptions

Exceptions are well-suited to error handling. They keep the 'happy path' free of
error propagation logic which should make them more efficient in the case that nothing
goes wrong. Indeed, [they are often the most efficient solution](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1947r0.pdf)
where there is high confidence that throwing is rare.

Exceptions are also a flexible solution which allow error handling code to be localised
within the call graph: imagine a program which needs to report defects differently
in separate sections of the code. In one section, errors are logged to a file. In
another section, errors are reported to a bug-tracking server. And in both sections,
the program is not allowed to access any IO. Such constraints can be overcome trivially
with just a couple of `catch` blocks.

However, such versatility is rarely necessary in real applications. And exceptions
bring other costs.

* Exceptions require additional [space](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1640r1.html),
  which can render them impractical in small systems such as microcontrollers,
* When thrown, they are costly in [time and determinism](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1886r0.html),
  which can break real-time system constraints, violating the End User Contract.
* Even when not thrown, they can make it difficult to reason about control flow,
  which has implications for both developers and tools.

### Abnormal Program Termination

A program can be stopped quickly via APIs such as `std::terminate` and `std::abort`.

A simple error-handling function can be used much like the assert routine:

```c++
// error handler function
template <typename... args>
[[noreturn]] void fatal(args&&... parameters)
{
  fmt::print(stderr, std::forward<args>(parameters)...);
  fmt::print("Try --help\n");
  std::abort();
}
```

Usage couldn't be much simpler:

```c++
  if (actual_num_params != expected_num_params) {
    fatal("Wrong number of arguments provided. Expected={}; Actual={}\n", actual_num_params, expected_num_params);
  }
```

Abnormal program termination is often discouraged in C++ programs.
The main reason is that it typically bypasses destructors.
That's not such a worry if the program is already in a bad
state, so terminating in reaction to contract violations is relatively palatable.
And destructors in a modern, well designed system are only important
while the process is running:
memory, file descriptors and peripherals should all be freed up by the system
once the owning process is ended.

So with caveats, this can be the best approach for reacting to violations of the
End User Contract — as well as contracts which cause UB. In profile,
Safety-Critical System With Redundancy, the requirement to 'fail fast' can be well
served by this approach. And in profile, Business-Critical Systems, the problems
associated with bypassing destructors may not be significant.

### Push UB Onto The End User

The program might be part of a larger system which is internal to a group of developers.
For example, essential data might be stored in a read-only filesystem.
Loss of this filesystem might be unrecoverable. In such a circumstance,
why not simply assume that the filesystem is there? Why not treat the End User Contract
in the same way as a C++ API contract?

This is entirely possible but the implications must be fully understood.
There is reduced freedom to invoke the program with ill-formed input.
There is heightened risk of compromise, corruption or critical failure if the
program is ever invoked outside the parameters of the End User Contract.
Even when it is possible, it is rarely worthwhile taking this risk.

## C++ API Author Strategies

### The Pit of False Security

When the advertised goal of a C++ API cannot be achieved,
it should be difficult for the user of the API to escape the disappointment.

Example 1:

```c++
auto load_file(string filename)
{
  ifstream file{filename};
  if (!file) {
    // bad! missing is not the same as empty
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

These two functions return incorrect results. And by returning anything at all,
the error is allowed to propagate further into the program.
By the time the value is observed to be incorrect,
control may have passed to a distant part of the program
where the cause of — and fix for — the problem are lost from view.

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
it is far better to clearly put the responsibility onto the caller:

* Make it clear that it's a bug for the API to be used disappointingly.
* Fulfil the obligations of a Test User Contract provider and help the user discover
  the bug.

Example 2 revisited:

```c++
// precondition: traffic_light must be red, amber or green
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
  // it's clear from the code that fall-through is a bug
  assert(false);
}
```

Note that the provider might be tempted to add a `default` clause to the switch statement.
This is not advised. There is no acceptable default behaviour here
so it's better not to express the intent that there is.

Example 2 (Slight Return):

```c++
// precondition: traffic_light must be red, amber or green
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
      // The compiler would not complain that there is no "case color::blue".
      assert(false);
  }
}
```

## Example Program

The following excerpts from the program hosted
[here](https://github.com/johnmcfarlane/eg-error-handling/blob/6bee393c245debf4ef921f8518b1b6a89a477b25/src/main.cpp),
illustrate the recommendations collected in this document.

### Assertion Logic

Here is an example assert function which illustrates the above Unambiguous Bug Strategies.
It is compatible with some modern tool-chains.

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
  // Just-about anything can go here as it will never be reached.
  // All Enforcement Profiles should assume this line isn't reached.
  // Further, the Performance-Critical/Resource-Constrained profile
  // may wish to hint to the compiler, e.g. with __builtin_unreachable().
#else
#error
#endif
}
```

### C++ API

The following function, `number_to_letter`, illustrates a C++ API which might make
use of the `eg_assert` facility:

```c++
constexpr auto min_number{1};
constexpr auto max_number{26};

// precondition: number is in range [1..26]
constexpr auto number_to_letter(int number)
{
  eg_assert(number >= min_number);
  eg_assert(number <= max_number);
  return char(number - min_number + 'A');
}
```

Calls to this API *must* observe the contract.
The contract is part of the interface of the API.
Contracts need to be communicated to API users.
Communication can be through documentation.

The contract is composed of conditions.
Some conditions can be formally expressed.
Some such conditions can be expressed in the code itself.
Some languages support format expression as part of the interface.
Many languages support expression of conditions within the implementation.

The `eg_assert` invocations above are examples of expressions within the implementation.

### Calling a C++ API

In general, APIs should not attempt to handle out-of-contract usage.
As with the `pull_away` example above, normalising bugs has many negative consequences.
The set of possible violations is likely to be vast
so logic to deal with it can dilute the 'intentional' code of a function.

The contract of `sanitized_run`,

```c++
// precondition: Requires validated data, i.e. number in the range 1<=number<=26.
void sanitized_run(int number)
{
  fmt::print("{}", number_to_letter(number));
}
```

with respect to `number` is no different to that of `number_to_letter`;
they both require the same range of values.
So even an assertion is of little value here.

### Validating Input

Code which digests input into the program is a very different matter.

The program's developer(s) cannot assume that its input is error-free.
In order to be confident that no Unambiguous Bugs can occur,
input must first be checked.

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
        stderr,
        "Wrong number of arguments provided. Expected={}; Actual={}\n",
        actual_num_params, expected_num_params);
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

  // The input is now successfully validated. If the program gets this far,
  // the End User Contract was not violated by the user.
  sanitized_run(number);

  return true;
}
```

In contrast with `sanitized_run`, `unsanitized_run` does a lot of validating of data
and little else.

Like ingested poison, .
This can be seen as analogous to digestion:
External matter is ingested, harmful or unwanted matter is rejected,
and if anything remains, it is accepted into the organism and processed.

It may be difficult to determine until quite far in the process whether input is well-formed.

It essentially does nothing else.
Even if the users of the program are 'friendly', they may make mistakes which the
program developer did not anticipate. Out-of-bounds errors, invalid pointers and
past-the-end iterators may all result if the possible violations by the user of the
End User Contract are not tested.

See End User Provider Strategies for further discussion.

### Type Safety is King

As if it needed repeating, the best time to prevent problems is early.
To that end, the program developer must use the type system to their advantage.
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

## Discussion

### API Contract Violation is Undefined Behaviour

It is a common misconception that the term _undefined behaviour_
refers only to violations of some subset of the ISO C++ Standard.

The standard [describes](https://eel.is/c++draft/defns.undefined) UB as:

> behavior for which this document imposes no requirements

In other words, the standard does *not* exclude, from its definition of UB,
violations of contracts outside of the ISO C++ Standard.

A more accurate and helpful view of UB is that it is one possible result of
a violation of any contract that occurs at run-time,
for which consequences are not described.

Part of the confusion is the belief that the definition behind an interface
grants the user licence to infer a broader contract.
Let's look again at the function `number_to_letter`.

```c++
// precondition: number is in range [1..26]
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

### Wide Or Narrow?

In the above example, `number_to_letter` is what is described as a narrow contract.
The user of the API must not deviate from the expected usage
and to do so necessarily introduces undefined behaviour.

One way to widen `number_to_letter`'s contract would be:

```c++
// returns corresponding uppercase letter iff number is in range [1..26]
constexpr auto number_to_letter(int number)
-> std::optional<char>
{
  if (number < min_number || number > max_number) {
    return std::nullopt;
  }

  constexpr auto lookup_table = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
  return lookup_table[number - min_number];
}
```

The code is now likely to be slower to comprehend, compile and execute.
The API is more difficult to use and the user is encouraged to deal at run-time
with bugs which should simply have been removed.
In general, wide contracts are the wrong choice for components of any
fully-digital automated system.

But when a human is in the loop, a wide contract is an essential UI feature.
Humans often require feedback in order to correct mistakes.
We consider two contracts which involve a human at run-time:

* the program user in the End User Contract, and
* the engineer in the Test User Contract.

However, the Test User Contract -- ever enigmatic -- relies on the narrowness of
the C++ API Contract in order to provide a wide contract of its own.
Thus by widening a C++ API Contract,
the contract author impedes the engineer's ability to identify bugs.

### Don't Optimize Until You Sanitize

C++ toolchains increasingly optimise programs assuming that they are correct.
Optimisations are not free. They demand strict adherence to contracts.
Historically, UB has been difficult to detect.

Fortunately, toolchains now also provide facilities for detecting some UB.
These include dynamic analysis tools known as sanitizers
which instrument code to test for bugs.
The beauty of undefined behaviour is that it allows this — and every other valid
bug-hunting tool — to be used in conforming code.

Meanwhile, modern software development practices emphasise automated testing regimes.
It is not uncomment for most of the APIs in a program
to be tested for provider C++ API Contract violation frequently during development.
Therefore, there is strong reason to use sanitizers as a matter of good practice.

If you wish to enable compiler optimisations in the program you provide to users
you are strongly advised to test your program code with sanitizers.
Combined with the approaches detained in this document, this will lead to significantly
less defective software without the necessity to compromise on safety or performance.

### Don't Rely on Sanitizers Alone

Unfortunately, sanitizers are far from perfect.
Some unambiguous bugs are prohibitively difficult to detect at run-time.

Further, sanitizers sometimes fail to instrument for UB. For example, if the logic
which identifies an instance of UB is located in the compiler's optimiser,
then instrumentation will not occur unless the optimiser is enabled. For this reason,
it is important to test code with settings as close as possible to production.

Finally, not all bugs are undefined behaviour.
Some bugs are only End User Contract violations.
They are sometimes described as errors in the 'business logic' of the program.
Because of this, unit testing will always play a vital role in a
healthy development process.

### UB Considered Helpful

Nevertheless, developers who aim for program correctness must recognise that a formal
notion of bugs which is machine-testable, is every bit as important as type safety.
It is undoubtedly better to discover programmer errors early in the development
process and this has led to a reliance on type safety over all else.
However, constant expressions illustrate that once compilers are able to detect
undefined behaviour, what was previously seen as a disadvantage now becomes beneficial.
And even if a bug escapes the build process, it is still better to identify it
during testing than let it survive to production.

## Conclusion

In short:

* You can choose bug handling strategy easily, change your mind or use different
  strategies in different builds.
* But you cannot so easily change error-handling strategies.
* Sometimes, a bug in a program isn't a bug, it's an error. This happens as part
  of the Test User Contract and it's important to understand the special properties
  of this contract.
* UB is not something that is confined to the ISO C++ Standard.
* UB is essential for efficiency, flexibility and for finding bugs.
* It is increasingly unreasonable to enable optimisations in software
  without first testing it using sanitizers.

## Acknowledgements

Thanks to Alicja Przybyś and Andrzej Krzemieński for much feedback, correction
and guidance.
