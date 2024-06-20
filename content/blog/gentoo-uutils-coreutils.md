---
title: Swapping GNU coreutils for uutils coreutils on Gentoo Linux
date: "2024-06-19"
---

[uutils coreutils](https://uutils.github.io/coreutils/) is a core utilities project (providing commands like `ls`/`cp`/etc) written in Rust and targetting compability with the GNU core utilities.

They have made impressive progress on their goal of passing all the tests in the GNU test suite, so I wondered what it would look like to run them as the primary coreutils on a full Linux system. Importantly, I didn't want to test this in an embedded application where the coreutils may be used primarily by humans as debug tools; I wanted to test this in an environment where the coreutils act as a fundamental part of the system.

[Gentoo Linux](https://www.gentoo.org/) uses the Portage package manager which has a heavy reliance on bash and thus on coreutils. It is also known to be very configurable, to the point of [calling itself a metadistribution](https://www.gentoo.org/get-started/about/). Despite having no previous exposure to Gentoo, I decided for these reasons to use it as the environment for this real world test of uutils coreutils.

## Gentoo flexibility makes this all incredibly easy, right?

Unfortunately, no..

> Portage, which is required by @system, hard depends on the GNU Coreutils. This means that busybox cannot be used as a replacement for the GNU Coreutils on Gentoo in most situations..
>
> [Source](https://wiki.gentoo.org/wiki/Busybox)

This post isn't about busybox, but the quote here makes it clear that Gentoo doesn't support swapping out coreutils "in most situations". What are most situations? They don't exactly say what they don't support, but I assume they do support embedded type use cases where Portage doesn't get installed on the final system. If we want a fully functional system, Portage included, we are going to have some more work to do.

## Virtual packages and the lack thereof

Portage supports the concept of [virtual packages](https://wiki.gentoo.org/wiki/Virtual_packages), which are a way to express that there is more than one underlying package which can fulfill a particular dependency. For examples:

* `virtual/editor` lists many console based text editors
* `virtual/ssh` lists `dropbear` and `openssh`

This would be a great fit for our use case of swapping GNU coreutils for the uutils version. A coreutils virtual package doesn't exist, but I could make one and put it in a local package repository, and then create an alternate [Portage ebuild file](https://gitweb.gentoo.org/repo/gentoo.git/tree/sys-apps/portage) that references the virtual package instead of directly depending on GNU coreutils. And I did all that. And then I found out that Portage isn't the only package that depends directly on GNU coreutils, [there are many](https://packages.gentoo.org/packages/sys-apps/coreutils/reverse-dependencies). That meant that installing any of those packages would try to re-install GNU coreutils, which wouldn't work because it conflicts with uutils coreutils.

The answer to this is replacing all depencies on GNU coreutils with the virtual package, but that seems like a lot of work just to figure out if this is at all feasible. Creating a virtual package is still probably the *right* thing to do, and I may come back to explore this further at some time, but for now I'm going to move on to the hacky approach.

## If it quacks like a duck

Instead of modifying all the packages that depend on GNU coreutils (known as the package `sys-apps/coreutils` in Gentoo), I modified the GNU coreutils package to instead install uutils coreutils. This is obviously incredible hacky, but this is an experiment so I'll take quick learning over fundamental soundness.

I did this by copying the contents of a [uutils coreutils ebuild](https://gitweb.gentoo.org/repo/gentoo.git/tree/sys-apps/uutils-coreutils) (ebuilds in Gentoo are files that describe how a package is built) into a local package repository under the name `sys-apps/coreutils`, causing it to override the original package file which would have installed GNU coreutils. I chose to increment the major version by one from the newest release of `sys-apps/coreutils` just to be sure that my local copy of the package would be considered an upgrade from the existing system install.

At that point, running `emerge --update sys-apps/coreutils` was enough to replace GNU coreutils with uutils coreutils on my system.

## Problem #1 - uutils command prefix

Well actually it wasn't quite as easy as running the update command. The existing uutils coreutils package that I copied the ebuild from was meant to install alongside GNU coreutils, so they prefix all the installed binaries with `uu-`. This is not desirable if we want to use these as our primary coreutils.

The fix here was to patch the ebuild to remove the `uu-` prefix.

## Problem #2 - uutils extra commands

The next attempt to update the coreutils package resulted in file collision errors from Portage. It turns out that uutils coreutils by default includes some binaries which on Gentoo are installed by packages other than the GNU coreutils package.

* `more` is installed by sys-apps/util-linux
* `hostname` is installed by sys-apps/net-tools
* `kill`/`uptime` is installed by sys-process/procps
* `groups` is installed by sys-apps/shadow

Fortunately the uutils Makefile allows setting an environment variable to specify any binaries that should be skipped when installing. The fix here was adding `SKIP_UTILS="more hostname kill uptime groups"` to the ebuild.

This was enough to get uutils coreutils installing as an "update" (but really a replacement) to GNU coreutils.

## Problem #3 - uutils missing commands

Things are this point seemingly work. I can `ls` and `mkdir` and etc. The first real test was using these coreutils as the base for installing another package.

Trying to run `emerge cowsay` resulted in an error about missing command `md5sum`. It turns on that GNU coreutils, at least as packaged on Gentoo, include a specific `md5sum` binary. uutils coreutils only includes the generic `cksum` binary.

I wrote a one line (plus a #!) bash script `cksum -a md5 --untagged $@` and modified the ebuild to install this as `md5sum` when installing uutils coreutils. But this was still not enough to get the first package installed using these coreutils.

## Problem #4 - `install-xattrs` unhappy with `install` being a symlink

Running `emerge cowsay` again, the `md5sum` errors were fixed but I was now receiving errors when the package manager ebuild files would try to run `install`. Similar to busybox, coreutils uutils was being installed as a single binary, and each entrypoint (i.e. `ls`) was just a symlink to that binary. The error was happening when the cowsay Makefile tried to run `install`, and it was as-if the underlying coreutils "multicall binary" didn't recognize it. This was puzzling because `install` definitely was a valid command.

It took running `emerge cowsay` under strace to realize `install` wasn't actually being called directly. Rather, for reasons I didn't fully track down but suspect to be related to the Gentoo [xattrs use flag](https://packages.gentoo.org/useflags/xattr), the `install-xattrs` binary was being called. This binary does some things on its own, and then calls the `install` binary. Only instead of calling it directly, it follows and resolves all symlinks, which in our case means it directly calls the uutils coreutils multicall binary rather than calling it through the `install` symlink. This caused the multicall binary to attempt to use the first argument (which happened to be `-d`) as the command name, and that command name didn't exist.

I didn't look closely at why `install-xattr` follows symlinks like this. I also didn't figure out exactly why `install-xattrs` is being substituted in for calling `install` directly as the Makefile default. Here again I took the easy way out and updated the ebuild to install full binaries for each application, rather than building one multicall binary and using a bunch of symlinks. An alternate approach would have been to use hard links rather than symlinks, but that would have been more work to implement.

Finally `cowsay` was installed, and we can claim victory - uutils coreutils can be used in place of GNU coreutils on Gentoo!

## Problem #5 - cowsay cannot find template files

If only things were so simple. I barely finished celebrating before I figured I should actually try to *run* cowsay. At which point rather than a cute cow I was greeted with this:

```
# cowsay hi
cowsay: Could not find cowfile for 'default.cow'!
```

I was stumped on this for a bit before realizing it wasn't a uutils coreutils problem at all. This Gentoo setup has `/bin`, `/sbin`, `/usr/bin`, and `/usr/sbin` all merged. Since I was running this as root (I was doing all this testing without a real Gentoo install, just chroot'ed into a stage3 tarball) `/usr/sbin` was on my path before `/usr/bin`, which broke an [expectation of the cowsay package](https://github.com/cowsay-org/cowsay/blob/d8c459357cc204723504e29be84607ceef2c5d42/cowsay#L85).

```
# /usr/bin/cowsay hi
 ____ 
< hi >
 ---- 
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```

Finally we can celebrate - uutils coreutils can be used in place of GNU coreutils on Gentoo (and the system can even install packages, and they *work*)!

## Problem #6 - more uutils missing commands

Let's try installing one more package, just for fun:

```
# emerge ripgrep

...
...
...

/var/tmp/portage/sys-apps/ripgrep-14.1.0/temp/environment: line 943: sha256sum: command not found
```

It turns out `md5sum` isn't the only command missing from uutils coreutils, when compared to GNU coreutils. This can probably be fixed using the same approach we took for `md5sum`, but at this point maybe it makes sense to find exactly what is missing rather than waiting to run into problems.

The correct answer here, as far as I know it, is to use `equery files sys-apps/coreutils` before replacing GNU coreutils to figure out exactly what is provided. Then run the same command affter "updating" to uutils coreutils. From there it'd be a matter of diff'ing those two lists and patching up any gaps.

## Try it out

If you want to play around with this, clone [this repo](https://github.com/JoshMcguigan/cascade), checkout `aac06b31d318542ee363d1299267a8bb49265bd9`, and run `just clean chroot`. This will pull down a Gentoo stage 3 tarball and replace the GNU coreutils with those by uutils. From there you'll be dropped into a chroot to test out the system.

