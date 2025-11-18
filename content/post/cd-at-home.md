---
title: Continuous Delivery At Home With NixOS
date: 2025-08-25
author: Spencer Balogh
description: Continuous integration and continuous delivery for my home lab
tags:
  - NixOS
  - CI/CD
  - Hydra
toc: true
---

I've fully realized the ["What's next?"](https://blog.decent.id/post/lower-compromises-zfs-encryption/#whats-next) section at the end of my ZFS encryption post, and I now have a full continuous integration / continuous delivery solution for my home lab. :tada:

There were a few humps to get over, many of them self imposed, but I wanted to document them and some thoughts around decisions I made.

<!--more-->

## Self imposed challenges

- Drive encryption everywhere
- Authenticate with hardware devices where possible
- Limit authorization where hardware authentication isn't possible
- No ingress from the public internet (this isn't actually a hard rule, I'll probably start using Cloudflare Tunnels once I have a good enough reason to, but nothing about his project so far or anything I've deployed requires them).

This isn't some sort of perfect security architecture. Especially physical security is pretty lax. I do delegate access to my GitHub account to software or systems that I don't own though, so code signing with a hardware key ensures people who aren't me cannot modify my infrastructure as code and get those changes automatically deployed (other than automatic updates that I would do regardless). Similar protections are placed on my user on my systems to limit damage if vulnerable or untrustworthy software is installed. It happens, even without making mistakes.

## Breaking it down

This is all the interaction of many ideas and systems, building on top of existing systems. This is just info on recent changes that got this across the finish line.

### Unattended reboots

AFAIK, NixOS doesn't have support for any live kernel patching, and I wouldn't really want to do it anyway outside a use case that demands it. So reboots are necessary to adopt kernel and systemd updates.

I had been doing remote ZFS dataset decryption over ssh. I wanted something automatic without storing plaintext ZFS keyfiles in my boot drive, so I [started storing a LUKS volume in my ZFS datasets that stores the dataset's encryption keyfiles](https://blog.decent.id/post/lower-compromises-zfs-encryption/) to move in that direction. The following enable unattended reboots on encrypted systems.

#### Secureboot

Newer hardware generally has support for this. I know some of my systems didn't initially support secureboot, or had limited support for specific operating systems. Over time they've all released firmware with support for enrolling keys, luckily.

[Lanzaboote](https://github.com/nix-community/lanzaboote) provides nix-community supported secureboot for NixOS.

Enabling secureboot with Lanzaboote is fairly straightforward. [sbctl](https://github.com/Foxboron/sbctl) is used to provision boot signing keys with `sbctl create-keys`. Once these keys are generated, enabling the lanzaboote nixos module and rebuilding will sign all of your EFI images. You can validate it's done with `sbctl verify`.

After this, you need to get your bios into secureboot setup mode. Once you do this, boot the system and enroll your self signing keys with `sbctl enroll-keys`. Several of my systems had to be poked one more time in the bios to enforce signing requirements at this point.

Up to date specifics can be found in the lanzaboote [quick start guide](https://github.com/nix-community/lanzaboote/blob/master/docs/QUICK_START.md).

<mark>On sbctl hardware keys:</mark> TPM shielded keys are available now, and yubikey support is coming soon as the project pushes toward the 1.0 release. However, I don't think they're really worth it in this case.

TPM shielded keys don't currently support password protection, and are limited to RSA2048 (unlike the file keys that utilize RSA4096).

Yubikey signing will support PIN protected RSA4096 keys via PIV, but I'm skeptical of the value of this even.

I want unattended updates, so any passwords or PINs would be stored in an environment file, so this is really just offering a layer of misdirection at best. The root user that can access root owned keys can access that environment file just the same. Unique keys are being provisioned for every machine as well, and they are only used for secureboot signing, so preventing exfiltration is low value as well.

I don't think using a hardware key here is really valuable unless you're:
- requiring human interaction
- implementing some sort of policy based protection around PINs / passwords
- or doing centralized image signing and enrolling the public keys on multiple machines

#### TPM drive decryption

`systemd-cryptenroll` is used to manage enrolling unlock methods for LUKS encrypted volumes. These volumes can have multiple unlock methods enrolled at the same time, and I'm using TPM devices specifically for unattended unlocks.

TPM devices are useful for this specific use case because they can enforce Platform Configuration Register (PCR) state. These can be used to ensure that your platform state hasn't changed or been tampered with. My specific enrollment looked like:

`systemd-cryptenroll --tpm2-device=auto --wipe-slot=tpm2 --tpm2-pcrs=7+12 <device>`

`--tpm2-device=auto` is fine as long as you don't have multiple TPM devices. I specifically chose to validate against PCR7 and PCR12.

PCR7 validates the system's secureboot state. This ensures that enforcement stays enabled, and the enrolled keys do not change.

PCR12 validates the system's kernel configuration. I allow kernel parameters to be dynamically set (I use this for systemd stage-1 debugging), but automatic unlocks are disabled when parameters are changed.

Info on all of the available PCRs can be found in the [Linux TPM PCR Registry](https://uapi-group.org/specifications/specs/linux_tpm_pcr_registry/).

<mark>Room for improvement</mark>:

I would use PCR4 (Boot loader and additional drivers; binaries and extensions loaded by the boot loader) and PCR11 (All components of unified kernel images (UKIs)) if it were easier to do so. The main issue is that these PCR values change with every generation, and you would need a way to pre-compute the register values (likely `systemd-measure`). All of this along with a repeat of `systemd-cryptenroll` would have to be included in your rebuild process. Something like this is included in the 2.0.0 release milestone of lanzaboote. It'll be a bit. I'll have to look through the milestones and see if there's anything I think I could take a swing at.

I don't think I'm concerned with hardware / system firmware specific PCRs (0-3, 5) for my use case. If I were more worried about physical tampering I'd probably consider more of these along with `dm-verity` to try to make truly immutable systems, but it's a lot of work and just overkill here.

### Hydra CI and build server

I deployed [hydra](https://github.com/NixOS/hydra) recently. It's [pretty easy to configure](https://github.com/hyperparabolic/nix-config/blob/e7cfd6cbcf09c4a66fa6340caf03c36c43835196/hosts/oak/services/hydra.nix), not too different than any other web service on NixOS.

Once this was running, I needed to create a hydraJobs output attribute in my flake that builds the toplevel derivation for each of my hosts:

```nix
...

  outputs = {
...
    hydraJobs = {
      hosts =
        lib.mapAttrs (_: cfg: cfg.config.system.build.toplevel)
        # I have a custom installation / debugging usb media in nixosConfigurations, don't build this in hydra
        (lib.attrsets.filterAttrs (n: v: !builtins.elem n ["iso"]) outputs.nixosConfigurations);
    };
...
```

There were a couple places I was importing from derivations in my nix config, which isn't great practice. IFD isn't permitted in hydraJobs during `nix flake check`, so this is the push I needed to finally get that cleaned up, and I've got guardrails in place to make sure it can't sneak back in again.

All of those toplevel derivations get evaluated on my hydra server. A **lot** of config is shared across multiple hosts. This ensures that any config or packages that aren't pulled from an upstream cache are only built once. I have a [nix-serve](https://github.com/edolstra/nix-serve) cache on the same host, so most updates are just a bunch of local network downloads.

This is also just a great platform to build on. `testers.runNixOSTest` is powerful test runner for complex integration tests where multiple machines interact. It's also a relatively small change to introduce new buildMachines to this setup to distribute building and testing or to introduce new systems for native multi-arch builds in the future.

### Automatic system upgrades

I ended up writing my own system upgrade script - [nixos-hydra-upgrade](https://github.com/hyperparabolic/nixos-hydra-upgrade).

There are a couple reasons for this. I didn't want multiple systems downloading and building the same code, so this upgrader polls my hydra instance and only upgrades to derivations that have successfully built there. It also has support for scheduling based on systemd.timers, and can perform health checks as a part of a system upgrade. All of this is configurable through a NixOS module. I ended up writing it in go. There are enough moving parts here that I wanted something easier to debug than a shell script.

With all of this, I have upgrades being rolled out incrementally. If some sort of change prevents systems from gracefully updating or rebooting then it'll more than likely just be my audio receiver MPD kiosk, and not my local dns. :sweat_smile:

I haven't had anything break this way yet, but those might be good candidates for tests if I want to prevent the same regression.

I have some ideas for improving this, and I'll probably build out update notifications next. Drop an issue in the repo if you have any other ideas.

### Automated repo updates

I wrote some [workflows](https://github.com/hyperparabolic/nix-config/blob/main/.github/workflows) for keeping dependencies up to date in my nix-config repo. In particular [flake-update.yml](https://github.com/hyperparabolic/nix-config/blob/main/.github/workflows/flake-update.yml) updates all of the flake inputs, performs a `nix flake check` on the updated inputs, and then opens a PR and merges it if that check is successful.

<mark>This is a little clunky</mark>. With different constraints I would prefer to implement this idea using a [GitHub App](https://docs.github.com/en/apps). I think they have a better access control model than PATs. However I'm building with a small amount of infrastructure, and no ingress from the public internet prevents the use of webhooks. These factors pushed me toward automating with GitHub Actions and accepting the clunkiness below.

It seems like GitHub really doesn't want you automatically writing and approving code changes without human interaction using GitHub Actions. However, I don't really have any meaningful input on these changes. If I'm updating manually, I might not even run a `nix flake check` before attempting to build the new generation, and these changes are going to go through my entire CI process before being deployed. So we got creative to work around this.

I enforce code signing on my nix-config repo. I can't even bypass this one as the admin. Consequently the author of any automated commits also needs to sign them. I don't really want any software protected keys associated with my personal account floating around, so this pushed me toward making a [bot account](https://github.com/hyperparabolic-bot) specifically for signing.

"Fine-grained" PATs don't support cross organization access yet, so I had to use classic PATs, which could be used to modify any files in repos the account is a contributor in. I don't really love this, so I started looking into ways to implement file level access control and landed on CODEOWNERS:

```
* @hyperparabolic

/.github/ @hyperparabolic 

/flake.lock @hyperparabolic @hyperparabolic-bot @hyperparabolic-bot-bot
```

That way, hyperparabolic-bot is only permitted to modify `flake.lock`. However, CODEOWNERS doesn't actually do anything without an associated branch rule. CODEOWNERS can be required to review review a pull request though. Enter the clunk: an author can't review their own PR, even if they are a codeowner, so I ended up making a second account just for approving my first bot's PRs. This ACL and approval process is the part that might be better suited with a GitHub App under different circumstances. It seems a bit more flexible for permissions, and checks on PR contents could be implemented as code, but this is where I landed on this pass.

![PR participants are images of Timmy and Tommy from Animal Crossing](/files/participants.png)

It's a little gross, but it works. On the bright side, I have these two Updating my flake.lock file ˡᵒᶜᵏ ᶠᶦˡᵉ...

On the off chance the `nix flake check` fails, the process stops short and I have an open PR with a review request alerting myself to it. Great.

## That wraps it up

My servers were already low maintenance. This brings them down to nearly zero maintenance. It's also a nice foundation to build other projects on top of. I'm happy with where it's landed overall.
