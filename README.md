# hyperlight-process

[![CI](https://github.com/youyuanwu/hyperlight-process/actions/workflows/ci.yml/badge.svg)](https://github.com/youyuanwu/hyperlight-process/actions/workflows/ci.yml)

Run [Hyperlight](https://github.com/hyperlight-dev/hyperlight) C guests
**natively**, without a hypervisor.

Hyperlight runs guest functions inside lightweight KVM micro-VMs.
`hyperlight-process` takes the same unmodified C guest, written against
Hyperlight's C API, and statically links it into ordinary Linux code. The guest
can then run in one of two native modes:

- **Subprocess:** the guest runs in a separate runner process; the host talks to
  it over stdin/stdout.
- **In-process:** the guest is linked directly into the host and called without
  IPC or serialization.

The host uses the same builder and typed `call` API as Hyperlight's `Sandbox`,
so switching between a micro-VM and native execution needs little code change.
Use it for development without KVM, environments where nested virtualization is
unavailable, testing guest logic, or trusted workloads where call overhead
matters more than isolation.

> [!WARNING]
> **Native execution is not a sandbox.** Guest code runs with the full
> privileges of its process: filesystem, network, and system calls. Crashes,
> memory corruption, and infinite loops are not contained; in-process mode can
> crash or corrupt the host itself. Only run trusted guests. Use Hyperlight's
> KVM micro-VMs when you need isolation.

## Choosing an execution mode

| | KVM micro-VM (Hyperlight) | Native subprocess | Native in-process |
|---|---|---|---|
| Isolation | Hardware VM | Separate OS process only | None |
| Requires `/dev/kvm` | Yes | No | No |
| `Add` call latency* | ~20 µs | ~16 µs | ~0.27 µs |
| Instance creation* | ~2.8 ms | ~0.55 ms | ~1.3 µs |
| Guest crash | Contained | Ends the runner process | Ends the host |

\* Measured on one nested-virtualized Azure VM; see the
[benchmark results](samples/benchmark/RESULTS.md). Results depend on payload
size and workload: for 64 KiB transfers the KVM path was faster than the
subprocess path.

## Requirements

- Linux x86_64.
- Rust, installed via [rustup](https://rustup.rs/). The pinned toolchain in
  [rust-toolchain.toml](rust-toolchain.toml) is installed automatically.
- Clang, LLVM `lld`, GNU Make, curl, and tar. On Ubuntu/Debian:

  ```sh
  sudo apt install build-essential clang lld curl ca-certificates
  ```

- For the KVM demo only: read/write access to `/dev/kvm`.

The build downloads the Hyperlight 0.19.0 C SDK and verifies its checksums.

## Quick start

Run the same C guest in each mode:

```sh
# Native subprocess, no KVM needed
make -C samples/c-guest run-native

# Native in-process, no KVM needed
make -C samples/c-guest run-inprocess

# Hyperlight KVM micro-VM, for comparison
make -C samples/c-guest
make -C samples/c-guest run
```

Each demo calls the guest's `Greet` and `Add` functions and services its
`HostLog` callback:

```text
guest -> host: Hello, Hyperlight! This is a Hyperlight C guest.
host <- guest: Hello, Hyperlight! This is a Hyperlight C guest.
host <- guest: Add(20, 22) = 42
```

## Using the library

Compile your C guest into a static library, for example with the `cc` crate in
a `build.rs`, and link it into a small runner executable:

```rust
unsafe extern "C" {
    fn hyperlight_main();
}

fn main() -> anyhow::Result<()> {
    unsafe { hyperlight_process::runtime::run(hyperlight_main) }
}
```

Then start and call it from the host:

```rust
use hyperlight_process::RunnerBuilder;

let mut runner = RunnerBuilder::from_executable("my-native-runner")
    .host_function("HostLog", |message: String| {
        println!("guest -> host: {message}");
        Ok(true)
    })
    .build()?;

let greeting = runner.call::<String>("Greet", "Hyperlight".to_owned())?;
let sum = runner.call::<i64>("Add", (20_i32, 22_i32))?;
runner.shutdown()?;
```

For in-process execution, link the guest into the host and use
`InProcessRunnerBuilder::new(hyperlight_main)` with the same `host_function`
and `call` API.

See the [library documentation](crates/hyperlight-process/README.md) for the
full API, supported types, the subprocess protocol, threading rules, and
safety requirements. The [native sample](samples/c-guest/native/) is a
complete working example, including its build script.

## Repository layout

| Path | Contents |
|------|----------|
| [crates/hyperlight-process](crates/hyperlight-process/) | The library: native C API runtime, subprocess client, and in-process runner |
| [samples/c-guest](samples/c-guest/) | A C guest with KVM, subprocess, and in-process hosts |
| [samples/benchmark](samples/benchmark/) | Criterion and latency benchmarks comparing all three modes |

A PostgreSQL-on-Hyperlight-Unikraft experiment that previously lived here is
now in [hyperlight-postgres](https://github.com/youyuanwu/hyperlight-postgres).

## Development

```sh
cargo build --workspace --all-targets --locked
make -C samples/c-guest test        # Native C guest unit tests
cargo test --workspace --locked     # Library, sample, and benchmark tests

# KVM tests (need /dev/kvm):
make -C samples/c-guest test-vm
cargo test -p hyperlight-benchmarks --locked --test workloads -- --ignored
```

On machines with limited memory, set `CARGO_BUILD_JOBS=1`. CI runs all of the
above, including the KVM tests.

## Compatibility

Pinned to **Hyperlight 0.19.0**. Hyperlight requires the host and guest
versions to match exactly, so the `hyperlight-*` crates and the C SDK in
[samples/c-guest/Makefile](samples/c-guest/Makefile) are upgraded together.

## License

[MIT](LICENSE). The downloaded Hyperlight C SDK is licensed separately under
Apache-2.0 and includes picolibc; see the
[Hyperlight licensing notes](https://github.com/hyperlight-dev/hyperlight/blob/v0.19.0/src/hyperlight_libc/third_party/README.md).
