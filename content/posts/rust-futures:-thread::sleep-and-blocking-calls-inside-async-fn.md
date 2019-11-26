---
title: "Rust futures: thread::sleep and blocking calls inside async fn"
date: 2019-11-25T10:24:11-05:00
draft: false
---
Lately, I've been seeing some common misconceptions about how Rust's `futures` and `async/await` work ("blockers", haha). There's an influx of new users excited for the major improvements that `async/await` brings, but stymied by basic questions. Concurrency is hard, even with `async/await`. Documentation is still being fleshed out, and the interaction between blocking/non-blocking can be tricky. Hopefully this article will help.

(This blog is mostly about specific pain points; for an overview of async programming in rust, go to the [book](https://rust-lang.github.io/async-book/index.html))

TLDR: Be careful using expensive blocking calls inside `async fn`! If you aren't sure, almost everything in the Rust `std` library is blocking, so watch out if you know if a call will take time!

While I think that anybody can make this mistake (it's basically silent until you introduce enough load to noticeably block the thread), beginners are especially prone to this. The following scenario may be a bit long-winded, but I think it's good to show how easy it is to reach for a blocking call inside an `async fn`.

## Don't sleep on `std::thread::sleep`

After working through a simple example, probably the first thing a new Rust async user will do is look for some proof that the program is really async. So let's take this [example](https://rust-lang.github.io/async-book/06_multiple_futures/02_join.html) from the Rust `async book`:
```
use futures::join;

async fn get_book_and_music() -> (Book, Music) {
    let book_fut = get_book();
    let music_fut = get_music();
    join!(book_fut, music_fut)
}
```

Even if you were to `log` inside `get_book` and `get_music`, there's no easy way to tell that they are run concurrently because any one run may produce output that happens to match the order in the code. You'd have to run the program several times in order to see that the logging order may flip (and what if it didn't flip?).

If you want to be able to see that `get_book` and `get_music` are run concurrently 100% of the time, you might want to log their starting times, and see that the starting times are the same. But wait, what if the starting times are still serial, but the fn runs so fast it still looks concurrent?

Let's introduce a _delay_! For example (in pseudocode for clarity):
```
async fn get_book() {
    println!("book start: time {}", current_time());
    std::thread::sleep(one_second);
    println!("book end: time {}", current_time());
}
```
With a delay of 1s inside `get_book` and `get_music`, we'd expect that in the concurrent case, we'll see output like:
```
book start: time 0.00
music start: time 0.00
book end: time 1.00
music start: time 1.00
```

In the serial case, we'd expect:
```
book start: time 0.00
book end: time 1.00
music start: time 1.00
music end: time 2.00
```

Which do you think will happen, the serial or concurrent case?

If you've read the title of this post, you might guess that `get_book` and `get_music` execute serially. But why!? Isn't everything inside an async fn supposed to be run concurrently?

Before I go move on to some explanation, here's a few other times this question was asked:

[reddit 1](https://old.reddit.com/r/rust/comments/e1gxf8/not_understanding_asyncawait_properly/)
[reddit 2](https://old.reddit.com/r/rust/comments/dtp6z7/what_can_i_actually_do_with_the_new_async_fn/)
[reddit 3](https://old.reddit.com/r/rust/comments/dt0ruy/how_to_await_futures_concurrently/)
[stackoverflow 1](https://stackoverflow.com/questions/52313031/how-to-run-multiple-futures-that-call-threadsleep-in-parallel)

So if you've made this mistake, no worries, many others have walked this path too.

## What went wrong? Why even async?
I'm not going to cover `futures` and `async/await` in depth here (the [book](https://rust-lang.github.io/async-book/index.html) is a good starting place). I just want to point out two possible sources of confusion:

#### `std::thread::sleep` is blocking?
It might not be obvious to newcomers that `std::thread::sleep` is blocking. I think that it's one of those things that's probably obvious in hindsight, but when trying to come to grips with a whole new paradigm of program execution, it's easy to overlook.

Even if you understand concurrency well generally, you might not know how `thread::sleep` is implemented specifically. Some high level reasoning and some examples (like the above) might get you there. But there's nothing in the [doc](https://doc.rust-lang.org/std/thread/fn.sleep.html) that says `"This call is BLOCKING and you shouldn't use it in async contexts"`, and non-systems programmers might not think much of `"Puts the current thread to sleep"`.

(Ironically, `thread::sleep` may be especially confusing if somebody's mental model of async programming is letting a `Future` "sleep" while letting other work happen).

#### what does `async` do?
But some might say, "So what if `thread::sleep` is blocking? Doesn't putting it in an `async fn` just take care of it?".

To paraphrase some online discussions, the thought is that `async` _makes_ everything inside a block or function async.

First, I'll just say that it makes sense to think this; part of the pitch of `async/await` is that it would make async easy for everybody. And if you understood concurrency models at a high level (the event loop, generally trying not to block threads), there might not be a particular reason why `async` wouldn't just work by making things async. That would definitely be the easiest and most ergonomic way.

Unfortunately, that's not how Rust's `async` paradigm works. `async` is powerful, but at its core it just provides a nicer way to handle `Future`s. And a `Future` does not just automatically move a blocking call to the side to allow other work to be done; it uses a totally separate system, with polling and async runtimes, to do the async dance. Any blocking call made within that system will still be blocking.

There might be some confusion because `async/await` is allows us to write code that _looks_ more like regular (blocking) code. That's where the `await` part of `async/await` comes in. When you `await` a future inside of an `async` block, it will be able to schedule itself off the thread and make way for another task. Blocking code might look similar, but can't be `await`ed because it's not a future, and won't be able to make "space" for another task.

So this won't block, but `await` lets you write code that looks pretty similar to blocking calls:
```rust
async {
    let f = get_file_async().await;
    let resp = fetch_api_async().await;
}
```

And this will block at each call:
```rust
async {
    let f = get_file_blocking();
    let resp = fetch_api_blocking();
}
```
And this won't compile:
```rust
async {
    let f = get_file_blocking().await;
    let resp = fetch_api_blocking().await;
}
```
And nothing happens here (you've got to `await` futures inside of `async`!):
```rust
async {
    let f = get_file_async();
    let resp = fetch_api_async();
}
```

Overall, it's better to think of `async` as something that allows `await` inside a function or a block, but doesn't actually make anything async.

## How to unblock
What if you want to unblock your async fn?

You could find an async alternative: while `thread::sleep` is blocking, you might use these (depending on which runtime ecosystem you choose):

- [async_std::task::sleep](https://docs.rs/async-std/1.1.0/async_std/task/fn.sleep.html)
- [tokio::time::Delay](https://docs.rs/tokio-timer/0.3.0-alpha.5/tokio_timer/struct.Delay.html)

Both `tokio` and `async_std` offer async alternatives for other blocking operations too, such as filesystem and tcp stream access.

The other option is to move the blocking call to another thread.

- [tokio_executor::threadpool::blocking](https://docs.rs/tokio-executor/0.2.0-alpha.5/tokio_executor/threadpool/fn.blocking.html)
- [async_std::task::spawn_blocking](https://docs.rs/async-std/1.1.0/async_std/task/fn.spawn_blocking.html)

This requires that your runtime have machinery (such as a thread pool) dedicated to offloading blocking calls.

I've also filed some issues to try to prevent others from falling into this trap:

- [async-book](https://github.com/rust-lang/async-book/issues/64)
- [clippy](https://github.com/rust-lang/rust-clippy/issues/4377)

## Conclusion
Hope that the this blog was able to clarify some things about how blocking calls interact with Rust's concurrency model! Feel free to give feedback.
