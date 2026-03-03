---
title: Windows VM Performance and Usability Tweaks
date: 2026-03-02
author: Spencer Balogh
description: Notes on performance oriented VMs on NixOS.
tags:
  - NixOS
  - libvirt
  - virtualization
toc: true
---

This post is a retrospective on how I've been running Windows VMs on NixOS. It contains a little bit of what and why around how I've tweaked my NixOS config, my libvirt configs, and my vm runtimes. It's more notes for me and assumes familiarity, but feel free to ask about anything in the comments.

I've been daily driving linux for a long while, but I've kept windows around in a VM for music software and games. I've also been thinking more about retiring my windows VM lately.

There's a fair bit of cost to it. Getting near-native performance in a VM requires a separate video card from your host (or unloading your video card's kernel modules, unbinding the EFI framebuffer on your host, and maybe ROM flashing depending on your card). Still cheaper than a separate PC, but not cheap, especially if you buy a motherboard that doesn't cheap out on iommu groups.

Also, when I started exploring this option for running windows proton didn't exist, and I wasn't as happy with options for managing wine applications. Running in a VM generally gave much better performance and compatibility than running software through wine.

Over time the difference has shrunk pretty drastically. More DRM has started explicitly disallowing running in a VM. Valve and the open source community around proton have done amazing work. The cost hasn't shrunk, and issues around power and heat management have only gotten worse as cards have become more power hungry and blower style cards have been phased out.

It's a balancing act. I definitely will lose some sandbox benefits if I go through with it, but I have also been more interested in hardware acceleration on my host over time. It's just become a lot less obvious of a decision over time for me.

Anyway though, this is just to collect all of the ideas I've incorporated in one place, since many of them don't fit cleanly in my nix config, and I might deprovision it if I work out a few emulation workflows to a happy place.

<!--more-->

## Documentation Sources

- https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF
- https://wiki.nixos.org/wiki/PCI_passthrough
- https://wiki.nixos.org/wiki/Libvirt
- https://passthroughpo.st/
- https://www.evonide.com/non-root-gpu-passthrough-setup/

The arch and nixos wikis are really up to date on these. Passthrough Post and the evonide posts are a bit more out of date, but a lot of core ideas are still there. Especially if you need to find information on hardware compatibility (iommu groups, cache architecture, etc), these are some of the places I'd check.

I'm sure there are other forum posts or blog posts I should credit, but the web is fleeting.

## Constraints

- I run qemu as a non root user.
- I run pipewire as a user session.

I'll get into details for specific places where I make custom changes to get around this. Otherwise, using libvirt with polkit generally resolves permissions issues around this. If you're running into situations where you're adding users to overly permissive extra groups or modifying device permissions with udev, you're probably making things much harder than you need to.

## Compatibility Tweaks

### Advertise Hardware

VRChat's docs has recommendations for being a good client in a VM with regards to its DRM [here](https://docs.vrchat.com/docs/using-vrchat-in-a-virtual-machine).

The general guidelines are to advertise your manufacturer's metadata, bios metadata, and vendor identifiers directly as they appear on your host machine. Not super complicated, and I found that for applications that are permitted to run in VMs this guidance kept everything happy. All of this information can be found using `dmidecode`, and stored in the `sysinfo` libvirt config.

I don't mess around with kernel level anti cheat even in my VMs. I found this alone was good enough, and I didn't have to get into other ideas people have posted like registry edits, spoofing hardware serials, or anything else.

### VM CPU Config

```
  <features>
    <acpi/>
    <apic/>
    <hyperv mode='passthrough'>
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
    <vmport state='off'/>
    <smm state='on'/>
  </features>
  <cpu mode='host-passthrough' check='none' migratable='on'>
    <topology sockets='1' dies='1' clusters='1' cores='8' threads='2'/>
    <cache mode='passthrough'/>
    <feature policy='require' name='topoext'/>
  </cpu>
```

I've heard hidden kvm state isn't even necessary with newer nvidia drivers, but I haven't bothered trying to turn it off.  I generally found passthrough mode on hyperv as good as any customized config.

### Fix the Damn Clock

```
  <clock offset='localtime'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
    <timer name='hypervclock' present='yes'/>
  </clock>
```

Windows expects local time from your hardware, not UTC.  I found fixing things here cleaner than trying to change anything on the linux side.

## Performance Tweaks

### VFIO Passthrough

The general idea behind this is that you pass through your entire video card (or other PCI slot hardware) to your vm for direct use.  This requires some extra care with video cards.  They aren't intended to be hot swapped, and the drivers assume they are loaded in a clean state.

Just about everything related to this is enabled through kernel modules, and configured through kernel parameters. A VFIO stub driver is attached to the video card before the vendor drivers can be loaded, ensuring the VM receives things in a clean state.

Kernel modules:
- `vfio_pci`
- `vfio`
- `vfio_iommu_type1`

Kernel parameters:
- `amd_iommu=on` (or intel)
- `vfio-pci.ids=<ids, comma delimited>` (ID has the form ####:####, can be found with `lspci -nv`)

As long as the drivers are stubbed, even the default gui config of PCI passthrough is fine.

### CPU Pinning

This one is a two part solution.  The first half is a QEMU hook to utilize `systemd.resource-control` (`man systemd.resource-control`) to restrict your host OS to specific CPU cores:

```bash
#!/usr/bin/env sh

guest=$1
command=$2

if [ "$guest" != "win11" ]; then
  echo "cpu-isolate-win11.sh ignoring other guest"
  exit 0
fi

if [ "$command" = "started" ]; then
  echo "cpu-isolate-win11.sh setting up host cpu isolation"
  systemctl set-property --runtime -- system.slice AllowedCPUs=2-3
  systemctl set-property --runtime -- user.slice AllowedCPUs=2-3
  systemctl set-property --runtime -- init.scope AllowedCPUs=2-3
elif [ "$command" = "release" ]; then
  echo "cpu-isolate-win11.sh restoring host cpu access"
  systemctl set-property --runtime -- system.slice AllowedCPUs=0-4
  systemctl set-property --runtime -- user.slice AllowedCPUs=0-4
  systemctl set-property --runtime -- init.scope AllowedCPUs=0-4
fi
```

The #-# on AllowedCPUs is just an example of some made up 4 core system without hyper-threading. `lscpu -e` can be used to inspect your CPU cores and cache topology. The goal is to pick cores so that your VM and host never share the same L1/L2/L3 cache. This improves performance by reducing cache misses, and can reduce the ability to analyze your host through side-channels.

Libvirt has the other half of this config:

```xml
  <vcpu placement='static'>16</vcpu>
  <iothreads>1</iothreads>
  <cputune>
    <vcpupin vcpu='0' cpuset='0'/>
    <vcpupin vcpu='1' cpuset='1'/>
    <emulatorpin cpuset='0'/>
    <iothreadpin iothread='1' cpuset='0'/>
  </cputune>
```

Again, a fake example for a 4 core system without hyper-threading, but you need pin entries for all of the cores that are left available after the qemu hook.

On my machine, L1 and L2 cache are exclusive to each core, and the L3 cache is shared by groups of 4 cores.  It's easy enough to isolate host and guest cache.  If your L3 cache is shared by more cores, you may want to emulate your L3 cache (`<cache level="3" mode="emulate"/>` in cpu config).

## Usability, Tricks, Workflow

### Sharing a Pipewire User Session

Sharing a pipewire socket directly with a VM has been by go-to solution for audio since I started using pipewire.  It handles both sinks and sources.  I've never had noise issues.  It can leverage the [effects chain of my host OS](https://github.com/hyperparabolic/nix-config/blob/8ad9bd2e4ba42a7894aab1a958b6fa4bc87eb092/modules/desktop-applications/easyeffects/input_filter_voice.json).  The audio quality is indistinguishable from a hardware passthrough as far as I can tell. It feels no compromises once you run through the setup steps.

Running as a non-root user, I couldn't get libvirt to accept a pipewire runtime directory that isn't owned by my libvirt user. I also use a pipewire user session, so my pipewire runtime directory is `/run/user/1000`, which I'd really prefer not to share the entirety of with my libvirt user. The best solution I found for this is setting up a bind mount, creating a replica of the socket for another user. I wanted systemd to manage this, so it would just be automatically created as a part of my libvirt service.  Unfortunately you can't make dependencies between systemd system and user sessions, so there's a little bit of waiting and polling, but it has worked without issue for me.

```nix
...
    systemd = {
      services = {
        prepare-run-qemu-libvirtd = {
          description = "Create run directory for qemu-libvirtd";
          wantedBy = ["libvirtd.service"];
          after = ["libvirtd.service"];
          script = ''
            mkdir -p /run/libvirt/qemu/pipewire
            chown -R qemu-libvirtd:qemu-libvirtd /run/libvirt/qemu/pipewire
            chmod 770 /run/libvirt/qemu/pipewire
          '';
          serviceConfig = {
            RemainAfterExit = "yes";
          };
        };
        wait-pipewire = {
          description = "Wait for spencer user pipewire socket";
          wantedBy = ["user@1000.service"];
          after = ["user@1000.service"];
          serviceConfig = {
            TimeoutStartSec = "infinity";
            # ConditionPathExists can't test the socket directly, since you're expected
            # to put a dependency on pipewire.socket instead. That lives in the user
            # session though so that dependency can't work. This works fine.
            ConditionPathExists = "/run/user/1000/pipewire-0.lock";
            RemainAfterExit = "yes";
            ExecStart = "${lib.getExe' pkgs.coreutils "sleep"} 1000";
          };
        };
      };
      mounts = [
        {
          description = "Share pipewire user socket with libvirtd via bind mount";
          what = "/run/user/1000/pipewire-0";
          where = "/run/libvirt/qemu/pipewire/pipewire-0";
          type = "none";
          options = "bind,rw";
          wantedBy = ["user@1000.service"];
          wants = [
            "wait-pipewire.service"
            "prepare-run-qemu-libvirtd.service"
          ];
          after = [
            "prepare-run-qemu-libvirtd.service"
            "user@1000.service"
            "wait-pipewire.service"
          ];
        }
      ];
    };
... 
```

Aside from that, there's no gui config for pipewire audio, but that's not too bad either:

```xml
    <audio id='1' type='pipewire' runtimeDir='/run/libvirt/qemu/pipewire/'>
      <input name='alsa_input.usb-some-device.HiFi__Mic1__source' streamName='win11-in' latency='20000'/>
      <output name='alsa_output.usb-some-device.HiFi__Line1__sink' streamName='win11-out' latency='20000'/>
    </audio>
```

The pipewire `alsa_*` device names can be found in your `pw-dump` output.

### evdev Input Devices

evdev lets you have a toggle-able passthrough for your input devices. The default behavior is to switch between your host and guest whenever you press left-ctrl + right-ctrl. The only downside of it is that it can't handle hot-swapping devices on the host. I still find it a lot more convenient than solutions like synergy, multiple devices, or a KVM switch. This is another one that can't be configured through a gui:

```xml
    <input type='evdev'>
      <source dev='/dev/input/by-id/usb-event-mouse'/>
    </input>
    <input type='evdev'>
      <source dev='/dev/input/by-id/usb-HID_Keyboard-event-kbd' grab='all' repeat='on'/>
    </input>
    <input type='mouse' bus='ps2'/>
    <input type='keyboard' bus='ps2'/>
    <input type='mouse' bus='usb'>
      <address type='usb' bus='0' port='4'/>
    </input>
```

`/dev` can just be browsed to find these devices.  I found the `event` devices were the best to pass through.

### Power Consumption and Heat Management

This has always been an issue, but it's become more severe as video cards have become more power hungry.  The vfio module tries to set devices in a low-power mode, but it's entirely up to the hardware on what that actually means.  Modern video cards do not do their own power or fan management and expect a driver to handle that, so they don't handle it very well :upside_down_face:.

It's important to either:
- Load and unload drivers on your host. This can have some stability issues. I don't do it personally, but some people do.
- Use `nvidia-persistenced` to apply some power management for video cards on headless machines. Services associated with this have to be disabled before you can start actually using the card, but this can be done fairly easily in VM hooks.
- Just keep the card mounted in a VM all the time.

I wish this was less jank.  It can really pump out a lot of heat and eat a lot of power if a VM crashes while unattended.  It will only keep getting worse if current trends continue.

### Disk Management - Small OS Disk

I found the best way to manage disk space is with a very small disk for your guest OS.  I ended up using a 100G zvol for this.  I get to use the same snapshot and backup mechanisms as my host OS, and it gets to leverage automatic compression.  The benefits are still the same if you go with a qcow disk image or similar.  It's obviously not as fast as a hardware passthrough, but it's more than fast enough.  It makes it cheap and easy to keep a backup around and revert your system to a "clean" state if something gets into an unhappy state.  It seems to happen more often than it should with driver installations...

I usually do a hardware passthrough for my main drive, but even if you install all of your software on a separate drive, AppData is still on your OS drive and loves to grow perpetually.  There doesn't seem to be an OS supported way to move this, but you can copy the files to another drive with a separate user, and then set up a directory junction (`mklink /d ...`) pointing to the content. This misbehaves if you don't replicate the file permissions of the original user's AppData.  I don't exactly trust this to not break, but you can get away with being adventurous with good backup and recovery practices :smile:.

## Everything Else

Everything else is easy to pull from the relevant config modules [oak passthrough](https://github.com/hyperparabolic/nix-config/blob/5b011a614e68aa5c00ca86d3b4311d86098fac21/modules/hosts/oak/virtualization/passthrough.nix) and [libvirt](https://github.com/hyperparabolic/nix-config/blob/5b011a614e68aa5c00ca86d3b4311d86098fac21/modules/libvirt/default.nix).

I've been diving deep on custom mime types to get some windows only software behaving well in file managers, and launching through wine as if it was a native linux app.  There might be a follow-up post on that if I learn enough and it's interesting.
