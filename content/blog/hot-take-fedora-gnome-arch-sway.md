---
title: Switching from Fedora & Gnome to Arch & Sway
date: "2020-06-18"
---

I've been a Linux user for ~1 year, switching from macOS which I had been using for a long time. I installed Fedora on a 4th generation X1 Carbon and everything just worked.

But of course Fedora is expected to work well on a ThinkPad. It was a safe choice but after a year it no longer felt like an exciting choice. To keep things interesting I decided I'd try out Arch Linux and with it the Sway window manager.

## Brief aside to discuss the Arch install

This isn't an Arch install tutorial, there are plenty of those, but I do want to point out a couple of things (mostly in case I want to look back at them in the future).

#### Wifi Setup

I did the Arch install over Wifi, which means connecting to Wifi using the zsh terminal provided by the Arch installer. I'm pretty comfortable in the terminal, but this was my first time ever connecting to Wifi without using a GUI.

Initially I tried using `wifi-menu`, which provides a nice terminal based UI, but for some reason it seemed to mis-identify my network as WPA rather than WPA2. Whether for that reason or another I wasn't able to connect using `wifi-menu`.

Ultimately I ended up using `wpa_passphrase` and `wpa_supplicant` as described [here](https://wiki.archlinux.org/index.php/wpa_supplicant#Connecting_with_wpa_passphrase).

After finishing the Arch install I rebooted the machine to realize that I had forgotten to install any network management utilities. The install system comes with a bunch of tools for this kind of thing (including `wifi-menu`, `wpa_passphrase`, and `wpa_supplicant`) but the default Arch install won't have any of them. To resolve this I had to boot back into the installer, connect to Wifi again, mount/chroot into the installed system, then install `networkmanager` which I could then use to permanently configure Wifi on my system.

#### Partitioning and Formatting HD

I tried to use cfdisk, which provides a TUI for setting up drive partitions, but I was nearly finished with the setup when I hit a wall setting up the bootloader. It turns out the version of cfdisk I was using defaulted (without asking) to a DOS partition table type, while I needed the GPT type for use on my UEFI system.

After figuring out what was happening I used fdisk and everything went fine from there.

#### Launch sway automatically on startup

This probably fits more into configuration than installation, nevertheless I think it is reasonable in the year 2020 to not call a new desktop OS install complete until it boots into a graphical environment automatically (or in my case at least after logging in).

I wanted a very minimal way to launch sway on startup, and I found a few variations of the script below online, but for some reason this was the only one that worked for me. Before this I was manually running `sway` from the tty.

```bash
if [[ -z $DISPLAY ]] && [[ $(tty) = /dev/tty1 ]]; then
	XKB_DEFAULT_LAYOUT=us exec sway
fi
```

#### Was it as difficult as they say?

For me I'd say the Arch install process lived up to the hype. The Arch wiki is, as everyone says, a very valuable resource but for me the [install guide](https://wiki.archlinux.org/index.php/Installation_guide) provided a bit of false confidence. It turns out that the reason the installation process is so manual is that it is very flexible, but you can't write a succinct guide walking through a very flexible process.

## Fedora vs Arch

The Arch wiki [describes Fedora as a distro for experienced users](https://wiki.archlinux.org/index.php/Arch_compared_to_other_distributions#Fedora), but that really didn't align with my experiences at all. I found the Fedora installer self-explanatory, and everything worked right out of the box.

Arch on the other hand is incredibly minimal, for example out of the box it doesn't include utilities to change the display brightness or audio volume. And even if those utilities were included, the default sway configuration doesn't map them to any keys. This minimalism provides an opportunity to choose the exact software you want running on your machine, but the time required to bootstrap a system is certainly something to think about when choosing a distro.

#### Package manager

I never had any issues with `dnf`, the package manager used by Fedora. I even upgraded major Fedora versions twice with zero issues.

That said, so far I am really liking `pacman` (the Arch package manager). For some reason `pacman` feels substantially faster than `dnf` (as in, packages are installed much more quickly).

I also appreciate the rolling release model, meaning in general I'll be able to run newer software versions on Arch than I would on Fedora, although I know this has its tradeoffs.

I've heard it said (although I can't find a reference for it now) that Arch has the largest package repository of any Linux distribution, especially if you include the `AUR`. That sounds great as an outsider, but after using the AUR for a short time it doesn't really seem fair to include packages from the AUR in with those from pacman. With AUR, you typically will still `git clone` a package to your machine and build it from source. Further, the package manager does not manage updates for AUR packages. The primary benefit of AUR over the normal process of installing software from source seems to be that with AUR you can skip the step of digging through the GitHub README to figure out how to build/install it.

## Gnome vs Sway

Fedora and Arch are very different, but day to day the biggest differences I'll experience are from changing window managers (or desktop environment in the case of Gnome) from Gnome to Sway.

While I've long had a keyboard centric workflow, I never felt the need for a tiling window manager because 99% of the time in Gnome I had two windows open, a web browser and a terminal, and both were full screen. I switched between them using my keyboard (alt-tab). If I needed multiple terminal windows, which wasn't often since I'm a heavy user of vim splits as well as the vim `:term`, I'd use the tab feature of Gnome terminal. There again I could switch between them with my keyboard, although the default key combination is more of a reach (ctrl-pgup/pgdown).

#### Configuration

Configuration was the win I wasn't expecting in this switch. Customizing Gnome involves downloading the Gnome tweak tool and navigating through a GUI. There are ways around this, but they are clumsy enough that this is the main approach suggested to do something like remap the caps lock key to ctrl.

On the flip-side, *everything* in Sway is configurable via a human-readable (and editable) text file. That includes not just remapping keys, but also things like keyboard repeat rate, display scaling, etc.

One very minor configuration option that I'm really happy to have is the ability to hide the mouse cursor some time after you stop moving it.

```
# Hide cursor after one second of inactivity.
seat * hide_cursor 1000
```

Another is the ability to hide all titlebars, which is surprisingly hard to do in Gnome. The configuration below hides titlebars for all applications unless I have more than one application on a given workspace while in tabbed mode (more to come on that).

```
# Hide titlebar, even while in tabbed mode if there is only a single window.
default_border none
hide_edge_borders --i3 smart
```

#### Sway for a person who likes fullscreen

I was a bit concerned using a tiling window manager as someone who never used tiled windows, but I made a few minor configuration tweaks and I'm now working seamlessly in Sway.

1. Set `tabbed` mode as the default, which makes new windows open as a tab on a given workspace rather than splitting the workspace.
2. On startup, open a web browser in workspace 1 and a terminal in workspace 2 (both full screen).
3. Configure alt-tab and alt-shift-tab to move forward and backward through workspaces.
4. I open multiple terminals, when needed, as tabs on workspace 2 and cycle between them with alt-h and alt-l (as in left/right in vim).

## Final thoughts

I never felt like my Fedora & Gnome setup was slow, but wow the Arch & Sway setup is very fast. I've also enjoyed the minimalism, which fits with my workflow as I tend to spend 99% of my time in a terminal or browser.

All that said, I think my favorite part of this new setup is that all configuration is done in plain text files. The sheer configurability of Sway certainly doesn't hurt either.

I'll be running Arch & Sway for a while.
