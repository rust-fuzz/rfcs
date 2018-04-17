- Start Date: 2018-04-16
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

We can now fuzz rust code with 3 different fuzzers (AFL, libFuzzer, and honggfuzz) through 3 different projects (afl.rs, libfuzzer-sys, and honggfuzz-rs respectively).

Even though the general principles of usage are the same in those projects, they use different conventions and APIs making it unnecessarily difficult to fuzz one project with all available fuzzers.

The minimal goal of this RFC is to harmonize the conventions, APIs and as much as possible the user interfaces.

A logical extension of this goal would be to build a new project abstracting all 3 fuzzers behind a common denominator API for an even easier use.
(to be more specific, I'm not thinking about cargo-fuzz but something lower level, which could then be used by cargo-fuzz)

# Motivation
[motivation]: #motivation

Differents fuzzers have different sets of strengths and weaknesses and therefore can be used in complementary ways to fuzz one codebase.

Being able to easily use all 3 fuzzers on one code base will help uncover more the rust ecosystem.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

to complete...

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

As a minimum, we should harmonize the code API and general project structure.

## Rust API

libfuzzer-sys (persistent fuzzing) : executable entry point provided by libfuzzer.a
```rust
fuzz_target!(|data: &[u8]| { ... }
```

afl.rs (forking fuzzing): executable entry point provided by rust
```rust
afl::read_stdio_bytes(|string| { ... }
```

honggfuzz-rs (persistent fuzzing): executable entry point provided by rust
```rust
loop {
  fuzz!(|data: &[u8]| { ... }
}
```
or
```rust
loop {
  fuzz(|data| { ... }
}
```

### Persistent vs forking fuzzing

- libfuzzer only works in persistent mode (by design)
- AFL supports forking and persistent (https://lcamtuf.blogspot.fr/2015/06/new-in-afl-persistent-mode.html) but persistent mode in [not implemented in AFL.rs](https://github.com/rust-fuzz/afl.rs/issues/131).
- honggfuzz supports both and honggfuzz-rs uses persistent. 
- persistent mode is (a lot)faster and the advantages of forking mode are not relevant for our use case.

I think we should port `afl.rs` to use persistent fuzzing for performance and consistency with the others.

### Entry point

- targets from `afl` and `honggfuzz` uses a file descriptor to read instructions from an external parent process. `afl.rs` and `honggfuzz-rs` let the fuzzed target define their own `main` and do some setup work and then call a function or a macro (defined by the rust fuzzing project) that will then call a function (defined in a C static library from the upstream project) that will read and interpret the instructions from the FD and pass a vector of bytes to a closure provided by the end-user.

- `libfuzzer` takes the form of a static C library linked to the fuzzed code. The `libfuzzer` static library provides the entry point of the final executable. `libfuzzer` then internally calls the function to fuzz in the executable.
This is very much incompatible with the other fuzzers and makes use of setup code [difficult](https://github.com/rust-fuzz/cargo-fuzz/issues/87).

I think we should try to find a way to start `libfuzzer` from rust code.

### macro vs function

- the macro has closure-like syntax allowing the user to choose the data type thanks to the `Arbitrary` crate.
- if you only want a slice of bytes, using a macro is unnecessary, a function suffice.

Maybe, we could provide both.

### interior or exterior loop

When the user provides a closure to fuzz to our function or macro, this closure needs to be called ad vitam aeternam (assuming persistent fuzzing).

The question is where to put this infinite loop?

interior loop:
```rust
// library code
pub fn fuzz<F>(closure: F) where F: Fn(&[u8]) {
  loop {
    let data = get_bytes_from_parent_fuzzer();
    closure(data);
  }
}

// user code
fn main() {
  fuzz(|data| {
    api_to_fuzz(data);
  })
}
```

exterior loop:
```rust
// library code
pub fn fuzz<F>(closure: F) where F: Fn(&[u8]) {
    let data = get_bytes_from_parent_fuzzer();
    closure(data);
}

// user code
fn main() {
  loop {
    fuzz(|data| {
      api_to_fuzz(data);
    })
  }
}
```

Advantages of the interior loop
- less user code
- less error-prone (user cannot forget to write the loop)

Advantages of the exterior loop
- more explicit behavior
- can work in more use cases, see example below.

An example that can only be implemented when using exterior loop:
```rust
fn main() {
  // this function provides an infinite stream of objects and we want to fuzz
  // each instance only once.
  some_function_from_user_api(|object_from_user_api|{
      fuzz!(|data| {
        object_from_user_api.method(data);
      })
    })
  }
}
```

### fuzzing function/macro name and namespace

`fuzz` / `fuzzing` / `fuzz_target` / `fuzzer::read_bytes` / `fuzzer::fuzz_with_bytes` ...


It has also be [proposed](https://github.com/rust-fuzz/afl.rs/issues/31#issuecomment-213767532) to use a syntax similar to `cargo bench`
```rust
#[fuzz]
fn test_fuzz(bytes: Vec<u8>) {
  ...
}
```

### Parallel fuzzing

- `AFL` and `libFuzzer` can be launched many times to use many cores but it requires some setup and the multiple processes needs to be started manually.
- `honggfuzz` takes care of everything.

We should try to do something about `AFL` and `libFuzzer` as nowadays CPUs often have >8 logic cores.


## project structure

example:
```
Cargo.toml
src/
  ...
target/
  debug/
  release/
fuzz/
  afl/
    artifacts/
    target/
  libfuzzer/
    artifacts/
    target/
  honggfuzz/
    artifacts/
    target/
      instrumented/
      not-instrumented/
      debug/
  seeds/
```

There are many questions:
- at the root crate directory, should we crate only one fuzzing directory (like `fuzz` or `fuzzing`) or more (`fuzz_targets` + `fuzz_workspace`)?
  - or even put it/them in the `target` directory?
- each fuzzer will create one or many builds, should we put them somewhere in the `fuzz` directory or in the top level `target` directory alongside `debug` and `release` ?
  - the latter solution will need some cooperation with the `cargo` team.
- should all fuzzers share their generated artifacts? (they are not 100% compatible)
- `fuzzer_name/{artifacts/target}` versus `{artifacts/target}/fuzzer_name` versus something else?
- + a lot of bikeshedding

## CLI

- [cargo tasks](https://aturon.github.io/2018/04/05/workflows/)
...

# Drawbacks
[drawbacks]: #drawbacks

- There is a risk of losing functionalities that are specific to some fuzzers.

# Rationale and alternatives
[alternatives]: #alternatives

`cargo-fuzz` provides a wrapper to easily crate fuzz targets by taking care of most of the boilerplate code creation. It could be possible to teach `cargo-fuzz` to generate boilerplate code compatible with all 3 fuzzers but I don't think it's the best solution.

I see providing a "high level wrapper" and a "low level generic API" as two orthogonal goals but both could be provided by `cargo-fuzz`.

# Prior art
[prior-art]: #prior-art

`cargo-fuzz` is working in a similar but IMO different design space.

see [alternatives](#alternatives)

# Unresolved questions
[unresolved]: #unresolved-questions

Some of the proposed solutions will probably require some help/assistance from the upstream fuzzers and from the cargo team.

