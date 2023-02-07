---
title: Investigating slow incremental no-change Rust builds
date: "2021-12-15"
---

I created a new Rust project recently (`cargo new hello`), and I happened to notice that repeated incremental builds, with no changes to the hello world project, were taking on the order of 100ms.

This is just on the edge of what [users experience as instantaneous](https://www.nngroup.com/articles/response-times-3-important-limits/). Of course in many instances this time is dwarfed by the actual compilation time, but if you are using something like the [cargo xtask](https://github.com/matklad/cargo-xtask) pattern (or if you simply `cargo run` a lot), this is adding meaningful latency to the execution of your command.

## Recreating the result with a local build

Given I wanted to attempt to improve the performance in this case, I first wanted to confirm that I could recreate this result with a local build of cargo.

```bash
# clone cargo and switch to that directory
..

# use the version of cargo installed on my machine to build cargo from source
cargo build --release

# switch to the directory containing the example hello world project
..

# use the built from source version of cargo to build the hello world project
time ../cargo/target/release/cargo build

# run the same command again to get the time for a no-change incremental rebuild
time ../cargo/target/release/cargo build
```

Unfortunately (or fortunately?) after rebuilding cargo from source I wasn't seeing the same performance issue as with the installed version of cargo. At first I thought perhaps cargo had been updated recently to improve performance here, so I ran `cargo -V` to get the exact commit sha of the currently installed version of cargo on my machine and rebuilt that from source - and it also performed better than the installed version of cargo. 

This was very surprising. At first I wondered if this could be the result of compiling cargo with a newer version of the rust compiler, but this performance delta seemed too large to be explained by that. I also considered that perhaps the installed version of cargo (which I had installed via rustup) used the statically linked musl-libc, but checking with `lld` showed that not to be the case.

# Further investigations into the rustup provided cargo binary

Running `strace cargo build` provided a hint about what was happening here.

```
execve("/home/josh/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/bin/cargo", ["/home/josh/.rustup/toolchains/st"..., "build"], 0x55ba190e0880 /* 41 vars */) = 0
```

The rustup provided "cargo" binary, located on my machine at `~/.cargo/bin/cargo` is not actually cargo. That binary is a `rustup` binary, which does things like check for project configuration files to determine which version of the rust toolchain to invoke, before proxying the command to the real cargo binary.

```
time /home/josh/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/bin/cargo build
```

Timing the actual cargo binary on my machine demonstrated that the performance issue I was noting was actually in rustup itself.

# Digging into rustup

So why does rustup have so much overhead in a hello world project with no project specific configuration? I first built rustup from source to check if the latest version has this issue. As of this writing, building rustup from HEAD does in fact provide a performance improvement, but there was still a larger overhead than I would expect.

From there I created a flamegraph of rustup while building the hello world project, and found it spent a lot of time parsing a very large toml file. However, building a basic project does not depend on the contents of this toml file.

I opened [this PR](https://github.com/rust-lang/rustup/pull/2917) against rustup which skips parsing the file when it isn't needed, resulting in an 80% reduction of the rustup overhead for this use case.

As part of reading this file rustup will also detect problems with this file and fix them if needed. I believe that was an unintended side effect, as I think any operation that doesn't depend on that file shouldn't be required to validate and fix it. However, the rustup test suite does rely on that side effect. For that reason this PR currently fails in CI, but I'm hopeful that someone more familiar with rustup can confirm my assumption here - as I'd be happy to clean up the tests to get this passing in CI.

## Conclusion

This was a quick, fun, real world investigation into a (minor) performance issue in a widely used tool. A potential fix has been proposed, but I'll need to develop a better understanding of rustup to determine whether the fix I've implemented here doesn't break some other use case.

Along the way I also learned something about how rustup works (tldr: `~/.cargo/bin/cargo` is not actually `cargo`).
