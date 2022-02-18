---
title: "Creating a base build of Snapcraft in Travis CI"
created_at: Fri Feb 18 2022 15:00:00 EDT
author: Montana Mendy
layout: post
permalink: 2022-02-18-snapcraft
category: news
excerpt_separator: <!-- more --> 
tags:
  - news
  - feature
  - infrastructure
  - community
---

![TCI-Snapcraft](https://user-images.githubusercontent.com/20936398/154757930-372583c3-eb7b-4ee6-9edc-fec0b6b0731e.png)



Sometimes you want to see how things work under the hood, and this is why I put together quickly how I used Snapcraft with Travis CI to get a LXD up and running, and download VLC and then get info on that version of VLC all inside Travis CI. 


<!-- more --> 


## Getting started 

Yes, yes I know what you're thinking, it's not called `.travis.yml`? We'll get to that, but first let's start with out a file called `snapchat.yaml`. This is how mine looks: 

```yaml
name: montana
version: mendy
summary: My name is Montana Mendy, I'm an Engineer here at Travis CI, and I'm just testing out Snapd with Travis.
description: |
 My name is Montana Mendy and I'm a Software Engineer at Travis CI, I'm trying new things with Travis CI and the Brave Browser.
grade: stable
confinement: strict

architectures:
  - build-on: amd64
  
parts:
  brave:
    plugin: dump
    source: https://github.com/brave/brave-browser/releases/download/v0.58.10/brave-browser-dev_0.58.10_amd64.deb
    source-type: deb
    override-pull: |
      snapcraftctl pull
      rm -rf etc/cron.daily/ 
      rm -rf usr/bin/brave-browser-dev
      chmod 4555 opt/brave.com/brave-dev/brave-sandbox
      unlink opt/brave.com/brave-dev/brave-browser
      sed -i 's|Icon=brave-browser|Icon=/opt/brave.com/brave-dev/product_logo_128\.png|g' usr/share/applications/brave-browser-dev.desktop
    after:
      - desktop-gtk3
    stage-packages:
      - gir1.2-gnomekeyring-1.0
      - libasound2
      - libgconf-2-4
      - libgl1-mesa-glx
      - libglu1-mesa
      - libgnome-keyring0
      - libcap2
      - libgcrypt20
      - libnotify4
      - libnspr4
      - libnss3
      - libpulse0
      - libxtst6
      - libxss1

apps:
  brave:
    command: bin/desktop-launch $SNAP/opt/brave.com/brave-dev/brave-browser-dev
    desktop: usr/share/applications/brave-browser-dev.desktop
    # Correct the TMPDIR path for Chromium Framework/Electron to
    # ensure libappindicator has readable resources.
    environment:
      TMPDIR: $XDG_RUNTIME_DIR
    plugs:
      - alsa
      - avahi-observe
      - browser-sandbox
      - camera
      - cups-control
      - desktop
      - gsettings
      - home
      - mount-observe
      - network
      - opengl
      - password-manager-service
      - pulseaudio
      - remove-media
      - screen-inhibit-control
      - unity7
      - upower-observe
      - x11

plugs:
  browser-sandbox:
    interface: browser-support
    allow-sandbox: true
  gtk-3-themes:
    interface: content
    target: $SNAP/data-dir/themes
    default-provider: gtk-common-themes
  icon-themes:
    interface: content
    target: $SNAP/data-dir/icons
    default-provider: gtk-common-themes
  sound-themes:
    interface: content
    target: $SNAP/data-dir/sounds
    default-provider: gtk-common-themes
```

## Your Travis CI Config

This is for the Brave Browser, but I've added things like `pulseaudio` to the Snap `plugs`. So let's push that into a directory called `snap`. Now let's head back to your root directory and make your `.travis.yml`, and this is what I coded out for my `.travis.yml`: 

```yaml
language: shell
dist: xenial
os: linux
group: edge 

env:
  global:
    - LC_ALL: C.UTF-8
    - LANG: C.UTF-8
    - SNAPCRAFT_ENABLE_SILENT_REPORT: y
    - SNAPCRAFT_BUILD_INFO: 1
    - SNAPCRAFT_BUILD_ENVIRONMENT: 'lxd'

addons:
  snaps:
    - name: snapcraft
      channel: stable
      confinement: classic
    - name: lxd
      channel: stable

script:
  - sudo usermod --append --groups lxd $USER
  - sudo /snap/bin/lxd.migrate -yes
  - sudo /snap/bin/lxd waitready
  - sudo /snap/bin/lxd init --auto
  - sudo apt install snapd
  - sudo snap install hello-world
  - sudo snap install nethack
  - snap version 
  - snap list
  - snap connections nethack
  - snap services lxd
  - sudo snap install --channel=edge vlc
  - which vlc 
  - snapcraft extensions
  - snap connections vlc
  - sudo systemctl enable --now snapd.socket
  - sudo journalctl -xeb | grep -i snap
```
So let's trigger a build by using `powerline` in Vim, and if your build is successful it should look something like this: 

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jbrgu8g8vqaytvdca02k.png)
 
Somethings you'll want to keep in mind with Snapcraft Multipass has 2 CPU's assigned to it, this can be variable depending on your settings:

```bash
$ export SNAPCRAFT_BUILD_ENVIRONMENT_CPU=8 
$ export SNAPCRAFT_BUILD_ENVIRONMENT_MEMORY=16G
``` 

You can put those in your `script:` hook in your `.travis.yml` file. 

## Migrating between bases 

A base snap is a special kind of snap that provides a run-time environment with a minimal set of libraries that are common to most Linux distributions. Think of it as a minimal Gentoo install. At the simplest level, in your `snapcraft.yaml` file you can do: 

```bash
- base: core18
+ base: core20
``` 

As you can see, only one of the base keywords need to be updated. 

## Enforcing Snap Policies 

One thing you may want to consider when making this, is enforcing some security policies, the way you do that is, yes you guessed it another `.yaml` file, so let's call this one `snap.yaml` and it would look a little like this:

```bash
name: montana
version: 1.0
apps:
  bar:
    command: mendy
  baz:
    command: dig
    daemon: simple
    plugs: [network]
```

If you run this with your Travis CI build and don't like what you see you can always set a `snap disconnect` conditional so you can disconnect. Don't forget you can add a Cron job directly from Snapcraft by adding this line in your `.travis.yml`: 

```bash
sudo snap set system refresh.timer=fri5,23:00-01:00
```
That's now a cron that will refresh Snap at a time of my choosing.

## Conclusion 

In some sense setting up Snapcraft for Travis CI was really fun, and got to see hands on how LXD containers are built from the ground up, setting your own policies, enforcing them and even picking your own plugins so you have a great foundation to build on when you're ready. 

Here's my repository: https://github.com/Montana/travis-snap-lxd, enjoy it. Do something with it! 

As always if you have any questions, any questions at all, please email me at [montana@travis-ci.org](mailto:montana@travis-ci.org).

Happy building!
