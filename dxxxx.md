---
title: "Why `constant_wrapper` is not a usable replacement for `nontype`"
document: pxxxx
date: today
audience: LEWG, LWG
author:
  - name: Bronek Kozicki
    email: <brok@incorrekt.com>
toc: true
toc-depth: 2
---

# Abstract

The lack of LEWG consensus during the Sofia to replace `nontype` with
`constant_wrapper` as a `function_ref` parameter has reportedly susprised
some members of LWG. Plausibly, it might have also surprised the wider C++ community.
This paper is an attempt to explain the process and the rationale behind the (lack of)
decision for such a change.

# Polls taken

During the discussion on [P3740R1](https://isocpp.org/files/papers/P3740R1.html) in Sofia,
LEWG took a series of polls which ultimately led to the decision to keep `nontype`
parameters in `function_ref` constructors and rename it, rather than replace `nontype`
with `constant_wrapper`, while opening a path for a possible NB comment which might
request such a change.

First, the following three polls were taken as popularity contest:

> **POLL:**  Forward “P3740R1: Last chance to fix `std::nontype`” selecting Option A
> (change from `nontype` to `constant_wrapper`, also add overloads to other function wrappers (`std::function`)) to LWG
>
> SF | F | N | A | SA
> ---|---|---|---|---
> 0  | 4 | 0 | 7 | 8
>
> Outcome: No consensus for change
>
> **POLL:** Forward “P3740R1: Last chance to fix `std::nontype`” selecting Option A
> (change from `nontype` to `constant_wrapper`, DO NOT add overloads to other function wrappers (`std::function`)) to LWG
>
> SF | F | N | A | SA
> ---|---|---|---|---
> 6  | 7 | 2 | 6 | 0
>
> Outcome: Weak consensus in favor
>
> **POLL:** Forward “P3740R1: Last chance to fix `std::nontype`” selecting Option B
> (rename `std::nontype` to `std::constant_arg`) to LWG
>
> SF | F | N | A | SA
> ---|---|---|---|---
> 7  | 9 | 2 | 2 | 1
>
> Outcome: Consensus in favour

As a result of the popularity contest, the Option B was assumed to have become the "status quo",
and the two options with (some) consensus have been pitted against each other in the final poll:

> **POLL:** Forward “P3740R1: Last chance to fix `std::nontype`” selecting Option A
> (change from `nontype` to `constant_wrapper`, DO NOT add overloads to other function wrappers
> (`std::function`)) instead of Option B (rename `std::nontype` to `std::constant_arg`) to LWG
>
> SF | F | N | A | SA
> ---|---|---|---|---
> 9  | 4 | 2 | 4 | 4
>
> Outcome: No consensus for change.

The final notes from the meeting do recognize that Option B has received fewer votes than the limited Option A:

> The second poll had no consensus, but got more votes toward the more extensive change
> (remove `nontype` and replace with `constant_wrapper` in `funtion_ref` only).
> This will need to be discussed again if an NB comment is submitted for C++26.

# Discussion

During the discussion leading to polls taken above, I have argued strongly against replacing
`nontype` as a parameter of `function_ref` with `constant_wrapper`, instead arguing for
renaming `nontype` to something that better reflects its use as a `function_ref` parameter.

This was informed by my [implementation experience](https://github.com/zhihaoy/nontype_functional/pull/13) replacing `nontype` with `constant_wrapper` in
reference implementations of `function_ref`, `move_only_function` and `function`. The problem was not
related to implementation (that was the easy part). It was the discovery of inconsistency
between different types of function wrappers which would potentially lead the users to write error-prone C++ code,
had such change been made in the standard library.

## Why `function_ref` takes the `nontype` parameter ?

The details can be found in [@P2472], but the short version is that it provides
`function_ref` with a type-erasing constructor from a member function or a free
function. The mechanics of this constructor is simple:

* initialize the exposition-only `thunk-ptr` with the address of a function which wraps the `invoke_r` call taking
  the value template parameter of `nontype` as a first parameter, and optionally second parameter to be dereferenced from `bound-entity`
* initialize the exposition-only `bound-entity` with the optional second constructor parameter (if it is a pointer) or
  its address (if it is a reference)

This allows the `function_ref` to, for example, wrap a pointer to a member function _and_ the object reference
to call it with.

## Would other function wrappers benefit from taking a `nontype` wrapper ?

Yes. In particular `move_only_function` and `copyable_function`, which both apply
small size optimization, would be able to use space more efficiently, if they performed type-erasure from
a `nontype` wrapper, when instantiating the wrapper to call target.

A similar optimization opportunity would arise if the function wrapper construtors recognized that
an invocable passed to its constructor is stateless (e.g. by application of `is_empty` trait), meaning that
taking a `nontype` parameter is not the only way to the same goal (i.e. size optimization).

## Is there a proposal to add such constructors to other function wrappers ?

Initial work has been done by Zhihao Yuan and (separately) by Tomasz Kamiński.

## What's the problem with replacing `nontype` with `constant_wrapper` ?

In short, `constant_wrapper` has its own `operator()`, with its own semantics which might, or might not,
be compatible with an arbitrary invocable used to instantiate it.

More specifically, `operator()` in `constant_wrapper` is defined in [@P2781] as:

```c++
    template<constexpr-param T, constexpr-param... Args>
      constexpr auto operator()(this T, Args...) noexcept
        requires requires(Args...) { constant_wrapper<T::value(Args::value...)>(); }
          { return constant_wrapper<T::value(Args::value...)>{}; }
```

Please note that the return type is an instance of `constant_wrapper`, which will fail compilation if the return type
returned from the invocation `T::value(Args::value...)` is _not_ a structural type. Also, the call parameters are all
expected to have `value` static data member, which is extracted inside the call. There is nothing wrong with this,
because the design goal of `constant_wrapper`  is to provide C++ users with
a _more useful_ compile-time constant (compared to `integral_constant`). However it also means that an invocable
returning an _arbitrary_ (including non-structural) type _cannot_ be used with `constant_wrapper::operator()`. This
is by design, and (in my humble opinion) it's _good_.

When trying to use `constant_wrapper` in place of `nontype`, the workaround for this limitation is obvious: do not
try to invoke the `constant_wrapper`, just unwrap its template parameter, via the `value` static data mamber.
This is what I did and it worked.

## It worked, so what's the problem ?

Currently no other function wrapper has this functionality.  However, the C++26 status quo is that
`constant_wrapper`, instantiated with a value of a compatible type, is also an invocable, and it can be used to construct any
function wrapper (with matching template parameters). The behaviour and semantics of such function wrapper will
be different compared to a `function_ref` constructed with `nontype`, because of the semantics `constant_wrapper::operator()`.
This difference will be most stark if the invocable used to instantiate `nontype` or `constant_wrapper`
is a functor user type with overloaded function call operators, or with a templated function call operator (as a niebloid might be).
In short, it is not safe to _just_ substitute `nontype` with a `constant_wrapper`.

Which also means that:

* if `nontype` was replaced by `constant_wrapper` as a construction parameter to `function_ref` _and_
* user performed a (seemingly) simple refactoring by swapping a standard function wrapper to another standard function wrapper (e.g. to benefit from the low cost of `function_ref` or to rely on data storage in other wrappers)

... any of the following might happen:

* program will continue to work as designed
* program will fail to compile
* program will continue to compile and "work", but with subtly changed behaviour

In order to avoid the last two from happening, we would have to do some of the following:

* revisit `operator()` in `constant_wrapper` i.e. [@P2781]
* revisit `nontype` parameter in `function_ref` i.e. [@P2472] and [@P0792]
* add similar overloads to other functional wrappers (see first poll taken)
* define specialization of `invoke` and `invoke_r` for `constant_wrapper` to use the extracted `value` rather than call `operator()` directly

## Wait, what about change in `invoke` `invoke_r` ?

The last option suggested above, which nobody is proposing and which (in my opinion) is _not_ sensible, would "fix"
the problem, preventing breakage when the user code is switched from one standard function wrapper to another (when the
constructor call relies on on `constant_wrapper` parameter), at the cost of imbuing the `constant_wrapper` with
_dual_ invocation semantics:

* direct call defined in `operator()`, as defined in [@P2781] _and_
* call into the value template parameter, via `value` static data member, with `invoke` and `invoke_r`

This dual semantics means that we have moved the problem from one place to another. A user would not
expect that a change in their code from `invoke` (or `invoke_r`) to function call syntax, might make the
program fail to compile or worse, change its behaviour.

This is just making things worse.

# Summary

LEWG did the right thing in Sofia by no approving replacing `nontype` constructor parameters in `function_ref` with
`constant_wrapper`. Had this been done, we would have to either revisit several other design decision taken previously
or risk users' code breaking (sometimes subtly) when they move between different standard function wrappers.

# Future

I suggest to pursue the following:

* Rename `nontype` to better reflect its current use in `function_ref` constructor (this is the _only_ use of `nontype` type in C++26). This _has_ to be done in C++26, possibly via NB comment
* Extend such newly named type with function call semantics that does not break user expectations
* Provide other function wrappers with the optimization for such a stateless constructor parameter

# Acknowledgments

Thanks to Jan Schultke, Tomasz Kamiński, Zhihao Yuan, Zach Laine and Gašper Ažman

