+++
title = "Test Server Setup"
slug = "TestServerSetup"
url = "/Developer/TestServerSetup/"
+++

## Preinstallation NixOS

Acquire [Nix or NixOS](https://nixos.org/download.html) ([AArch64 22.05 ISO](https://hydra.nixos.org/job/nixos/release-22.05-aarch64/nixos.iso_minimal.aarch64-linux))

### If you're using Nix from another distro

As root, logout and back in as prompted and then acquire installation tools and git, and setup the correct channel:

    nix-shell -p nixos-install-tools git
    nix-channel --remove nixpkgs
    nix-channel --add https://nixos.org/channels/nixos-22.05 nixos
    nix-channel --update

### If you're using NixOS installation media

Acquire git: `nix-shell -p git`

## Format Disks

Format disks according to the [UEFI guide](https://nixos.org/manual/nixos/stable/index.html#sec-installation-partitioning-UEFI)

For the AArch64 machines we have boot on `nvme0n1p1`, and root on RAID0 `nvme[01]n1p3`:

    # lsblk

    NAME        MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
    nvme1n1     259:0    0 894.3G  0 disk
    ├─nvme1n1p1 259:2    0     1G  0 part
    └─nvme1n1p2 259:3    0 893.3G  0 part
    └─md127     9:127  0   1.7T  0 raid0 /nix/store
                                        /
    nvme0n1     259:1    0 894.3G  0 disk
    ├─nvme0n1p1 259:4    0     1G  0 part  /boot
    └─nvme0n1p2 259:5    0 893.3G  0 part
    └─md127     9:127  0   1.7T  0 raid0 /nix/store
                                        /

## System setup

Read **BUT DO NOT RUN**, the installation steps in the [manual](https://nixos.org/manual/nixos/stable/index.html#sec-installation-installing).

Mount the partitions as steps 1-3 instruct.

Clone our repo in to `/etc/nixos` on the target machine:

    cd /mnt
    mkdir -p etc
    mkdir -p tmp # needed for a bug in the installer

    git clone https://github.com/YellowOnion/nixos-test-farm.git nixos
    
    cd etc/nixos

Run our custom setup script (this automates `nixos-generate-config`):

    ./setup.sh <new HostName for machine>
    
    
given the command `./setup.sh farm2` this will create 2 files, `farm2.nix` `farm2-hw.nix`, and symlink those to `configuration.nix` and `hardware-configuration.nix` respectively. `farm.nix` will have `common.nix` imported, and host name set.

`common.nix` comes with a bunch of tools and ssh keys setup for root make changes if needed.

## Installation

Run

    nixos-install --no-root-passwd && poweroff --reboot
    # the flag is optional but but the installer will
    # prompt for root password before rebooting otherwise.
    
## Troubleshooting

If the install process doesn't work or after a system update something broke for what ever reason follow steps skipping the disk format and cloning, make sure you mount boot in `/mnt/boot`, make your fixes and re-run `setup.sh` if you made any hardware changes, and finally rerun the installation process, this will repair your OS.

## Postinstall

TODO
