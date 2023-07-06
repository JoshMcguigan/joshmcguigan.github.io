---
title: A surprising fact about ldd
date: "2023-07-05"
---

[ldd](https://man7.org/linux/man-pages/man1/ldd.1.html) is a utility to print the shared objects required by a program or shared object, as well as the location where the objects are found. An example invocation, stolen from the man page and lightly edited, follows:

```sh
$ ldd /bin/ls
    linux-vdso.so.1 (0x00007ffcc3563000)
    libc.so.6 => /lib64/libc.so.6 (0x00007f87e4e92000)
    /lib64/ld-linux-x86-64.so.2 (0x00005574bf12e000)
```

The first line lists [vdso](https://man7.org/linux/man-pages/man7/vdso.7.html), which is a special shared object injected by the kernel. No path to this object is listed because, unlike other shared objects, it doesn't actually exist on disk.

The second line lists `libc` and shows that it is found at `/lib64/libc.so.6`. This path is determined by the rules of the dynamic linker itself, [ld-linux.so](https://man7.org/linux/man-pages/man8/ld.so.8.html), which is listed on the third line.

However, running that command on my machine results in the following (again lightly edited):

```sh
$ ldd /bin/ls
    linux-vdso.so.1 (0x00007fff2e1f3000)
    libc.so.6 => /usr/lib/libc.so.6 (0x00007f79ca26c000)
    /lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007f79ca4ab000)
```

Note the dynamic linker, which is specified by `/bin/ls` as the absolute path `/lib64/ld-linux-x86-64.so.2`, is shown as being found at `/usr/lib64/ld-linux-x86-64.so.2`. Setting aside the fact that these are actually the same files (by the magic of symbolic links) because the linker/ldd doesn't know this, why is something which is specified as an absolute path being looked up at all?

## shared objects choose their dynamic linker

Shared objects in the elf format are able to choose their dynamic linker.

```
$ readelf -l /bin/ls | grep interpreter
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

Based on this, I would have assumed that `ldd` would somehow base its output on the specific linker requested (as "interpreter") by the program or shared object being queried.

## `ldd` implementation

In a [glibc](https://www.gnu.org/software/libc/) based environment, `ldd` is a shell script. See for yourself with `cat $(which ldd)`.

```sh
$ cat $(which ldd) | grep RTLDLIST=

RTLDLIST="/usr/lib/ld-linux.so.2 /usr/lib64/ld-linux-x86-64.so.2 /usr/libx32/ld-linux-x32.so.2"
```

A surprising fact about `ldd` is that its behavior is not based on the dynamic linker requested by the program (or shared object) being queried. Rather `ldd` invokes a standard linker from one of a few expected locations, and asks that linker for the linking data in order to present it to the user.


## `ldd` by hand

Effectively all the `ldd` script is doing is invoking the standard linker with some special environment variables set so that it prints the linking data that the user cares about.

```sh
$ LD_TRACE_LOADED_OBJECTS=1 /usr/lib64/ld-linux-x86-64.so.2 /bin/ls
    linux-vdso.so.1 (0x00007ffed0e9b000)
    libc.so.6 => /usr/lib/libc.so.6 (0x00007fd1526ea000)
    /lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007fd152929000)
```

Now that we are invoking the dynamic linker ourselves, we can choose the one which is actually requested by the program (`/bin/ls` in our example).

```sh
$ LD_TRACE_LOADED_OBJECTS=1 $(readelf -l /bin/ls \
    | grep interpreter \
    | cut -d':' -f2 \
    | tr -d ' ]') \
    /bin/ls

    linux-vdso.so.1 (0x00007ffc971ca000)
    libc.so.6 => /usr/lib/libc.so.6 (0x00007f127ca05000)
    /lib64/ld-linux-x86-64.so.2 (0x00007f127cc44000)
```

## `ld-linux.so` no longer mapped

When we ask the linker that `/bin/ls` requested (as its "interpreter"), `ld-linux.so` is no longer shown as mapped from one absolute path to the other. This helps explain what is happening in the cases where `ldd` does show the absolute path to the linker as being mapped somewhere else - in a real execution of this program the kernel would use the dynamic linker requested, there would be no lookup involved. But due to the implementation of `ldd`, specifically how it always uses a standard linker rather than the one specified by the shared object or program being queried, it can sometimes look like the dynamic linker itself is being found at some alternate path.

## Conclusion

I'm not aware of any cases of programs specifying a non-standard dynamic linker, however, if one did, there is no guarantee that the output from `ldd` will be accurate when run on that program.

Of course, even if `ldd` was willing to take the security risk of running a potentially untrusted dynamic linker, there is no guarantee that linker would provide the linking data that the user is requsting when run with the same environment variables set. For that reason, there is no general way to know how an arbitrary dynamic linker might map the set of shared objects requested by a program to specific file paths on your machine without running the program.

## A note about musl

This investigation was focused on `glibc` based platforms, however `musl` `ldd` is implemented similarly (they just use a symlink to point `/bin/ldd` to the dynamic linker rather than creating a wrapper shell script).
