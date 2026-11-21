---
layout: post
title: "Parsing Physical Quantities from CSV with C++26 Reflection and mp-units"
author: Daniel Gómez Vergel
date: 2026-11-21
last_modified_at: 2026-11-21
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
  style="float: right; width: 280px; height: auto; margin: 0 0 1rem 2rem;">
Experimental data in science and engineering are often stored in CSV files as sequences
of numerical values, with each column representing a physical quantity whose unit is either
implicit or indicated in a header. Turning these raw values into strongly typed physical
quantities involves both parsing the serialized data and constructing the corresponding
objects using a physical quantities library. In this article, we will accomplish this with C++26,
particularly its reflection facilities, and mp-units.[^1]

For example, consider a CSV file containing measurements from a projectile-launch experiment
involving parabolic motion (see the figure), with each row storing the launch angle
<code>&#x03D5;</code> in degrees, the measured initial speed `v` in miles per hour, and the
horizontal range to the impact point `r` in centimeters:

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

The deserialization of this CSV file into a container of physical quantities will follow the
simple scheme below:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-1.html %}
</div>

Here, the `shot_data` aggregate provides the strongly typed description of a CSV row: each data
member associates a CSV column with a physical quantity and its unit. More specifically, every
member is an instance of `mp_units::quantity<R, Rep>`, where R identifies the quantity and its unit,
while `Rep` specifies the type used to store its numerical value and defaults to `double`.

A single call to `qcsv::load_quantities<shot_data>`, provided by `quantity_csv.hpp`, returns either
the complete dataset as a `std::vector<shot_data>` or a loading or parsing error, represented through
`std::expected`.[^2]

Running this program produces the following output:

<div class="dgv-terminal-out">0.0°, 5.7 mi/h, 61.5 cm
5.0°, 5.7 mi/h, 64.5 cm
10.0°, 5.6 mi/h, 72.0 cm
15.0°, 5.6 mi/h, 77.3 cm
20.0°, 5.6 mi/h, 82.0 cm
25.0°, 5.4 mi/h, 85.5 cm
30.0°, 5.5 mi/h, 88.1 cm
</div>

<div class="dgv-note" markdown="1">Based on the ISO 80000 standard (International System of
Quantities, ISQ), <b>mp-units</b> is a modern C++ library that provides compile-time dimensional analysis,
unit conversions, and strong type safety for physical quantities. Beyond dimensional correctness,
it validates operations between quantities according to their physical meaning and mathematical
relationships. mp-units provides six distinct layers of safety: dimension safety, unit safety,
representation safety, quantity kind safety, quantity safety, and mathematical space safety.[^3]
Furthermore, mp-units is being proposed for standardization in C++29 through proposal P3045.[^4]
</div>

The rest of the article develops the header-only library in `quantity_csv.hpp`. Concatenating the
code blocks in order produces the complete header. The project has been tested with GCC 16.2.

## Implementation

We begin by including the standard library headers required by the implementation, along with
the `mp-units` core header:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-2.html %}
</div>

Next, we define the error types that the library functions may report through `std::expected`
objects. When parsing a CSV row, we distinguish three possible errors: (i) a value cannot be
parsed because of an invalid format, (ii) an expected value is missing, or (iii) the row contains
more values than expected. A `parse_error` stores an error code identifying the failure and the
CSV column at which it occurred. When processing an entire CSV input, a `row_error` additionally
records the line number at which the parsing error occurred.

Beyond errors associated with parsing individual rows, loading a CSV input may also fail while
reading from the input stream. A `read_error` records the line at which the read operation failed.
When loading directly from a file, an additional failure may occur if the file cannot be opened;
this condition is represented separately by `open_file_error`:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-3.html %}
</div>

Loading a CSV dataset may fail in different ways depending on the input source:

* `load_error`: Loading from an existing input stream may fail either while reading from the
stream (`read_error`) or while parsing a row (`parse_error`).
* `file_load_error`: Loading directly from a file additionally accounts for failure to open
the file (`open_file_error`).

<div class="dgv-cb">
{% include physical-quantities-csv/cb-4.html %}
</div>

As mentioned before, a physical quantity is typically expressed in terms of a numerical value and
a unit of measurement. We therefore begin by introducing the concepts that constrain the underlying
numerical representation of a quantity. The value is considered parsable if it can be converted
directly from a character sequence using `std::from_chars`,[^5] or alternatively extracted from an
input stream using `operator>>`:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-5.html %}
</div>

The following `parse_value` function parses a numerical value using the strategy appropriate
for its underlying representation type. Whenever possible, conversion is performed directly
on the character sequence with `std::from_chars`, providing a lightweight, locale-independent
path. Otherwise, the value is extracted through a `std::ispanstream` imbued with the classic C
locale.[^6] In both cases, floating-point values use a period (`.`) as the decimal separator.
Leading and trailing ASCII whitespace is ignored. Empty or whitespace-only fields are reported
as missing values, while any unparsed characters remaining after conversion result in a format
error:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-6.html %}
</div>

The next step is to constrain the aggregate type used to represent a CSV row and, more
importantly, to verify at compile time that each of its non-static data members is
a parsable mp-units quantity whose numerical value can be populated from a CSV field.
For simplicity, the `plain_aggregate` concept restricts row types to default-initializable
aggregate classes with no base classes and at least one non-static data member.

<div class="dgv-note">Note how C++26 reflection is used to inspect the aggregate at compile time,
discovering the identifiers and types of its non-static data members and verifying that it has
no base classes.
<hr>
The C++26 reflection system can be understood as an interaction between two domains: the program
domain, comprising types, members, expressions, and other language constructs; and the meta-domain,
comprising <code>std::meta::info</code> values. The <code>^^</code> reflection operator projects
program elements into the meta-domain, where their properties can be inspected and the resulting
reflections can be manipulated, for example by querying types, identifiers, members, or template
arguments. Splicers, written <code>[: refl :]</code>, provide the complementary operation: they
reintroduce (reify) entities represented by reflections into the program in the grammatical
contexts where those entities are permitted.
</div>

<div class="dgv-cb">
{% include physical-quantities-csv/cb-7.html %}
</div>

Having established the aggregate's structural requirements, we now validate its individual data
members. The `parsable_quantity` concept requires each member to satisfy `mp_units::Quantity`, to
be non-const so that its numerical value can be modified, and to have a representation type `rep`
satisfying `parsable_value`.

`first_invalid_member` uses a C++26 `template for` expansion statement[^7] to inspect the
reflected members in declaration order. It returns the first member whose type does not
satisfy `parsable_quantity`, or an empty `std::meta::info`[^8] if all members are valid.

Rather than expressing the validity of the entire aggregate as a boolean constraint,
`assert_valid_members` uses the result of `first_invalid_member` to produce a targeted compile-time
diagnostic. Its `static_assert` reports the identifier and type of the offending member, together
with the type of the enclosing aggregate:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-8.html %}
</div>

<div class="dgv-note">Consider the introductory example: if the type of <code>shot_data::v</code>
is changed from <code>speed_type</code> to <code>double</code>, compilation fails with a diagnostic
that reports the identifier and type of the offending data member: *static assertion failed: qcsv
data member 'v' of type double in shot_data is not a parsable mp_units::quantity*.
</div>

With the compile-time requirements in place, we can now use the `parse_row` function to parse
an individual CSV row into an aggregate `T` of physical quantities. The function first validates at
compile time that the aggregate's members are suitable physical quantities for parsing and then
splits the input row using the requested delimiter.
A default-initialized instance of `T` is populated, in declaration order, using a C++26
`template for` expansion statement. For each reflected data member, `template for` expands its body
at compile time, generating the code that parses the corresponding CSV field into the quantity's
numerical representation at runtime.

<div class="dgv-cb">
{% include physical-quantities-csv/cb-9.html %}
</div>

Here, the call to `numerical_value_ref_in(q.unit)` ensures that the numerical value read from the
CSV field is interpreted in the unit carried by the data member.[^9]

<div class="dgv-note">Reflection plays a key role in this design. It allows <code>parse_row</code>
to take the aggregate type itself as a template argument and inspect its non-static data members,
discovering their types and identifiers at compile time. An alternative design could pass the
physical quantity types separately through a template parameter pack. Keeping them instead as
named members of a single aggregate is particularly convenient: the type provides both the
structure of a CSV row and meaningful names for the physical quantities it contains.
</div>

To make the structured errors convenient to present to users, we provide a set of `describe`
overloads that translate each type of error into a human-readable diagnostic. Errors carrying
contextual information use `std::format` to include the relevant line and column numbers:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-10.html %}
</div>

<div class="dgv-note">For instance, an empty second field on line 10 of the CSV, <code>10, , 72.0</code>
produces the message *"CSV error: line 4, column 2: missing value"*, whereas a malformed second
filed, <code>10, 5.64a, 72.0</code>, produces *"CSV error: line 4, column 2: invalid numeric format"*.
</div>

Finally, we provide two overloads of `load_quantities` for loading a complete CSV dataset: one
operates on an existing input stream, while the other takes a `std::filesystem::path` and opens
the corresponding file. If the input is configured to contain a header, the first line is consumed
and skipped. Each subsequent line is parsed with `parse_row` and, on success, the resulting
aggregate is appended to the output vector.

The operation has all-or-nothing semantics: `load_quantities` returns the complete dataset only
if every data row is parsed successfully. Otherwise, it returns the first error encountered,
including any available location information, and discards all previously parsed values:

<div class="dgv-cb">
{% include physical-quantities-csv/cb-11.html %}
</div>

Finally, we make the `load_error` and `file_load_error` variants formattable by specializing
`std::formatter`[^10]. Both specializations inherit the common formatting logic from `error_formatter`,
allowing these error types to be passed directly to `std::format`, `std::println`, and other
facilities based on `std::formatter`:

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
[^4]: www.open-std.org - [Quantities and units library](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p3045r10.html)
[^5]: cppreference.com - [std::from_chars](https://en.cppreference.com/cpp/utility/from_chars)
[^6]: cppreference.com - [std::basic_ispanstream](https://en.cppreference.com/cpp/io/basic_ispanstream)
[^7]: cppreference.com - [template for expansion](https://cppreference.com/cpp/language/template_for)
[^8]: cppreference.com - [std::meta::info](https://cppreference.com/cpp/meta/info)
[^9]: mp-units - [Why can't I convert a double& to a quantity&?](https://mpusz.github.io/mp-units/latest/getting_started/faq/?h=numerical_value_ref_in#why-cant-i-convert-a-double-to-a-quantity)
[^10]: cppreference.com - [std::formatter](https://en.cppreference.com/cpp/utility/format/formatter)
