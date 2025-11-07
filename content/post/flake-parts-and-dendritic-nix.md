---
title: flake-parts and dendritic nix
date: 2025-11-07
author: Spencer Balogh
description: A retrospective on incrementally migrating to flake-parts and dendritic nix.
tags:
  - NixOS
  - flake-parts
  - nix flake
  - dendritic nix
toc: true
---

I came across [dendritic nix](https://github.com/mightyiam/dendritic) a while ago. I didn't know enough about flake-parts at the time to really get my head wrapped around it, but I definitely sympathized with the problems people thought it helped solve in the surrounding discussion.

I kept seeing it, and I kept poking at the composing parts, and eventually I got it. I'm currently almost finished migrating [my dotfiles](https://github.com/hyperparabolic/nix-config) to the dendritic pattern. I mostly just need to re-write some of my docs, revise my bootstrap scripts for the new repo structure, and take care of a few flake level stragglers (overlays, templates), but my migration of internal nixos and home-manager modules is complete!

This blog post gives a high level overview flake parts, and the dendritic pattern. It gets into some of the ideas I find the most useful, and also gives an overview of my migration strategy and a retrospective on this change.

<!--more-->

## The relevant parts of flake-parts

I think [flake-parts](https://flake.parts/index.html) does a pretty poor job selling itself. The documentation is a little scant, and most of the examples they provide are a bit trivial. I didn't really understand the appeal of it until I started looking at other people's repos that were using it, and even then it took a moment to digest it.

The general idea behind flake-parts is that it enables you to split up your flake into flake-parts modules. Each of these modules has access to *any* flake or flake-parts attribute, and `flake-parts.lib.mkFlake` will take those imported flake-parts modules and compose them into a flake that is compatible with the entire nix flake ecosystem. That's everything that is important to dendritic nix at least, and the parts I've adopted so far.

The main impact of flake-parts on a repo focused on defining nixosConfigurations is that it gives you freedom to structure the files that compose your flake however you see fit. I personally just took all of my top-level flake attributes and put them in a [flake-parts directory](https://github.com/hyperparabolic/nix-config/tree/main/modules/flake-parts). Nothing too wild yet. My current use is just smaller files that indicate their enclosed feature with a name, and a `perSystem` attribute that hides `forEachSystem` boilerplate.

In other contexts, in particular reproducible software development, I think flake-parts has a lot more utility. [flake-parts.flakeModules](https://flake.parts/options/flake-parts-flakemodules) can define flake-parts modules that can be imported into an external flake, enabling flake level code re-use. [Partitioning](https://flake.parts/options/flake-parts-partitions.html) also your flake allows you to separate development dependencies so that they are only loaded for development related attributes.

[flake-parts.modules](https://flake.parts/options/flake-parts-modules.html) is another optional feature of flake-parts. `flake-parts.modules` creates a `flake.modules` attribute that publishes [nixos modules](https://nixos.wiki/wiki/NixOS_modules), and this module is what enables...:

## dendritic nix

<mark>Naming kind of sucks here.</mark> You might need to re-read some of that. `flake-parts.flakeModules` is the import that enables reusable *flake-parts modules*, which are flake level definitions imported by flake-parts. `flake-parts.modules` is the import that enables `flake.modules`, which are `nixos modules` imported as a part of your *nixosConfigurations*.  `flake-parts.module` != `flake-parts module`

The whole idea behind dendritic nix is to make every file in the repo a flake-parts module, and utilize `flake.modules` to define all of your nixos, home-manager, and / or darwin nix files as modules. No new dependencies or anything, it's just a usage pattern for flake-parts. This idea has a couple really neat features:

### Files are not specialized

My old config had at the flake level:

- Imports that were nixos modules
- Imports that were shared nixos modules
- Imports that were shared home-manager modules
- Imports that were functions that return an attribute set
- Imports that were plain attribute sets

Most had slightly different ways they had to be defined. Some of them have different import syntax. All of them have specific places they must be imported.

With dendritic nix, every file is a flake-parts module, and they can all be imported in a single import statement with no differentiation.

### A file can contain any number of modules and a module can be split into any number of files

A file can contain any number of modules, and any of those modules can be imported separately. Modules can be defined across multiple files, and still gets imported with a single statement without having to manage a hierarchy of imports.

I find this especially nice for configuration that doesn't live strictly in a nixos or home-manager context. Those modules can be co-located in the same file. Files can be organized strictly by configuration concern or feature without having to worry about import hierarchy, and without sacrificing using nixos and home-manager independently. Features can be freely split into separate files as a feature grows, or split into variant modules in a single file if necessary.

### Automatic importing

Due to the features above, most people who use this pattern set up automatic importing of modules. [vic/import-tree](https://github.com/vic/import-tree) is commonly used to recursively import all files in a directory in the flake's top-level import. These auto-import features come with semantics like ignoring files prefixed with an underscore (_) to prevent importing a file if necessary.

### Easy access to context outside the module 

Modules co-located in a single file have access to a shared `let in` statement. Additionally, every attribute of the top-level flake can be accessed in any module, without having to jump through hoops passing them through [specialArgs](https://wiki.nixos.org/wiki/NixOS_modules#Passing_custom_values_to_modules). This includes any self-defined attributes, allowing flake level options. Most dendritic repos I've seen define `flake.meta` as a freeoform attribute set, and use that for defining global options for every system defined in the flake.

### Modules are a part of outputs

Modules aren't useful outside of your config unless you structure them to be useful outside your config, but everything is exported as an output without having to jump through hoops of defining `nixosModules` or `homeManagerModules` at the flake level.

## My migration

My flake relies on services that are defined in my flake. Downtime of my nix-serve cache would slow down builds considerably. Downtime of my internal hydra, notification, or observability tooling would mean manually inspecting servers for health.

I'm also quite attached to my git history. When your files have to be organized by import strucutre, looking at commit history can be the best way to hunt down all of the config that is associated with a change. I also don't keep around a lot of cruft, so it's the easiest way to get a hold of past iterations on an idea.

Between these, I knew I couldn't do a migration from scratch and had to do an incremental migration.

Just about all of the commits associated with the migration are denoted with `feat(dendritic)` and most larger changes happened in PRs. If you want to see what this looks like in a [fairly extensive repo that I've been iterating on for two years now](https://github.com/hyperparabolic/nix-config), the migration commits are easy to find there. It took about 80 commits spread across about 3 weeks. I wasn't in a huge rush, and I was never blocked from making other changes by the migration. It would be a lot more complicated and likely justify more concentrated effort in a repo with regular merge conflicts.

### Start with structure

It doesn't take much to start the incremental migration. My first step was transitioning to `flake-parts`. `flake-parts` has a [flake](https://flake.parts/options/flake-parts.html#opt-flake) option that just accepts raw flake output attributes.  I took my entire flake and provided it to (flake-parts.lib.mkFlake's flake attribute unedited)[https://github.com/hyperparabolic/nix-config/pull/31/commits/8fcc90905676cf0fe29fb36eed0884220698dfda].


The next step is (setting up automatic importing of a modules folder)[https://github.com/hyperparabolic/nix-config/pull/31/commits/50bef40b2a27e6c36dca543e3c5dbe2504e09d1a]. At this point you have a lot of modules that aren't dendritic, but that's really it. In my [first dendritic pr](https://github.com/hyperparabolic/nix-config/pull/31) I migrated most of my flake (exceptions for pieces I wasn't sure I would keep), and a small sampling of default and opt-in nixos and home-manager modules to make sure I liked the structure I was setting up.

I did write my own automatic importing. I didn't know if I would run into anything weird during the migration, so I wrote my own that I could freely adjust if needed. At the end I landed on this:

```nix
...
  outputs = {
    self,
    flake-parts,
    ...
  } @ inputs: let
    inherit (inputs.nixpkgs) lib;
    importNixFiles = dir:
      lib.filesystem.listFilesRecursive dir
      |> builtins.filter (f: lib.hasSuffix ".nix" (builtins.toString f));
  in
    flake-parts.lib.mkFlake {inherit inputs;} ({...}: {
      imports =
        importNixFiles ./modules
        |> builtins.filter (f: !lib.hasInfix "/_" (builtins.toString f));
...
```

The `|>` syntax is the [pipe-operators nix experimental feature](https://nix.dev/manual/nix/2.28/development/experimental-features#xp-feature-pipe-operators). You may have seen something similar to this in other functional programming languages. This is mostly syntactic sugar for lib.pipe, which is turn is just syntactic sugar for `foldl'` (`pipe = builtins.foldl' (x: f: f x);`). I find it a bit easier to write and parse personally.

### Preserving git history

Moving files without changing them and then converting to a flake-parts module ensures that `git log --follow` can get history from the old nixos modules. These commits put your repo into a state where it can't be evaluated, but I just made these changes in a PR to ensure my main branch was never in that state.

### Take it piece by piece and compare derivations

Make small incremental changes and validate them before switching to a new config. `system.build.toplevel` is a nixosConfiguration generated attribute that contains your entire system in a single derivation (useful if you want to build your system as a part of a CI/CD process as well!).

```
$ nix build .#nixosConfigurations.<name>.config.system.build.toplevel && nix run nixpkgs#dix /nix/var/nix/profiles/system result
```

If you're on the system you're developing, replacing `<name>` with the name of your nixosConfiguration and running that command will highlight any changes made since you last performed a `nixos-rebuild switch` or booted a new `nixos-rebuild boot` entry. You can also build another system locally, you'll just have to build your own diffing command and keep track of your own links (check out `nix build --help`).

I tried out both [dix](https://github.com/faukah/dix) and [nix-diff](https://github.com/Gabriella439/nix-diff) diffing tools, and generally found `dix` to have the better level of verbosity for me. There are generated elements of your derivation that can change order every build. This is really anything generated using functions like `map`, `genAttrs`, `mapAttrs`, `attrValues`, or similar iteration. For me, this regularly included order-related noise around PATH, flake repositories, man files, and fish autocompletions. You might like the extra verbosity of `nix-diff` or something else if you have less order thrash than me. Try out a couple and see what you like.

### Keep it simple, refactor later

The dendritic pattern gives you a lot of flexibility and expression when it comes to refactoring. I mostly just made my modules equivalent to my existing configuration's import trees. Over time I think I'll move more of my literals to `let in` statements or even options to cut down on host specific config. It feels less tedious to do this with this structure in place.

## Building ideas on this foundation

I don't think these ideas are impossible with nix and flakes alone, but changing the way you write a config changes the way you think about writing a config. It's also certainly less syntactically annoying. I'll list any novel ideas I come up with here:

### Global nixos / home-manager options

A note on options names: `options.this.*` indicates options that defined by this flake and really only intended for consumption by this flake. `options.hyperparabolic.*` indicates something written with flexibility and intention for re-use externally (even if it is defined in this flake).

I really like having useful development tooling. Options that are defined at the flake level are very convenient, but flakes don't have a type system. Anything that isn't defined in the [flake schema](https://nixos.wiki/wiki/flakes#Flake_schema) is treated as structurally unknown, and anything other than `flake-parts` with debug enabled will treat it as structurally unknown. Even those `flake-parts` definitions aren't particularly useful to nixd, as just about everything defined by `flake-parts` is a lazy attribute set. This isn't a bad thing, it's really kind of fundamental to flake-parts. It generally doesn't restrict function or behavior ever.

If you want type checking, asserts, warnings, or LSP support associated with your options, have to put them into your nixos / home-manager modules (or build your own validation and tooling, but no thank you).

My [this module](https://github.com/hyperparabolic/nix-config/tree/main/modules/this) contains options that are just generic config containers that get consumed by other modules. I've come up with an idea for sharing options that can't be cleanly separated to either nixos or home-manager there.

[monitors.nix](https://github.com/hyperparabolic/nix-config/blob/main/modules/this/monitors.nix) is an example of a file that employs this pattern. All of the options are defined in this file are defined in a `let in` statement and defined and validated into both `flake.modules.nixos.this` and `flake.modules.homeManager.this`.

There's a separate module below called `this-share-home`. It has a definition in a separate file `this-share-home.nix`:

```nix
topLevel: {
  flake.modules.nixos.this-share-home = {...}: {
    home-manager = {
      sharedModules = with topLevel.config.flake.modules.homeManager; [
        this
      ];
    };
  };
}
```

and at the bottom of `montiors.nix`:

```nix
{
  flake.modules.nixos.this-share-home = {config, ...}: {
    config = {
      assertions = [
        {
          assertion = (lib.length config.this.monitors) == (lib.length config.home-manager.users.spencer.this.monitors);
          message = "this.monitors: only define this.monitors in nixos module if using this-share-home";
        }
      ];

      home-manager = {
        sharedModules = [
          {
            this.monitors = config.this.monitors;
          }
        ];
      };
    };
  };
}
```

`sharedModules` is some home-manager special sauce. Anything defined in this list is a module that will be imported for all users in `home-manager.users.*`. `this-share-home` is a separate module that will automatically import the corresponding homeManager module, and duplicate options from the nixos level into the homeManager equivalent options. This could also be pretty easily tweaked to only set those options for specific users.

This feels like the lowest sacrifice compromise I've found for this problem so far. You still get to leverage the type system, implement warnings / assertions, and you still get autocompletion support with `nixd`. Since it's opt-in, this doesn't interfere with standalone homeManager use. You can just ignore it and import the homeManager module separately instead and duplicate the options values in both contexts.

## Open issues

[#32](https://github.com/hyperparabolic/nix-config/issues/32)

This isn't really a unique problem to dendritic nix, but I do think it will likely accelerate a problem unique to heavy module use for me.

`nixd` evaluates a specific system's configuration to detect options and provide auto-completion. The dendritic pattern pushes you toward leveraging conditional importing of modules to control what config is included in a system. As you write more modules with options, the odds of having a sytem that includes every module decreases if you have multiple systems. home-manager module options also only show up in `nixd` if they are defined in `home-manager.sharedModules`.

Right now oak is exposed to every option so it's the host I use for [configuring nixd](https://github.com/hyperparabolic/nix-config/blob/main/modules/core/helix/languages.nix#L75) for now, but that might not be the case moving forward.

I might try to make a dummy host that imports every module but never gets evaluated to work around this. Conflicting `mkForce` options would make this idea more complicated, but this could be worked around be replacing `mkForce` with `mkOverride` and twiddling with values close to 50, or defining those options with `mkStrict` in that system's config as they come up. This might be the lowest effort, and it confines weirdness to a single file or directory, but it's finnicky.

I might have to bite the bullet and make all of my modules global imports, and utilize options to enable and disable them at the host level. Certainly a bit more effort, and it requires changes across many modules, but it's elegant.

I'm putting it off until I refactor more and have to make a decision, and I'll mull it over in the meantime. I'll have to see how evaluation performance looks when I get there. Let me know if you have any other thoughts on it. I'd love to hear any of them.


## Thoughts

I had already been considering refactoring more of my config into `nixosModules` and `homeManagerModules`. There's value to being able to tweak behavior with options, and I found this especially useful while incrementally adopting new boot and zfs changes in my zfs nixosModule. The dendritic pattern fills the same want and was easier to implement.

Also, eventually I know I'll run into a situation where I need to use nix as a package manager on another os, and I'll want to leverage this repo and home-manager to keep the familiarity of home setups. Dendritic nix has made it relatively intuitive to clean up some of the ugliest corners of my repo where I've run into pain points around this.

I think repo-level bootstrapping will only be easier. You can add a whole nixosConfiguration with stub configs *anywhere* in the modules directory. Organization into the `hosts` and `flake-parts` module directories is just book-keeping to keep things sane in the long term, but a top level stub module is perfect for something you're going to refactor away anyway like a `_hardware-configuration.nix`. Those scripts won't need to be refactored next time I decide I want to reorganize differently.

I like it overall. I think there's some nice flexibility that should help adopt some ideas I've been toying with (SDN and private config partitions in particular). I'm less annoyed with new problems than problems I had before, and I don't think it's just new problem smell. It's not a revolution, but a different pattern makes you think about your problems differently, and I think this particular pattern makes it a smaller lift to write better modules.

## Links

Rewriting things into my own words helps me refine and retain thoughts, but a lot of this is a re-implementation of other people's thoughts and ideas.

Here are some of the places I learned from and drew inspiration from:

- https://github.com/mightyiam/dendritic (especially "Usage in the wild")
- https://vic.github.io/dendrix/Dendritic.html 
- https://tsawyer87.github.io/posts/top_level_attributes_explained/

The community in the matrix channel listed in the dendritic repo is super nice and enthusiastic about sharing ideas.

Thank you for all of them.
