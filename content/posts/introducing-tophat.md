---
title: "Introducing Tophat"
date: 2020-12-30T12:54:48-05:00
draft: false
---
# Overview
[Tophat](https://github.com/hwchen/tophat) is a small, pragmatic, and flexible async http server library for Rust.

Its main points:

- no async runtime dependencies.
- works with `AsyncRead/AsyncWrite`.
- `Fn(Request, ResponseWrite) -> ResponseWritten` instead of `Fn(Request) -> Response` allows more observability of response lifecycle.

For a quick example, go to bottom of this post.

I want to first thank the Rust async and web community for all the work they've done. Tophat is largely a remix of code from [async-h1](https://github.com/http-rs/async-h1), with a good sprinkling of libraries from the [hyperium](https://github.com/hyperium) project.

# Rust Async Background
For those readers who are not as familiar with the Rust async ecosystem, this section may provide more background on `tophat`'s design choices.

There are two major async ecosystems for Rust: `tokio` and `async-std`. Because Rust does not have a bundled standard async runtime, each ecosystem provides its own.

Libraries developed for one ecosystem may not work with the other. The roadblocks are generally:

- Usage of different `AsyncRead/AsyncWrite` traits in each ecosystem. This can be overcome with a shim like tokio's [compat](https://docs.rs/tokio-util/0.6.0/tokio_util/compat/index.html).
- Usage of `spawn` for new tasks, tied to the executor of an ecosystem. There's currently no easy solution, creating a bridge requires a trait in std.

Library authors who want to support multiple runtimes have three options:

- Have the library start its own runtime in the background if necessary.
- Use feature flags to switch runtime-specific code on/off in the library.
- Reduce the surface area of the library: removing internal spawns tied to a runtime and requiring tasks to be spawned in user code. This strategy will work if the library can, for example, accept just `AsyncRead/AsyncWrite`.

At the moment, many libraries are supporting option number two: feature flags. Perhaps this is tenable when there's only two runtimes to support. But if Rust would like to grow a diverse async ecosystem, this path would become more difficult as library authors have to support new code for each runtime.

For me, the first option (starting a new runtime just for a library) doesn't have the best feel to me, even if it's possible. Rust is a language which values control and explicitness over resources. While this option may be more convenient for users, it pushes a major resource-handling choice into the background.

The last option, reducing the surface area of the library, is the most appealing to me when possible. While it requires more work from the user, it's straightforward and transparent.

# Motivation for tophat
Since learning Rust, I've been increasing excited about moving down the stack. While my day job is as a backend/data engineer, I've used Rust as a way to learn about lower-level implementations. For one project, I wanted to learn more about https servers: parsing requests, server architecture, etc. I became quite excited looking at `async-h1` because it was a clear and easy codebase to navigate. It also chose option three (from "how to support multiple runtimes" above"), where it receives a stream that's `AsyncRead/AsyncWrite`.

At the same time, I was also learning more about async executors. My codebase of choice was [smol](https://github.com/smol-rs/smol). It's not one of the major async runtimes, but it was designed to be small, flexible, and performant. It was not difficult to get `async-h1` running on `smol`. However, I noticed that in the process, I pulled in `async-std` as a dependency from `async-h1`, and which started a separate runtime. I got this itch, where I wanted to get `async-h1` running without pulling in `async-std`.

However, I didn't really act on it right away. I guess I needed more motivation than just making `async-h1` runtime-agnostic. That came a little while later, when I was reading the issues for `hyper`, `tokio`'s http server.

I noticed an issue for [increasing observability](https://github.com/hyperium/hyper/issues/2181). Basically, `hyper`'s service architecture takes response handlers with an simplified signature of `Fn(Request) -> Response`. This means that the handler doesn't have knowledge of the outcome of sending the `Response`. Instead, the simplified signature `Fn(Request, ResponseWrite) -> ResponseWriten` would allow the handler to see the outcome of writing the response. (Also, note that this is similar to the handler signature that Golang chose).

Some use cases for this pattern, mentioned in the issue:

- Logging the outcome of a request to metrics.
- Logging the timing of a response (time-to-last-byte).
- Logging the amount of transmitted bytes (to diagnose failures).

It seemed like implementing this pattern would be a little different (for Rust async); and I thought that this, plus being async-runtime agnostic, would be a fun and worthwhile challenge. It might even be potentially useful to others!

# pre-1.0
I've finally reached a point where I feel comfortable presenting `tophat` more publicly. While it's definitely not 1.0, it should be useable for basic apis. There's even some conveniences, like a router and error-response handling!

Feature list:

- no async runtime deps
- HTTP/1.1 only
- #[deny(unsafe_code)]
- fast enough, probably
- basic router, cors, identity features
- server sent events
- a `Glitch` system for conveniently converting errors to error-responses (e.g. 500) for simple error-handling cases.

I've got some [examples](https://github.com/hwchen/tophat/tree/main/examples) ready, including one for a basic [api](https://github.com/hwchen/tophat/tree/main/examples/postgres) using tokio-compat, tokio-postgres, and deadpool.

Some tasks for the near future:

- keep improving correctness
- fuzzing
- more docs/examples
- improve logging

If you're interested in [tophat](https://github.com/hwchen/tophat), issues and PRs are definitely welcome!

# Example
This code snippet is a bit long, so I didn't want to put it at the top.

Besides `tophat`, this example uses `smol` as the runtime.

```rust
use smol::{Async, Task};
use std::net::TcpListener;
use async_dup::Arc;
use tophat::server::accept;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = Async::<TcpListener>::bind("127.0.0.1:9999")?;

    smol::block_on(async {
        loop {
            let (stream, _) = listener.accept().await?;
            let stream = Arc::new(stream);

            let task = smol::spawn(async move {
                let serve = accept(stream, |_req, resp_wtr| async {
                    resp_wtr.send().await
                }).await;

                if let Err(err) = serve {
                    eprintln!("Error: {}", err);
                }

            });

            task.detach();
        }
    })
}
```
