---
layout: post
title: "Parsing Physical Quantities from CSV with C++26 Reflection and mp-units"
author: Daniel Gómez Vergel
date: 2026-11-21
last_modified_at: 2026-11-24
categories: [C++26, csv, mp-units, physical-quantities, reflection]
permalink: /2026/11/21/physical-quantities-csv/
excerpt: >
  A small header-only library for parsing CSV data directly into strongly typed
  physical quantities using C++26 reflection and mp-units.
---

{% include post-categories.html %}

## Introduction

<img src="{{ '/assets/images/physical-quantities-csv/fig-1.webp' | relative_url }}"
  alt="Parabolic projectile motion"
  style="float: right; width: 290px; height: auto; margin: 0 0 1rem 2rem;">
Experimental data in science and engineering are commonly stored in CSV files, where each column
represents a physical quantity whose unit is either implicit or specified in a header.
Converting these raw values into strongly typed physical quantities requires parsing the serialized
data and constructing the corresponding objects using a physical quantities library. In this article,
we will accomplish this with C++26, particularly its reflection facilities, and mp-units.[^1]

For example, consider a CSV file named `parabolic_throwing_data.csv` containing measurements from a
projectile motion experiment (see the figure). Each row contains the launch angle `ϕ` in degrees,
the measured initial speed `v` in miles per hour, and the horizontal range `r` in centimeters:

<div style="clear: both;"></div>
<div class="dgv-sheet"><pre>ϕ (deg), v (mi/h), r (cm)
0, 5.68, 61.5
5, 5.66, 64.5
10, 5.64, 72.0
15, 5.61, 77.3
20, 5.59, 82.0
25, 5.42, 85.5
30, 5.50, 88.1</pre>
</div>

The following simple scheme illustrates how the CSV file may be deserialized into a container of physical
quantities using the small library developed in this article:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-1.html %}
</div>

Here, `shot_data` represents a CSV row as an aggregate whose data members correspond, in declaration
order, to the columns of the file. Each member is a `mp_units::quantity<R, Rep>` object,
where `R` identifies both the quantity (e.g., `isq::speed`) and its unit (e.g., `mi/h`), while `Rep`
is the type used to store its numerical value (`double` by default).

A single call to `qcsv::load_quantities<shot_data>`, provided by `quantity_csv.hpp`, returns a `std::expected`[^2]
containing either the complete dataset as a `std::vector<shot_data>` or a loading or parsing error.

Running this program produces the following output:

<div class="dgv-sheet"><pre>0.0°, 5.7 mi/h, 61.5 cm
5.0°, 5.7 mi/h, 64.5 cm
10.0°, 5.6 mi/h, 72.0 cm
15.0°, 5.6 mi/h, 77.3 cm
20.0°, 5.6 mi/h, 82.0 cm
25.0°, 5.4 mi/h, 85.5 cm
30.0°, 5.5 mi/h, 88.1 cm</pre>
</div>

<div class="dgv-note" markdown="1">Based on the ISO 80000 standard (International System of
Quantities, ISQ), <b>mp-units</b> is a modern C++ library that provides compile-time dimensional analysis,
unit conversions, and strong type safety for physical quantities. Beyond dimensional correctness,
it validates operations between quantities according to their physical meaning and mathematical
relationships. mp-units provides six distinct layers of safety: dimension safety, unit safety,
representation safety, quantity kind safety, quantity safety, and mathematical space safety.[^3]
Furthermore, mp-units is being proposed for standardization in C++29 through proposal P3045.[^4]
</div>

The rest of the article develops the header-only library in `quantity_csv.hpp`. Concatenating the code blocks
in order produces the complete header, which is also available on [Compiler Explorer](https://godbolt.org/z/ccPnPKrz6)
together with the `main()` function from this introduction, ready to compile with GCC 16.2.

## Implementation

We begin by including the standard library headers required by the implementation, along with
the `mp-units` core header:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-2.html %}
</div>

Next, we define the error types that the library functions may report. When parsing a CSV row, we
distinguish three possible errors: (i) a value cannot be parsed because of an invalid format (enumerator `format_error`),
(ii) an expected value is missing (`missing_value`), or (iii) the row contains more values than expected
(`extra_value`). A `parse_error` stores an error code identifying one of these failures, together with the number of
the CSV column in which it occurred. When processing an entire CSV input, a `row_error` additionally records
the number of the line on which the parsing error occurred. Columns and lines are both numbered starting from `1`.

Beyond errors associated with parsing individual rows, loading a CSV input may also fail while
reading from the input stream. A `read_error` records the line at which the read operation failed.
When loading directly from a file, an additional failure may occur if the file cannot be opened;
this condition is represented separately by `open_file_error`:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-3.html %}
</div>

The failures that may occur when loading a CSV dataset depend on the input source. We therefore define
two variant types to represent the possible failures in each case:

* Loading from an input stream may fail either while reading from the stream (`read_error`) or while
parsing a row (`row_error`). These alternatives are represented by the `std::variant` type `load_error`.

* Loading from a file may additionally fail because the file cannot be opened (`open_file_error`).
All three failures are represented by the `std::variant` type `file_load_error`.

<div class="dgv-cb">
{% include physical-quantities-csv/cb-4.html %}
</div>

As mentioned earlier, a physical quantity is typically expressed in terms of a numerical value and
a unit of measurement. For parsing purposes, it is convenient to introduce a concept that constrain the underlying
numerical representation of a quantity. In our case, a type satisfies `parsable_value` if its values can be converted
directly from a character sequence using `std::from_chars`,[^5] or alternatively extracted from an input stream
using `operator>>`:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-5.html %}
</div>

The following `parse_value` function template parses a numerical value of any type `T` that
satisfies `parsable_value`, using the strategy appropriate for that type. Whenever possible,
conversion is performed directly from the character sequence using `std::from_chars`. This
avoids the overhead associated with stream-based conversion and makes the process independent of
the locale. Otherwise, the value is extracted through a `std::ispanstream` imbued with the classic C
locale.[^6] In both cases, floating-point values use a period (`.`) as the decimal separator. Leading and
trailing ASCII whitespace is ignored. Empty or whitespace-only fields are reported as missing values, while
any unparsed characters remaining after conversion result in a format error:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-6.html %}
</div>

The next step is to constrain the aggregate type used to represent a CSV row and, more
importantly, to verify at compile time that each of its non-static data members is
a parsable mp-units quantity whose numerical value can be populated from a CSV field.
For simplicity, the following `plain_aggregate` concept admits only aggregate classes that are
default-initializable, have no base classes, and declare at least one non-static data member:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-7.html %}
</div>

<div class="dgv-note">Note how C++26 <b>reflection</b> is used to inspect the aggregate at compile time,
discovering its non-static data members and verifying that it has no base classes.
<hr>
The C++26 reflection system can be understood as an interaction between two domains: the program
domain, comprising types, members, expressions, and other language constructs; and the meta-domain,
comprising <code>std::meta::info</code> reflection values. The <code>^^</code> reflection operator projects
program elements into the meta-domain, producing prvalues of type <code>std::meta::info</code> that represent them.
These reflection values can then be queried, for example to obtain information about types, identifiers, members,
or template arguments. Splicers, written <code>[: <em>refl</em> :]</code>, provide the complementary operation: they
reintroduce (reify) entities represented by reflections into the program in the grammatical contexts where
those entities are permitted.
</div>

Having established the aggregate's structural requirements, we now validate its individual data members.
This validation is organized into three components:

* `parsable_quantity` is a concept that requires the type of a member to satisfy `mp_units::Quantity`, to
be non-const, and to have a representation type `rep` satisfying `parsable_value`.

* `first_invalid_member` uses a C++26 `template for` expansion statement[^7] to inspect
the reflected members in declaration order. It returns the first member whose type does not
satisfy `parsable_quantity`, or an empty `std::meta::info`[^8] if all members are valid.

* `assert_valid_members` takes a type `T` satisfying `plain_aggregate` as a template parameter and uses
`first_invalid_member` to emit a compile-time diagnostic if `T` has a data member whose type does not
satisfy `parsable_quantity`. In that case, the `static_assert` message reports the member's
identifier and type, along with the enclosing aggregate type. Unlike the diagnostic of a failed constraint
on the whole aggregate, which would only report that `T` does not satisfy it, this message points
directly at the offending member. If all members are valid, the `static_assert` has no effect and its
message is an unevaluated operand.

<div class="dgv-cb">
{% include physical-quantities-csv/cb-8.html %}
</div>

<div class="dgv-note">Consider the introductory example: if the type of <code>shot_data::v</code> were
changed from <code>speed_type</code> to <code>double</code>, compilation would fail with a diagnostic that
reports the identifier and type of the offending data member: <code>"static assertion failed: qcsv: data member
'v' of type double in shot_data is not a parsable mp_units::quantity"</code>.
</div>

Having defined the compile-time requirements, we can now implement the `parse_row` function template to parse
an individual CSV row into an aggregate `T` of physical quantities. The function first validates at
compile time that the aggregate's members are parsable physical quantities, and then splits the input row
using the specified delimiter.
A default-initialized instance of `T` is populated, in declaration order, using a C++26
`template for` expansion statement. For each reflected data member, `template for` expands its body
at compile time, generating the runtime code that parses the corresponding CSV field into that member:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-9.html %}
</div>

Here, the call to `numerical_value_ref_in(q.unit)` provides a reference to the numerical value held by
quantity `q`, so it can be populated directly from the CSV field.[^9]

<div class="dgv-note">Reflection plays a key role in this design. It allows <code>parse_row</code>
to take the aggregate type itself as a template argument and inspect its non-static data members,
discovering their types and identifiers at compile time. An alternative design could pass the
physical quantity types separately through a template parameter pack. Keeping them instead as
named members of a single aggregate is particularly convenient: the type provides both the
structure of a CSV row and meaningful names for the physical quantities it contains.
</div>

To make structured errors easier to present to users, we provide a set of `describe`
overloads that translate each error type into a human-readable diagnostic. Errors carrying
contextual information use `std::format` to include the relevant line and column numbers:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-10.html %}
</div>

<div class="dgv-note">For instance, an empty second field on line 4 of the CSV shown in the introduction, <code>10, , 72.0</code>,
produces the message <code>"CSV error: line 4, column 2: missing value"</code>, whereas a malformed
second field, <code>10, 5.64a, 72.0</code>, produces <code>"CSV error: line 4, column 2: invalid field value"</code>.
</div>

At this point, we are ready to provide two overloads of `load_quantities` that load a complete CSV dataset into
a vector of aggregates. One operates on an existing `std::istream` input stream, while the other takes a
`std::filesystem::path` and opens the corresponding file. If the input is configured to contain a header,
the first line is consumed and skipped. Each subsequent line is parsed with `parse_row` and, on success,
the resulting aggregate is appended to the output vector.

The operation has all-or-nothing semantics: `load_quantities` returns the complete dataset only
if every data row is parsed successfully. Otherwise, it returns the first error encountered,
including any available location information, and discards all previously parsed values:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-11.html %}
</div>

Finally, we make the `load_error` and `file_load_error` variant types formattable by specializing
`std::formatter`.[^10] Both specializations inherit the common formatting logic from `error_formatter`,
so these error types can be passed directly to `std::format`, `std::println`, and other formatting facilities:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-12.html %}
</div>

---
Note: This post draws on and adapts a physical-quantity parsing library that I originally
presented in an earlier post on my Spanish-language C++ blog on Blogger:
[mp-units v2 - Biblioteca de magnitudes y unidades físicas](https://dgvergel.blogspot.com/2026/04/mp-units-v2.html).

## Bibliography

[^1]: mp-units - [Home](https://mpusz.github.io/mp-units/latest/)
[^2]: cppreference.com - [std::expected](https://en.cppreference.com/cpp/utility/expected)
[^3]: mp-units - [Safety features](https://mpusz.github.io/mp-units/latest/getting_started/safety_features/?h=safety)
[^4]: www.open-std.org - [P3045 - Quantities and units library](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p3045r10.html)
[^5]: cppreference.com - [std::from_chars](https://en.cppreference.com/cpp/utility/from_chars)
[^6]: cppreference.com - [std::basic_ispanstream](https://en.cppreference.com/cpp/io/basic_ispanstream)
[^7]: cppreference.com - [template for expansion](https://cppreference.com/cpp/language/template_for)
[^8]: cppreference.com - [std::meta::info](https://cppreference.com/cpp/meta/info)
[^9]: mp-units - [Why can't I convert a double& to a quantity&?](https://mpusz.github.io/mp-units/latest/getting_started/faq/?h=numerical_value_ref_in#why-cant-i-convert-a-double-to-a-quantity)
[^10]: cppreference.com - [std::formatter](https://en.cppreference.com/cpp/utility/format/formatter)


