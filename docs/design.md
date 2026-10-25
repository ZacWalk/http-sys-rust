# httpsys design

What the `httpsys` crate does, why it is fast and how it stays safe.

## What it does

- **`Handler` trait** using `async fn` in traits, with a blanket impl for
  closures returning `impl Future<Output = Response>`.
- **`Request`** with cooked accessors that borrow straight out of the
  kernel-filled receive buffer — `method()`, `url()`, known headers, and a
  `body()` that returns a `Body` borrowing that same buffer whenever the body
  arrived with the headers, which is the usual case.
- **`Response`** builder over `bytes::Bytes`, with custom headers and an
  opt-in http.sys kernel response cache.
- **Graceful shutdown** that stops the queue, drains in-flight handlers up to
  `ServerConfig::shutdown_timeout`, then aborts. `Drop` does it for you.

## How it goes fast

Each of these is a deliberate choice, and each one is measurable:

| Technique | What it replaces |
| --- | --- |
| A dedicated IOCP drained in batches with `GetQueuedCompletionStatusEx` | `BindIoCompletionCallback`, which pays a Win32 thread-pool dispatch per completion |
| `FILE_SKIP_COMPLETION_PORT_ON_SUCCESS` on the queue handle | An IOCP round trip for every receive http.sys could satisfy inline |
| `HTTP_RECEIVE_REQUEST_FLAG_COPY_BODY`, adaptively | A second syscall to fetch the body of every request that has one |
| Receive buffers that resize to the traffic each slot sees | A fixed 16 KiB buffer that forces uploads down the streaming path forever |
| `HTTP_RECEIVE_REQUEST_ENTITY_BODY_FLAG_FILL_BUFFER` | One syscall per partially-filled body chunk |
| Pooled receive buffers, body buffers and `OVERLAPPED` state | A heap allocation per request, per body and per response |
| A fixed set of request slots, each owning a request end to end | A `tokio::spawn` and a semaphore acquire per request |

## Safety

Every async operation owns an `OVERLAPPED` whose strong reference is handed
to the kernel and reclaimed exactly once — by a completion thread, or by the
submitting task when the kernel completes inline or refuses the call.
Dropping a future before completion issues `CancelIoEx` and blocks until the
completion fires, so the kernel never writes into a buffer Rust has already
reused.

## Tuning

`ServerConfig` defaults suit a general-purpose server. Two knobs matter for
upload-heavy work:

- `request_buffer_bytes` (16 KiB) — the initial receive buffer. Anything that
  fits, headers and body together, arrives in a single syscall.
- `max_request_buffer_bytes` (128 KiB) — how far a slot's buffer may grow when
  it keeps seeing larger bodies. Raising it trades memory for syscalls.

## The kernel response cache

`Response::cache_in_kernel_for_secs` lets http.sys answer repeat requests from
kernel mode without ever waking your handler. It is deliberately **not** used
in the [benchmark](results.md), because axum has no equivalent and the result
would not mean anything. Try it with:

```ps
.\target\release\httpbench.exe serve --engine httpsys --kernel-cache
```

## Testing

```ps
cargo test --workspace          # unit tests, no http.sys required
cargo test -p httpsys --test server --release -- --ignored --test-threads=1
```

The `--ignored` set binds a real http.sys queue under
`http://+:80/Temporary_Listen_Addresses/`, so it needs no elevation and no
setup — only a free port 80. It covers routing, verbs, inline bodies, a 1 MiB
streamed body verified by checksum, oversized headers, and graceful shutdown.

### Loom

The `OverlappedWrap` completion handshake is model-checked:

```ps
$env:RUSTFLAGS = "--cfg loom"
cargo test -p httpsys --test loom --release -- --test-threads=1
Remove-Item Env:\RUSTFLAGS
```
