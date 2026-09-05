---
layout: post
title: "Parallel Directory Traversal"
author: Daniel Gómez Vergel
date: 2026-08-24
categories: [C++, C++26, concurrency]
permalink: /2026/08/24/parallel-directory-traversal/
excerpt: >
  This article presents a parallel directory traversal
  algorithm implemented in C++26 using modules and concurrent
  programming techniques.
---

{% include post-categories.html %}

### Introduction

In this article, we will examine a concurrent programming example in C++ in detail, using it as a case
study to explore dynamic task generation, work-completion detection, and the safe aggregation of partial
results.

Given a directory path, our goal is to produce a statistical summary of the contents of its
directory tree. For each file extension encountered (`.txt`, `.zip`, and so on), we will determine both
the number of files and their cumulative size. The analysis will also report the total number of
subdirectories discovered during the traversal.

In this implementation, we will parallelize the directory tree traversal itself. Each worker thread
will process directories retrieved from a shared concurrent queue, discover any subdirectories they
contain, and dynamically push them into the queue so that they can later be processed by any
available worker. As a result, multiple threads will be able to explore different branches of the
directory tree simultaneously while accumulating partial statistics that will ultimately be merged
into a single global result.

This approach will allow us to generate summaries such as the following for a given root directory:

<div class="dgv-terminal-out"> .docx:  16 files (147.938.289 bytes)
  .pdf:  28 files (72.191.677 bytes)
  .png:    1 file (90.575 bytes)
 .xlsx:    1 file (88.162 bytes)
_____________________________________
contains: 46 files, 5 folders
size: 210.1 MiB (220.308.703 bytes)
</div>

<div class="dgv-note">We will use the C++26 standard, particularly contracts and <code>std::optional<T&></code>,
together with the module system introduced in C++20. The latter allows us to organize the code efficiently into well-defined
components. If necessary, adapting the solution to the traditional structure based on header (<code>.hpp</code>) and
implementation (<code>.cpp</code>) files would be straightforward.
</div>

### Module 1: dynamic_task_queue

We begin by implementing a blocking concurrent work queue, `dynamic_task_queue<T>`, specifically
designed for scenarios in which tasks can dynamically generate additional tasks during their execution.
This data structure greatly simplifies the parallelization of graph traversals, including the directory
hierarchies considered in this article.

This class keeps pending tasks of type `T` in a private standard queue named `tasks_`, of type `std::queue<T>`.
In addition, it maintains a counter called `active_` that tracks the number of tasks currently being
processed. This counter is incremented whenever a task is acquired and decremented when processing of that task completes.
As a result, the queue can automatically detect global completion, which occurs when there are neither
pending tasks nor tasks in progress. The global termination condition is therefore: `tasks_.empty() and active_ == 0`.

Both the queue and the counter are protected by a `std::mutex`[^1], while a `std::condition_variable`[^2] is
used to block worker threads whenever no work is available and to wake them up when new tasks are
added or when global completion is detected.

<div class="dgv-note">It is important to note that an empty <code>tasks_</code> queue does not
necessarily imply that the computation has finished: a worker thread may still be processing a task
and could generate additional tasks at a later stage. The purpose of the <code>active_</code> counter is precisely
to distinguish between these two situations.
</div>

The queue interface will be based on an `acquire/complete` protocol, where every task acquired by a worker
through the `acquire()` member function must eventually be reported as completed through a matching call to the `complete()`
function. In addition, `complete()` may register any new tasks discovered during the processing of the acquired task.
Specifically:

* `acquire()`: If work is available, it retrieves a task from the front of the queue, removes it from the
pending-work list, and internally increments the `active_` task counter. If there are no pending tasks but
other workers are still processing work, the call blocks until new work becomes available or the computation finishes.
When there are neither pending nor active tasks left, the function returns an empty result (`std::nullopt`)
that signals that no further work can be generated and that the worker may terminate.

* `complete()`: Atomically performs two actions: (i) It records the completion of a task previously acquired
via `acquire()` by decrementing the `active_` counter, and (ii) it pushes newly discovered tasks into
the queue. If no pending or active tasks remain after the operation, it signals global completion
so all workers can finish. If there is pending work (`tasks_.empty() == false`), it
wakes up a blocked worker.

To prevent the programmer from having to call `complete()` manually and to guarantee proper task
completion even in the presence of exceptions, `acquire()` does not directly return a task object of type
`T`. Instead, it returns a `std::optional` containing an instance of an auxiliary class named
`acquired_task`. This class acts as an RAII handle representing a task currently in progress. In addition to
the acquired value, it stores a vector of new tasks discovered during processing. When the object goes
out of scope, its destructor automatically invokes `complete()`, transferring the accumulated new tasks
to the queue and signaling the completion of the original task. Thus, the `acquire/complete` protocol
is enforced by design through RAII:

<div class="dgv-cb">
{% include parallel-directory-traversal/cb-1.html %}
</div>

### Module 2: statistics.directory

This module implements the parallel analysis of a directory tree. Given a root directory (`root`),
it produces an object of type `directory_statistics` containing:

* A `std::map` associative container that maps each file extension encountered (`.txt`, `.zip`, and so on)
to an `extension_statistics` object, which stores both the number of files with that extension
present in the directory tree and their cumulative size. The use of `std::map` ensures that file extensions
appear alphabetically in generated reports by default, improving readability.
* The total number of subdirectories discovered.
* The total number of errors encountered during the analysis, whether due to (i) failure to open a
directory, (ii) inability to retrieve information about a specific entry, or (iii) inability to continue
iterating through a directory. These errors are accumulated in a single counter for simplicity of
implementation, but they could easily be broken down into separate counters.

The `run_directory_statistics()` function is responsible for coordinating the parallel execution.
It initializes a `dynamic_task_queue<filesystem::path>` with the root directory, where each task
represents a directory pending analysis within the directory tree. The function then launches a configurable
number of worker threads, `num_workers`, each of which produces a `directory_statistics` object containing the
partial statistics gathered during its execution. Specifically, `num_workers - 1` workers are launched
asynchronously using `std::async`[^3], while the main thread acts as an additional worker processing tasks
from the shared queue.

Each worker executes the `process_directories()` function, which repeatedly acquires directories from the queue
through `acquire()`, inspects their contents, and updates the worker's local `directory_statistics` object
with the statistics of the files found, grouped by extension and cumulative size. Subdirectories discovered during
the traversal are not processed immediately; instead, they are registered as new tasks within the `acquired_task`
object associated with the current directory. Upon destruction, the object automatically transfers the
accumulated tasks to the queue and signals completion of the original task. As we previously mentioned, this
behavior is implemented using RAII.

<div class="dgv-note">Each pending directory <code>dir</code> is represented as a task stored in the
<code>dynamic_task_queue&lt;filesystem::path&gt;</code>, with <code>std::filesystem::is_directory(dir) == true</code>.
As workers discover new subdirectories, they are dynamically added to the queue as additional tasks, whereas regular
files are processed directly and never become tasks themselves.
</div>

The implementation of `process_directories()` is based on `std::filesystem::directory_iterator`[^4], which
iterates over the entries contained within a directory but does not visit its
subdirectories. The iteration order is unspecified by the standard, except that each directory entry
is visited exactly once.

<div class="dgv-note">As noted earlier, under this design each worker maintains its own
<code>directory_statistics</code> object, accumulating statistics only for the directories it processes.
</div>

Once the traversal has completed, the partial results are merged by `merge_statistics()`, which
aggregates the file counts and cumulative sizes for each extension, together with the total number of
subdirectories visited.

<div class="dgv-cb">
{% include parallel-directory-traversal/cb-2.html %}
</div>

### Auxiliary modules

Before turning to the main program, we will define three simple auxiliary modules. Two of them provide thin
wrappers around selected C library facilities, while the third offers formatting utilities for program output.
We begin by examining the compatibility modules:

* `c_tools.exit_codes`: Exposes the standard termination values `EXIT_SUCCESS` and `EXIT_FAILURE`
as `constexpr` variables within the `c_tools` namespace. Its purpose is to facilitate the use of these values from
modular code without relying directly on the macros defined in `<cstdlib>`.

* `c_tools.standard_streams`: Provides `noexcept` functions that return `std::FILE*` pointers to the standard
`stdin`, `stdout`, and `stderr` streams defined in `<cstdio>`.

<div class="dgv-cb">
{% include parallel-directory-traversal/cb-3.html %}
</div>

<div class="dgv-cb">
{% include parallel-directory-traversal/cb-4.html %}
</div>

The remaining auxiliary module is:

* `format_tools`: Extends `std::format` through custom `std::formatter` specializations, allowing binary
sizes to be displayed using more readable units (KiB, MiB, and GiB) and integral values to be formatted
with thousands separators:

<div class="dgv-cb">
{% include parallel-directory-traversal/cb-5.html %}
</div>

### Benchmarks

The following `main()` function benchmarks the various modules developed throughout
this article. It performs the statistical analysis of a directory tree specified as a command-line
argument, using an increasing number of worker threads and measuring the execution time in each case.

Based on these measurements, several performance metrics are computed and displayed in the terminal, including the
speedup relative to the sequential execution (*i.e.*, a single worker thread) and the percentage improvement
achieved when increasing the number of workers:

<div class="dgv-cb">
{% include parallel-directory-traversal/cb-6.html %}
</div>

<div class="dgv-note" markdown="1">The number of worker threads <code>num_workers</code> defaults to
<code>hardware_concurrency()</code>[^5], which
provides an estimate of the concurrency level available on the system, typically matching the number
of hardware threads. However, this value should be interpreted as an indicative upper bound and not
necessarily as the optimal number of workers. In practice, performance may saturate at a lower
thread count due to factors such as contention on the shared queue, the underlying storage device,
or limitations imposed by the file system itself. In particular, different storage technologies may
scale quite differently: modern NVMe SSDs are designed to handle significantly higher degrees of
concurrent access than mechanical drives.
</div>

As an example, on a system equipped with an 11th Gen Intel Core i5-1135G7 processor running at 2.40 GHz,
8 hardware threads, and an SK hynix HFM256GD3HX015N SSD, the following results were obtained when
analyzing a test directory tree:

<div class="dgv-terminal-out">Workers  Time (ms)    Speedup    vs. 1 worker   vs. previous
      1      16331       1.0×            0.0%              -
      2      10194       1.6×           37.6%          37.6%
      3       7777       2.1×           52.4%          23.7%
      4       6242       2.6×           61.8%          19.7%
      5       4994       3.3×           69.4%          20.0%
      6       4545       3.6×           72.2%           9.0%
      7       4281       3.8×           73.8%           5.8%
      8       4125       4.0×           74.7%           3.6%
____________________________________________________________
contains: 136.613 files, 7.695 folders
size: 7.2 GiB (7.679.411.327 bytes)
errors: 0
</div>

Here, the "vs. 1 worker" column expresses the percentage reduction in execution time relative to the
sequential version.

The associative container within the final `directory_statistics` result can be used, among other things,
to generate breakdowns like the one shown in the introduction or to retrieve statistics for a specific
file extension. For instance, to determine the total number of `.txt` files and their cumulative size in
a `root` directory, we would write:

<div class="dgv-cb">
{% include parallel-directory-traversal/cb-7.html %}
</div>

---

Note: This post is an English translation of my earlier post originally published in Spanish on Blogger as [Programación Concurrente IX](https://dgvergel.blogspot.com/2026/08/programacion-concurrente-ix.html).

### Bibliography

[^1]: cppreference.com – [std::mutex](https://en.cppreference.com/cpp/thread/mutex)
[^2]: cppreference.com – [std::condition_variable](https://en.cppreference.com/cpp/thread/condition_variable)
[^3]: cppreference.com – [std::async](https://en.cppreference.com/cpp/thread/async)
[^4]: cppreference.com – [std::filesystem::directory_iterator](https://en.cppreference.com/cpp/filesystem/directory_iterator)
[^5]: cppreference.com – [std::thread::hardware_concurrency](https://cppreference.com/cpp/thread/thread/hardware_concurrency)
