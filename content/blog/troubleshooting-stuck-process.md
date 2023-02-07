---
title: Troubleshooting a stuck process
date: "2021-02-28"
---

I was recently troubleshooting an issue where a process seemed to be stuck, not making any progress. Eventually I was able to track down the problem, but along the way I found a few quick ways to start gathering information about a running process.

These tips apply only to Linux systems. I'm sure other operating systems have similar tools, but I am not familiar with them so I won't mention them here.

I use `sleep` as an example program to "troubleshoot" throughout this article. You will probably want to replace `sleep` in my examples below with something more interesting.

## strace

strace prints out the system calls made by a particular process. One option when using strace is to start the program from within strace:

```bash
# trace the command `sleep 10`
$ strace sleep 10

execve("/usr/bin/sleep", ["sleep", "10"], 0x7ffc0b292728 /* 54 vars */) = 0
...
clock_nanosleep(CLOCK_REALTIME, 0, {tv_sec=10, tv_nsec=0}, 0x7ffd8fa9ef30) = 0
...
``` 

strace has lots of output so I removed most of it, but watching the output live you can see which system call your process is getting stalled on. In this case it is `clock_nanosleep`.

Sometimes you realize you want to see what a process is doing after the process has already started - in that case you can use strace to attach to an existing process by PID:

```bash
# start `sleep 10` in the background
$ sleep 10 &

$ pgrep sleep | xargs -n1 sudo strace -p 
```

Here we use `pgrep` to get the process ID of the `sleep` process, then use strace to attach to that process. Be aware that `pgrep` uses a regular expression to match the process name, so you might want to use `pgrep ^sleep$` or `pidof sleep` for stricter matching. 


The downside to attaching strace to a running application is that if the process has already called into a blocking system call, as is likely the case here with sleep, you won't see anything with strace until that system call returns and a next system call is invoked.

## procfs kernel stack

If you use strace to attach to a running process and find it stalled on a blocking syscall (so strace is not providing any output), you can use procfs to determine the currently running syscall, as well as the full kernel stack trace.

```bash
# start `sleep 10` in the background
$ sleep 10 &

$ pgrep sleep | xargs -I % sudo cat /proc/%/stack
[<0>] hrtimer_nanosleep+0xca/0x1a0
[<0>] common_nsleep+0x40/0x50
[<0>] __x64_sys_clock_nanosleep+0xd1/0x140
[<0>] do_syscall_64+0x33/0x40
[<0>] entry_SYSCALL_64_after_hwframe+0x44/0xa9
```

Here again we can see we are stuck in the `clock_nanosleep` system call, but this time we are able to figure it out while the process is blocked on the system call, whereas with strace we would have had to start tracing *before* the process calls the blocking system call.

## userspace trace

Now you might want to know where in the application code the syscall was called from. Or an alternate case would be that you find the application isn't stuck in kernel space but rather is stuck executing application code.

You can use `gdb` to see the current userspace stack trace.

```bash
# start `sleep 10` in the background
$ sleep 10 &

$ pgrep sleep | xargs -n1 sudo gdb --batch -ex "thread apply all bt" -p
0x00007f8b1b29f0da in clock_nanosleep@GLIBC_2.2.5 () from /usr/lib/libc.so.6

Thread 1 (process 132170 "sleep"):
#0  0x00007f8b1b29f0da in clock_nanosleep@GLIBC_2.2.5 () from /usr/lib/libc.so.6
#1  0x00007f8b1b2a4357 in nanosleep () from /usr/lib/libc.so.6
#2  0x000055fae2c404c8 in ?? ()
#3  0x000055fae2c40282 in ?? ()
#4  0x000055fae2c3d251 in ?? ()
#5  0x00007f8b1b1ffb25 in __libc_start_main () from /usr/lib/libc.so.6
#6  0x000055fae2c3d32e in ?? ()
[Inferior 1 (process 132170) detached]
```

The usefulness of this information depends on the existence of debug symbols, so you may not get much out of it unless you can re-compile the application with debug symbols included.

```bash
$ pgrep rust-analyzer | xargs -n1 sudo gdb --batch -ex "thread apply all bt" -p

# ... omitting lots of output from other threads

Thread 1 (LWP 113543 "rust-analyzer"):
#0  0x00007fc9b871ba9d in syscall () from /usr/lib/libc.so.6
#1  0x0000561397f843aa in std::sys::unix::futex::futex_wait () at /rustc/cb75ad5db02783e8b0222fee363c5f63f7e2cf5b//library/std/src/sys/unix/futex.rs:25
#2  std::sys_common::thread_parker::futex::Parker::park () at /rustc/cb75ad5db02783e8b0222fee363c5f63f7e2cf5b//library/std/src/sys_common/thread_parker/futex.rs:50
#3  std::thread::park () at /rustc/cb75ad5db02783e8b0222fee363c5f63f7e2cf5b//library/std/src/thread/mod.rs:885
#4  0x0000561397f6a673 in crossbeam_channel::context::Context::wait_until ()
#5  0x0000561397f6a36e in crossbeam_channel::context::Context::with::{{closure}} ()
#6  0x0000561397f6aa07 in crossbeam_channel::select::run_select ()
#7  0x00005613972e3c6f in rust_analyzer::main_loop::<impl rust_analyzer::global_state::GlobalState>::run ()
#8  0x0000561397371fe2 in rust_analyzer::main_loop::main_loop ()
#9  0x00005613971ed850 in rust_analyzer::main ()
#10 0x0000561397207e63 in std::sys_common::backtrace::__rust_begin_short_backtrace ()
#11 0x00005613972083a9 in std::rt::lang_start::_$u7b$$u7b$closure$u7d$$u7d$::h01cd5294bcfc3945 ()
#12 0x0000561397f96347 in core::ops::function::impls::{{impl}}::call_once<(),Fn<()>> () at /rustc/cb75ad5db02783e8b0222fee363c5f63f7e2cf5b/library/core/src/ops/function.rs:259
#13 std::panicking::try::do_call<&Fn<()>,i32> () at /rustc/cb75ad5db02783e8b0222fee363c5f63f7e2cf5b//library/std/src/panicking.rs:379
#14 std::panicking::try<i32,&Fn<()>> () at /rustc/cb75ad5db02783e8b0222fee363c5f63f7e2cf5b//library/std/src/panicking.rs:343
#15 std::panic::catch_unwind<&Fn<()>,i32> () at /rustc/cb75ad5db02783e8b0222fee363c5f63f7e2cf5b//library/std/src/panic.rs:396
#16 std::rt::lang_start_internal () at /rustc/cb75ad5db02783e8b0222fee363c5f63f7e2cf5b//library/std/src/rt.rs:51
#17 0x00005613971ee8f2 in main ()
```

This demonstrates what the output might look like for an appication compiled with debug symbols.

## Conclusion

These tools probably aren't enough to immediately root cause an issue, but they are a great way to gather some initial information which can help guide further troubleshooting effort.
