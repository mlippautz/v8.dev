---
title: 'High-performance garbage collection for C++'
author: 'Anton Bikineev, Omer Katz ([@omerktz](https://twitter.com/omerktz)), and Michael Lippautz ([@mlippautz](https://twitter.com/mlippautz)), C++ memory whisperers'
avatars:
  - 'anton-bikineev'
  - 'omer-katz'
  - 'michael-lippautz'
date: 2020-05-15
tags:
  - memory
  - internals
  - cppgc
description: 'This post describes the Oilpan C++ garbage collector, its usage in Blink, and how it optimizes sweeping, i.e., reclamation of unreachable memory.'
---

In the past we have [already](https://v8.dev/blog/trash-talk) [been](https://v8.dev/blog/concurrent-marking) [writing](https://v8.dev/blog/tracing-js-dom) about garbage collection for JavaScript, the document object model (DOM), and how all of this is implemented and optimized in V8.Not everything in Chromium is JavaScript though, as most of the browser and its Blink rendering engine where V8 is embedded are written in C++. JavaScript can be used to interact with the DOM that is then processed by the rendering pipeline. Because the C++ object graph around the DOM is heavily tangled with Javascript objects, the Chromium team switched a couple of years ago to a garbage collector, called [Oilpan](https://www.youtube.com/watch?v=_uxmEyd6uxo), for managing this kind of memory. Oilpan is a garbage collector written in C++ for managing C++ memory that can be connected to V8 using [cross-component tracing](https://research.google/pubs/pub47359/) that treats the tangled C++/JavaScript object graph as one heap. This post is the first in a series of Oilpan blog posts which will provide an overview of the core principles of Oilpan and  its C++ APIs. For this post we will cover some of the supported features, explain how they interact with various subsystems of the garbage collector, and do a deep dive into concurrently reclaiming objects in the sweeper. Most excitingly, Oilpan is currently implemented in Blink but moving to V8 in the form of a [garbage collection library](https://chromium.googlesource.com/v8/v8.git/+/HEAD/include/cppgc/). The goal is to make C++ garbage collection easily available for all V8 embedders and more C++ developers in general.

## Background

Oilpan implements a [Mark-Sweep](https://en.wikipedia.org/wiki/Tracing_garbage_collection) garbage collector where garbage collection is split among two phases: *marking* where the managed heap is scanned for live objects, and *sweeping* where dead objects on the managed heap are reclaimed. We’ve covered the basics of marking already when introducing [concurrent marking in V8](https://v8.dev/blog/concurrent-marking). To recap, scanning all objects for live ones can be seen as graph traversal where objects are nodes and pointers between objects are edges. Traversal starts at roots which are registers, native execution stack (which we will call stack from now on), and other globals, as described [here](https://v8.dev/blog/concurrent-marking#background). C++ is not different to JavaScript in that aspect. In contrast to JavaScript though, C++ objects are statically typed and thus cannot change their representation at runtime. C++ objects managed using Oilpan leverage this fact and provide a description of pointers to other objects (edges in the graph) via visitor pattern. The basic pattern for describing Oilpan objects is the following:

```cpp
class LinkedNode final : public GarbageCollected<LinkedNode> {
 public:
  LinkedNode(LinkedNode* next, int value) : next_(next), value_(value) {}
  void Trace(Visitor* visitor) const {
    visitor->Trace(next_);
  }
 private:
  Member<LinkedNode> next_;
  int value_;
};

LinkedNode* CreateNodes() {
  LinkedNode* first_node = MakeGarbageCollected<LinkedNode>(nullptr, 1);
  LinkedNode* second_node = MakeGarbageCollected<LinkedNode>(first_node, 2);
  return second_node;
}
```

In the example above, `LinkedNode` is managed by Oilpan as indicated by inheriting from `GarbageCollected<LinkedNode>`. When the garbage collector processes an object it discovers outgoing pointers by invoking the `Trace` method of the object. The type `Member` is a smart pointer, that is syntactically similar to e.g. `std::shared_ptr`, which is provided by Oilpan and used to maintain a consistent state while traversing the graph during marking. All of this allows Oilpan to precisely know where pointers reside in its managed objects.

Avid readers probably noticed —and may be scared— that `first_node` and `second_node` are kept as raw C++ pointers on the stack in the example above. Oilpan does not add abstractions for working with the stack, relying solely on conservative stack scanning to find pointers into its managed heap when processing roots. This works by iterating the stack word-by-word and interpreting those words as pointers into the managed heap. This means that Oilpan does not impose a performance penalty for accessing stack-allocated objects. Instead, it moves  the cost  to the garbage collection time where it scans the stack conservatively. Oilpan as integrated in the renderer tries to delay garbage collection until it reaches a state where it’s guaranteed to have no interesting stack. Since the web is event based and execution is driven by processing tasks in event loops, such opportunities are plentiful.

Oilpan is used in Blink which is a large C++ codebase with lots of mature code and thus also supports:

- Multiple inheritance through mixins and references to such mixins (interior pointers).
- Triggering garbage collection during executing constructors.
- Keeping objects alive from non-managed memory through `Persistent` smart pointers which are treated as roots.
- Collections covering sequential (e.g. vector) and associative (e.g. set and map) containers with compaction of collection backings.
- Weak references, weak callbacks, and [ephemerons](https://en.wikipedia.org/wiki/Ephemeron).
- Finalizer callbacks that are executed before reclaiming individual objects.

## Sweeping for C++

Stay tuned for a separate blog post on how marking in Oilpan works in detail. For this article we assume marking is done and Oilpan has discovered all reachable objects with the help of their `Trace` methods. After marking all reachable objects have their mark bit set. Sweeping is now the phase where dead objects (those unreachable during marking) are reclaimed and their underlying memory is either returned to the operating system or made available for subsequent allocations. In the following we show how Oilpan’s sweeper works, both from a usage and constraint perspective, but also how it achieves high reclamation throughput.

The sweeper finds dead objects by iterating the heap memory and checking the mark bits. In order to preserve the C++ semantics, the sweeper has to invoke the destructor of each dead object before freeing its memory. Non-trivial destructors are implemented as finalizers.

From the programmer’s perspective, there is no defined order in which destructors are executed, as the iteration used by the sweeper does not consider construction order. This imposes a restriction that finalizers are not allowed to touch other on-heap objects. This is a common challenge for writing user-code that requires finalization order as managed languages generally do not support order in their finalization semantics (e.g. Java). Oilpan uses a clang plugin that statically verifies, among many other things, that no heap objects are accessed during destruction of an object:

```cpp
class GCed : public GarbageCollected<GCed> {
 public:
  void DoSomething();
  void Trace(Visitor* visitor) {
    visitor->Trace(other_);
  }
  ~GCed() {
    other_->DoSomething();  // error: Finalizer '~GCed' accesses
                            // potentially finalized field 'other_'.
  }
 private:
  Member<GCed> other_;
};
```

For the curious: Oilpan provides pre-finalization callbacks for complex use cases that require access to the heap before objects are destroyed. Such callbacks impose more overhead than destructors on each garbage collection cycle though and are only used sparingly in Blink.

## Incremental and concurrent sweeping

Now that we have covered the restrictions of destructors in a managed C++ environment, it is time to look at how Oilpan implements and optimizes the sweeping phase in more detail. Before diving into details it is important to recall how programs in general are executed on the web. Any execution, e.g., JavaScript programs but also garbage collection, is driven from a main thread by dispatching tasks in an [event loop](https://en.wikipedia.org/wiki/Event_loop). The renderer, much like other application environments, supports background tasks that run concurrently to the main thread to aid processing any main-thread work.

Starting out simple, Oilpan originally implemented stop-the-world sweeping which ran as a part of the garbage collection finalization pause interrupting executing of the application on the main thread:

![Stop-the-world sweeping](/_img/high-performance-cpp-gc/stop-the-world-sweeping.svg)

For applications with soft real-time constraints the determining factor when dealing with garbage collection is latency. Stop-the-world sweeping may induce a significant pause time resulting in user-visible application latency. As the next step to reduce latency, sweeping was made incremental:

![Incremental sweeping](/_img/high-performance-cpp-gc/incremental-sweeping.svg)

With the incremental approach, sweeping is split up and delegated to additional main-thread tasks. In the best case, such tasks are executed completely in [idle time](https://research.google/pubs/pub45361/), avoiding interfering with any regular application execution. Internally, the sweeper divides work into smaller units based on a notion of pages. Pages can be in two interesting states: *to-be-swept* pages that the sweeper still needs to process, and *already-swept* pages that the sweeper already processed. Allocation only considers already-swept pages and will refill local allocation buffers (LABs) from free lists that maintain a list of available memory chunks. For getting memory from a free list the application will first try to find memory in already-swept pages, then try to help processing to-be-swept pages by inlining the sweeping algorithm into allocation, and only request new memory from the OS in case there is none.

Oilpan has used incremental sweeping for years but as applications and their resulting object graphs grew bigger and bigger, sweeping started to impact application performance. To improve over incremental sweeping we started leveraging background tasks for concurrent reclamation of memory. There are two basic invariants used to rule out any data races between background tasks executing the sweeper and the application allocating new objects:

- The sweeper only processes dead memory which is by definition not reachable by the application.
- The application only allocates on already-swept pages which are by definition not being processed by the sweeper anymore.

Both invariants ensure that there should be no contender for the object and its memory. Unfortunately, C++ heavily relies on destructors which are implemented as finalizers. Oilpan enforces finalizers to run on the main thread to assist developers and rule out data races within the application code itself. To solve this issue, Oilpan defers object finalization to the main thread. More concretely, whenever the concurrent sweeper runs into an object that has a finalizer (destructor), it pushes it onto a finalization queue that will be processed in a separate finalization phase, which is always executed on the main thread also running the application. The overall workflow with concurrent sweeping looks like this:

![Concurrent sweeping using background tasks](/_img/high-performance-cpp-gc/concurrent-sweeping.svg)

Since finalizers may require accessing all of the object’s payload, adding the corresponding memory to the free list is delayed till after executing the finalizer. If no finalizers are executed, the sweeper running on the background thread immediately adds the reclaimed memory to the free list.

# Results

Background sweeping has shipped in Chrome M78. Our [real-world benchmarking framework](https://v8.dev/blog/real-world-performance) shows a reduction of main thread sweeping time by 25%-50% (42% on average). See a selected set of line items below.

![Main thread sweeping time in milliseconds](/_img/high-performance-cpp-gc/results.svg)

The remainder of time spent on the main thread is for executing finalizers. There’s ongoing work on reducing finalizers for heavily-instantiated object types in Blink. The exciting part here is that all these optimizations are done in application code as sweeping will automatically adjust in the absence of finalizers.

Stay tuned for more posts on C++ garbage collection in general and Oilpan library updates specifically as we move closer to a release that can be used by all users of V8.
