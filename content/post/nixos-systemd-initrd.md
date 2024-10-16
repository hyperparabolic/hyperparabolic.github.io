---
title: systemd initrd on NixOS
date: 2024-10-14
author: Spencer Balogh
description: Notes on transitioning to systemd initrd on NixOS, and debugging systemd during boot.
tags:
  - NixOS
  - Systemd
  - Boot process
toc: true
---

I recent transitoned to using `boot.initrd.systemd.enable = true` on NixOS. It wasn't terrible to figure out, but I did have some migrations to figure out for my boot process customizations. This is the blog post I wish I had to read first regardless :sweat_smile:.

For many system configurations, transitioning to a systemd based initrd process will *just work* with the one config line above. This post gets into config patterns that typically require migration, what my personal migration looked like, and advice for debugging boot processes for any migration.

<!--more-->

<mark>A few words of caution:</mark>
- Systemd based initrd is not feature complete yet. Not every system is going to be fully supported yet. [Project tracking for this feature can be found here](https://github.com/orgs/NixOS/projects/66). My general experience with it at this point is that it's stable enough for daily use, but there are features that aren't implemented, and there are open and unknown bugs.
- I've only gone through this process with systemd-boot, and unified kernel images. I'm not sure if there are any missing details here for GRUB or legacy boot users. 

Maybe don't implement this one in production unless you're *adventurous* and have good health check and rollback processes.

## What is this anyway?

How do you mount a disk without linux available to `mount` it? Your bootloader is probably executing an EFI executable that mounts an in memory file-system used for bootstrapping, probably with initramfs (these aren't hard and fast rules, older systems without UEFI would still be relying on legacy boot, and these are the people the note above is warning).

The default boot process in NixOS implements this as a procedural script. Optionally, you can utilize `boot.initrd.systemd` and its associated options to replace this default boot process with a systemd based initialization process. If you want to know more details about this process, you can execute `man bootup` or look at that document [here](https://www.freedesktop.org/software/systemd/man/latest/bootup.html).

### Why is this anyway?

Along with all of the complexity of systemd, you get all of the complexity of systemd :wink:. 

Implementing customizations requires knowledge of implementing systemd services, but you get to leverage this as well. Systemd imlpements an interface for manging complex dependencies between services. Generally, more processes are able to occur concurrently without explicitly implementing concurrency. Systemd may also abstract away a lot of complex implementation. For example, systemd-networkd trivializes management of network interfaces during startup time, and processes that must implement polling (external network dependencies as an example) may implement polling processes using systemd (`Restart*` and `StartLimit*` service config).

This doesn't necessarily mean that boot processes are definitely faster. There is overhead to this concurrency management. For me, the limiting factor is usually disk decryption regardless. The bootup time isn't meaningfully long compared to decryption password entry. I personally am switching to this to leverage dependency management in boot process customizations in the future.

Another more obscure benefit is that a lot more state from stage-1 survives the transition to stage-2. I haven't found any use for this, but it's interesting and potentially useful.

## When is migration required?

If you customize your stage1 boot with scripts, those scripts need to be migrated to systemd services. In the current default boot, typically these scripts are inserted using `pre*Commands` and `post*Commands`. These allow the insertion of scripts before or after a few distinct phases of the boot process like LVM discovery, mounting `/dev/*` device nodes, mounting filesystems, network initialization, or opening LUKS devices.

This search query doesn't find those options exclusively, but most of these options are listed at the top of [this NixOS options query](https://search.nixos.org/options?channel=unstable&from=0&size=50&sort=relevance&type=packages&query=boot.initrd+pre*commands+post*commands).

## Debugging systemd scripts

There are a lot of systemd units that are always going to be present on every system. A lot of these are documented in `man bootup`. However, there are also services that get generated based on your hardware. Systemd services that replace scripts may need to reference these services. I found it was easiest to find out more about these services by just querying them on a system during the boot process.`systemctl` can be used just like on a live system to explore systemd units from the boot process. If you aren't familiar with this, try `systemctl status` and `systemctl list-units`.

<mark>Do your own risk assessment.</mark> You may decide that you don't want to leave these debugging holes open all the time.

### Remote debugging with SSH

This was the easiest way to debug for me personally. I was exploring one booting system while referencing docs and writing config on another. This is really easy if you have a drive decryption step, since boot will already wait at the password prompt. If your system boots successfully and you need to debug, then you may need to write a service that intentionally fails.

#### Network config:

<mark>EDIT:</mark> This seems to just be a regression? It is intended that setting `boot.initrd.network.enable = true;` (notice, not `boot.initrd.systemd.network.enable`) should automatically configure the intird network based on `networking.interfaces.*`. Try that before this.

Network config changes slightly with a systemd based initd. Network interfaces don't automatically get set up based on your `networking.interfaces.*` config, and you will instead need to write a systemd networkd based config. Here's a simple example of what that might look like:

```nix
  boot = {
    kernelModules = [
      # you may need to enable drivers for your network device here like "igb"
    ];
    initrd = {
      kernelModules = [
        # you may need to enable drivers for your network device here like "igb"
      ];
      systemd = {
        enable = true;
        network = {
          enable = true;
          networks.<your network device name here> = {
            enable = true;
            name = <your network device name here>;
            DHCP = "yes";
          };
        };
      };
    };
  };
```

The name of your device can be found by running `ip addr`. Wired interfaces will have names like `enp#s#`. Additional config is required if you're using wifi or your network requires authentication. The config here supports the same features as `systemd.networkd`, and more details on that config can be found [on the NixOS wiki](https://wiki.nixos.org/wiki/Systemd/networkd).

#### SSH config:

This part is identical to ssh config in default boot.

`boot.initrd.secrets` inserts secrets files into the boot process. This is being used to copy a host key into the boot process. This key file can be generated with `ssh-keygen`, and you should really be sure that this file has appropriate permissions (read only access, and only for the root user directly). Replace `/secrets/boot/ssh/ssh_host_ed25519_key` in the sample below with whatever key flavor and location you end up using.

I'd also strongly recommend using a non-default port here to avoid your ssh clients warning you about host key changes.

```nix
  boot = {
    initrd = {
      secrets = {
        "/secrets/boot/ssh/ssh_host_ed25519_key" = "/secrets/boot/ssh/ssh_host_ed25519_key";
      };
      network = {
        ssh = {
          enable = true;
          port = 2222;
          hostKeys = [/secrets/boot/ssh/ssh_host_ed25519_key];
          authorizedKeys = config.users.users.<your user>.openssh.authorizedKeys.keys;
        };
      };
    };
  };
```

Once this is all configured, you can ssh into the booting system with `ssh -p 2222 root@<ip address>` and query with `systemctl`.

#### Alternatives: debug kernel parameters

Debugging can also be done on a single system. Use of these kernel parameters requires setting `boot.initrd.systemd.emergencyAccess` (either true for unauthenticated access, or set a password hash).

`rd.systemd.unit=rescue.target` forces your system into rescue mode in stage 1.

`rd.systemd.debug_shell` starts a debug shell on tty9 (<kbd><kbd>ctrl</kbd> + <kbd>alt</kbd> + <kbd>F9</kbd></kbd>).

## Gotchas

There are a few differences in the systemd initrd shell environment compared to the default one. I'll mention a few of them here. You could switch to the older defaults, or just be mindful of the changes.

### Different tooling in the default shell

A lot of the process management tooling that was previously available is no longer included. Everything is a systemd unit in the systemd initrd, so you can just use `systemctl` for process management instead.

I found I didn't need it, but if there is some binary you require in your initrd environment, `boot.initrd.systemd.storePaths` allows you to specify individual binaries or entire packages to copy to your initrd environment's `/nix/store` fo use in systemd services, or `boot.initrd.systemd.initrdBin` and `boot.initrd.systemd.extraBin` can add binaries to your debug shell's `PATH`.

### Different home directory for the root user

The root user's home directory moved from `/root/` to `/var/empty/`.

## What migration was required for me?

I had two different post commands that needed to be replaced. I used `boot.initrd.postResumeCommands` to trigger my ZFS rollbacks for impermanence, and I used `boot.initrd.network.postCommands` to populate my initrd root user's `.profile` with commands to decrypt my ZFS encryption dataset, and kill the local decryption command (`zfs load-key rpool/crypt; killall zfs`).

Both of these were pretty easy to replace with systemd oneshot services. The only tricky part was finding where they should fit in the systemd dependency graph to get executed at the correct time.

### Impermanence rollback

I utilize ZFS rollbacks to drive my [impermanence](https://github.com/nix-community/impermanence) setup. I might post about this at some later point, but the general idea is that your filesystem is ephemeral, and you use impermanance to explicitly persist files between reboots. These persisted directories act similarly to docker file volumes, and eliminate config drift outside of when you explicitly allow it.

This rollback script used `boot.initrd.postResumeCommands` before. The service config replacement for this looks like:

```nix
  boot.initrd.systemd.services.zfs-rollback = mkIf (cfg.enable && cfg.rollbackSnapshot != null) {
    description = "Rollback ZFS root dataset to blank snapshot";
    wantedBy = [
      "initrd.target"
    ];
    after = [
      # this is a dynamically generated service, based on the zpool name
      "zfs-import-rpool.service"
    ];
    before = [
      "sysroot.mount"
    ];
    path = with pkgs; [
      zfs
    ];
    unitConfig.DefaultDependencies = "no";
    serviceConfig.Type = "oneshot";
    script = ''
      zfs rollback -r ${cfg.rollbackSnapshot} && echo "zfs rollback complete"
    '';
  };
```

### ZFS remote password decryption

This script used to populate the root user's `.profile` during `boot.initrd.network.postCommands`. The service config replacement for this looks like:

```nix
  boot.initrd.systemd.services.zfs-remote-unlock = {
    description = "Prepare for ZFS remote unlock";
    wantedBy = ["initrd.target"];
    after = ["systemd-networkd.service"];
    path = with pkgs; [
      zfs
    ];
    serviceConfig.Type = "oneshot";
    script = ''
      echo "systemctl default" >> /var/empty/.profile
    '';
  };
```

This queries for a ZFS password automatically upon `ssh` into the initrd. Most of these decyrption prompts are now using `systemd-ask-password` for password prompts, so `systemd-tty-ask-password-agent` can be used to hook into that prompt, or just `systemctl default`.

## Let me know how it goes for you

I'd happily add anybody else's lessons learned here as well.

Thanks to [ElvishJerricco](https://discourse.nixos.org/u/ElvishJerricco), the primary author of stage 1 systemd boot for feedback on this article.

## Future plans

This gives you the ability to script anything, at any point during the boot process

I don't know if I'll make this migration proactively or just passively whenever I image new systems here from now on, but I plan to use this to make my drive encryption strategies consistent across all of my machines. Right now my desktops utilize ZFS native encryption. My laptops that leave the house use ZFS on top of LUKS for better key management options. I want to try to use LUKS on a ZVol to store file encryption keys for my encrypted datasets instead, and this should be usable to script that unlock process. This should give me all of the key management and unlock options of LUKS, the performance of native ZFS encryption (scrubs are slow on ZFS on LUKS), and encrypted `zfs send` / `zfs recceive` backups. We'll see how it goes when I get there.

