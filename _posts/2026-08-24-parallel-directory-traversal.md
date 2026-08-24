---
layout: post
title: "Parallel directory traversal: A dynamic queue approach"
date: 2026-08-24
categories: [concurrency, cpp26, dynamic task queue]
---

## Introduction

In this article, we will take an in-depth look at a concurrent programming example in C++.

Given the path of a directory, our goal is to produce a statistical summary of the contents of its
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
  .png:   1 files (90.575 bytes)
 .xlsx:   1 files (88.162 bytes)
_____________________________________
contains: 46 files, 5 folders
size: 210.1 MiB (220.308.703 bytes)
</div>

We will use the C++26 standard, in particular contracts and `std::optional<T&&>`, together with the
module system introduced in C++20, which allows us to organize the code into well-defined, self-contained
components. This choice is motivated by considerations of code simplicity, clarity, and modernity.
Should it be necessary, adapting the solution to the traditional structure based on header (`.hpp`) and
implementation (`.cpp`) files would be straightforward.

## Module #1: concurrency_tools.dynamic_task_queue

We begin by implementing a blocking concurrent work queue, `dynamic_task_queue<T>`, specifically
designed for scenarios in which tasks can dynamically generate additional tasks during their execution.
Such a structure greatly simplifies the parallelization of graph traversals, including the directory
hierarchies considered in this article.

This class stores pending tasks of type `T` in a private standard queue named `tasks_`, of type `std::queue<T>`.
In addition, it maintains a counter called `active_` that tracks the number of tasks currently being
processed. This counter is incremented whenever a task is acquired and decremented when processing completes.
As a result, the queue can automatically detect global completion, which occurs when there are neither
pending tasks nor tasks in progress. Formally, the termination condition is `tasks_.empty() && active_ == 0`.

Both the queue and the counter are protected by a `std::mutex`, while a `std::condition_variable` is
used to block worker threads whenever no work is available, and to wake them up when new tasks are
added or when global completion is detected. It is important to note that an empty queue does not
necessarily imply that the computation has finished: a worker thread may still be processing a task
and could generate additional tasks at a later stage. The purpose of the `active_` counter is precisely
to distinguish between these two situations.

The queue interface will be based on an `acquire/complete` protocol, where every task acquisition by
a worker via the `acquire()` member function must be paired with a subsequent call to the `complete()`
function. The latter may register new derived tasks. Specifically:

* `acquire()`: If work is available, it pops a task and internally increments the active task counter
`active_`. If there are no pending tasks but other workers are still processing work, the call blocks
until new work appears or the computation finishes. When there are no pending or active tasks left,
it returns an empty result (`std::nullopt`) to indicate that the exploration is complete and workers
can terminate.

* `complete()`: Atomically performs two actions: (i) It logs the completion of a task previously acquired
via `acquire()` by decrementing the `active_` counter, and (ii) it pushes newly discovered tasks into
the queue. If no pending or active tasks remain after the operation, it notifies the global completion
of the computation so all workers can exit. If there is pending work (`tasks_.empty() == false`), it
wakes up a blocked worker.

To prevent the programmer from having to call `complete()` manually and to guarantee proper task
completion even in the presence of exceptions, `acquire()` does not directly return an object of type
`T`. Instead, it returns an optional value (`std::optional`) of an auxiliary `acquired_task` handle
class. This class acts as an RAII object representing a task currently in progress. In addition to
the acquired value, it stores a vector of new tasks discovered during processing. When the object goes
out of scope, its destructor automatically invokes `complete()`, transferring the accumulated new tasks
to the queue and signaling the completion of the original task. Thus, the `acquire/complete` protocol
is guaranteed by design using the RAII technique:

<div class = "dgv-cb">{% include parallel-directory-traversal/cb-1.html %}</div>

To be continued...
