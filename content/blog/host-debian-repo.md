---
title: Maintaining your own Debian package repository
date: "2024-08-15"
---

After a few years of using rolling release Linux distributions I migrated to Debian for some stability. But despite the warnings about [breaking debian](https://wiki.debian.org/DontBreakDebian) there were times when I really needed software that was newer than the Debian stable repositories offered.

There are lots of ways to accomplish this, but I wanted to see what it was like staying within the `apt` package management world (as opposed to installing another package manager like `nix`). More broadly I wondered what it might look like to layer a repository of very up to date end user software on top of a stable debian base, all using `apt`.

Warning: This idea is barely half baked. I am going to be stepping away from it for a while so I wanted to write some of it down while I still remember. You can see the current state of my work on this in the [git repo here](https://github.com/JoshMcguigan/community-repos).

## Building \*.deb

I won't go into much depth about how to actually create the packages. There are lots of ways to do it depending on how closely you want to follow the upstream [debian packaging guidelines](https://wiki.debian.org/Packaging). For myself, the goal was to build packages which wouldn't break the system; I wasn't trying to uphold any additional promises. To keep things simple this basically meant I wouldn't install any dynamic libraries, since these could conflict with other packages. By sticking to mostly statically linked executables, or executables which link only to upstream debian libraries, I should be able to avoid breaking any debian system.

For rust programs, I found [cargo deb](https://crates.io/crates/cargo-deb) to be helpful.

## Creating the debian repository

Debian repositories are setup to be hosted by a static file server, so long as the directory structure is organized in just the right way. Fortunately there is a tool called [reprepro](https://wiki.debian.org/DebianRepository/SetupWithReprepro) which makes generating and maintaining (while you add or remove packages) this directory structure easy.

```sh
$ mkdir debpkgs

$ reprepro -S unknown -b debpkgs includedeb bookworm path/to.deb
```

You do need [some config](https://github.com/JoshMcguigan/community-repos/tree/main/conf) in the `debpkgs` directory for this to work, but otherwise it is really easy.

## Hosting the debian repository

I chose to host the repository on an object store (specifically I chose cloudflare r2). This keeps the hosting really simple (no underlying OS to maintain), and it also allows easy updating by mounting the object store onto a build machine when I want to add packages.

## Building packages in CI

A potential concern with a binary package repo is that malware could be injected into the packages (either by the owner of the repo, or an attacker). To mitigate this, I wanted to build all the packages in CI, so a consumer of these packages could fully audit the environment they run in.

Making this all [work in github CI](https://github.com/JoshMcguigan/community-repos/blob/main/.github/workflows/deploy.yml) was probably the most time consuming part of this project.

* Environment secrets are used so the CI job has access to cloudflare r2 credentials and a gpg key
* The r2 object store is mounted onto the file system via rclone/fuse
* Any packages which are configured but which do not exist in the object store are built (build the binary, package it into a deb, and use reprepro to update the files on the object store)
  * This step is done (very poorly) by a tool I prototyped [pkgr](https://github.com/JoshMcguigan/community-repos/blob/main/pkgr/src/main.rs)

## Consuming packages from this repo

The github repository publishes a public key which can be used to verify the source of the packages, and has [instructions](https://github.com/JoshMcguigan/community-repos/tree/main?tab=readme-ov-file#setup-debian-bookworm-to-use-these-packages) which explain how you can setup your debian machine to use this package repo.

For now only a single package is published, and it'll likely stay that way for a while. But I didn't find any other resources that describe how to host and maintain a debian package repository utilizing CI tooling, so I wanted to share what I came up with. I'm sure there are many things I'm doing sub-optimally here, and I'd be thrilled to receive feedback in that regard. 

