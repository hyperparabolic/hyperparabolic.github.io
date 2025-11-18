---
title: NixOS Audio Receiver with Airplay
date: 2025-09-23
author: Spencer Balogh
description: A funky little music kiosk for my living room
tags:
  - NixOS
  - Audio
  - Pipewire
  - MPD
toc: true
---

I've been bought into the Apple ecosystem for a while now for phones. This might be a little surprising given the rest of the content on my blog, but I don't really enjoy the experience of mobile phones (screentime says 20 minutes daily this week). I've done both Apple and Android, I don't really like either as a development platform, and I find ios just stays out of the way a bit better. Anyway though, part of this was buying into the AirPlay ecosystem.

Regardless of what audio streaming technology you use, I've always felt the market for products that connect "legacy" media is lacking. Just about everything that streams audio from a physical input is priced as a luxury product, and it's rare to not have some kind of additional ecosystem lock-in that comes with it. I've been iterating on several different homebrewed solutions for a while now.

Past iterations were built with a raspberry pi running Darkice / Icecast, streaming an internet radio station to my phone and allowing mobile to stream to devices over AirPlay. I found that solution a little finicky / fragile, and was always interested in removing my phone from the equation. I kept iterating on it and finally landed on a really nice franken-solution utilizing PipeWire and MPD. As a bonus, it also streams internet radio stations, can be controlled remotely by Home Assistant, and has a local user interface.

All of this is running on my NixOS host redbud. This system is a retired laptop that I use as a metrics server, so I moved it to my living room and it lives under my record player so it can fill another niche.

<!--more-->

## PipeWire's Role

[PipeWire](https://www.pipewire.org/) is more or less the glue that holds everything together here. It acts as a unifying interface for linux audio, providing secure access support for the PulseAudio, JACK, and ALSA interfaces. It provides a graph based system for routing audio between hardware devices and applications for all of these systems. A basic config for this is easy enough to set up on NixOS. Mine looks something like this:

```nix
{
  services.pipewire = {
    enable = true;
    alsa.enable = true;
    alsa.support32Bit = true;
    pulse.enable = true;
    jack.enable = true;
  };
}
```

Out of the box though, this is missing a piece. PipeWire supports RAOP Discovery and streaming, otherwise known as Airplay. I have this configured as a separate addon config:

```nix
{pkgs, ...}: {
  # raop (airplay) discovery support for pipewire
  services = {
    avahi.enable = true;
    pipewire = {
      package = pkgs.pipewire.override {
        raopSupport = true;
      };
      raopOpenFirewall = true;
      extraConfig.pipewire = {
        "99-raop-discovery" = {
          "context.modules" = [
            {
              name = "libpipewire-module-zeroconf-discover";
              args = {};
            }
            {
              name = "libpipewire-module-raop-discover";
              args = {};
            }
          ];
        };
      };
    };
  };
}
```

Avahi is needed to support discovering Airplay devices. The rest of this is just enabling the RAOP feature set. This is the minimum config for functionality. Additional pipewire / wireplumber rules can restrict what devices you want to discover and connect to.

### PipeWire Pain

PipeWire is designed to run as a systemd user session service. It does have support for a system level service. In this case, all users of the pipewire group can get simultaneous access to the pipewire socket. I personally chose not to do this. The developers recommend against it, as it's in direct conflict with the security model they've developed for pipewire. I also try to keep as much of my system running in user space as possible just as personal policy. This does mean that we need a user session though, and any software that needs to interact with pipewire needs shared access to that user session. The upcoming section on MPD configuration gets into some more details around this...

## MPD (Music Player Daemon) Provides Control

MPD is an audio player with a server-client architecture. The server portion of the config looks something like this:

```nix
{
  config,
  ...
}: {
  services = {
    mpd = {
      enable = true;
      user = "spencer";
      extraConfig = ''
        default_permissions "read,add,control"
        restore_paused "yes"
        input {
          plugin "curl"
        }
        input {
          plugin "alsa"
          default_device "hw:1,0"
        }
        audio_output {
          type "pipewire"
          name "pipewire output"
        }
      '';
    };
  };

  users.users.spencer = {
    # mpd starts without logged in user
    linger = true;
  };

  systemd.services.mpd.environment = {
    # share environment for pipewire socket
    XDG_RUNTIME_DIR = "/run/user/${toString config.users.users.spencer.uid}";
  };
}
```

Notice that we're running as my "spencer" user. This is necessary so that MPD can access the user session pipewire socket. This could be a dedicated user, but I'm using the same user I use for ssh connections for convenience. Sharing XDG_RUNTIME_DIR is also necessary to allow mpd to discover the pipewire socket.

The rest of the config is fairly straight forward. The alsa input plugin allows you to stream from a hardware input. The pipewire audio_output allows mpd to play to the pipewire RAOP sink. The curl plugin allows audio to be streamed over the internet, this is a nice little perk that enables internet radio stations among other things.

The alsa `hw:1,0` default device is a `hw:<card #>,<device #>`. These values can be discovered with `aplay -l` and are hardware specific.

### Remote Access to PipeWire via SSH

I'm not going to get into configuring SSH access here, but users can only access ALSA devices over a remote session if they are a member of the `audio` group. This is configured easily enough like this:

```nix
{
  
  users.users.spencer = {
    extraGroups = ["audio"];
  };
}
```

With this set up, `aplay` and `wpctl` both behave just like they would locally while connected to a ssh session. This can be used to manually configure pipewire.

### Remote Access to MPD

You definitely don't want MPD to be exposed to the public internet. I keep all of my most trusted devices connected to a tailscale vpn, and only expose MPD over that vpn interface. This is *good enough* for me.

```nix
{
  config,
  ...
}: {
  services = {
    mpd = {
      network = {
        listenAddress = "any";
        port = 6600;
      };
    };
  };

  # remote access only permitted via vpn
  networking.firewall.interfaces."tailscale0" = {
    allowedTCPPorts = [config.services.mpd.network.port];
  };
}
```

I usually just use `mpc` to control this via the CLI. If your hardware device isn't showing up in your playlist, you can add it with `mpc add alsa://hw:1,0`.

Once this was all set up I also added this MPD server to Home Assistant.  I find that particular client is a little inflexible for configuration, but it allows easy access to play / pause controls, supports streaming any audio content Home Assistant has access to, and can play text-to-speech notifications. Neat.


### Keeping the User Session Alive and Local Access

There are a lot of different ways you could keep the user session alive. `loginctl enable-linger` alone *should* be enough to take care of this.

I ended up deciding to take advantage of the fact that the host this all runs on is a laptop though. [cage](https://github.com/cage-kiosk/cage) can be used to run a desktop application as a "kiosk". I use this to run a graphical mpd client on redbud automatically on tty1. This establishes a user session on boot without manually logging in, gives a quick an easy way to acesss play / pause / volume controls, and keeps the session locked down to the mpd client only.

```nix
{
  lib,
  pkgs,
  ...
}: {
  services = {
    cage = {
      enable = true;
      user = "spencer";
      extraArguments = [
        # no decorations
        "-d"
        # allow tty switching
        "-s"
      ];
      program = "${lib.getExe pkgs.ymuse}";
    };
  };

  users.users.spencer = {
    # loginctl enable-linger as a backup if the kiosk is failing
    linger = true;
  };
}
```

I didn't look too long for local players before landing on ymuse. I find it's good enough, but cage doesn't handle popup windows very gracefully. Let me know if you have any other recommendations for minimal apps.

## Keeping It Declarative

I use impermanence on all of my hosts, and try to keep as much of my config as possible completely declarative. I ended up writing some wireplumber rules to make this host more likely to do the right thing as audio devices are connected / disconnected.

```nix
{
  services.pipewire = {
    wireplumber.extraConfig = {
      # disable audio devices,
      "50-disable-audio-device" = {
        "monitor.alsa.rules" = [
          {
            matches = [
              {
                # disable internal audio devices to prevent feedback potential
                "device.name" = "alsa_card.pci-0000_00_1f.3";
              }
            ];
            actions = {
              update-props = {
                "device.disabled" = true;
              };
            };
          }
        ];
      };
      "60-out-priority" = {
        "monitor.alsa.rules" = [
          {
            matches = [
              {
                "node.name" = "raop_sink.Sonos-542A1B89DEA5*";
              }
            ];
            actions = {
              update-props = {
                # normal input priority is sequential starting at 2000
                "priority.driver" = "3000";
                "priority.session" = "3000";
              };
            };
          }
        ];
      };
    };
  };
}
```

This just bumps up the priority of my preferred RAOP devices, and disables the laptop's internal audio devices to ensure that there's no potential for those to be selected.

This is all completely optional, and it's fine to just control devices with `wpctl`, but I find it keeps things behaving just a little more predictably.

## All Done!

![mpd kiosk photo](https://i.imgur.com/iat5hnu.png)

This is definitely the best iteration of this project I've come up with. Let me know if you've done something similar or can think of any enhancements.
