---
title: "Replacement function"
document: D2826R0
date: today
audience:
  - EWG
author:
  - name: Gašper Ažman
    email: <gasper.azman@gmail.com>
toc: true
toc-depth: 2
---

# Introduction

This paper introduces a way to add a function to an overload set with a given signature.

Example:

```cpp
struct S {};
struct U { operator S() const {return{};} };

int g(S) { return 42; }
void f(U) = g;
long f(S) { return 1l; }
int h() {
  return f(U{}); // returns 42
}
```

# Motivation and prior art

There are many cases in c++ where we want to add a function to an overload set
without wrapping it.

A myriad of use-cases are in the standard itself, just search for
_expression_equivalent_ and you will find them.

The main thing we want to do in all those cases is:

- step 1: detect we are in a particular case
- step 2: call the appropriate function for that case

The problem is that the detection of the case and the appropriate function to
call often don't fit into the standard overload resolution mechanism of C++,
which necessitates a wrapper, such as a customization point object, or some
kind of dispatch function.

This dispatch function, however it's employed, has no way to distinguish
prvalues from other rvalues and therefore cannot possibly emulate the
copy-elision aspects of _expression-equivalent_.

This paper proposes a way to do at least that. However, in doing so, it opens a
surprising number of other doors.

## Related Work

Roughly related were Parametric Expressions [@P1221R0], but they didn't
interact with overload sets very well.

This paper is strictly orthogonal, as it doesn't give you a way to rewire the
arguments, just substitute the function that's actually invoked.

# Proposal

We propose a new kind of function definition of the form

_function-body_:
  ...
  `=` _constant-expression_ `;`

where the _constant-expression_ is (evaluates to) a pointer-to-function,
pointer-to-member-function, or denotes a function (not an overload set).

Example (illustrative):

```cpp
int takes_long(long i) { return i + 1; };
auto takes_int(int) = g;

void test1() {
  takes_int(1); // calls takes_long and implicitly converts 1 into a long.
}

template <typename T>
struct Container {
  auto cbegin() const -> const_iterator;
  auto begin() -> iterator;
  auto begin() const = &cbegin; // saves an instantiation
};
```

That's pretty much it - you already know how it works.

However, let's spell the rules out, just for clarity:

We have two parts to this function definition: the declarator, and the constant expression.

The function signature as declared in the declarator participates in overload
resolution normally (the constant expression is not part of the immediate
context).

If overload resolution picks this signature, then, *poof*, instead of
substituting this signature into the call-expression, the replacement function
is substituted. If this renders the program ill-formed, so be it.

# Why this syntax

- syntax noted actually already exist, just look for =delete
- it works for `0` as "not-provided" and =delete as no body, consistently

# Nice-to-have properties

- removal of library-defined dispatch from callstacks
- optimizers have to work way less hard
- major symbol reduction
- no issues with expressions - we don't modify expressions (compare: parametric expressions)
- No extra work for overload resolution - this feature has no bearing on
  overload resolution, as the `@_constant-expression_@` is evaluated at
  declaration time.

# Use-cases

## deduce-to-type

```cpp
struct A {};
struct B : A {};
struct C { operator A() const { return {}; } };
template <typename T>
struct Box { T value; };

string f(A const&) { return "I'm an A"; }
auto f(std::derives_from<A> auto&&) = static_cast<string(*)(A const&)>(f);
```

## programming overload resolution
## expression-equivalent for `std::strong_order`
## boxing arguments for type-elision
## interactions with `__builtin_calltarget()`
## interactions with templates
## interaction with "fixing surrogate deduction"
## programmable UFCS

- if we get pmfs are callable

## The "Rename Function" refactoring becomes possible in large codebases

- we can finally rename a function (as opposed to the whole overload set) and not break things

See talk by Titus Winters (TODO find talk reference).

## The "rename function" refactoring becomes ABI stable

- we can finally move overload sets around and not break ABI in some cases
  since we basically gain function aliases.

## Changing the return type of a conversion function

- make example for changing return type of conversion function

## Witness tables for wrapper types (a-la Carbon)

- since you can do argument replacement, you can deduce-to-trait and generate witness tables



---
references:
  - id: TaxonomyRevzin
    citatation-label: TaxonomyRevzin
    title: "What is unified function call syntax anyway?"
    author:
      - family: Revzin
        given: Barry
    URL: https://brevzin.github.io/c++/2019/04/13/ufcs-history/
  - id: D2823R0
    citation-label: D2823R0
    title: "Declaring hidden non-friend functions to be found by argument-dependent-lookup"
    author:
      - family: Baker
        given: Lewis
    URL: https://wg21.link/D2823R0
  - id: D2822R0
    citation-label: D2822R0
    title: "Providing user control of associated entities of class types"
    author:
      - family: Baker
        given: Lewis
    URL: https://wg21.link/D2822R0
  - id: P2826R0
    citation-label: P2826R0
    title: "Replacement functions"
    author:
      - family: Ažman
        given: Gašper
    URL: https://wg21.link/P2826R0
---
