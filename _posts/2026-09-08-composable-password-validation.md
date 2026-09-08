---
layout: post
title: "Compile-Time Policy Composition for Password Validation"
author: Daniel Gómez Vergel
date: 2026-09-08
categories: [C++26, concepts, policy-based-design, bitset, template-for]
permalink: /2026/09/08/composable-password-validation/
excerpt: >
  A composable password validation framework in C++26 built around policy-based design and pipeline
  composition.
---

{% include post-categories.html %}

### Introduction

The goal of this article is to design a simple password validator that can be configured by composing
independent validation policies. As a baseline requirement, our validator will verify that a password
length falls within an allowed range, `[Min_sz, Max_sz]`, and that it does not contain ASCII whitespace characters.
Additional rules can then be added to require the presence of digits, lowercase letters,
uppercase letters, and/or special characters.

A traditional object-oriented solution could rely on dynamic polymorphism and the Decorator pattern[^1]<sup>,</sup>[^2],
allowing validation policies to be selected and composed at runtime. In this article, we will take a
different approach: policy selection will be performed entirely at compile time, making the validator
configuration part of the type itself. As a result, the implementation will avoid virtual dispatch
and perform no dynamic allocations.

The following example illustrates how validation policies can be composed using the pipeline syntax.
Starting from a validator that enforces only length constraints and the absence of whitespace
characters (`check_0`), additional rules can be added through composition. Validator `check_1`
requires at least one digit, whereas `check_2` additionally requires the presence of both lowercase
and uppercase letters:

<div class="dgv-cb">
{% include composable-password-validation/cb-1.html %}
</div>

<div class="dgv-note">Although our library is fully <code>constexpr</code>-friendly and can therefore
validate passwords at compile time, as demonstrated by the <code>static_assert</code> declarations
above, its primary use case is runtime validation. The same validator object can be used in either
context without any changes to the API. See the <a href="#runtime-validation">last section</a> of the
article for an example of execution at runtime.
</div>

For simplicity, the entire library will be implemented as a single `password_validator` module, so that
all of its functionality is made available through a single `import password_validator;` declaration.
The complete module can be reconstructed simply by concatenating the code fragments presented in the
following sections.

### Character-level policies

The following listing introduces the set of character-level validation policies that can be composed
to build custom password validators. Each policy is implemented as a stateless function object
providing an `operator()(char)` predicate that checks whether a given character satisfies a particular
validation rule. The available policies are:

* `Digit_character`: the password must contain at least one digit.
* `Lower_case_character`: the password must contain at least one lowercase letter.
* `Upper_case_character`: the password must contain at least one uppercase letter.
* `Special_character`: the password must contain at least one special character from the set `$!?#@%&*^-`.

<div class="dgv-cb">
{% include composable-password-validation/cb-2.html %}
</div>

For convenience, we also provide constant instances of each policy. As explained later, this will allow us
to select rules directly within pipeline expressions:

<div class="dgv-cb">
{% include composable-password-validation/cb-3.html %}
</div>

### Validation engine

We can now implement a variadic password validator parameterized by both the allowed length bounds and
the set of character policies selected by the user.

First, the `Character_policy` concept defines the requirements that every policy type must satisfy.
In particular, a policy must be default-constructible and invocable with a `char` argument.
The invocation must be `noexcept`, and its result must be convertible to `bool`:

<div class="dgv-cb">
{% include composable-password-validation/cb-4.html %}
</div>

The `Password_validation` class template defined below takes the minimum (`Min_sz`) and maximum (`Max_sz`)
allowed password lengths as non-type template parameters, together with a policy parameter pack (`Policies...`,
possibly empty) specifying the requirements that a password must satisfy. A `static_assert` ensures
that the lower bound does not exceed the upper bound. The public `policy_count` constant exposes
`sizeof...(Policies)`, that is, the number of validation policies configured at compile time.

The validator's call operator, `operator()(std::string_view)`, performs the actual password validation
in three stages:

1. Verify that the password length lies within the allowed range `[Min_sz, Max_sz]`.
2. Reject passwords containing ASCII whitespace characters.
3. Verify that every policy in `Policies...` has been satisfied by at least one character in the password.

The password is processed sequentially, with each character updating a validation state that
records which policy requirements have already been satisfied. This state is stored in a `std::bitset<policy_count>`[^3], whose bits are in one-to-one correspondence with the policies in the
`Policies...` pack. Each bit indicates whether the corresponding requirement has been met. 
As soon as all bits become set, validation succeeds and processing stops immediately; otherwise, the
scan continues until the end of the password.

<div class="dgv-cb">
{% include composable-password-validation/cb-5.html %}
</div>

Given a password character `c`, the `update()` function updates the validation state by
evaluating only those policies that have not yet been satisfied. The implementation relies on a C++26
`template for` expansion[^4], causing the compiler to expand the loop body for every policy
in the pack at compile time. The pack indexing expression `Policies...[Idx]` retrieves the `Idx`-th policy
type within the parameter pack. Whenever a policy `P` returns `true` for the current character, the
corresponding bit in the `std::bitset` is set.

Since bits are only ever set and never reset, the number of satisfied requirements can only increase
during the traversal. As an optimization, once the bit associated with a policy has been set, that
policy is excluded from all subsequent evaluations.

<div class="dgv-note"> The new C++26 <code>template for</code> statement (formally known as an expansion statement) allows a compound statement to be replicated at compile time for each element of: (i) an
expression list (enumerating expansion), (ii) any entity that can be decomposed through structured
bindings (that is, a tuple-like type; this form is known as a destructuring expansion), and (iii) a
range whose size is known at compile time (iterating expansion).
</div>

### Pipeline composition

The following overload of `operator|` allows validation policies to be added using a
pipeline-style syntax. Starting from a `Password_validation` object, the operator appends a new policy
to the set of requirements and returns a new validator type that includes the additional policy. Before
the new policy is added, a `static_assert` checks at compile time that a policy of the same type has
not already been added:

<div class="dgv-cb">
{% include composable-password-validation/cb-6.html %}
</div>

<div class="dgv-note"> As discussed earlier, each application of <code>operator|</code> produces a
new <code>Password_validation</code> type with an extended policy pack. As a consequence, the entire
validator configuration is encoded in its template arguments: no additional runtime storage
is required, and policy composition introduces no runtime overhead.
</div>

### Runtime validation

Finally, as an example, the following `main()` function shows how to use the validator with strings
entered at runtime:

<div class="dgv-cb">
{% include composable-password-validation/cb-7.html %}
</div>

---

### Bibliography

[^1]: wikipedia.org – [Decorator pattern](https://en.wikipedia.org/wiki/Decorator_pattern)
[^2]: Marius Bancila. 2018. *The Modern C++ Challenge*. Packt Publishing.
[^3]: cppreference.com – [std::bitset](https://en.cppreference.com/cpp/utility/bitset)
[^4]: cppreference.com – [template for expansion](https://cppreference.com/cpp/language/template_for)
