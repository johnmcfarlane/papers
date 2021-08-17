# Contractual Disappointment in C++

Reply-to: John McFarlane <[sg21@john.mcfarlane.name](mailto:sg21@john.mcfarlane.name)>

> "Oh who can say how subtle and safe one feels" (John Betjeman, False Security)

This document offers advice on writing robust programs in C++.
It draws on personal experience in domains including interactive entertainment,
large-scale server systems, and safety-critical devices.

Contracts and their consequences are the main tool used here to frame run-time failure.
Through their lens I attempt to show that careful understanding of interfaces
is the most effective way to ensure usability, efficiency *and* reliability.

## Introduction

### Disappointment

[P0157R0, Handling Disappointment in C++](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0157r0.html),
observes that run-time errors are just one kind of disappointment
that [a C++ program](https://eel.is/c++draft/basic.link#def:program) might elicit.
I argue that all disappointment involves the violation of a contract
and that the correct strategy for handling a violation depends on two things:

1. certain attributes of the contract, and
1. the usage profile of the program.

### Bugs and Errors

Section 4.2 of [P0709R4, Zero-overhead Deterministic Exceptions: Throwing values](https://wg21.link/p0709r4),
"Proposed cleanup: Don’t report logic errors using exceptions",
makes clear the distinction between errors and bugs:

* A _bug_ is the violation of a C++ API Contract or the ISO C++ Standard.
* An _error_, is when an interface couldn’t do what it advertised.

A third category, _abstract machine corruption_, is identified by P0709R4,
which includes heap exhaustion and stack overflow.
These are special cases of disappointment which may be caused by bugs,
e.g. memory leaks and infinite recursion respectively.
Alternatively, they may be errors,
e.g. a program executed on a system with inadequate resources.

There is one time when the bug versus error distinction breaks down.
That is when considering the Test User Contract.
When testing for contract violations,
the user wishes for bugs to be treated as errors.

### Vulnerability

While not a specific concern of this this document,
security depends upon successful handling of disappointment.

[NIST SP 500-268, Source Code Security Analysis Tool Function Specification Version 1.1](https://doi.org/10.6028/NIST.SP.500-268v1.1),
states that a security failure is caused by vulnerabilities.
Applying the terminology of NIST SP 500-268 to this document:

* Failure is a source of disappointment.
* An implementation vulnerability is a bug.
* An operation vulnerability is an error.

Thus, this document is pertinent to the subject of security.

## Contracts

### Contract Attributes

For our purposes, a contract is an _agreement_ between a _user_, and
a _provider_ about the run-time behaviour of some or all of a program.
_User contract violation_ is what most commonly leads to disappointment.

While those four attributes affect how the developer should deal with disappointment,
other attributes bear mentioning.

#### Provider Contract Violation

Both the user and the provider have contractual obligations.
However, this document concentrates on user contract violation for two reasons:

1. It's simpler to focus in on one side of the contract.
2. The user is typically the less experienced and more error-prone of
   the two parties. Therefore, violation by the user is more common.

For example, in the case of the ISO C++ Standard (below),
the toolchain provider is far less likely to be the cause of a contract
violation by virtue of the significant rigour, effort and feedback poured into such
tools.

#### Contract Author

Authors are often also providers, sometimes users and occasionally neither.

### Types of Contracts

In a C++ program, some contracts that matter are as follows.

#### End User Contract

The overarching contract that a program must fulfil is to its end user.
The other contracts exist in support of this fulfilment.

* agreement: program documentation, possibly including a `--help` option
  or a _man_ page
* provider: the program developer
* user: the program user
* violation by user: input such as command-line parameters, UI interaction,
  data files, and network traffic is validated and errors are handled through
  program control flow, typically leading to rejection.

Where possible, rejection should be accompanied by meaningful feedback
which helps the user correct the input and increases the chance of later success.
For example,
meaningful feedback might take the form of a diagnostic emitted on the error stream,
followed by exit with non-zero status.

Input that is not rejected must be incapable of causing violations
of any of the other contracts.
Such violations could result in security vulnerabilities
or unsafe program behaviour in safety-critical applications.
Fuzzers can help to test whether the program fails to reject erroneous input.

#### ISO C++ Standard

A typical language specification for a C++ program is a revision of
[the ISO C++ Standard](https://isocpp.org/std/the-standard).

* agreement: a recognised set of standing documents including a C++ standard
  such as International Standard ISO/IEC 14882:2020(E), technical specifications,
  etc..
* provider: implementer of the language and its accompanying libraries
* user: the program developer
* violation by user: undefined behaviour

Note: the ISO C++ Standard also diagnoses violation by user at compile time.
However, the focus of this document is run-time disappointment.

#### C++ API Contract

Note: For most intents and purposes, the library incorporated into
the ISO C++ Standard can be considered to be a collection of C++ APIs.

* agreement: documentation (including self-documentation)
* provider: C++ API implementer (often also the program developer)
* user: the program developer
* violation by user: undefined behaviour

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
// postcondition: returns square root of a
auto sqrt(double a)
{
  return 42.;
}
```

are outside the scope of this document.
The former is left to the compiler to enforce.
The latter is identified through unit testing and by scrutinising the code.
All things being equal, good APIs favours the former and avoid the latter.

#### Test User Contract

During development, the program — or portions of it —
may be built for testing purposes.

Commonly, tests aim to detect provider contract violations of

* the End User Contract using whole-program or black box testing
  (e.g. functional testing), and
* the C++ API Contract using unit tests, which exercise individual APIs.

However, it is also important to test for user contract violations of
the ISO C++ Standard, and C++ API Contract.
This can be achieved using assertions and toolchain-specific tools:

* Assertions can be configured to trap provider *and* user C++ API Contract violations.
* Dynamic analysis tools such as sanitizers can trap
  ISO C++ Standard violation by the program developer.

Where testing leads to trapping of a contract violation,
this should be treated in the same way that user errors are treated
in the End User Contract, e.g.
by terminating with a helpful diagnostic and non-zero exit status.

***Important:***
It is important to understand that what is considered a bug outside of testing
(e.g. signed integer overflow or performing binary search on an unsorted sequence)
now becomes merely an error.
Bugs written by the program developer which are trapped during testing
are *not* a violation of the Test User Contract.

* agreement: documentation of dynamic analysis tools and/or sanitizers
* provider:
  * analysis tool providers whose tools flag violations — especially of the
    ISO C++ Standard — or
  * C++ API Contract providers who are encouraged to assert that their APIs are used
    correctly.
* user: an engineer who may be some combination of
  * the program developer testing a C++ API Contract using unit tests,
  * the test engineer testing the End User Contract, or
  * the dev-ops engineer testing either of the above as part of a CI pipeline
* violation by user:
  * If a C++ API Contract violation could reasonably be identified (e.g. through
    an assertion) but is not, then the program developer failed to use the tool effectively.

Examples of contract trapping techniques that are commonly employed are

* API contract violation tests (e.g. assertions, pre-conditions and post-conditions),
* API contract fulfilment tests (e.g. unit tests), and
* ISO C++ Standard contracts (e.g. by enabling sanitizers
  and flags such as `_GLIBCXX_ASSERTIONS` and `_ITERATOR_DEBUG_LEVEL`).

### Types of Contract Violation

From the above contracts, we can identify interesting categories.

#### Dynamically-Enforceable C++ API Contracts

This is the subset of C++ API Contracts which it is impossible to test at compile
time, but which it is possible to test at run-time.

* agreement: documentation (ideally self-documentation)
* provider: C++ API implementer (often also the program developer)
* user: the program developer
* violation by user: undefined behaviour, trappable

#### Unambiguous Bugs

This contractual category is the constituency of violations from

* ISO C++ Standard, and
* Dynamically-Enforceable C++ API Contracts

that can be identified programmatically and unambiguously at run-time.

* agreement: constituent contract agreements
* provider: constituent contract providers
* user: the program developer
* violation by user: undefined behaviour, trappable

Examples of violations include

* behaviour designated as undefined behaviour by the ISO C++ Standard, e.g.
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
(aside from those related to computational complexity).

Further, a subset of them can be identified through static analysis,

```c++
int main()
{
  return 1/0;
}
```

and in a constant expressions, they can become compiler errors:

```c++
constexpr auto a{1/0};
```

## Unambiguous Bugs in Detail

### Unambiguous Bug Strategies

Broadly, there are four strategies that can be applied
in situations where a violation might occur in a running program.
They can be followed more effectively with help from modern toolchains.
Examples of features used to deal with bugs can be found in Appendix A.

#### Trap Enforcement Strategy

The program can stop. (It may also emit a diagnostic message or break in a debugger.)

Dynamically-Enforceable ISO C++ Standard violations require
system or toolchain support to trap. For example,
null pointer dereferences may be trapped by systems with virtual memory, and
out-of-bounds array lookup may be trapped by dynamic analysis tools such as sanitizers.

Dynamically-Enforceable C++ API Contract violations must be trapped using assertions.

Disadvantages:

* Code may need to be instrumented, incurring run-time cost.
* On violation, the program completely fails in its task.

Advantages:

* No further incorrect behaviour will be exhibited by the program;
  the user is protected from unbounded risk.
* The developer is compelled to confront bugs as a priority.

Because of these advantages, trapping makes a good default strategy.
This is the default chosen by [the `assert` macro](https://en.cppreference.com/w/cpp/error/assert)
and by most sanitizers.

#### Nonenforcement Strategy

The program continues past the bug, effectively ignoring the contract violation.
For ISO C++ Standard contract violations, neither optimisation nor instrumentation
is enabled.
For C++ API Contract violations, assert statements behave as if they are not there:

```c++
#define ASSERT(condition) ((void)0)
```

Advantages:

* Legacy programs in production can continue to function as before.
* Lack of run-time checking leads to a fairly lean binary.

Disadvantages:

* Tools have limited ability to identify bugs
  because contract violation is effectively normalised.
* Any violation means the program exhibits UB.
* It is unsafe for the compiler to assume the contract is not violated.

Nevertheless, it is a popular choice in some domains
where the cost of fixing or terminating a program is great
and the risk of continuing with undefined behaviour is low.

Some optimisations assume software is free from Unambiguous Bugs.
It is unwise to enable any such optimisations in a program which
applies this strategy to any contract violation.

#### Log-And-Continue Strategy

Additional to the Nonenforcement Strategy, the program can emit a run-time diagnostic.

Advantages:

* Developers can gain feedback helpful in addressing defects.

Disadvantages:

* The disadvantages of the Nonenforcement Strategy apply.
* Diagnostics may increase code size considerably.

#### Prevention Enforcement Strategy

The program can assume that defects have been eliminated.
Assuming bugs do not occur, no contract check will fail,
so no such check is required at run-time.
Further, users can communicate this assumption to the compiler
in the form of optimisation flags.

Finally, user-defined assertions may also communicate this assumption to the optimiser:

```c++
#define ASSERT(condition) ((condition) ? static_cast<void>(0) : __builtin_unreachable())
```

Advantages:

* Optimiser is given hints which are often necessary to fulfil
  the zero-overhead principle.
* Sanitizer seamlessly maximises the number of violations it is able to trap.

Disadvantages:

* Developers must first test their code thoroughly
  using the Trap Enforcement Strategy and
  taking full advantage of the Test User Contract.
* Developers who skip thorough testing but still enable optimisations may see
  pronounced deterioration in the behaviour of their program.

Concerns about this deterioration are raised in ([P2064R0](https://wg21.link/p2064r0))
suggesting than an alarming amount of defective code reaches production
without first being tested using the Trap Enforcement Strategy.
It is always unwise to release software that contains undefined behaviour.
Optimising such programs only throws fuel on an already-lit fire.

### Enforcement Profiles

Now that we have established some Unambiguous Bug Strategies,
how do we choose between them?
We can use examples of C++ programs built for different domains
to speculate on which strategies are best suited to different situations.

#### Tester

Somebody who is testing the program, e.g.

* a developer iterating on a feature or investigating a bug,
* a QA/test engineer, or
* a DevOps engineer programming a CI runner to perform automated tests

will prefer Trap Enforcement Strategy because bugs are a likelihood
and performance and stability are secondary concerns.

#### Safety-Critical System With Redundancy

A safety-critical system *with* backup/redundancy
will also prefer Trap Enforcement Strategy
because allowing the program to continue in an incorrect state
is an unacceptable risk.
Examples include life-support systems and autonomous vehicle controllers.

#### Safety-Critical System Without Redundancy

A safety-critical system *without* backup/redundancy
might consider Prevention Enforcement Strategy, or Log-And-Continue Strategy.

However, logging may not be viable for some safety-critical embedded controllers.
Such minimal systems may not have the facility to log errors or to off-board them.

And despite thorough testing, doubts regarding Unambiguous Bugs may linger.
Optimisations which assume fulfilment of contract violations
may represent a small — but unnecessary — risk.

Thus, Nonenforcement Strategy or Log-And-Continue Strategy are preferable.

#### Performance-Critical/Resource-Constrained

Applications which are performance-critical or resource-constrained
may favour Prevention Enforcement Strategy.

Applications include

* embedded controllers,
* high-frequency trading,
* image processing,
* interactive entertainment,
* machine learning, and
* scientific simulation.

#### Business-Critical Systems

Where business interests are at stake but user safety is not,
the program developer has leeway to choose from a wider range of strategies.
If they are prepared to perform thorough testing, the Prevention Enforcement Strategy
will reap financial benefits in terms of compute costs and reliability.

But where competing financial concerns limit the effort put into defect prevention
either the Trap Enforcement Strategy or The Log-And-Continue Strategy may be preferable.

For example, a RESTful server which persists between requests
may eventually identify bad state that has lain unnoticed for a while.
The developer can choose what happens on discovery:

* Kill the process and restart it in the hope that the problem is mitigated.
* 'Soldier on' and hope the problem doesn't cause some catastrophe or other.

This choice will be affected by factors such as the scale of the fleet,
exposure to malicious agents and the cost of incorrect behaviour.
Either way, bugs should not go unnoticed.

## Handling Errors

The End User Contract provider must account for user contract violations.
In other words, the program must handle errors.

Unfortunately, this is not a straightforward task.
A wide variety of contrasting and conflicting error-handling approaches are available.
Section 2 of [P0709R4, Zero-overhead Deterministic Exceptions: Throwing values](https://wg21.link/p0709r4),
"Why do something: Problem description, and root causes"
does a good job of describing the C++ error-handling landscape.

### Software Terrain

Having identified the subset of disappointment that counts as errors,
the problem remains: how to deliver the bad news to the user.
This news must often travel across large sections of the code.
As with bugs, Enforcement Profiles affect the choice of solution.
Additionally, the choice of error handling technique is affected
by other prominent features of the code...

#### Batch Processing

The simplest kind of code involves a finite amount of processing and no user interaction.
Flow of information typically follows flow of control within a call graph
and results — disappointing or otherwise — can be returned on completion.

Entire programs can be written this way.
Diagnostic messages can be emitted at the point where the error is first detected.
And only a failure code need be returned by the `main` function.

#### Realtime Processing

Some programs such as services, applications, control systems, and interactive simulations
execute indefinitely.

Realtime programs undergo an initialisation phase
in which the starting state of the system is established.
During this phase, they behave like a batch process, and
can exit if inputs are ill-formed.

However, once the initialisation phase is ended,
the realtime phase — sometimes called the 'main loop' — is entered.
During the this phase, a realtime program, cannot simply exit on error.
It must do what it can to convey disappointment to the user and carry on.

#### Libraries

Libraries add a new dimension. A library might be used across many programs.
Some of those programs might be batch and some might be realtime.
Thus, either the applicability, or the choice of error-handling strategy is limited.

#### Multithreading

The addition of threads to a program makes matters worse.
Now there may be concurrent initialisation phases and realtime phases.

### Error Handling Techniques

Now that a flavour of the different types of terrain are established,
techniques can be evaluated based on how well they cover different terrains.

#### Diagnostics

While not directly related to sending disappointment back to an external process,
the logging or display of detailed error messages to the user is very important.
A clear message in human-readable form is what the user needs
in order to fix most errors.

In an interactive program, this may be presented in the form of a dialogue box,
an output window or a status bar.
Elsewhere, logging/console output is the medium through which the users is reached.
Printing a message to the error stream is usually adequate.

```c++
std::cerr << std::format("failed to open file, \"{}\"\n", filename);
```

If the program can carry on past this line of code as normal,
what we are dealing with probably isn't an error at all.
If this is an error, typical control flow will be dramatically altered
and further choices will need to be made irrespective of any diagnostics...

#### Return Values

Some programs are simple enough that single scalar return values suffice as
a method for sending news of failure back to the calling process.

```c++
// print file's size or return false
auto file_size(char const* filename)
{
  std::ifstream in(filename, std::ios::binary | std::ios::ate);
  if (!in) {
    std::cerr << std::format("failed to open file \"{}\"\n", filename);
    return false;
  }

  std::cout << std::format("{}\n", in.tellg());
  return true;
}
```

There is a small cost here on every call
and explicit logic in the calling code needs to handle the disappointment.
But it's a tempting choice for batch programs reporting errors via exit codes.
Even in realtime and library code,
there is no more efficient way to report disappointment to the caller.

Additionally, the receiving logic is placed where there is greater context.
Further down the call stack, there are richer diagnostics to be emitted:

```c++
auto config_file_size()
{
  if (!file_size("default.cfg")) {
    std::cerr << "failed to get the size of the config file";
  }
}
```

A problem arises if the function already returns a result.
C++ functions only return one object
and the most readable functions return the desired result as that object
(e.g. file size as an integer).

```c++
// return file's size
auto file_size(char const* filename)
{
  std::ifstream in(filename, std::ios::binary | std::ios::ate);
  if (!in) {
    std::cerr << std::format("failed to open file \"{}\"\n", filename);
    // how is the disappointment returned now?
  }

  return in.tellg();
}
```

To some degree or other, types such as `expected`, `outcome` and `optional` facilitate
returning of both the desired result combined with the successfulness:

```c++
auto file_size(char const* filename)
-> std::optional<std::ifstream::pos_type>
{
  std::ifstream in(filename, std::ios::binary | std::ios::ate);
  if (!in) {
    std::cerr << std::format("failed to open file \"{}\"\n", filename);
    return std::nullopt;
  }

  return in.tellg();
}
```

#### Exceptions

Exceptions are well-suited to error handling. They keep the 'happy path' free of
error propagation logic which should make them more efficient in the case that nothing
goes wrong. Indeed, [they are often the most efficient solution](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1947r0.pdf)
where there is high confidence that throwing is rare.

Exceptions are also a flexible solution which allows error handling code to be localised
within the call graph: imagine a program which needs to report defects differently
in separate sections of the code. In one section, errors are logged to a file. In
another section, errors are reported to a bug-tracking server.
And in both sections, the program is not allowed to access any IO,
so emitting a diagnostic at moment of initial disappointment isn't possible.
Such constraints can be easily overcome with one or two `catch` blocks.

However, such versatility is rarely necessary in real applications. And exceptions
bring other costs.

* Exceptions require additional [space](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1640r1.html),
  which can render them impractical on small systems such as microcontrollers,
* When thrown, they are costly in [time and determinism](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1886r0.html),
  which can break real-time system constraints, violating the End User Contract.
* Even when not thrown, they can make it difficult to reason about control flow,
  which has implications for efficiency.

For batch programs on well-resourced systems, they are a very good choice.
For resource-constrained or realtime systems or libraries which target those systems
the costs and benefits of exception handling should be considered.

#### Abnormal Program Termination

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
    fatal("Wrong number of arguments provided. Expected={}; Actual={}\n", expected_num_params, actual_num_params);
  }
```

Abnormal program termination is often shunned in C++ programs.
The main reason is that it typically bypasses destructors.
That may not be the biggest worry in a program which is already in a bad
state, so terminating in reaction to Unambiguous Bugs is relatively palatable.
And destructors in a modern, well designed system are only important
while the process is running:
memory, file descriptors and peripherals should all be freed up by the system
once the owning process is ended.

So this can often be the best approach for reacting to violations of the
End User Contract — as well as to Unambiguous Bugs. In profile,
Safety-Critical System With Redundancy, the requirement to 'fail fast' can be well
served by this approach. And in profile, Business-Critical Systems, the concerns
associated with bypassing destructors may not be significant.

#### Push UB Onto The End User

The program might be part of a larger system which is internal to a group of developers.
For example, essential data might be stored in a read-only filesystem.
Reliance on this filesystem might be a given. In such a circumstance,
why not simply assume that the filesystem is there? Why not treat its absence
as if it were undefined behaviour?

In other words, why not apply the Prevention Enforcement Strategy to errors?
Why not assume that the End User Contract is never violated?

This is entirely possible but the implications must be fully understood.
There will be reduced freedom to invoke the program with ill-formed input.
There is heightened risk of compromise, corruption or critical failure if the
program is ever invoked outside the parameters of the End User Contract.
Even when it is possible, it is rarely worthwhile taking this risk.

## C++ API Author Strategies

We will briefly explore some ways to design our C++ API Contracts
so they are set up for success.

### The Pit of False Security

When the advertised goal of a C++ API cannot be achieved,
it should be difficult for the user of the API to ignore the disappointment.
Defending against failure by allowing the caller to proceed as normal is unhelpful.

Example 1:

```c++
auto load_file(std::string filename) -> std::vector<std::byte>
{
  std::ifstream file{filename};
  if (!file) {
    // bad! missing is not the same as empty
    return std::vector<std::byte>{};
  }
  ...
}
```

Example 2:

```c++
auto car::pull_away(color traffic_light)
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

The best documentation is the API itself.
In a strongly-typed language, the types of a declaration are a powerful form of communication.

Example 1 revisited:

```c++
auto load_file(std::string filename) -> std::optional<std::vector<std::byte>>
{
  std::ifstream file{filename};
  if (!file) {
    // better; user is made aware that the result isn't always some bytes
    return std::nullopt;
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
  the bug, e.g. with assertions.

Example 2 corrected:

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

Note that the provider might be tempted to place the assertion in a `default` clause.
This is not advised. There is no acceptable default behaviour here
so it's better not to express the intent that there is.

## Example Program

The following excerpts from the program hosted
[here](https://github.com/johnmcfarlane/eg-error-handling/blob/6bee393c245debf4ef921f8518b1b6a89a477b25/src/main.cpp),
illustrate the recommendations collected in this document.

### Assertion Logic

Here is an example assert function which illustrates different Enforcement Profiles.
It is compatible with some modern tool-chains.

```c++
constexpr void eg_assert(bool condition)
{
  if (condition) {
    return;
  }

#if defined(TRAP_STRATEGY) // Trap Enforcement Strategy
  // stop the program
  std::terminate();
#elif defined(NONENFORCEMENT_STRATEGY) // Nonenforcement Strategy
  // do nothing
#elif defined(LOG_AND_CONTINUE_STRATEGY) // Log-And-Continue Strategy
  // print a helpful diagnostic
  std::fputs("a C++ API violation occurred\n", stderr);
#elif defined(PREVENTION_STRATEGY) // Prevention Enforcement Strategy
  // make it very clear that the program must not reach this line
  __builtin_unreachable();
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
Some of those conditions can be expressed in the code itself.
Some languages support formal expression as part of the interface.
Many languages support expression of conditions within the API implementation.

The `eg_assert` invocations above are examples of expressions within the implementation.

### Calling a C++ API

In general, C++ APIs should not attempt to handle out-of-contract invocation.
As with the `pull_away` example above, normalising bugs has many negative consequences.
The set of possible violations is likely to be vast
so logic to deal with it can dilute the essential code of a function.

The contract of `sanitized_run`,

```c++
// precondition: Requires validated data, i.e. number in the range 1<=number<=26.
void sanitized_run(int number)
{
  fmt::print("{}", number_to_letter(number));
}
```

with respect to `number` is no different to that of `number_to_letter`.
They both require the same range of values.
So even an assertion is of little value here.

### Validating Input

Code which accepts external input into the program is a very different matter.

The developer shouldn't assume that program input is error-free.
Even if the users of the program are 'friendly', they may make mistakes which the
program developer did not anticipate.

In order to be confident that no Unambiguous Bugs can occur,
input must first be checked.
If the possible violations by the user of the End User Contract are not tested,
out-of-bounds array access, invalid pointers and past-the-end iterators may all result.

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
It can be difficult to determine until quite far in the process whether input is
well-formed. But as a general rule, it is better to fail fast.

This can be seen as analogous to digestion:
External matter is ingested, harmful or unwanted matter is rejected,
and if anything remains, it is accepted into the organism and processed.
The sooner poisons are rejected, the less harm they can do.

Because of the effort involved in validating input, it is highly recommended
that developers leverage mature, 3rd-party libraries designed for this purpose.
Command-line argument parsers such as [Lyra](https://github.com/bfgroup/Lyra)
can be used to replace most of `unsanitized_run`.
Libraries that handle data interchange formats can also save effort and reduce risk.

However, application-agnostic libraries
can only transforms data from one format to another.
There may still be errors in the input which are application-specific.
For example, the range check of `number` in `unsanitized_run` must still be performed.

### Type Safety is King

Well-written APIs mean that most bugs do not survive compilation.
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

### C++ API Contract Violation is Undefined Behaviour

It is a common misconception that the term _undefined behaviour_
refers only to violations of some subset of the ISO C++ Standard.

The standard [describes](https://eel.is/c++draft/defns.undefined) UB as:

> behavior for which this document imposes no requirements

This implies that, the standard does *not* exclude, from its definition of UB,
violations of contracts outside of the ISO C++ Standard.

A more accurate and helpful view of UB is that it is one possible result of
a violation of any contract that occurs at run-time,
for which consequences are not described.

Part of the confusion is the belief that the definition behind an interface
grants the user licence to infer a broader contract.
Let's look again at the function `number_to_letter`.

```c++
constexpr auto min_number{1};

// precondition: number is in range [1..26]
constexpr auto number_to_letter(int number)
{
  return char(number - min_number + 'A');
}
```

It so happens that we can see the definition of `number_to_letter`. Even if we couldn't,
we might feel confident in guessing roughly how it was implemented.
And we might feel confident to say that the function does not
exhibit undefined behaviour — even when the precondition is violated.

This is categorically false.

Firstly, the author has intentionally limited the 'warranty' of the contract.
They are declining to guarantee the result in the event of contract violation.
Put another way, they decline to define the behaviour.

Secondly, the provider is under no obligation
to have tested the behaviour outside of contract.
So any and all assurances gathered through automated testing
do not apply to out-of-contract use.
And in the absence of testing, it's easy to miss a failure case,
[e.g.](https://godbolt.org/z/5e3cvj95f):

```c++
// signed integer overflow violates ISO C++ Standard, is already UB
number_to_letter(0x7fffffff);
```

Finally, the provider reserves the right to change the implementation as they choose.
Assumptions drawn from the original definition are already false.
But they become yet more dangerous when applied to a new definition. For example:

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
UB is just a way for authors to limit contracts so that providers can deliver
better implementations.

### Wide Or Narrow?

In the above example, `number_to_letter` is what is described as a narrow contract.
The user of the API can — but must not — deviate from the expected usage.

One way to widen `number_to_letter`'s contract would be:

```c++
// returns corresponding letter iff number is in range [1..26]
constexpr auto number_to_letter(int number)
-> std::optional<char>
{
  if (number < min_number || number > max_number) {
    return std::nullopt;
  }

  return char(number - min_number + 'A');
}
```

The code is now likely to be slower to comprehend, to compile and to execute.
The API is more difficult to use and the user is encouraged to deal at run-time
with bugs which should simply have been removed.
In general, wide contracts are the wrong choice for components of any
fully-digital, automated system.

But when a human is in the loop, a wide contract is an essential UI feature.
Humans often require feedback in order to identify mistakes.
Humans participate in two contracts related to running programs:

* the program user in the End User Contract, and
* the engineer in the Test User Contract.

Paradoxically, the Test User Contract relies on the narrowness of
the C++ API Contract in order to provide a wide contract of its own.
Thus by widening a C++ API Contract,
the contract author impedes the engineer's ability to identify bugs.

### Don't Optimize Until You Sanitize

C++ toolchains increasingly optimise programs by assuming their correctness.
These optimisations are not free.
They are provided to users in exchange for not violating the ISO C++ Standard.

Historically, such contract violations have been difficult to detect.
Fortunately, modern toolchains provide facilities for their detection.
These include dynamic analysis tools known as sanitizers
which instrument code to test for bugs at run-time.
The main benefit of undefined behaviour is that it allows such tools
to accurately identify bugs.

Meanwhile, modern software development practices emphasise automated testing regimes.
It is not uncommon for most of the APIs in a program
to be tested for provider contract violation frequently during development.
Such tests are an opportunity to enable sanitizers.

If you wish to enable compiler optimisations in the program you provide to users
you are strongly advised to ***test your program code with sanitizers***.
Combined with the approaches detailed in this document, this will lead to significantly
fewer software defects without the need to compromise safety or performance.

If your toolchain provider doesn't offer sanitizers in its latests products,
question whether it is of suitable quality.

Notably, the author's experience is that many tools which are certified for use with
safety-critical software are often outdated versions of free open-source software,
lacking years' worth of new features and fixes.
They rely on a business model which predates modern software practices —
such as incremental releases, regression testing and test-driven development.
They tend to deliver substandard products as a consequence.

As a professional, you must decide whether they are acceptable for the task at hand.

### Use All the Tools

As well as enabling sanitizers and static analysers and elevating compiler warnings,
using multiple toolchains will increase the range of bugs for which you test.
Even if your target compiler does not have those facilities, some (though not all)
bugs will be detected by testing your software on consumer-grade system using
free operating systems and toolchains, or in emulators that more closely resemble
the target platform.

### Don't Rely on Sanitizers Alone

Unfortunately, sanitizers are far from perfect.
Some unambiguous bugs are prohibitively difficult to detect at run-time.

Sometimes, sanitizers fail to instrument for UB as advertised.
For example, if the logic which recognises the possibility of UB
is located in the compiler's optimiser, and if optimisation is not enabled,
instrumentation may not occur. For this reason,
it is important to test code with settings that are as close to release as possible.

Finally, not all bugs are undefined behaviour.
Some bugs violate only the End User Contract.
They are sometimes described as bugs in the 'business logic' of the program.
Because of this, unit testing will always play a vital role in a
healthy development process.

### A Carefully Controlled Vagueness

Nevertheless, the notion of Unambiguous Bugs is every bit as important as type safety.
Certainly, it is better to discover bugs early in the development process
and this is why static typing is so beneficial.
However, discovering bugs during program execution is a necessary fallback.

If behaviour of a program is defined at the point where an Unambiguous Bug occurs,
the behaviour must be diagnostic in nature. Otherwise, the bug is

* harder for tools to identify,
* harder for engineers to identify,
* no longer diagnosed in constant expressions, and
* normalised as OK (as per [Hyrum's Law](https://www.hyrumslaw.com/)).

There is often a 'least worst' defined behaviour for a bug,
e.g. signed integer modulo behaviour, or a valid-but-wrong iterator value
returned from a binary search performed on an unordered sequence.
Often, these are the default behaviour in unoptimised configurations.

Bespoke behaviour definitions can be locked in, e.g. using GCC's `-fwrapv` flag.
They prevent detection of bugs and may pessimise bug-free programs.
But where bugs slip through the net (which they do), they can reduce attack vectors.
***Nothing about UB prevents toolchains from implementing these behaviours.***
Feel free to seek them out as a last line of defence in safety-critical applications.

## Conclusion

In short:

* Use assertions to formally express C++ API Contracts.
* Then, you can easily choose a bug handling strategy, change your mind, or use different
  strategies for different contracts and in different builds.
* You cannot so easily change error-handling strategies.
* Sometimes, a bug in a program isn't a bug; it's an error. This happens as part
  of the Test User Contract, and it's important to understand the irregularities
  of this contract.
* UB is not something that is confined to the ISO C++ Standard.
* UB is essential for efficiency, flexibility and for finding bugs.
* It is increasingly unreasonable to enable optimisations in software
  without first testing it using sanitizers.

## Acknowledgements

Thanks to Alicja Przybyś and Andrzej Krzemieński for much feedback, correction
and guidance.

## Appendix A - Toolchain-Specific Recommendations

This section introduces some of the toolchain-supported features
which can be used to help enforce the Unambiguous Bug Strategies

* **Trap** Enforcement Strategy,
* **Non**enforcement Strategy,
* **Log**-And-Continue Strategy, and
* **Pre**vention Enforcement Strategy,

using three popular compilers:

* Clang,
* GCC, and
* MSVC.

To some extent, different strategies can be applied to different bugs
within the same build of a program.
For example, the developer may wish to trap C++ API Contract violations
but prevent ISO C++ Standard violations.
Or they may wish to prevent all user contract violations with the sole
exception of signed integer overflow.

Regardless, this guide is only an example of the choices available in modern toolchains.
A clear understanding of the interplay between the features of your individual toolchain,
and the contracts they affect, are both required to successfully handle disappointment.

<table>
  <tr>
    <td><b><i>flag</i> or <code>intrinsic</code></b></td>
    <td><b>Clang</b></td>
    <td><b>GCC</b></td>
    <td><b>MSVC</b></td>
    <td><b>Trap</b></td>
    <td><b>Non</b></td>
    <td><b>Log</b></td>
    <td><b>Pre</b></td>
    <td><b>Description</b></td>
  </tr>

  <tr>
    <td><i>-Werror</i></td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td>✓</td>
    <td>✓</td>
    <td>✓</td>
    <td>✓</td>
    <td>turn warnings into errors</td>
  </tr>

  <tr>
    <td><i>/WX</i></td>
    <td></td>
    <td></td>
    <td>✓</td>
    <td>✓</td>
    <td>✓</td>
    <td>✓</td>
    <td>✓</td>
    <td>turn warnings into errors</td>
  </tr>

  <tr>
    <td><i>-Wall</i>, <i>-Wextra</i> and <i>-Wpedantic</i></td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td>✓</td>
    <td>✓</td>
    <td>✓</td>
    <td>✓</td>
    <td>enable many warnings</td>
  </tr>

  <tr>
    <td><i>/W4</i></td>
    <td></td>
    <td></td>
    <td>✓</td>
    <td>✓</td>
    <td>✓</td>
    <td>✓</td>
    <td>✓</td>
    <td>enable many warnings</td>
  </tr>

  <tr>
    <td><i>-D_LIBCPP_ENABLE_NODISCARD</i></td>
    <td>✓</td>
    <td></td>
    <td></td>
    <td>✓</td>
    <td>✓</td>
    <td>✓</td>
    <td>✓</td>
    <td>enable some warnings</td>
  </tr>

  <tr>
    <td><i>-fsanitize=undefined,address</i> etc.</td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td>✓</td>
    <td></td>
    <td></td>
    <td></td>
    <td>trap ISO C++ Standard user contract violations<sup>†</sup></td>
  </tr>

  <tr>
    <td><i>-fwrapv</i></td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td>✓</td>
    <td></td>
    <td></td>
    <td></td>
    <td>trap signed integer overflow</td>
  </tr>

  <tr>
    <td><i>-D_LIBCPP_DEBUG=1</i></td>
    <td>✓</td>
    <td></td>
    <td></td>
    <td>✓</td>
    <td></td>
    <td></td>
    <td></td>
    <td>trap Standard Library user contract violations</td>
  </tr>

  <tr>
    <td><i>-D_GLIBCXX_ASSERTIONS</i></td>
    <td></td>
    <td>✓</td>
    <td></td>
    <td>✓</td>
    <td></td>
    <td></td>
    <td></td>
    <td>trap Standard Library user contract violations</td>
  </tr>

  <tr>
    <td><i>/D_ITERATOR_DEBUG_LEVEL=2</i></td>
    <td></td>
    <td></td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td></td>
    <td></td>
    <td>trap Standard Library user contract violations</td>
  </tr>

  <tr>
    <td><code>__builtin_unreachable()</code></td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td>✓</td>
    <td></td>
    <td></td>
    <td>✓</td>
    <td>flag Unambiguous Bugs to compiler<sup>‡</sup></td>
  </tr>

  <tr>
    <td><code>__assume(false)</code></td>
    <td></td>
    <td></td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td></td>
    <td>✓</td>
    <td>flag Unambiguous Bugs to compiler<sup>‡</sup></td>
  </tr>

  <tr>
    <td><i>-DNDEBUG</i></td>
    <td>✓</td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td>✓</td>
    <td>✓</td>
    <td>✓</td>
    <td>disable <code>assert</code> macro</td>
  </tr>

  <tr>
    <td><i>-O0</i></td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td></td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td>disable optimisations<sup>&#42;</sup></td>
  </tr>

  <tr>
    <td><i>/Od</i></td>
    <td></td>
    <td></td>
    <td>✓</td>
    <td></td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td>disable optimisations<sup>&#42;</sup></td>
  </tr>

  <tr>
    <td><i>-fwrapv</i></td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td></td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td>disable signed integer overflow</td>
  </tr>

  <tr>
    <td><i>-O</i>, <i>-O1</i>, <i>-O2</i>, <i>-O3</i>, <i>-Os</i>, <i>-Ofast</i>
or <i>-Og</i></td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td>✓</td>
    <td></td>
    <td></td>
    <td>✓</td>
    <td>optimise code</td>
  </tr>

  <tr>
    <td><i>/O1</i>, <i>/O2</i>, <i>/Os</i>, <i>/Ot</i> or <i>/Ox</i></td>
    <td></td>
    <td></td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td></td>
    <td>✓</td>
    <td>optimise code</td>
  </tr>

</table>

Notes:

† Use sanitizer flags ***in addition to*** optimisation flags
to trap common occurrences of ISO C++ Standard user contract violations.
Additional [Clang sanitizers](https://clang.llvm.org/docs/index.html) are dedicated
to catching bugs at run-time. Some of them work with GCC as well as Clang.
Tools like Valgrind can also catch some of these bugs.

‡ When used as part of the Trap Enforcement Strategy,
must be used ***instead of*** explicit bug trapping code,
and ***in conjunction with*** a sanitizer such as UBSan.

&#42; Some optimisations may be safely enabled without further degrading
behaviour of defective code, e.g. `-finline-functions` and `/Ob2`.
