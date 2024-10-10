---
title: Why NixOS?
date: 2024-10-10
author: Spencer Balogh
description: Why NixOS?
tags:
  - NixOS
toc: true
---

I've been using linux for quite a while. This has included personal, academic, and professional use. I started using NixOS on my physical machines last year, and I already have a really difficult time imaginging not using NixOS. I wanted to put together a post to talk about what I think is really unique, interesting, and compelling about it.

This gets a bit into some of the internals. I'll try to keep it general and understandable without prior knowledge of NixOS or the Nix language.

<!--more-->

## When NixOS

NixOS has been around for bit. It started as a research project over 20 years ago. I don't know exactly when it enered my radar, but I feel like I've been vaguely aware of it for quite a while.

I really stared to get intereseted in NixOS when I read about the introduction and more specific details about Flakes. They were introduced as an experimental feature along with a new nix CLI in 2021. I had already worked pretty extensively with nodejs package management with package-lock.json, and dove deep into the intimates of package management in that ecosystem. I really apprecaiate reproducible systems more due to these experiences, and flakes felt like an amazing tool for making nix reproducible to me. I started occasionally experimenting with NixOS VMs and reading more not long after this.

I knew enough that I took the plunge on physical machines last year.

## Flakes

NixOS can be used without flakes, but I haven't used it without them personally. Flakes are still technically an *experimental feature*. What does that actually mean?

"Experimental" here is more about communicating promises of API stability. It's still experiemental because there may be breaking changes with flakes going forward. Personally I don't mind the idea that I might have to rewrite some flakes, and I think the utility of them is nice enough to justify their use. The fact that they have been experimental for so long does come with some downsides. Experimental features do have to be explicitly opted into. The [official nixos documentation](https://nixos.org/learn/) also only covers "stable" features, so external documentation like the [NixOS wiki](https://wiki.nixos.org/) or other blogs need to be referenced to find a lot of details about it. I got a lot of value out of [zero-to-nix](https://zero-to-nix.com/) and the [NixOS & Flakes Book](https://nixos-and-flakes.thiscute.world/) personally.

Regardless, I'm not alone in adopting them. Version pinning is powerful, and especially in source code driven projects its seen a lot of adoption. More projects are being created with flakes than without at the moment, and there are RFCs in place detailing incremental process to stabilization of these features. At this point it's a matter of when and not if they will be stabilized.

## What's Compelling About NixOS?

### Configuration management is the first class default

Package installation and configuration in NixOS is primarily driven by configuration written with [Nix the programming language](https://nix.dev/). This is the exact same language that is also used to write derivations (read: packages, derivations are build tasks, we'll talk about this is a second). Nix is not just bash scripting installs. It's a powerful functional programming language. This means that you can implement complex conditional behaviors in your config, and your config can be stored in version control to keep history.

This does mean that NixOS isn't for everybody. It's a bit more difficult to install a package on NixOS than just running `sudo apt get ...`. Learning enough nix to manage a basic config isn't too tough, and simple configurations don't really even need to utilize functional programming features, but it still does require opening and editing a config file.

#### Reasonable escape hatches

We'll get into why in a second, but you can't just download and run a linux binary on NixOS. This is a dealbreaker for some people as well, but there are some reasonable ways to get around this without updating your config.

Tools like `nix shell` and `nix-shell` allow you to temporarily install a package and run it without needing to make it a permanent part of your system config. If you've ever used `npx` or `yarn dlx` in the javascript ecosystem, it's a lot like that. This is a great way to get a hold of uncommonly used tools. `nix shell nixpkgs#usbutils --command lsusb -tv` installs and executes `lsusb` to output information about your USB hardware. This tool automatically gets cleaned up the next time you run garbage collection.

Dev shells and `nix develop` let you install development dependencies. These can be specific versions of development tools and library dependencies for your projects. This is a lot like managing multiple versions of `npm` with `nvm`, and project specific dependencies with `npm` and `package.json`. These can even be loaded automaticaly with tools like `direnv`. To make loading these development tools automatic when you enter a project directory.

I don't do this personally, but Flatpak can be used on nixos for managing user space applications.

I'm not a free and open software purist (`nixpkgs.config.allowUnfree = true;`), but I don't like running random binaries from the internet. I don't have any reason to do this personally, but there are methods to run binaries. You can patch the binary to pack a loader and libraries (and write a derivation to do this in a repeatable way), use FHS to simulate a classic linux shell, sandbox applications with `steam-run`, or run programs directly with `nix-ld`. This doesn't need to be a dealbreaker if you can't avoid it.

### The nix store

All of the software on your system gets installed to the nix store. This is a filesystem location at `/nix/store`, and every derivation is stored in the filesystem here.

#### Reproducible Builds - Derivations are everything

Nixos packages are declared using derivations.  What does that even mean?

<mark>Note:</mark> I'm skipping a few details here. `stdenv.mkDerivation` is a wrapper around `bultins.derivation`, and some ideas here are simplified. I'm just trying to indicate what I think is valuabe here, not every intimate detail.

Most packages in nixpkgs (the nix packages repository) are declared using `stdenv.mkDerivation` documented [here](https://nixos.org/manual/nixpkgs/stable/#sec-using-stdenv).  The general idea behind a derivation is that it's a set of attributes that describes how to build a package. Typically this consists of at least the package name, the package version, any code or binary sources (this includes hash checksums to verify the content), buildInputs (dependencies), a buildPhase script, and an installPhase script. This set of attributes indicates everything that is required to install a package.

When a derivion is built, it gets persisted to the nix store, located at `/nix/store/<entry>`. Nix store entries are specified as `$packageHash-$packageName-$version`. The `$packageHash` is a function of the set of environment variables, the inputs, and the derivation attributes that specify the package. This means that different source or different config produces an entirely different package! This doesn't make NixOS immune to supply chain attacks, but if you are concerned with software supply chain issues this is a huge amount of information and context to detect issues (at least until source itself or your tooling is attacked 😬).

#### Handling multiple verions of dependencies

This is elaborating on another benefit of the nix store and how packages are built.

Derivations are all built into the nix store. What does this actually look like in a live system? As an example, let's look at curl. It's a binary that can be used to make network requests from the command line, and it's also a library that allows other programs to make network requests via libcurl. The built curl binary lives in the nix store. How does NixOS know where to find this?

```
$ which curl   
/run/current-system/sw/bin/curl
$ ls -l /run/current-system/sw/bin/curl
lrwxrwxrwx 6 root root 67 Dec 31  1969 /run/current-system/sw/bin/curl -> /nix/store/6r0bn0dkvlvhicyvair205s07m92dpaz-curl-8.9.1-bin/bin/curl
```

The system's equivalent to `/bin/curl` is a symlink into the nix store where the current live system's curl binary lives.

This is even more interesting when you look at libcurl:

```
$ ls -al /nix/store/*-curl-*

...

/nix/store/2888pjmiry2b58gcjn0bv3p4g4d4il4k-curl-8.7.1

...

/nix/store/lphchxfs3iqwk1i3jy6wflkzmklk7wp4-curl-8.9.1
/nix/store/s8mwi42ihglv70gxkaz238adlk76s3dm-curl-8.9.1
/nix/store/x6ssc2mmx1kb52gchksqbzg5c2y0z7lf-curl-8.9.1

...

```

My current packages and dev environments have dependencies on multiple different versions of libcurl, but on top of that they rely on multiple different libcurl 8.9.1 shared objects. These might be differently configured versions of the same libcurl source code, or these might even be different versions of libcurl code that aren't differentiated with a different release version / tag. Personally I think this is a worthwhile trade-off. RAM and disk space are cheap, and I'd rather sacrifice a small amount of each so that consumers of libcurl can get the exact version they want. This is a small cost to avoid issues that are typical with dynamic linking even when dynamic linking is explicitly expected.  I don't think I'm alone with this risk assessment.  Most build systems for modern programming languages seem to be pushing toward static linking for most use cases.

Due to package pinning with flakes, all of my NixOS systems have the exact same curl library versions (minus whatever is only installed with `nix develop`), regarldless of what version of nixpkgs was live at build time.

#### Rollbacks - a layer of assurance around rolling release

Part of the reason that I have multiple versions of libcurl is that I'm utilizing `nixos-unstable`. This is the version of `nixpkgs` that lives in the `nixos-unstable` branch on github. This probably isn't quite as extreme as it sounds. `nixos-unstable` is unstable in the way software is "unstable". Config for this branch is permitted to make changes that would necessitate a semantic versioning `MAJOR` version bump. Most updates nixos-unstable don't actually make any breaking changes to packages, so the maintenance overhead of using this branch isn't actually a whole lot higher than targeting a versioned release. `nixos-unstable` also gets fully built and populated into the cache.nixos.org, so it doesn't install any slower than a versioned release. It's just functionally the "rolling release" of NixOS.

I like using a rolling relase operating system. Especially with respect to virtualization, I've been an aggressive adopter of new libvirt / qemu / kvm features. It isn't perfect and sometimes things break, but I can usually fix things when they break, and I've learned a lot about linux fixing these breaks. Occasionally I've been tired of this break / fix cycle and moved away from distributions like Arch Linux or Manjaro. NixOS makes it incredibly easy to deal with issues with misconfiguration and potentially broken packages via rollbacks.

Your live system is a set of links to your current versions in the nix store. When you update your system with `$ nixos-rebuild ...` any new software versions or configurations are built and persisted to the nix store, and additionally a new entry is added to your boot loader, and this new entry includes any boot process changes that are necessary, but also initialization for your new set of nix store links. All of the prior packages and boot entries still remain. If your system breaks with an upate you're a reboot away from reverting to a prior stable state, and old package / boot entries are persisted until they are garbage collected.

I've also found that this gives you a lot of freedom to experiment aggressively. I just transitioned all of my computers to using [systemd inside my initrd](https://systemd.io/INITRD_INTERFACE/), and I was never concerned about breaking anything permanently.

#### Flakes, an open ecosystem

Building from source is very first class in nixos with flakes. Nixpkgs itself can be consumed as a [flake](https://github.com/NixOS/nixpkgs/blob/master/flake.nix), and the primary thing that's special about nixpkgs is that they get built and stored into cache.nixos.org, and that cache is automatically trusted by default.

All of the same tooling is available to other projects. Any project can publish a flake and can even host their own binary cache for their packages to make their packages accessible without needing to be a part of nixpkgs.

## A work in progress

All of these features are pretty compelling to me. I've gradually been convering all of my infracstructure over to nixos. I'll have some other blog posts about how I've implemented my home lab, and lessons learned along the way coming soon.
