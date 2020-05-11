---
layout: post
title:  "Purely Functional Package Management"
date: 2020-05-09 21:08:26 +0700
categories: nix
---

{:.article-content}
Being a fan of pure functional programming, I'm intrigued by the prospect of
applying those concepts to package management on my systems. Since I'm mostly
on macOS these days, I'm going to start with the [Nix][1] package manager for
my initial foray into this new and exciting world.

{:.article-content}
I've been a dedicated user of [Homebrew][2] since [Max][3] first released it.
However, after watching Tim Steinbach's &#955;C 2019 presentation [Sane System
Management with NixOS - I Do Declare!][4], I found myself motivated to give Nix
a try. The talk is well worth a watch. Tim does a good job of making the case
for using Nix. For me, some highlights from the talk were

* One single configuration file for kernel, services, and updates
* Atomic upgrades and rollbacks
* Side-by-side installation of different versions of the same package
* (Re-)installation within minutes
* Bitwise reproducibility of the entire system

### Quickstart

By default, the Nix installer will perform a single-user installation. However,
the documentation *highly* recommends opting in to the mult-user installation.

From a terminal, install multi-user Nix:
```
$ sh <(curl https://nixos.org/nix/install) --daemon
```

Since Nix installs everything in the Nix store, removal is quite easy if we
find that we don't actually like the project. On macOS:
```
$ sudo rm -rf \
      /etc/profile/nix.sh \
      /etc/nix \
      /nix \
      ~root/.nix-profile \
      ~root/.nix-defexpr \
      ~root/.nix-channels \
      ~/.nix-profile \
      ~/.nix-defexpr \
      ~/.nix-channels

$ sudo launchctl unload /Library/LaunchDaemons/org.nixos.nix-daemon.plist
$ sudo rm /Library/LaunchDaemons/org.nixos.nix-daemon.plist

# rollback global etc files modified during install
$ mv /etc/profile.backup-before-nix /etc/profile
$ mv /etc/bashrc.backup-before-nix /etc/bashrc
$ mv /etc/zshrc.backup-before-nix /etc/zshrc
```

### Migrating off of homebrew
Before we start messing with our installed packages, its a good idea to dump
our homebrew packages so we can restore everything if things go sideways with,
or we decide we don't actually like, Nix.

```
$ brew bundle dump
```

This will generate a `Brewfile` in the current working directory, which we can
squirrel away in the event we need it later. We can restore our homebrew managed
packages with:

```
$ brew bundle
```

Hopefully we won't need it as I anticipate not wanting to come back after
we free our mind and take the red pill, but better to be safe and prepared just
in case we change our mind.

{:.article-content}
[Salar Rahmanian][7] put together a quick guide on getting started migrating
off homebrew. In it he provides a command that lists the formula installed by
homebrew and the formulae installed that specify the formula as a dependency.
```
$ brew list -1 \
    | while read formula; do echo -ne "\x1B[1;34m $formula \x1B[0m"; brew uses $formula --installed \
    | awk '{printf(" %s ", $0)}'; echo ""; done
```

Once you've visualized your homebrew footprint you can use the CLI to look for
a similarly named package:
```
$ nix-env -qaP | grep -i <packagename>
```

{:.article-content}
or [search packages][8] online to find a suitable Nix flavored package.

{:.article-content}
Another helpful resource that goes deeper into Nix is Geoffrey Huntley's
[Mastering Nix Workshop][9]. This coupled with [Mastering NixOS][10] offer a
treasure trove of learnings I plan on mining next.

### Next Steps
{:.article-content}
In Tim's talk, he mentions how he uses [NixOS][5] in addition to the Nix package
manager. NixOS is a GNU/Linux distribution that allows you to achieve a fully
declarative system configuration model. [Nix modules for darwin][6] attempts to
achieve similar things for macOS.

{:.article-content}
Once I've fully digested these concepts, I'd like to codify the best practices in
my [dotfiles][11] repo and get to an even more idealized place of consistent,
functional, and sane systems management.


[1]:  https://nixos.org/nix/
[2]:  https://brew.sh/
[3]:  https://mxcl.dev/
[4]:  https://www.youtube.com/watch?v=_LDzO5_d1a0
[5]:  https://nixos.org/
[6]:  https://github.com/LnL7/nix-darwin
[7]:  https://www.softinio.com/post/moving-from-homebrew-to-nix-package-manager
[8]:  https://nixos.org/nixos/packages.html
[9]:  https://github.com/ghuntley/workshops/tree/master/nix-workshop
[10]: https://github.com/ghuntley/workshops/tree/master/nixos-workshop
[11]: https://github.com/jopecko/dotfiles