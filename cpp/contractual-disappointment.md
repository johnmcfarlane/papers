# Contractual Disappointment in C++

Reply-to: John McFarlane <[sg21@john.mcfarlane.name](mailto:sg21@john.mcfarlane.name)>

> “make it run, make it right, make it fast” (Kent Beck's dad, circa 1975)

## Introduction

Writing robust software in C++ is challenging.
Effective use of modern tools and guidelines helps a great deal.
And successive revisions of the language and the standard library
add features which help the user prevent problems.
But inevitably, upsets happen during program operation.
This document examines what program developers can do to help.

Contracts and their implications are the main tool used here to frame run-time failure.
Through their lens I attempt to show that careful understanding of interfaces
is the most effective way to ensure usability, efficiency *and* reliability.

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
makes clear the distinction between bugs and errors.
In terms of contracts:

* A _bug_ is the violation of the [C++ API Contracts](#c-api-contracts)
  or the [C++ Standard](#c-standard).
* An _error_, is when an interface couldn’t do what it advertised.

A third category, _abstract machine corruption_, is identified by P0709R4,
which includes heap exhaustion and stack overflow.
These are special cases of disappointment which may be caused by bugs,
e.g. memory leaks and infinite recursion respectively.
Alternatively, they may be errors,
e.g. a program executed on a system with inadequate resources.

There is one occasion when the bug versus error distinction breaks down.
That is when considering the [Test User Contract](#test-user-contract).
When testing for contract violations,
the user wishes for bugs to be treated as errors.

### Vulnerability

While not a specific concern of this this document,
security depends upon successful handling of disappointment.

[NIST SP 500-268,](https://doi.org/10.6028/NIST.SP.500-268v1.1)
[Source Code Security Analysis Tool Function Specification Version 1.1](https://doi.org/10.6028/NIST.SP.500-268v1.1),
states that a security failure is caused by vulnerabilities.
Applying the terminology of NIST SP 500-268 to this document:

* Failure is a critical source of disappointment.
* An implementation vulnerability is a bug.
* An operation vulnerability is an error.

Failure of developers to achieve the goals aimed for in this document
is a common cause of vulnerabilities.

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
1. The user is typically the less experienced and more error-prone of the two parties.

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

#### C++ Standard

A typical language specification for a C++ program is a revision of
[the ISO C++ Standard](https://isocpp.org/std/the-standard).

* agreement: a recognised set of standing documents including a C++ standard
  such as International Standard ISO/IEC 14882:2020(E), technical specifications,
  etc..
* provider: implementer of the language and its accompanying libraries
* user: the program developer
* violation by user: undefined behaviour

Note: the C++ Standard also diagnoses violation by user at compile time.
However, the focus of this document is run-time disappointment.

#### C++ API Contracts

* agreement: documentation (including self-documentation)
* provider: C++ API implementer (often also the program developer)
* user: the program developer
* violation by user: undefined behaviour

It is helpful to divide [C++ API Contracts](#c-api-contracts) into three subcategories:

1. statically-enforceable,

   ```c++
   double add(double a, double b);

   auto f()
   {
     return add("2", "2");
   }
   ```

1. manually-enforceable,

   ```c++
   void print(char const* str, int str_len);

   auto f()
   {
     // print the word 'three'
     print("three", 3);
   }
   ```

1. and dynamically-enforceable:

   ```c++
   double sqrt(double x);

   void f()
   {
     sqrt(-4);
   }
   ```

The first is the most desirable: the compiler can enforce developer mistakes.
The second is the least desirable:
the developer catches these through unit testing and code review.
The third, [Dynamically-Enforceable C++ API Contracts](#dynamically-enforceable-c-api-contracts),
carries challenges for enforcement and opportunities for optimisation.
It is discussed in more detail below.

Note: For most intents and purposes, the library incorporated into
the [C++ Standard](#c-standard) can be thought of as a collection of C++ APIs.

#### Test User Contract

During development, the program — or portions of it —
may be built for testing purposes.

Commonly, tests aim to detect _provider_ contract violations of

* the [End User Contract](#end-user-contract) using whole-program or black box testing
  (e.g. functional tests), and
* [C++ API Contracts](#c-api-contracts) using unit tests,
  which exercise individual APIs.

However, it is also important to test for _user_ contract violations of
the [C++ Standard](#c-standard), and [C++ API Contracts](#c-api-contracts).
This can be achieved using assertions and toolchain-specific tools:

* Assertions can be configured to trap provider *and*
  user [C++ API Contracts](#c-api-contracts) violations.
* Dynamic analysis tools such as sanitizers can trap
  [C++ Standard](#c-standard) violations by the program developer.

Where testing leads to trapping of a contract violation,
this should be treated in the same way that user errors are treated
in the [End User Contract](#end-user-contract), e.g.
by terminating with a helpful diagnostic and non-zero exit status.

***Important:***
It is important to understand that what is considered a bug outside of testing
(e.g. signed integer overflow or performing binary search on an unsorted sequence)
now becomes merely an error.
Bugs written by the program developer which are trapped during testing
are *not* a violation of the [Test User Contract](#test-user-contract).

* agreement: documentation of dynamic analysis tools and/or sanitizers
* provider:
  * analysis tool providers whose tools flag violations — especially of the
    [C++ Standard](#c-standard) — or
  * [C++ API Contracts](#c-api-contracts) providers who are encouraged
    to assert that their APIs are used correctly.
* user: an engineer who may be some combination of
  * the program developer testing [C++ API Contracts](#c-api-contracts)
    using unit tests,
  * the test engineer testing the [End User Contract](#end-user-contract), or
  * the dev-ops engineer testing either of the above as part of a CI pipeline
* violation by user:
  * If a [C++ API Contracts](#c-api-contracts) violation could reasonably be identified
    (e.g. through an assertion) but is not,
    then the program developer failed to use the tool effectively.

### Types of Contract Violation

From the above contracts, we can identify interesting categories.

#### Dynamically-Enforceable C++ API Contracts

This is the subset of [C++ API Contracts](#c-api-contracts)
which it is impossible to test at compile time,
but which it is possible to test at run-time.

* agreement: documentation (ideally self-documentation)
* provider: C++ API implementer (often also the program developer)
* user: the program developer
* violation by user: undefined behaviour, trappable

#### Unambiguous Bugs

This contractual category is the constituency of violations from

* [C++ Standard](#c-standard), and
* [Dynamically-Enforceable C++ API Contracts](#dynamically-enforceable-c-api-contracts)

that can be identified programmatically and unambiguously at run-time.

* agreement: constituent contract agreements
* provider: constituent contract providers
* user: the program developer
* violation by user: undefined behaviour, trappable

Examples of violations include

* behaviour designated as undefined behaviour by the [C++ Standard](#c-standard),
  e.g.
  * integer divide-by-zero,
  * out-of-bounds array lookup,
  * out-of-lifetime object access,
  * calling `std::vector::front()` on an empty object,
  * calling `upper_bound` on an unsorted sequence, and
  * passing iterators from separate sequences to `std::for_each`, and
* [C++ API Contracts](#c-api-contracts) violations for which a test
  could be expressed in-code.

The cost involved in testing for these violations varies.
But in theory, all of them can be automatically checked
without changing the behaviour of the program or violating other contracts
(aside from those related to computational complexity).
And it is reasonable to test many of them during development.

Further, a subset of them can be identified through static analysis,

```c++
int main()
{
  return 1/0;
}
```

and in constant expressions, they become compiler errors:

```c++
constexpr auto a{1/0};
```

## Unambiguous Bugs in Detail

### Unambiguous Bug Strategies

Broadly, there are four strategies that can be applied
in situations where a violation might occur in a running program.
Modern toolchains can help in following these strategies.
(Examples of features used to deal with bugs can be found in Appendix A.)

#### Trap Enforcement Strategy

The program can stop. (It may also emit a diagnostic message or break in a debugger.)

To trap Dynamically-Enforceable [C++ Standard](#c-standard) violations requires
system or toolchain support. For example,
null pointer dereferences may be trapped by systems with virtual memory, and
out-of-bounds array lookup may be trapped by dynamic analysis tools such as sanitizers.

[Dynamically-Enforceable C++ API Contracts](#dynamically-enforceable-c-api-contracts)
violations should be trapped using assertions.

Disadvantages:

* On violation, the program completely fails in its task.
* Code may need to be instrumented, incurring run-time cost.
* Some violations are prohibitively expensive to trap.

Advantages:

* When detected, no further incorrect behaviour will be exhibited by the program;
  the user is protected from unbounded risk.
* Feedback is timely and targetted, the developer is compelled to address bugs promptly
  and they are empowered to avoid mistakes in future.
* Automated tests can identify a great many [Unambiguous Bugs](#unambiguous-bugs)
  in code that uses modern tools and guidelines
  (such as the [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)).
* Fuzz testing relies on [Unambiguous Bugs](#unambiguous-bugs)
  to identify implementation vulnerabilities.

#### Nonenforcement Strategy

The program continues past the bug, effectively ignoring the contract violation.
For [C++ Standard](#c-standard) contract violations, neither optimisation nor instrumentation
is enabled.
For [C++ API Contracts](#c-api-contracts) violations,
assert statements behave as if they are not there:

```c++
#define ASSERT(condition) ((void)0)
```

Advantages:

* Legacy programs in production can continue to function as before.
* Lack of run-time checking leads to a fairly lean binary.

Disadvantages:

* Tools have limited ability to identify bugs
  because contract violation is effectively normalised.
* [C++ API Contracts](#c-api-contracts) violations in constant expressions
  are not compiler errors.
* Any violation results in UB.
* It is unsafe to enable compiler optimisations that assume contracts are not violated.

#### Log-And-Continue Strategy

Additional to the [Nonenforcement Strategy](#nonenforcement-strategy),
the program can emit a run-time diagnostic.

Advantages:

* The advantages of the [Nonenforcement Strategy](#nonenforcement-strategy) apply.
* Developers can gain feedback helpful in addressing defects.

Disadvantages:

* The disadvantages of the [Nonenforcement Strategy](#nonenforcement-strategy) apply.
* Diagnostic code may increase binary size considerably.

#### Prevention Enforcement Strategy

The only truly safe course of action is to eliminate all contract violations.
While it's wise to assume that all programs contain bugs,
their incidence can reduce dramatically by

* observing modern practices - as documented in guidelines like the
  [CCG](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)
  and enforced by analysers like
  [Clang-Tidy](https://clang.llvm.org/extra/clang-tidy),
* fulfilling the [Test User Contract](#test-user-contract) by writing asserts, and
* following a thorough testing regime, while
* taking full advantage of the [Trap Enforcement Strategy](#trap-enforcement-strategy).

The benefits of zero bug tolerance make Prevention Enforcement Strategy compelling.
Besides safety and security, opportunities for optimisation present themselves:

* Assuming assertions hold means that their run-time checks can be elided.
* Assuming no violation of the [C++ Standard](#c-standard)
  allows for aggressive optimisation.
* Through compiler hints, the same optimisations can be applied to asserts.

For example, assertion macros can use compiler-specific hints, e.g.

```c++
#define ASSERT(condition) ((condition) ? static_cast<void>(0) : __builtin_unreachable())
```

* to trap [C++ API Contracts](#c-api-contracts) violations using UB sanitizers, and
* to streamline C++ API implementations using optimizers.

Advantages:

* The program operates correctly, reliably and securely.
* The program operates with minimum overhead.

Disadvantages:

* Great onus is placed on the developer to ensure bugs are eradicated.
* Failure to minimise bugs before enabling optimisations
  increases the likelihood of observable defects and vulnerabilities.

### Enforcement Profiles

Now that we have established some [Unambiguous Bug Strategies](#unambiguous-bug-strategies),
how do we choose between them?
The answer depends on the End User and the type of application being built.
Some examples follow.

#### Tester

Somebody who is testing the program, e.g.

* a developer iterating on a feature or investigating a bug,
* a QA/test engineer, or
* a DevOps engineer programming a CI runner to perform automated tests

will prefer [Trap Enforcement Strategy](#trap-enforcement-strategy) because bugs
are a likelihood and performance and stability are secondary concerns.

#### Safety-Critical System With Redundancy

A safety-critical system *with* backup/redundancy
will also prefer [Trap Enforcement Strategy](#trap-enforcement-strategy)
because allowing the program to continue in an incorrect state
is an unacceptable risk.
Examples include life-support systems and autonomous vehicle controllers.

#### Safety-Critical System Without Redundancy

A safety-critical system *without* backup/redundancy
might consider Prevention Enforcement Strategy, or Log-And-Continue Strategy.

However, logging may not be viable for some safety-critical embedded controllers.
Such minimal systems may not have the facility to log errors or to off-board them.

And despite thorough testing,
doubts regarding [Unambiguous Bugs](#unambiguous-bugs) may linger.
Optimisations which assume fulfilment of contract violations
may represent a small — but unnecessary — risk.

Thus, [Nonenforcement Strategy](#nonenforcement-strategy) or
Log-And-Continue Strategy are preferable.

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

Where business interests are at stake but safety and resources are less of a concern,
the program developer has leeway to choose from a wider range of strategies.
If they are prepared to perform thorough testing, the Prevention Enforcement Strategy
will reap benefits in terms of compute costs and reliability.

But where competing financial concerns limit investment in prevention,
either the [Trap Enforcement Strategy](#trap-enforcement-strategy) or
The Log-And-Continue Strategy may be preferable.

For example, a RESTful server which persists between requests
may eventually identify bad state in the form of a contract violation.
The developer can choose what happens on discovery:

* Kill the process and restart it in the hope that the problem is mitigated.
* 'Soldier on' and hope the problem doesn't cause some catastrophe or other.

This choice will be affected by factors such as the scale of the fleet,
exposure to malicious agents and the cost of incorrect behaviour.
Either way, bugs should not go unnoticed.

## Handling Errors

The previous section focused on bugs: contract violations by the developer.
In this section, we turn out attention to errors:
contract violations by the End User — who may not be a developer.

Error handling requires more forethought; choices here are harder to reverse.
A wide variety of contrasting and conflicting error-handling approaches are available.
(See Section 2 of [P0709R4, Zero-overhead Deterministic Exceptions: Throwing values](https://wg21.link/p0709r4)
for a good description of the C++ error-handling landscape.)

### Software Terrain

Having identified the subset of disappointment that counts as errors,
the problem remains: how to deliver the bad news to the user.
This news must often travel great distances across the code.
As with bugs, Enforcement Profiles affect the choice of solution.
Additionally, the choice of error handling technique is affected
by other prominent characteristics of the code...

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
During this phase, a realtime program cannot simply exit on error.
It must do what it can to convey disappointment to the user, and then carry on.

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

While not directly related to control flow,
the logging or display of disappointment is very important to the user.
A clear message in human-readable form is what is needed
in order to get beyond most errors.

The message should not include implementation details, such as source location:
it should be easy to find the source of the diagnostic from the message text itself.

If the program is interactive, delivery may be via a dialogue box,
an output window or a status bar.
Otherwise, logging or console output are the media through which the user is reached.
Printing a message to the error stream is usually adequate.

```c++
std::cerr << std::format("failed to open file, \"{}\"\n", filename);
```

If the program can proceed past this line of code as normal,
there probably isn't an error at all.
But if there is, typical control flow will be dramatically altered
and further design choices must be made...

#### Return Values

Some programs are simple enough that single scalar return values suffice as
a method for sending news of failure back to the calling process.

```c++
// print file's size or return false
auto print_file_size(char const* filename)
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
auto print_config_file_size()
{
  if (!print_file_size("default.cfg")) {
    // in this function, we know the nature of the file
    std::cerr << "failed to print the size of the config file\n";
  }
}
```

A problem arises if the function already returns a result.
C++ functions only return one object
and the most readable functions return the desired result as that object,
e.g. file size as an integer:

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
returning of the desired result combined with the successfulness:

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
where there is high confidence that disappointment is rare.

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
  which can break real-time system constraints, violating the [End User Contract](#end-user-contract).
* Even when not thrown, they can make it difficult to reason about control flow,
  which has implications for efficiency.

For batch programs on well-resourced systems, they are a very good choice.
For resource-constrained or realtime systems or libraries which target those systems,
the costs and benefits of exception handling should be considered more carefully.

#### Abnormal Program Termination

A program can be stopped quickly via APIs such as `std::terminate` and `std::abort`.

A simple error-handling function can be used much like an assert routine:

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
state, so terminating in reaction to [Unambiguous Bugs](#unambiguous-bugs)
is relatively palatable.
And destructors in a modern, well designed system are only important
while the process is running:
memory, file descriptors and peripherals should all be freed up by the system
once the owning process is ended.

So this can often be the best approach for reacting to violations of the
[End User Contract](#end-user-contract) — as well as to [Unambiguous Bugs](#unambiguous-bugs).
In profile,
Safety-Critical System With Redundancy, the requirement to 'fail fast' can be well
served by this approach. And in profile, Business-Critical Systems, the concerns
associated with bypassing destructors may not be significant.

#### Push UB Onto The End User

The program might be a constituent part of a larger system.
For example, essential data might be stored in a read-only filesystem.
Reliance on this filesystem might be a given.
In such a circumstance, why not simply assert that the filesystem is there?
Why not treat its absence as an Unambiguous Bug?

This is entirely possible but the implications must be fully understood.
There will be reduced freedom to invoke the program with ill-formed input.
There is heightened risk of compromise, corruption or critical failure if the
program is ever invoked outside the parameters of the [End User Contract](#end-user-contract).
Even when it is possible, it is rarely worthwhile taking this risk.

## C++ API Contracts Strategies

We will briefly explore some ways to design and implement [C++ API Contracts](#c-api-contracts)
which can help to deal with disappointment.

### The Pit of False Security

When the advertised goal of a C++ API cannot be achieved,
it should be difficult for the user of the API to ignore the disappointment.
Defending against failure by silently proceeding as normal is unhelpful.

Example 1:

```c++
auto load_file(std::string filename)
-> std::vector<std::byte>
{
  std::ifstream file{filename};
  if (!file) {
    // Bad: missing is not the same as empty!
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
      // Bad: suppresses compiler warnings; silences sanitizers!
      // Illogical: do we wait on a blue traffic light?
      return false;
  }
}
```

These two functions return incorrect results. And by returning anything at all,
the error is allowed to propagate further into the program.
By the time the caller is aware there is a bug, it may be much harder to find.

There are two main solutions to this kind of anti-pattern.

### Disappointment By Type

The best documentation is the API itself.
In a strongly-typed language, the types of a declaration are a powerful form of communication.

Example 1 revisited:

```c++
// The nodiscard attribute has very wide applicability.
[[nodiscard]] auto load_file(std::string filename)
-> std::optional<std::vector<std::byte>>
{
  std::ifstream file{filename};
  if (!file) {
    // Better: user is made aware that the result isn't always some bytes.
    return std::nullopt;
  }
  ...
}
```

### Disappointment By Contract

Example 1 showed disappointment that could reasonably happen in a bug-free program.
But when the [C++ API Contracts](#c-api-contracts) user
could reasonably avoid the disappointment,
it is far better to clearly put the responsibility onto the caller:

* Make it clear that it's a bug for the API to be used disappointingly.
* Fulfil the obligations of a [Test User Contract](#test-user-contract) provider
  and help the user discover the bug, e.g. with assertions.

Example 2 corrected:

```c++
// precondition: traffic_light must be red, amber or green
[[nodiscard]] auto pull_away(color traffic_light)
{
  switch (traffic_light) {
    case color::red:
      return false;
    case color::amber:
      return false;
    case color::green:
      return true;
  }

  // Better: program will trap if user passes an unhandled value.
  // It's clear from the code that fall-through is a bug.
  ASSERT(false);
}
```

Note that the provider might be tempted to place the assertion in a `default` clause.
This is not advised. There is no acceptable default behaviour here
so it's better not to express the intent that there is.

### Be Decisive

Having decided that a potential disappointment is a bug,

* _do_ assert that the bug does not happen,
* _don't_ treat it as an error, and
* _don't_ write code to handle it or to defend against it.

Consider a program in which `ASSERT` assumes no [Unambiguous Bugs](#unambiguous-bugs)
(as described in Prevention Enforcement Strategy):

```c++
// precondition: traffic_light must be red, amber or green
[[nodiscard]] auto pull_away(color traffic_light)
{
  switch (traffic_light) {
    case color::red:
      return false;
    case color::amber:
      return false;
    case color::green:
      return true;
  }

  // This line is unreachable.
  ASSERT(false);

  // Bad: this code also can never be reached.
  // The compiler may even optimise it away!
  std::fprintf(stderr, "mysterious enum value, %d\n", (int)traffic_light);
  return false;
}
```

Concerns about assumption-based optimisation are raised in [P2064R0, Assumptions](https://wg21.link/p2064r0).
Examples from P2064R0 suggest than a lot of defective code reaches production
without first being tested using the [Trap Enforcement Strategy](#trap-enforcement-strategy).
Instead of fixing the offending code, the paper recommends that asserts aren't assumed.
That is the approach taken by both the [Nonenforcement Strategy](#nonenforcement-strategy)
and the Log-And-Continue Strategy.
But for other strategies, it is excessive and harms performance.

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

The `eg_assert` arguments above are examples of expression within the implementation.

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
Duplicate assertions here are harmless but of less value.

### Validating Input

Code which accepts external input into the program is a very different matter.

The developer shouldn't assume that program input is error-free.
Even if the users of the program are 'friendly', they may make mistakes which the
program developer did not anticipate.

In order to be confident that no [Unambiguous Bugs](#unambiguous-bugs) can occur,
input must be thoroughly vetted.
If the possible violations by the user of the [End User Contract](#end-user-contract)
are not tested, all manner of undefined behaviour may result.

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

Here we see `std::span` used to corral the program's implicitly-bound input parameters:

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

### C++ API Contracts Violation is Undefined Behaviour

It is a common misconception that the term _undefined behaviour_
refers only to violations of some subset of the [C++ Standard](#c-standard).

The standard [describes](https://eel.is/c++draft/defns.undefined) UB as:

> behavior for which this document imposes no requirements

It does not say that the behaviour cannot be defined elsewhere.
It also does not say that all other behaviour is defined.

Part of the confusion is the belief that the definition behind an interface
grants the user licence to infer a broader contract.
Let's look again at the function, `number_to_letter`.

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
// signed integer overflow violates C++ Standard, is already UB
number_to_letter(0x7fffffff);
```

Finally, the provider reserves the right to change the implementation as they choose.
Behaviour inferred from the original definition is already false.
But it becomes yet more dangerous when applied to a new definition. For example:

```c++
constexpr auto number_to_letter(int number)
{
  constexpr auto lookup_table = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
  return lookup_table[number - min_number];
}
```

Here, the implementer changed the definition — something they are entitled to do.
Now, almost every possible [C++ API Contracts](#c-api-contracts) violation
is also a [C++ Standard](#c-standard) violation.

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

* the program user in the [End User Contract](#end-user-contract), and
* the engineer in the [Test User Contract](#test-user-contract).

Paradoxically, the [Test User Contract](#test-user-contract)
relies on the narrowness of [C++ API Contracts](#c-api-contracts)
in order to fulfil a wide contract of its own.
Thus by widening [C++ API Contracts](#c-api-contracts), the [Contract Author](#contract-author)
impedes the engineer's ability to identify bugs.

### Don't Optimize Until You Sanitize

C++ toolchains increasingly optimise programs by assuming their correctness.
These optimisations are not free.
They are provided to users in exchange for not violating the [C++ Standard](#c-standard).

Historically, such contract violations have been difficult to detect.
Fortunately, modern toolchains provide facilities for their detection.
These include dynamic analysis tools known as sanitizers
which instrument code to test for bugs at run-time.
The main benefit of undefined behaviour is that it allows such tools
to accurately identify bugs.

Meanwhile, modern software development practices emphasise automated testing regimes.
It is not uncommon for most of the APIs in a program
to be tested for [Provider Contract Violation](#provider-contract-violation)
frequently during development.
Such tests are an opportunity to enable sanitizers.

If you wish to enable compiler optimisations in the program you provide to users
you are strongly advised to ***test your program code with sanitizers***.
Combined with the approaches detailed in this document, this will lead to significantly
fewer software defects without the need to compromise safety or performance.

If your toolchain provider doesn't offer sanitizers in its latests products,
question whether it is of suitable quality.

### Use All the Tools

As well as enabling sanitizers and static analysers and elevating compiler warnings,
using multiple toolchains will increase the range of bugs for which you test.
Even if your target compiler does not have those facilities, some (though not all)
bugs will be detected by testing your software on consumer-grade system using
free operating systems and toolchains, or in emulators that more closely resemble
the target platform.

### Don't Rely on Tools Alone

Unfortunately, sanitizers are far from perfect.
Some [Unambiguous Bugs](#unambiguous-bugs) are
prohibitively difficult to detect at run-time.

Sometimes, sanitizers fail to instrument for UB as advertised.
For example, if the logic which recognises the possibility of UB
is located in the compiler's optimiser, and if optimisation is not enabled,
instrumentation may not occur. For this reason,
it is important to test code with settings that are as close to release as possible.

Finally, not all bugs are undefined behaviour.
Some bugs violate only the [End User Contract](#end-user-contract).
They are sometimes described as bugs in the 'business logic' of the program.
Because of this, unit testing will always play a vital role in a
healthy development process.

### A Carefully Controlled Vagueness

Nevertheless, the notion of [Unambiguous Bugs](#unambiguous-bugs)
is every bit as important as type safety.
Certainly, it is better to discover bugs early in the development process
and this is why static typing is so beneficial.
However, seeking bugs during program execution only increases fortification.

If behaviour of a program is defined at the point where an Unambiguous Bug occurs,
the behaviour must be diagnostic in nature. Otherwise, the bug is

* harder for tools to identify,
* harder for engineers to identify,
* no longer diagnosed in constant expressions, and
* normalised (as per [Hyrum's Law](https://www.hyrumslaw.com/)).

There is often a 'least worst' defined behaviour for a bug,
e.g. signed integer modulo behaviour, or a valid-but-wrong iterator value
returned from a binary search performed on an unordered sequence.
Often, these are the default behaviour in unoptimised programs.

Such behaviour can be locked in, e.g. using GCC's `-fwrapv` flag.
This prevents detection of bugs and may pessimise bug-free programs.
But where bugs slip through the net (which they do), it can reduce attack vectors.
***Nothing about UB prevents toolchains from implementing these behaviours.***
Feel free to seek them out as a last line of defence in safety-critical applications.

## Conclusion

In short:

* Use modern C++ guidelines and tools to prevent bugs before they are written.
* Use assertions to formally express [C++ API Contracts](#c-api-contracts).
* Then, you can easily choose a bug handling strategy, change your mind, or use different
  strategies for different contracts and in different builds.
* You cannot so easily change error-handling strategies.
* Sometimes, a bug in a program isn't a bug; it's an error. This happens as part
  of the [Test User Contract](#test-user-contract),
  and it's important to understand the irregularities
  of this contract.
* UB is not something that is confined to the [C++ Standard](#c-standard).
* UB is essential for efficiency, flexibility and for finding bugs.
* It is increasingly unreasonable to enable optimisations in software
  without first testing using sanitizers.

## Acknowledgements

Thanks to Alicja Przybyś and Andrzej Krzemieński for much feedback, correction
and guidance.

## Appendix A - Toolchain-Specific Recommendations

This section introduces some of the toolchain-supported features
which can be used to help enforce the [Unambiguous Bug Strategies](#unambiguous-bug-strategies)

* **Trap** Enforcement Strategy,
* **Non**enforcement Strategy,
* **Log**-And-Continue Strategy, and
* **Pre**vention Enforcement Strategy,

using three popular compilers:

* Clang,
* GCC, and
* MSVC.

To some extent, different strategies can be applied to different bugs
within the same build of a program. For example,
the developer may wish to trap [C++ API Contracts](#c-api-contracts) violations
but prevent [C++ Standard](#c-standard) violations.
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
    <td><i>-Wall</i>, <i>-Wconversion</i>, <i>-Wextra</i> and <i>-Wpedantic</i></td>
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
    <td>trap <a href="#c-standard">C++ Standard</a> user contract violations<sup>†</sup></td>
  </tr>

  <tr>
    <td><i>-ftrapv</i></td>
    <td>✓</td>
    <td>✓</td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td></td>
    <td>avoid; <a href="https://gcc.gnu.org/bugzilla/show_bug.cgi?id=61893">broken
on GCC</a></td>
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
to trap common occurrences of [C++ Standard](#c-standard) user contract violations.
Additional [Clang sanitizers](https://clang.llvm.org/docs/index.html) are dedicated
to catching bugs at run-time. Some of them work with GCC as well as Clang.
Tools like Valgrind can also catch some of these bugs.

‡ When used as part of the [Trap Enforcement Strategy](#trap-enforcement-strategy),
must be used ***instead of*** explicit bug trapping code,
and ***in conjunction with*** a sanitizer such as UBSan.

&#42; Some optimisations may be safely enabled without further degrading
behaviour of defective code, e.g. `-finline-functions` and `/Ob2`.
