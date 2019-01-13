---
layout: post
comments: true
title: "Implementation Strategies for First-Class Continuations"
description: ""
category:
tags: []
date: 2014-11-19
---

There are many features a programming language can have. The ones we
will be touching today are closures, proceedable exceptions, and first
class continuations. These have one thing in common - they are really
really hard (impossible) to implement using traditional C-like stack
architecture, where every function call allocates new frame and every
function return pops a frame.

The reason closures, proceedable exceptions, and continuations cannot be
implemented using traditional stack is that the frame can outlive its
invocation. Closures can reference local variables defined within
a function (more precisely, variables defined in the lexical scope
the closure is created in). Function can return such closure,
and the closure can be invoked later. Local variables references by the
closure have to be alive when closure is invoked. Moreover, the
variables must be shared between potentialy multiple accessing closures.

Basic exceptions, like those present in Java, when thrown, look for the
handler up the stack frames and destroy the frames which don't have the
handler. For proceedable exceptions, we cannot destroy the frames, we
need to evaluate exception handler and depending on the evaluation the
exception can be returned, continued, aborted, or retried.

And for continuations, we must be able to store the current continuation
(entire currently performed computation)
off to the side and start it later. This can be used to implement green
threads, exceptions, http sessions and many other interesting things. We
even can return multiple times from a single function.

In this post I will summarize various techniques of how to solve this
issues when implementing our own language runtime. I will talk about
continuations, as the their requirements close
over proceedable exceptions and closures. I hereby certify that none of
this is from my head :) Most of it
can be read (in more depth, clarity, and precision) in [Implementation
Strategies for First Class
Continuations](http://link.springer.com/article/10.1023/A:1010016816429)
by William D. Clinger, Anne H. Hartheimer, and Eric M. Ost, and in [An
Empirical and Analytic Study of Stack vs. Heap Cost for Languages With
Closures](http://journals.cambridge.org/action/displayAbstract?fromPage=online&aid=1295104&fileId=S095679680000157X)
by Andrew W. Appel and Zhong Shao.

## Baseline

Let's start with a simple example of continuation usage. Consider this
(kind reader will excuse ugliness of this code as it serves as a simple
example, not real world code). The goal is to tell whether a list
contains an element. In Scheme, there is no `break`, `continue`, or
`return` as it is in Java and other languages. If we were to write the
code in Java using Scheme syntax, it would look like this:

```scheme
(define (search wanted? lst)
  (for-each (lambda (element)
              (if (wanted? element)
                (return element)))
            lst)
  #f)
```

We would like the `return` function to stop the computation and return
with the value of element. But we can get to the similar code using
continuations. The code looks like this:

```scheme
(define (search wanted? lst)
  (call-with-current-continuation
    (lambda (return)
      (for-each (lambda (element)
                  (if (wanted? element)
                    (return element)))
                lst)
    #f)))
```

In Scheme, the mechanism that allows working with continuations
`call-with-current-continuation`. This function gets one argument,
a function, which is called with one argument, a function. In the
example we call the function `return`, and that is exactly what it does.
When called, it returns a value from the
`call-with-current-continuation`. The computation of lambda is stopped.

This looks like a `goto`, but this mechanism is more powerful. We can
store `return` somewhere and call it later. Or we can call it from
completely different location in program. Or we can call it multiple
times.

One possible implementation of `call-with-current-continuation`:

```scheme
(define (call-with-current-continuation f)
  (let ((k (creg-get)))
    (f (lambda (v) creg-set! k) v)))
```

where `creg-get` converts implicit continuation to scheme object with
unlimited extent. `creg-set!` takes such object and installs it as the
continuation for the currently executing procedure. The operation
performed by `creg-get` is called *capture*. Operation performed by
`creg-set!` is called *throw*. The procedure, that is passed to `f`, is
called an *escape* procedure, because a call to the excape procedure
will allow control to bypass the implicit continuation.

Simplest implementation strategy for call-cc is to allocate each frame
on the heap and manage them with gc. With this strategy, `creg-get`
returns the content of a continuation register (frame pointer), and
`creg-set!` stores its argument into the register.

Other strategies focus on improving ordinary function calls faster,
while (unavoidably) making captures and throws slower. That is
a reasoable thing to do, function calls happen much more often than
captures and throws.

## Costs

There are penalties we have to pay for having and using continuations.
Direct cost is a number of instructions required to create a single
continuation frame, link it to the continuation frame being extended,
and to dispose the frame.

It is reasonable to assume that one continuation frame is created for
every hundred machine instructions that are executed. Under this
assumption, each unit of direct cost will correspond to about 1% of the
overall execution time.

Indirect cost is any cost that is not part of its direct cost.

strategy has zero overhead, when direct cost for programs
not using call-cc is not greater than on ordinary stack-based
implementation.

## Strategies

### The gc strategy

Simplest one, always stores frames on the heap. It is practical when
fast gc is used, but is not zero-overhead.

The gc is of great importance in this strategy. Some form of compacting
or copying gc with quick new object allocation is a must.

### The spaghetti strategy

The spaghetti stack is a separate heap in which storage is reclaimed by
ref. counting instead of gc. It was more efficient than gc strategy with
nongenerational gc. It is not zero-overhead. Overall, it is probably
slower than gc strategy in all cases, when modern gc is involved.

### The heap strategy

Frames are allocated on the heap, similarly to gc strategy, but a free
list of uncaptured frames is also used. When a frame is needed, it is
taken from the list, if available. It is not a zero-overhead strategy.
It is only practical when all frames are of same size. At the end, it is
comparable to gc strategy.

### The stack strategy

Active continuation is represented as a contiguous stack in an area of
storage we call stack cache. When continuation is captured, a copy of
the entire stack cache is made and stored in the heap.  When
a continuation is thrown, the stack cache is cleared and the
continuation is copied back to the stack cache.  It is a zero-overhead
strategy. It performs well for normal cc scenarios, but poorly for
scenarios when cc are captured and thrown repeatedly. Also, it can
increase the asymptotic storage space. It also prevents allocating
storage for mutable variables within continuation frame (there may be
multiple copies of the frame around)

### The chunked-stack strategy

Maintaining a small bound on the size of the stack cache, and copying
portions of the stack cache into the heap or back again as the stack
cache overflows and underflows. This reduced the worst case latency of
captures and throws.

This works well with generational gc because of limiting the size of
thack cache -> limiting the root set that the gc must scan on each
cycle. The portion of continuation that resides in the heap will be
scanned only when its generation is collected.

When the limited stack cache overflows, the copying of continuation to
the heap can slow execution down even when call-cc is not used. This
shouldn't happen often though.

### The stack/heap strategy

All frames are allocated on the stack, but when the continuation is
captured, entire stack is copied to the heap and stack is cleared.
Because active continuation can reside either on stack or heap, we have
to check on every return whether the frame should be popped off the
stack.

This is quite useful strategy, performing well for both single
evacuation and recapture scenarios. There is always only one copy of
a single frame.

### The incremental stack/heap strategy

Similar to stack/heap, but when returning through a continuation frame
that isn't in the stack cache a trap occurs and copies the frame into
the stack cache. This can be done by maintaining a permanent
continuation frame at the bottom of the stack cache. This frame's return
address points to system code that copies one or more frames from the
heap into the stack cache, and immediatelly returns through the first of
those continuation frames.

### The Hieb-Dybvig-Bruggeman strategy

This strategy uses multiple stack segments that are allocated in the
heap. The stack segment that contains the current continuation serves as
the stack cache.

When a continuation is captured, the stack cache is split by allocating
a small data structure that points to the current continuation frame
within the stack cache. This structure represents the captured
continuation.

### Mateu's coroutines

Variation of the gc strategy in which two youngest generations of
a generational gc are devoted entirely to continuation frames. It may
improve the performance by:

* The youngest generation acts as a stack cache, which can be small
  enough to fit in the primary cpu cache entirely.
* Continuation frames have a bigger mortality rate than other objects,
  so dedicating the youngest generations entirely to frames decreases
  the number of objects that must be copied into an older generation.
* The roots for frames-only cycle consist of the register that holds the
  current cc, and the data structure taht represents the current set of
  threads.

## Summary of costs

First table shows the costs of each strategy in terms of number of
machine instructions per continuation frame created plus the number of
cycles lost per continuation frame due to cache misses.

| Strategy       | gc         | heap       | stack     | stack/heap | inc. s/h  |
| -------------- | ---------- | ---------- | --------- | ---------- | --------- |
| Creation       | 3.1        | 2.0 - 3.0  | 1.0       | 1.0 - 3.0  | 1.0 - 3.0 |
| Frame pointers | 2.0        | 2.0        | 0.0       | 0.0        | 0.0       |
| Disposal (pop) | 1.4        | 4.0        | 1.0       | 3.0        | 1.0       |
| Direct Cost    | 6.5        | 8.0 - 10.0 | 2.0       | 4.0 - 6.0  | 2.0 - 4.0 |
| Indirect Costs | < 10.7     | < 6.2      | < 1.4     | < 4.1      | < 1.5     |
| **Total Cost** | 6.5 - 17.2 | 8.0 - 16.2 | 2.0 - 3.4 | 4.0 - 10.1 | 2.0 - 5.5 |


Second table uses asymptotic notation to express the cost of capturing
the frame, the cost of capturing it again, the cost of throw, and the
cost of return. These costs are expressed in terms of the size N of the
continuation that is contained within the stack cache, and the number of
frames M that are contained within the continuation being thrown to.

| Strategy      | gc   | heap | stack | stack/heap | inc. s/h | HDB  |
|---------------|------|------|-------|------------|----------|------|
| First capture | O(1) | O(M) | O(N)  | O(N)       | O(N)     | O(1) |
| Recapture     | O(1) | O(1) | O(N)  | O(1)       | O(1)     | O(1) |
| Throw         | O(1) | O(1) | O(N)  | O(1)       | O(1)     | O(1) |
| Returns       | O(1) | O(N) | O(1)  | O(N)       | O(N)     | O(N) |

Indirect costs are hard to estimate, and vary greatly upon the program
being executed and the compiler used.

| Strategy             | gc     | heap  | stack | stack/heap | inc. s/h |
|----------------------|--------|-------|-------|------------|----------|
| Cache write misses   | < 5.1  | 0     | 0     | 0          | 0        |
| Cache read misses    | < 1.0  | 0     | 0     | 0          | 0        |
| Not reusing frames   | < 4.6  | < 5.4 | 0     | < 3.2      | 0        |
| Heap variables       | 0      | 0     | < 0.6 | 0          | < 0.6    |
| Stack-cache overflow | 0      | 0     | 0     | < 0.1      | < 0.1    |
| Copying and sharing  | 0      | < 0.8 | < 0.8 | < 0.8      | < 0.8    |
| **Indirect cost**    | < 10.7 | < 6.2 | < 1.4 | < 4.1      | < 1.5    |

## Multitasking

Implementing green threads using continuations is very easy. The main
point is that we can change active continuation often. When
continuations are used this way, the resulting performance can be much
different (as the cost of evacuation of continuation to the heap is paid
even when programs don't use continuations explicitly).

When context switches happen often, roughly every 128 function calls,
the overhead of gc strategy is becoming smaller than the overhead of
frame evacuation of other strategies.

## Conclusion

Authors of Implmementation Strategies for First-Class Continuations
recommend the incremental stack/heap and Hieb-Dybvig-Bruggeman
strategies for implementing first class continuations in
almost-functional languages as a best compromise between performance,
multitasking performance and difficulty of implementation.

The gc strategy is the simplest one to implement, and it is shown that
it should perform reasonably well. Therefore for
[femtoscheme](https://github.com/mhlopko/femtoscheme), I decided to go
with gc strategy. At least, I will have a chance to profile cache misses
:) If you're implementing language runtime only for the fun and
exploratory reasons, I would suggest going with the gc strategy too.
