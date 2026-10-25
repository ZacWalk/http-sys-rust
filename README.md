# http.sys Rust wrapper and benchmarking

An async Rust server built directly on the Windows [`http.sys`] kernel HTTP
stack, plus a benchmark harness that measures it head-to-head against
[`axum`]. Windows only.

[`http.sys`]: https://learn.microsoft.com/en-us/windows/win32/http/http-server-api-overview
[`axum`]: https://github.com/tokio-rs/axum

```
crates/httpsys/    the library: server, request/response types, IOCP plumbing
tools/httpbench/   the harness: both servers, load generator, charts
```

## The `httpsys` crate

```rust
use httpsys::{Request, Response, ServerBuilder, status};

fn main() -> httpsys::Result<()> {
    let mut builder = ServerBuilder::new()?;
    builder.route("http://+:8080/api/", |request: Request| async move {
        match request.body().await {
            Ok(body) => Response::ok_text(format!("{} bytes", body.len())),
            Err(_) => Response::new(status::BAD_REQUEST),
        }
    })?;

    let mut server = builder.run();
    // ... do other work ...
    server.shutdown();
    server.join();
    Ok(())
}
```

- **Async handlers:** closures returning a future, or the `Handler` trait.
- **Borrowed requests:** the method, URL, headers and usually the body are
  read straight out of the buffer the kernel filled.
- **Built for throughput:** a batched IOCP, inline completions, pooled
  buffers and fixed request slots.
- **Safe cancellation:** dropping an operation's future cancels it in the
  kernel and waits for the completion, so the kernel never writes into memory
  Rust has reused.

[Design](docs/design.md) explains each technique, the tuning knobs and the
kernel response cache.

## Results

32 logical cores, loopback:

| | httpsys | axum | |
| --- | ---: | ---: | --- |
| Peak throughput, small requests | **96.4k req/s** | 95.1k req/s | |
| Throughput at 256 connections | **96.4k req/s** | 89.3k req/s | httpsys holds its peak |
| Median latency at 1 connection | 84 µs | **67 µs** | axum wins when idle |
| Ingest at 256 KiB bodies | 6167 MiB/s | **6770 MiB/s** | |
| Ingest at 8 MiB bodies | 1973 MiB/s | **2278 MiB/s** | |
| Server CPU at 8 MiB bodies | **6.10 cores** | 18.32 cores | 2.6x more efficient per byte |

If you serve many small requests, or care what large uploads cost in CPU,
http.sys is the better substrate; for the last 10% of raw upload bandwidth, a
plain socket is. [Results](docs/results.md) has the charts, what they mean and
how to run `httpbench` yourself.

## Testing

```ps
cargo test --workspace          # unit tests, no http.sys required
cargo test -p httpsys --test server --release -- --ignored --test-threads=1
```

The `--ignored` tests bind a real http.sys queue on port 80 without
elevation. [Design](docs/design.md#testing) covers them and the loom model
check of the completion handshake.

## License

MIT OR Apache-2.0.
