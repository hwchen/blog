---
title: "Rust Async Foundation"
date: 2020-04-04T21:25:21-04:00
draft: true
---
# Background
Rust async. Async syntax landed on stable, `Future` trait stabilizied in std library. Async ecosystem is growing, with established (tokio) and upcoming (async-std) and possibly

There's two routes to getting a strong grasp on async:
- understand the high-level model of async and futures. This will let you write application code. This will let you chain async operations together, and know when an operation might block the async runtime, causing unexpected behavior.
- understand the machinery that implements and supports this model. For most application writers and library users, they don't have to know these details. They'll just need to be familiar with the futures combinators and the pitfalls of the model.

But it's fun to know how everything works! There's plenty of articles about the high level, so let's jump right in.

# Overview
The supporting machinery is a reactor and an executor. Reactor receives events. Executor schedules tasks (futures) to execute. Their interaction drives the async model. It's the decoupling of receiving an event (e.g. IO) and using computational power. When they're coupled, the thread of execution gets blocked. Using the async loop, it manages tasks and events separately so that the thread doesn't get blocked.

# Executing a `Future`

What's required to execute a future? An executor.
The most basic is `block_on`, which basically just 

## Key concepts
- Future
- Pin
- Wake
- Context
- Park
- Poll

These concepts are often mentioned in discussions, or defined without context.

My goal here is to tie them all together, to understand how the work together to provide the foundation of Rust async.

## I basically understand
- Futures
- Pin
- Poll
The most confusing part: what is a waker, why do we need it.
- How reactor and executor talk to each other: through wakers! Executor send waker to task, then the task registers the waker with an event source (or whatever is doing the waking, like the end of a computation). On wake, the executor then moves the task to the "active pool" that it's polling. Then executor will poll.

## Don't understand
- Park: Why park? Is this specific to threads? It's one strategy for sleeping and waking an executor that's blocking and on a dedicated thread.

If Poll returns pending, then executor should move the task to the "inactive" pool. This can only be done safely if the future is properly constructed to wake the task again, otherwise the task will never be polled again. This situation can arise if either the future was constructed incorrectly, leading to wake being called incorrectly; or, if ...?

# From tokio discord:
Hi, trying to understand Futures and Waker better. I'm under the impression that Waker::wake() is called when a Future is ready. However, here it's called when the future is still Pending: https://github.com/tokio-rs/tokio/blob/master/tokio/src/stream/all.rs#L38. Can somebody explain why it's being called even when the future is not ready to return?
GitHub
tokio-rs/tokio
A runtime for writing reliable asynchronous applications with Rust. Provides I/O, networking, scheduling, timers, ... - tokio-rs/tokio
jebrosenToday at 1:29 PM
@hwchen wake should be called when more progress can be made, not necessarily when the whole thing is ready. Here, "more progress" means "poll the next future in the stream to see if it's ready"
Alice RyhlToday at 1:30 PM
In this case it is asking the scheduler to poll it again as soon as it can
jebrosenToday at 1:34 PM
The other common way to implement this is with a loop { }, however that starves the executor if the stream returns a lot of futures that are already Ready
hwchenToday at 1:40 PM
Ok, I think I understand. I guess I got a little confused between the current poll and the next one. For some reason I was thinking that wake would be called, then Pending would be returned, and that would be it. But Pending is just what's returned for the current poll, and wake asks the scheduler to poll again, which would move the future forward.
Thanks for the help!

# References:
- https://www.reddit.com/r/rust/comments/cfvmj6/is_a_contextwaker_really_required_for_polling_a/
- https://boats.gitlab.io/blog/post/wakers-i/
- https://stjepang.github.io/2020/01/25/build-your-own-block-on.html
- https://rust-lang.github.io/async-book/02_execution/04_executor.html
- https://doc.rust-lang.org/beta/std/future/trait.Future.html
- https://www.reddit.com/r/rust/comments/anu8w4/futures_03_how_does_waker_and_executor_connect/
- https://www.reddit.com/r/rust/comments/bws280/the_waker_pattern_in_async_doesnt_feel_very/
- https://www.reddit.com/r/rust/comments/f05qiy/announcing_cooked_wakers/
- https://rust-lang.github.io/async-book/02_execution/03_wakeups.html

# futures executor
- Is it true that run_executor, the blocking call, is only used to block the calling thread. But then all the polling that is done inside is non-blocking? I think that the rest of the pool is always polled; there is no pool of "awake" or "asleep" tasks, so there never has to be a worry about...
-
- But wait, then why does there need to be `thread::park` for the overall blocking executor call if polling tasks happens in a naive loop? Ah, maybe it's because each task in the pool is passed the ability to `wake` the thread when it's ready to proceed. But how does the executor poll when it's thread is parked? It's because the executor doesn't poll only ready tasks. But it polls all tasks in the pool, but only when any one task wakes the whole thread. Is this true? But wait, what happens when thread gets unparked, but then the first future returns `Pending`? won't that park the thread again? Or will parking only happen at the end of one iteration through the loop?

I need to trace where `park` is called. Currently I only see it in `run_executor`, which just parks every time Poll is not ready. which means that each task can call unpark when it's time for it to do work? But then my question is how the executor knows which task to poll. Because if it just polls any random future, it may park the thread. But the issue won't happen if the whole pool gets polled when any one task is ready.

To answer above question, when a future returns `Pending`, that doesn't park the thread; the whole thread only get parked if the polling the whole pool returns pending. Ah, I think this makes sense now. That way each run parks, then if any one future wakes up the executor, all futures get polled again.

https://users.rust-lang.org/t/understanding-async-executors-futures-local-pool/40632 confirms my understanding
