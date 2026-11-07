---
layout: post
title: "Algorithm Design with Concepts, Ranges, and Projections"
author: Daniel Gómez Vergel
date: 2026-10-31
last_modified_at: 2026-11-06
categories: [C++, algorithms, concepts, projections, ranges]
permalink: /2026/10/31/algorithm-design-with-concepts/
excerpt: >
  In this post, we consider the problem of implementing the well-known insertion sort algorithm using generic programming techniques. We examine its implementation in light of the introduction of concepts, ranges, and projections into the C++20 language standard and later versions.
---

{% include post-categories.html %}

### Introduction

In this post, we will use the well-known insertion sort algorithm as a case study in the design of generic algorithms.
We will explore how concepts, ranges, and projections, introduced in C++20 and further developed in subsequent
standards,[^1]<sup>,</sup>[^2]<sup>,</sup>[^3]<sup>,</sup>[^4] can be used to develop a flexible generic implementation of the
algorithm, capable of operating on a variety of sequence containers such as `std::vector` and `std::list`. For reference, we begin
with brief working definitions:

<div class="dgv-def">A <b>concept</b> is a named compile-time predicate that defines a set of constraints on template arguments
and can be used to restrict the interfaces of class and function templates, as well as to select the most appropriate function
overloads and template specializations. These constraints are checked at compile time and generally provide clearer and more
informative diagnostic messages when they are not satisfied. Beyond expressing syntactic requirements, concepts are intended
to model meaningful semantic categories and properties that can be reused across generic code. Although concepts can express both
syntactic and semantic requirements, compilers generally check only the former; semantic requirements must generally be ensured by
the programmer.
<hr>
A <b>range</b> designates a sequence of elements, potentially unbounded, over which a computation is to be performed through
iteration. It is specified by an iterator and a sentinel denoting the beginning and end of the computation, respectively.
The sentinel need not have the same type as the iterator.
<hr>
A <b>projection</b> is a transformation applied by an algorithm to an element before its value is inspected.
</div>

### Insertion sort

Let `R` be a range to be sorted according to a given binary predicate `comp`, which
must induce a strict weak ordering on the values being compared. That is, we assume that `comp`
satisfies the following properties:

* Irreflexivity: `comp(a,a)` is false for every `a`.
* Transitivity: `comp(a,b)` and `comp(b,c)` imply `comp(a,c)`.
* If we define `equiv(a,b)` as `!comp(a,b) and !comp(b,a)`, then `equiv` must also be transitive:
`equiv(a,b)` and `equiv(b,c)` imply `equiv(a,c)`.

Consequently, `equiv` partitions the elements into equivalence classes, on which `comp` induces a strict total ordering.

Let us assume that the range `R` contains at least two elements, denoted in order of occurrence by `r₀, r₁, ...`.
This notation does not imply random access.
During its first iteration, insertion sort must order the first two elements of `R`,
that is, if `comp(r₁, r₀)` evaluates to `true`, the two elements must be swapped;
otherwise, they remain in their original positions. Next, we insert the third element, `r₂` (if it exists),
into the correct position within the already sorted subrange `{r₀, r₁}`. At this point, the first
three elements of `R` are correctly ordered. We then insert the fourth element, `r₃` (if it exists), into its
proper position with respect to the already sorted subrange `{r₀, r₁, r₂}`, and so on
until the entire range `R` has been sorted.[^5]

The following figure shows, from left to right and from top to bottom, the successive iterations involved
in sorting the sequence `R = {5, 3, 1, 2}` in ascending order:

<div style="text-align: center; margin: 2em 0;">
  <img src="{{ '/assets/images/algorithm-design-with-concepts/fig-1.webp' | relative_url }}"
       alt="Figure 1"
       style="max-width: 100%; height: auto;">
</div>

<div class="dgv-note">Insertion sort remains useful for small sequences and performs particularly
well on nearly sorted data. Its relative ease of implementation also makes it a convenient case study
for generic algorithm design. For general-purpose sorting, however, a standard library algorithm such as
<code>std::ranges::sort</code> will usually be a more appropriate choice. Interestingly, implementations
of standard sorting algorithms commonly use insertion sort internally to handle small subranges.
</div>

Let us consider a half-open range `R = [first, last)` to be sorted using insertion sort,
where `first` is an iterator of type `I` and `last` is a sentinel of type `S` (which need not be the same
type as `I`, as we shall see in a later example). Let `proj` denote a projection to be applied to the
elements of the range.  It may, for example, be a function object or a pointer to a data member.
As we have already mentioned, a projection allows the algorithm to apply transformations to range elements
before comparing their values. This makes it possible, for instance, to sort objects according
to the ordering of one of their data members.  Let `comp` denote the binary predicate evaluated on the projected values of
the range elements. The range `R` is said to be ordered with respect to `comp` and `proj` if, for every iterator `i` referring to
one of its elements, `std::invoke(comp, std::invoke(proj, *std::ranges::next(i, n)), std::invoke(proj, *i))` evaluates to
`false` for every positive integer offset `n` such that `std::ranges::next(i, n)` still denotes a valid element of the range.

<div class="dgv-note">The use of the standard function template <code>std::invoke</code> is justified in
this generic programming context by the variety of callable entities that may be used to represent <code>comp</code>
and <code>proj</code>, including function objects, function pointers, and pointers to members.
</div>

We shall introduce the following interface for the `insertion_sort` algorithm, making the constraints imposed on its
argument types explicit. In what follows, we will use the `using namespace std;` directive throughout the code to improve
readability:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-1.html %}
</div>

As we can see, we have specified a number of constraints both within the template parameter list
(enclosed in angle brackets) and through the `requires` clause. In order of appearance:

* The iterator type `I` must model at least the `std::forward_iterator` concept. In particular, it must be copyable,
incrementable (`++`), dereferenceable when it denotes an element, and support multiple independent passes over the same range.
Note that <code>std::forward_iterator</code> does not require the iterator to support modification of the elements it refers to;
that requirement is imposed separately by <code>std::permutable</code> (see below).

* The type `S` must model `std::sentinel_for<S, I>`, allowing a value of type `S` to denote the end of a range whose
iterator type is `I`. In particular, `S` must be semiregular, and objects of types `I` and `S` must be weakly equality-comparable.
One of its semantic requirements is that, whenever `bool(first != last)` evaluates to `true`, `first` must be dereferenceable, and
incrementing it must leave `[first, last)` as a valid range. As noted before, `S` need not be the same type as `I`.

* The types `I`, `Comp`, and `Proj` must jointly model the `std::sortable` concept,[^6] which is
defined as the conjunction of the concepts `std::permutable` and `std::indirect_strict_weak_order` (see the listing below).
Among other requirements, `std::permutable` ensures that the iterator type supports the indirect movement, storage,
and swapping of elements needed to rearrange them. Meanwhile, `std::indirect_strict_weak_order` requires the comparison predicate
to induce a strict weak ordering on the projected elements; here, `std::projected<I, Proj>` provides an indirectly readable type
whose reference type is the result of applying `Proj` to `std::iter_reference_t<I>`, the reference type of `I`.

<div class="dgv-note">
The compiler can check that <code>Comp</code> can be used as a predicate and that the required expressions are well-formed, but it
cannot verify the semantic requirements of a strict weak ordering (including irreflexivity, transitivity, and transitivity of
the induced equivalence relation), which must therefore be ensured by the programmer.
</div>

Omitting namespace qualification for clarity, `std::sortable` is defined essentially as follows:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-2.html %}
</div>

<div class="dgv-note">Note that we do not require objects of type <code>std::iter_value_t&lt;I&gt;</code> to be copyable.
Copyability would be a stronger requirement than necessary and would prevent the algorithm from supporting
move-only types. We may wish, for example, to sort exclusive-ownership smart pointers such as <code>std::unique_ptr</code>
according to the values of the objects they own, a case that we will examine later.
</div>

The default sorting policy adopted by the `insertion_sort` algorithm is provided by `std::ranges::less`,
while the default projection is the function object type `std::identity`, whose `operator()` returns
its argument unchanged.

When generating a specific instance of the function template, the compiler will verify that the supplied
template arguments satisfy the stated constraints, issuing a meaningful diagnostic message otherwise.
Our version of the algorithm can operate on a wide variety of iterators, including the forward iterators of
`std::forward_list`, the bidirectional iterators of `std::list`, and the contiguous iterators of containers
such as `std::array` and `std::vector`:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-3.html %}
</div>

Next, we shall implement the body of `insertion_sort` as a composition of well-known standard
algorithms.[^7]<sup>,</sup>[^8]<sup>,</sup>[^9]<sup>,</sup>[^10] This provides a direct and concise formulation
that supports forward ranges:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-4.html %}
</div>

First, we take into account that if the range to be sorted, `[first, last)` , is empty
or contains only a single element, the algorithm can terminate immediately. The compiler selects the
first branch **(A)**  when `S` and `I` model `std::sized_sentinel_for` (as is the case, for example, for a pair of
random-access iterators), in which case their distance can be obtained in constant time. Otherwise, the second
branch **(B)** is used. This is typically the case, for example, with iterator pairs from `std::list` or `std::forward_list`.

At the beginning of each `for` loop iteration, the subrange `[first, i)` is sorted. The current element,
denoted by `i`, is then inserted into its proper position, after which `[first, std::ranges::next(i))` is sorted.
To locate the insertion position `j`, we distinguish two strategies according to the capabilities of the iterator type:
* `I` models `std::bidirectional_iterator`: the sorted subrange `[first, i)` is traversed in reverse order, starting
from the element immediately preceding `i`. The `std::ranges::find_if_not` algorithm **(C)** stops as soon as it finds
an element that the current element does not precede according to `comp` and `proj`. The `base()` member of the resulting
reverse iterator then yields the position immediately after that element.
* `I` is only a forward iterator: the `std::ranges::upper_bound` algorithm **(D)** performs a binary search on the
sorted subrange `[first, i)` to locate the insertion position.

In either case, the resulting iterator `j` denotes a position after any previously processed elements equivalent to the
current one according to `comp` and `proj`. More precisely, `!bool(std::invoke(comp, std::invoke(proj, *i), std::invoke(proj, *k)))`
evaluates to `true` for every iterator `k` in the subrange `[first, j)`.

The `std::ranges::rotate` algorithm **(E)** then moves the element denoted by `i` to position `j` by means of a cyclic
permutation of `[j, std::ranges::next(i))`.

<div style="text-align: center; margin: 2em 0;">
  <img src="{{ '/assets/images/algorithm-design-with-concepts/fig-2.webp' | relative_url }}"
       alt="Figure 2"
       style="max-width: 100%; height: auto;">
</div>

<div class="dgv-note">It is worth noting, in particular, how every operation performed within the function
body conforms exactly to the constraints imposed on its arguments. Also note that the comparator and projection
objects are passed by value, consistently with the interfaces of the standard Ranges algorithms.
</div>

We shall also provide an overload of the algorithm that operates directly on a range, rather than on
the iterator-sentinel pair used by the first version:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-5.html %}
</div>

In this version, we require the type `R` to satisfy the `std::ranges::forward_range` concept and
subsequently require the range iterator type `std::ranges::iterator_t<R>`, the comparator type `Comp`,
and the projection type `Proj` to satisfy the `std::sortable` concept.

### Example 1: Movable but non-copyable elements

The following code shows how the previous algorithm can be used to sort an array of movable but non-copyable elements,
in this case `std::unique_ptr<int>` smart pointers. The comparator, defined as a lambda expression, takes two such pointers
and dereferences them to establish the natural ordering of the integers they point to:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-6.html %}
</div>

The memory layout of the array in the example would follow a scheme like the one below, before (left) and after (right) sorting,
with ownership of the managed integers being transferred as the smart pointers are moved:

<div style="text-align: center; margin: 2em 0;">
  <img src="{{ '/assets/images/algorithm-design-with-concepts/fig-3.webp' | relative_url }}"
       alt="Figure 3"
       style="max-width: 100%; height: auto;">
</div>

Alternatively, the array could be sorted using a projection that dereferences the pointers:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-7.html %}
</div>

### Example 2: Projections

As we have seen, projections allow the sorting algorithm to specify which value derived from each range element is
to be used for comparison. As a further example, the following simple code sorts a vector of `Book` objects
according to one of their public data members, `price`, using the `std::ranges::greater` function object to sort the
books in descending order:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-8.html %}
</div>

### Example 3: Sentinels

The following example sorts a C-style null-terminated string and illustrates a situation in which
the iterator type `I` (a `char*`) and the sentinel type `S` (named `Zstr_sentinel`) are different:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-9.html %}
</div>

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-10.html %}
</div>

Here, the null character belongs to the underlying array but acts as the end marker, and is therefore
not part of the range being sorted. A more detailed explanation of the `Zstr_sentinel` type is available
in the literature.[^11]

### Example 4: Compile-time sorting

We will conclude this article by putting C++’s compile-time computation capabilities to the test once again. The reader will no doubt have noticed that the `insertion_sort` overloads have been declared `constexpr`. This allows them to be evaluated at compile time when invoked in a context that requires a constant expression.[^12]

As an example, let us define the following `Sorted_array<T, N, Comp, Proj>` class template, which publicly inherits from `std::array<T, N>`.
Given an array rvalue, a comparator, and a projection, the class constructor will store and sort the data by calling `insertion_sort`:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-11.html %}
</div>

Let us consider the following simple program:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-12.html %}
</div>

The generated x86-64 assembly (compiled with GCC 16.2 using -O3 and -DNDEBUG) shows that the sorting of the array `entries` has been
completed at compile time; its elements already appear in descending order, and no run-time sorting code is generated:[^13]

<div class="dgv-terminal-out">.LC0:
        .string "%d\n"
"main":
        push    rbx
        xor     ebx, ebx
.L2:
        mov     esi, DWORD PTR "entries"[rbx]
        mov     edi, OFFSET FLAT:.LC0
        xor     eax, eax
        add     rbx, 4
        call    "printf"
        cmp     rbx, 20
        jne     .L2
        xor     eax, eax
        pop     rbx
        ret
"entries":
        .long   12
        .long   7
        .long   5
        .long   1
        .long   -3
</div>

<div class="dgv-note">Remarkably, none of the machinery used to sort the array appears in the generated run-time code:
the executable contains only the already sorted data and the instructions required to print it.
</div>

Interestingly, the Clang 23.1.0 compiler is able to unroll the `for` loop used for printing and produce
a sequence of calls to `printf`.[^14] Such a transformation may improve execution speed in some circumstances,
at the cost of increasing the size of the generated code:

<div class="dgv-terminal-out">main:
        push    rbx
        lea     rbx, [rip + .L.str]
        mov     rdi, rbx
        mov     esi, 12
        xor     eax, eax
        call    printf@PLT
        mov     rdi, rbx
        mov     esi, 7
        xor     eax, eax
        call    printf@PLT
        mov     rdi, rbx
        mov     esi, 5
        xor     eax, eax
        call    printf@PLT
        mov     rdi, rbx
        mov     esi, 1
        xor     eax, eax
        call    printf@PLT
        mov     rdi, rbx
        mov     esi, -3
        xor     eax, eax
        call    printf@PLT
        xor     eax, eax
        pop     rbx
        ret
.L.str:
        .asciz  "%d\n"
</div>

---
Note: This post is a translated and expanded version of an earlier article that I originally published in Spanish on Blogger: [Programando con C++20 (Parte I): Concepts](https://dgvergel.blogspot.com/2020/06/programando-cpp20-parte-i.html).

### Bibliography

[^1]: cppreference.com - [Constraints and concepts](https://en.cppreference.com/w/cpp/language/constraints)
[^2]: cppreference.com - [Concepts library](https://en.cppreference.com/w/cpp/concepts)
[^3]: cppreference.com - [Iterator library](https://en.cppreference.com/w/cpp/iterator)
[^4]: cppreference.com - [Rages library](https://en.cppreference.com/w/cpp/ranges)
[^5]: Cormen T. H., et al., *Introduction to Algorithms*. 3rd Edition, MIT Press (2009).
[^6]: cppreference.com - [std::sortable](https://en.cppreference.com/w/cpp/iterator/sortable)
[^7]: cppreference.com - [std::ranges::find_if_not](https://en.cppreference.com/cpp/algorithm/ranges/find)
[^8]: cppreference.com - [std::ranges::upper_bound](https://en.cppreference.com/cpp/algorithm/ranges/upper_bound)
[^9]: cppreference.com - [std::ranges::rotate](https://en.cppreference.com/cpp/algorithm/ranges/rotate)
[^10]: Stack Overflow - [How to implement classic sorting algorithms in modern C++?](https://stackoverflow.com/questions/24650626/how-to-implement-classic-sorting-algorithms-in-modern-c)
[^11]: foonathan::​blog() - [C++20’s Iterator Sentinels](https://foonathan.net/2020/03/iterator-sentinel/)
[^12]: cppreference.com - [Constant expression](https://en.cppreference.com/w/c/language/constant_expression)
[^13]: Compiler Explorer - [Article's code (compiled with GCC)](https://godbolt.org/z/cxr96T5rY)
[^14]: Eric Niebler - [Quicksort of an array of ints at compile-time using C++17 constexpr](https://godbolt.org/z/wtsYKF)
