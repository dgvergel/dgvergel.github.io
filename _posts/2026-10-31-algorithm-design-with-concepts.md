---
layout: post
title: "Algorithm Design with Concepts"
author: Daniel Gómez Vergel
date: 2026-10-31
last_modified_at: 2026-10-31
categories: [C++, algorithms, concepts, projections, ranges]
permalink: /2026/10/31/algorithm-design-with-concepts/
excerpt: >
  In this post, we consider the problem of implementing the well-known insertion sort algorithm using generic programming techniques. We examine its implementation in light of the introduction of concepts, ranges, and projections into the C++20 language standard and later versions.
---

{% include post-categories.html %}

### Introduction

In this post, we will consider the problem of implementing the well-known insertion sort algorithm using generic programming techniques, so that it can be used with a variety of sequence containers such as `std::vector` or `std::list`. We will examine its implementation in light of the introduction of concepts, ranges, and projections into the C++20 language standard and later versions [^1]<sup>,</sup>[^2]<sup>,</sup>[^3]<sup>,</sup>[^4].

<div class="dgv-note">A concept is a named compile-time predicate which is part of the interface of a class or function template and constrains the types of data with which it may operate. The early detection of any violation of such constraints, during the initial stages of template instantiation, will generally lead to error messages that are easier to understand and trace. Rather than serving merely as syntactic constraints, the goal is to model genuine semantic categories that can be reused by the programmer.
</div>

For the sole purpose of making our code easier to read, we will introduce the following declaration:

<div class="dgv-terminal-out">using namespace std;
</div>

### Insertion sorting

Let `R` be a range to be sorted according to a given binary predicate `comp`, which
must establish a strict weak ordering over the operands it compares. That is, we assume that `comp`
satisfies the following properties:

* Irreflexivity: `comp(a,a)` is false for every `a`.
* Transitivity: `comp(a,b)` and `comp(b,c)` imply `comp(a,c)`.
* If we define `equiv(a,b)` as `!comp(a,b)` and `!comp(b,a)`, then `equiv` must also be transitive:
`equiv(a,b)` and `equiv(b,c)` imply `equiv(a,c)`.

Under the above assumptions, `equiv` constitutes an equivalence relation, and `comp` induces a strict
weak ordering on the equivalence classes established by `equiv`.

Let us consider the general case in which the `R` range contains at least two elements. During its
first iteration, insertion sort must order the first two elements of `R`, that is, if `comp(R[0], R[1])`
evaluates to `false`, the two elements must be swapped; otherwise, they remain in their original positions.
Next, we insert the third element, `R[2]` (if it exists), into the correct position within the already
sorted subrange `{R[0], R[1]}`. At this point, the first three elements of `R` are correctly ordered.
We then insert the fourth element, `R[3]` (if it exists), into its proper position with respect to the
already sorted subrange `{R[0], R[1], R[2]}`, and so on until the entire range `R` has been sorted.

The following figure shows, from left to right and from top to bottom, the successive iterations involved
in sorting the set of natural numbers `R = {5, 3, 1, 2}` in ascending order:

<div style="text-align: center; margin: 2em 0;">
  <img src="{{ '/assets/images/algorithm-design-with-concepts/fig-1.webp' | relative_url }}"
       alt="Figure 1"
       style="max-width: 100%; height: auto;">
</div>

Let `n` denote the number of elements to be sorted. This algorithm has a time complexity of `O(n)`
in the best-case scenario (when the input sequence is
already sorted) and `O(n²)` in both the average and worst-case scenarios (when the initial sequence is
arranged in the reverse of the desired order) [^5].

Admittedly, the appeal of insertion sort lies primarily in its relative ease of implementation.
In production code, however, more efficient sorting algorithms, such as quicksort or mergesort,
are generally preferred.

Let us consider a half-open range `R = [first, last)` to be sorted using insertion sort,
where `first` is an iterator of type `I` and `last` is a sentinel of type `S` (not necessarily the same
type as `first`, as we shall see in a later example). Let `proj` denote a projection to be applied to
the elements of the range. For example, it may take the form of a unary function object whose
`operator()` accepts a reference to a constant element of the range and returns a suitable transformation
of that element. Such projections make it possible, for instance, to sort containers of aggregates
according to the natural ordering of one of their data members. Let `comp` denote the binary predicate evaluated on
the projected values of the range elements. The range `R` is said to be ordered with respect to `comp` and `proj` if,
for every iterator `i` referring to one of its elements, `invoke(comp,invoke(proj,*ranges::next(i,n)),invoke(proj,*i))`
evaluates to `false` for every non-negative integer `n` such that `std::ranges::next(i, n)` still
refers to a valid element of the range. The use of the standard function template `std::invoke` is justified in
this generic programming context by the variety of callable entities that may be used to represent `comp` and `proj`,
including function objects and function pointers.

We shall introduce the following signature for the `insertion_sort algorithm`, making explicit in its
interface the constraints imposed on the types with which it operates:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-1.html %}
</div>

As we can see, we have specified a number of constraints both within the template parameter list
(enclosed in angle brackets) and through the requires clause. In order of appearance:

1. The iterator type `I` must satisfy at least the forward iterator requirements, supporting the
usual comparison, dereference, and increment operations (`++` in both its prefix and postfix forms).
2. The type `S` must act as a sentinel for `I`. This means that the expression `first == last` must be
well-formed and that, whenever `bool(first != last)` evaluates to `true`, `first` must be dereferenceable
and `[++first, last)` must in turn denote a valid range.
3. The triplet `<I, Comp, Proj>` must satisfy the standard constraint `std::sortable` [^6], which is
defined as the conjunction of the concepts `std::permutable` and `std::indirect_strict_weak_order` (see the listing below).
Among other requirements, these concepts mandate that objects of type `std::iter_value_t<I>`,
referenced through the iterators, be movable and swappable through swap operations, and that the predicate
type `Comp` establish a strict weak ordering over the projected elements. In practice, however, the
compiler can only verify that `Comp` defines a valid binary relation between such elements. The semantic
requirements, namely irreflexivity and transitivity, must be guaranteed by the programmer.

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-2.html %}
</div>

Note that we do not require objects of type `std::iter_value_t<I>` to be copyable, a more restrictive
condition that would subsume the previous movability requirement. This is because we wish to support,
for example, the sorting of ownership-enforcing smart pointers such as `std::unique_ptr` according to
the values they reference, a case that we will examine later.

The default sorting policy adopted by the `insertion_sort` algorithm is provided by `std::ranges::less`,
while the default projection is the function object type `std::identity`, whose `operator()` returns
its argument unchanged.

When generating a specific instance of the function template, the compiler will verify that the supplied
template arguments satisfy the stated constraints, issuing a meaningful diagnostic message otherwise.
With our version of the algorithm, we are able to operate on forward iterators, including those of a
`std::forward_list`, a bidirectional `std::list`, and contiguous containers such as `std::array` and
`std::vector`. The following static assertion provides evidence of this:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-3.html %}
</div>

Next, we shall implement the body of `insertion_sort` as a composition of well-known standard algorithms[^7]<sup>,</sup>[^8]:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-4.html %}
</div>

As an optimization, we take into account that if the range to be sorted, `[first, last)` , is empty
or contains only a single element, the algorithm can terminate immediately. The compiler will select
control statement `(A)` whenever the distance between `first` and `last` can be computed in constant
time, that is, when the concept `sized_sentinel_for<S, I>` evaluates to `true` at compile time
(as is the case, for example, for a pair of random-access iterators). Otherwise, statement `(B)`
will be used instead, as happens with pairs of forward or bidirectional iterators.

The preceding `for` loop maintains the subrange `[first, i]` in sorted order at every iteration,
assuming that the single-element subrange `[first, next(first))` is already sorted.
The `std::ranges::upper_bound` algorithm `(C)` performs a binary search to locate the furthest
iterator `j` within `[first, i]` such that `!bool(invoke(comp, invoke(proj,*i),invoke(proj,*k)))`
evaluates to `true` for every iterator `k` in the subrange `[first, j)`. The `std::ranges::rotate`
algorithm (D) then places the element pointed to by `i` at position `j` by means of a cyclic permutation.

<div style="text-align: center; margin: 2em 0;">
  <img src="{{ '/assets/images/algorithm-design-with-concepts/fig-2.webp' | relative_url }}"
       alt="Figure 2"
       style="max-width: 100%; height: auto;">
</div>

It is worth noting, in particular, how every operation performed within the function body conforms
exactly to the constraints imposed on its arguments. Also note that the comparator and projection
objects are captured by value, in accordance with the standard specification of this class of
algorithms.

We shall also provide an overload of the algorithm that operates directly on a range, rather than on
the iterator-sentinel pair used by the first version:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-5.html %}
</div>

In this version, we require the type `R` to satisfy the `std::ranges::forward_range` concept and
subsequently require the range iterator type `std::ranges::iterator_t<R>`, the comparator type `Comp`,
and the projection type `Proj` to satisfy the `std::sortable` concept. In particular, the expression
`ranges::end(r)` will return an object that models `std::sentinel_for<std::ranges::iterator_t<R>>`.

### Example 1: Movable but non-copyable elements

The following code shows how the previous algorithm can be used to sort an array of movable but non-copyable elements, in this case, `std::unique_ptr<int>` smart pointers. The comparator, defined as a lambda expression, takes two such pointers and dereferences them to establish the natural ordering of the integers they point to:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-6.html %}
</div>

The memory layout of the array in the example would follow a scheme like the one below, before and after sorting (transferring ownership of the integers when necessary):

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

As we have seen, projections allow the sorting algorithm to apply transformations to the elements of
the range before comparing them. As a further example, the following simple code sorts a vector of `Book`
objects based on one of their public attributes, the price, using `ranges::greater` to sort them in descending order:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-8.html %}
</div>

### Example 3: Sentinels

The following example, which sorts a C-style null-terminated string (zero-terminated string), illustrates a situation in which
the iterator type `I` (a `char const*`) and the sentinel type `S` (named `Zstr_sentinel`) are different:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-9.html %}
</div>

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-10.html %}
</div>

A more detailed explanation of the `Zstr_sentinel` type can be found in reference[^9].

### Example 4: Compile-time sorting

We will conclude this article by putting C++’s compile-time computation capabilities to the test once again. The reader will no doubt have noticed that the `insertion_sort` sorting algorithm overloads have been declared with the `constexpr` specifier. This allows them to be executed at compile time, provided that the call constitutes a constant expression[^10].

As an example, let us define the following `Sorted_array<T, N>` class template, which publicly inherits from `std::array<T, N>`. Given an arbitrary array, a comparator, and a projection, the class constructor will store and sort the data by calling `insertion_sort`:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-11.html %}
</div>

Let us consider the following simple program:

<div class="dgv-cb">
{% include algorithm-design-with-concepts/cb-12.html %}
</div>

Its x86-64 assembly code (compiled with GCC 16.2 with -O3 optimization and -DNDEBUG) shows that the global integer array entries (with static storage duration and stored in the data segment) is constructed and sorted in descending order at compile time[^11]:

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

It is striking that, in this case, the code in our article ultimately results in such a small set of instructions.

Interestingly, the Clang 23.1.0 compiler is able to unroll the `for` loop used for printing and produce a sequence of calls to `printf`[^12]. This optimization can improve the execution speed of our program at the cost of increasing the size of the executable:

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

### Bibliography

[^1]: cppreference.com - [Constraints and concepts](https://en.cppreference.com/w/cpp/language/constraints)
[^2]: cppreference.com - [Concepts library](https://en.cppreference.com/w/cpp/concepts)
[^3]: cppreference.com - [Iterator library](https://en.cppreference.com/w/cpp/iterator)
[^4]: cppreference.com - [Rages library](https://en.cppreference.com/w/cpp/ranges)
[^5]: Cormen T. H., et al., *Introduction to Algorithms*. 3rd Edition, MIT Press (2009).
[^6]: cppreference.com - [std::sortable](https://en.cppreference.com/w/cpp/iterator/sortable)
[^7]: cppreference.com - [std::rotate](https://en.cppreference.com/w/cpp/algorithm/rotate)
[^8]: Stack Overflow - [How to implement classic sorting algorithms in modern C++?](https://stackoverflow.com/questions/24650626/how-to-implement-classic-sorting-algorithms-in-modern-c)
[^9]: foonathan::​blog() - [C++20’s Iterator Sentinels](https://foonathan.net/2020/03/iterator-sentinel/)
[^10]: cppreference.com - [Constant expression](https://en.cppreference.com/w/c/language/constant_expression)
[^11]: Compiler Explorer - [Test with GCC](https://godbolt.org/z/dKqqaofY1)
[^12]: Eric Niebler - [Quicksort of an array of ints at compile-time using C++17 constexpr](https://twitter.com/ericniebler/status/1270832838754004992)
