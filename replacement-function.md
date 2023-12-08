---
title: "Replacement function"
document: D2826R2
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

This paper introduces a way to redirect a function call to a different function
without a forwarding layer.

Example:

```cpp
int g(int) { return 42; }
int f(long) { return 43; }
auto g(unsigned) = &f; // handle unsigned int with f(long)

int main() {
    g(1); // 42
    g(1u); // 43
}
```

One might think that declaring

```cpp
int g(unsigned x) { return f(x); }
```

is equivalent; this is far from the case in general, which is exactly why we
need this capability. See the motivation section for the full list of issues
this capability solves.

First, however, this paper introduces what it _does_.

# Status

This paper is in early stages of exploration. The R1 will be presented to EWGI
in Kona, 2023.


# Proposal

We propose a new kind of function definition of the form

_function-body_:
  ...
  `=` _constant-expression_ `;`

where the _constant-expression_ is an expression that designates a function
(such as a reference-to-function or reference-to-member-function).

We will call this function the **target**.

The associated function cannot have been declared previously.

A call expression that resolves to a function thus declared instead resolves to
the target.
This may render the program ill-formed.

**Notes:**

- The target need not have the same type as the function declaration that was
  selected would suggest.
- This feature is *not* the same as calling the target function from the body
  (see example with immovable types).

**OPEN QUESTION:** should we allow pointer-to-function and
pointer-to-member-function too? It's technically a bit weird, type-wise, but syntax-wise it's nicer.
This paper allows pointer-to-function and pointer-to-member-function, and uses
it.

**OPEN QUESTION:** what should we call this feature? Function aliases? GDR points
out that "replacement function" is close to "replacable function", which is a
term of art already.

## Example 1: simple free function aliases

We could implement overload sets from the C library without indirection:

```cpp
// <cmath>
auto   exp(float)       = &expf; // no duplication 
double exp(double)      { /* normal definition */ }
auto   exp(long double) = &expl; // no duplication
```

This capability would make wrapping C APIs much easier, since we
could just make overload sets out of individually-named functions.

## Example 2: simple member function aliases

```cpp
template <typename T>
struct Container {
  auto cbegin() const -> const_iterator;
  auto begin() -> iterator;
  auto begin() const = &cbegin; // saves on template instantiations
};
```

## Example 3: computed target

`fmt::format` goes through great lengths to validate format string
compatibility at compile time, but *really* does not want to generate code for
every different kind of string.

This is extremely simple to do with this extension - just funnel all
instantiations to a `string_view` overload.

```cpp
template <typename FmtString, typename... Args>
    requires compatible<FmtString, Args...>
auto format(FmtString const&, Args const&... args) 
    = static_cast<std::string(*)(std::string_view, Args const&...)>(&format);
```

Contrast with the best we can realistically do presently:

```cpp
template <typename FmtString, typename... Args>
    requires compatible<FmtString, Args...>
auto format(FmtString const& fmt, Args const&... args) -> std::string {
    return format(std::string_view(fmt), args...);
}
```

The second example results in a separate function for each format string (which
is, say, one per log statement). The first provably never instantiates different
function bodies for different format strings.

## Example 4: deduce-to-baseclass

Imagine we inherited from `std::optional`, and we wanted to forward
`operator*`, but have it be treated the same as `value()`; note this is *far*
easier to done with the help of `__builtin_calltarget` [@P2825R0], so we'll use
it here (it's just "give me the function pointer to the top-most call expression"):

```cpp
template <typename T>
struct my_optional : std::optional<T> {
    template <typename Self>
    auto operator*(this Self&& self) = __builtin_calltarget(std::declval<Self>().value());
};
```

Without `__bultin_calltarget`, the computation becomes a *lot* uglier, because
we need to conjure the function pointer type for each cv-ref qualification, so
I'm not even going to include it here.


## Example 5: immovable argument types

Consider having an argument type that must be in-place constructed:

```cpp
#include <type_traits>
#include <utility>

template <typename T>
struct pin {
    T value;

    pin(auto&&... vs) 
        requires(requires { T{std::forward<decltype(vs)>(vs)...}; })
        : value{std::forward<decltype(vs)>(vs)...} {}
    pin(pin&&) = delete;
};

template <typename T>
void takes_pinned(pin<T> x) {}

template <typename T>
void takes_pinned_adapter(pin<T> x) {
    takes_pinned(std::forward<decltype(x)>(x));
}

template <typename T>
void takes_pinned_alias(pin<T>) = takes_pinned<T>;

template <typename T>
void takes_pinned_alias2(T&&) = takes_pinned<std::decay_t<T>>;

int main() {
    takes_pinned<int>(3); //  what we have to write (A)
    takes_pinned(3); // what we want to write, errors
    takes_pinned(pin<int>{3}); // works
    takes_pinned_adapter(pin<int>{3}); // error, can't forward pin<T>
    takes_pinned_alias<int>(3); // works with this paper
    takes_pinned_alias2(3); // also works with this paper, same as line (A)
}
```

## Discussion

There are two parts such a function definition: the declarator, and the body.

The function signature as declared in the declarator participates in overload
resolution. (the constant expression is not part of the immediate
context).

If overload resolution picks this signature, then, *poof*, instead of
substituting this signature into the call-expression, the replacement function
is substituted. If this renders the program ill-formed, so be it.

## What if the constant expression evaluates to null?

That renders calling that overload ill-formed (and thus effectively means the
same as a programmable `= delete`).

`virtual`, in which case you can now spell conditionally-implemented overrides.
Yay. (Note: whether a class is abstract or not is decided when the vtable entry
is tried to be synthesized and fails).

For virtual functions, though, the signature needs to match in the same way as
signatures of overrides have to match the virtual interface declaration,
otherwise we have a problem with ABI.

Example:

```cpp
template <bool enable_implementations>
struct A {
  long g(int) { return 42; }
  long h(int) = enable_implementations ? &g : nullptr;
  virtual long f(int) = enable_implementations ? &g : nullptr;
};
struct Concrete : A<true> {};
struct Abstract : A<false> {};
struct Concrete2 : Abstract { long f(int) override { return 3; } };
void impl() {
  Concrete x;  // ok
  x.h(2);      // ok, 42

  Concrete2 z; // ok
  z.f(2);      // ok, 3
  z.h(2);      // Error, h is explicitly deleted

  Abstract y;  // Error, f(int) is abstract
};
```

# Why this syntax

- syntax noted actually already exist, just look for =delete
- it works for `0` as "not-provided" and `= delete` as "no body", consistently

# Nice-to-have properties

- removal of library-defined dispatch from callstacks
- optimizers have to work way less hard
- major symbol reduction
- no issues with expressions - we don't modify expressions (compare: parametric expressions)
- No extra work for overload resolution - this feature has no bearing on
  overload resolution, as the `@_constant-expression_@` is evaluated at
  declaration time.

# Related Work

Roughly related were Parametric Expressions [@P1221R1], but they didn't
interact with overload sets very well.

This paper is strictly orthogonal, as it doesn't give you a way to rewire the
arguments, just substitute the function that's actually invoked.


# Use-cases

There are many cases in c++ where we want to add a function to an overload set
without wrapping it.

A myriad of use-cases are in the standard itself, just search for
_expression_equivalent_ and you will find many.

The main thing we want to do in all those cases is:

- step 1: detect a call expression pattern
- step 2: dispatch to the appropriate function for that case

The problem is that the detection of the pattern and the appropriate function to
call often don't fit into the standard overload resolution mechanism of C++,
which necessitates a wrapper, such as a customization point object, or some
kind of dispatch function.

This dispatch function, however it's employed, has no way to distinguish
prvalues from other rvalues and therefore cannot possibly emulate the
copy-elision aspects of _expression-equivalent_. It is also a major source of
template bloat. The proposed mechanism also solves a number of currently
unsolved problems in the language (see the [use-cases section](#use-cases)).


## deduce-to-type

The problem of template-bloat when all we wanted to do was deduce the type
qualifiers has plagued C++ since move-semantics were introduced, but gets much,
worse with the introduction of (Deducing This) [@P0847R7] into the
language.

This topic is explored in Barry Revzin's [@P2481R1].

This paper provides a way to spell this though a specific facility for this
functionality might be desirable as a part of a different paper.

Observe:

```cpp
struct A {};
struct B : A {};

template <typename T>
auto _f_impl(T&& x) {
  // only ever gets instantiated for A&, A&&, A const&, and A const&&
};

// TODO define cvref-derived-from
template <typename D, typename B>
concept derived_from_xcv = std::derived_from<std::remove_cvref_t<D>, B>;

template <std::derived_from<A> T>
auto f(T&&) = &_f_impl<copy_cvref_t<T, A>>;

void use_free() {
  B b;
  f(b);                // OK, calls _f_impl(A&)
  f(std::move(b));     // OK, calls _f_impl(A&&)
  f(std::as_const(b)); // OK, calls _f_impl(A const&)
}
```

Or, for an explicit-object member function:

```cpp
struct C {
  void f(this std::same_as<C> auto&& x) {} // implementation
  template <typename T>
  auto f(this T&& x) = static_cast<void (*)(copy_cvref_t<T, C>&&)>(f);
  // or, with __builtin_calltarget (P2825) - helps if you just want to write the equivalent expression
  template <typename T>
  auto f(this T&& x) = __builtin_calltarget(std::declval<copy_cvref_t<T, C>&&>().f());
};
struct D : C {};

void use_member() {
  D d;
  d.f();                // OK, calls C::f(C&)
  std::move(d).f();     // OK, calls C::f(C&&)
  std::as_const(d).f(); // OK, calls C::f(C const&)
}
```

### Don't lambdas make this shorter?

Observe that we could also have just defined the inline lambda as the _constant-expression_:

```cpp
struct A {};
struct B : A {};

// saves on typing?
template <std::derived_from<A> T>
auto f(T&&) = +[](copy_cvref_t<T, A>&&) {};

void use_free() {
  B b;
  f(b);                // OK, calls f-lambda(A&)
  f(std::move(b));     // OK, calls f-lambda(A&&)
  f(std::as_const(b)); // OK, calls f-lambda(A const&)
}
```

While at first glance this saves on template instantiations, it's not really
true (but it might save on executable size if the compiler employs COMDAT
folding).

The reason is that the lambdas still depend on `T`, since one can use it in the
lambda body.

This is very useful if we need to manufacture overload sets of _functions with
identical signatures_, such as needed for type-erasure.

Continued example:

```cpp
template <typename T>
int qsort_compare(T const*, T const*) = +[](void const* x, void const* y) {
  return static_cast<int>(static_cast<T const*>(x) <=> static_cast<T const*>(y));
  //                                  ^^                           ^^
};


// special-case c-strings
int qsort_compare(char const* const*, char const* const*) = +[](void const* x, void const* y) {
  auto const x_unerased = static_cast<char const* const*>(x);
  auto const y_unerased = static_cast<char const* const*>(y);
  return strcmp(*x_unerased, *y_unerased);
};

void use_qsort_compare() {
  std::vector<int> xs{1, 4, 3, 5, 2};
  ::qsort(xs.data(), xs.size(), sizeof(int),
          __bultin_calltarget(qsort_compare(std::declval<int*>(), std::declval<int*>()));
  std::vector<char const*> ys{"abcd", "abdc", "supercalifragilisticexpielidocious"};
  ::qsort(ys.data(), ys.size(), sizeof(char const*),
          __bultin_calltarget(qsort_compare("", "")));
}
```

So, while lambdas don't turn out to be a useful "template-bloat-reduction"
strategy, they *do* turn out to be a very useful strategy for type-erasure.


## expression-equivalent for `std::strong_order`

`std::strong_order` needs to effectively be implemented with an
[`if constexpr` cascade](https://github.com/llvm/llvm-project/blob/4a4b8570f7c16346094c59fab1bd8debf9b177e1/libcxx/include/__compare/strong_order.h#L42) (link to libc++'s implementation).

This function template would probably be much easier to optimize for a compiler if it were factored as follows:

```cpp
inline constexpr auto struct {
    template <typename T, typename U>
    static consteval auto __select_implementation<T, U>() { /* figure out the correct function to call and return the pointer to it */ }
    template <typename T, typename U>
    static auto operator(T&&, U&&) = __select_implementation<T, U>();
} strong_order;
```

Implementing it like that allows one to dispatch to functions that don't even
take references for built-in types, eliding not just a debug/inlining scope,
but also the cleanup of references.

It allows one to dispatch to ABI-optimal versions of the parameter-passing
strategy. It lets us make function objects truly invisible to debuggers.


## interactions with `__builtin_calltarget()`

While [@P2826R0] makes this paper much easier to utilize, in the presence of
this paper, it becomes truly invaluable. Computing the calltarget while having
to know the exact function pointer type of the called function is very
difficult, and needless, since the compiler is very good at resolving function
calls.

## First-class overload sets

We can lift overload sets into function arguments (as per the point of invocation):

(TODO: propose this syntax...)

```cpp
auto call(auto&& fn, auto&&... args) -> decltype(auto) {
    return fn(std::forward<decltype(args)>(args)...);
}

call([](auto&&...args) = __builtin_caltarget(overloaded(std::forward<decltype(args)>(args)...)));
```


## programmable UFCS

If we get a new revision of [@P1214R0], (`std::invoke`-in-the-language), one
could substitute a free function call with a pointer-to-member-function, or
vice versa, thus enabling both call syntaxes as desired.

## The "Rename Overload Set" / "Rename Function" refactoring becomes possible in large codebases

In C++, "rename function" or "rename overload set" are not refactorings that
are physically possible for large codebases without at least temporarily
risking overload resolution breakage.

However, with this paper, one can disentangle overload sets by leaving removed
overloads where they are, and redirecting them to the new implementations,
until they are ready to be removed.

(_Reminder:_ conversions to reference can lose fidelity, so trampolines in
general do not work).

One can also just define

```
auto new_name(auto&&...args) 
    requires { __builtin_calltarget(old_name(FWD(args)...))(FWD(args)...) }
    = __builtin_calltarget(old_name(FWD(args)...));
```

and have a new name for an old overload set (as resolved at the declaration of `new_name`).

## The "rename function" refactoring becomes ABI stable

- we can finally move overload sets around and not break ABI in some cases
  since we basically gain function aliases.


# FAQ

## Can I declare the function with a different return type from the assigned?

Yes. The declared return type (or auto) is used only for overload resolution.

## Why not require `auto` as the return type of the declaration?

There are situations in the language where the return type matters for overload
resolution.

Examples are conversion operators and explicit casts. See `[temp.deduct.funcaddr/1]`.


# Acknowledgements

- Gabriel Dos Reis for pointing out return types do matter for overload resolution.


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
---
