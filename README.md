```markdown
# 🌟 Beautiful NixOS Setup with Flakes & Home Manager 🌟

Welcome, friend! 👋  
This is a **friendly, beginner-oriented guide** (in README form) to installing **NixOS** using **flakes** from the very beginning, and setting up **Home Manager** for your user environment — all in one clean, declarative, reproducible setup.

This guide assumes you're starting fresh (new machine or reinstall). It's designed to be safe, step-by-step, and full of explanations. No rushing — we'll take it slow and steady. 💆‍♂️

Why this combo?
- **NixOS**: Fully declarative OS — your entire system is defined in code.
- **Flakes**: Reproducible, pinned dependencies. No more "it works on my machine" surprises.
- **Home Manager**: Declarative user config (dotfiles, packages, shell, etc.) integrated directly into your NixOS setup.

Let's go! 🚀

## Table of Contents
- [Preparation](#preparation)
- [Boot the NixOS Installer](#boot-the-nixos-installer)
- [Partition & Format Disks](#partition--format-disks)
- [Mount Partitions](#mount-partitions)
- [Generate Hardware Config](#generate-hardware-config)
- [Set Up Your Flake Config Repo](#set-up-your-flake-config-repo)
- [Example Flake Structure](#example-flake-structure)
- [Install NixOS with the Flake](#install-nixos-with-the-flake)
- [First Boot & Final Steps](#first-boot--final-steps)
- [Applying Changes Later](#applying-changes-later)
- [Tips & Troubleshooting](#tips--troubleshooting)

## Preparation

1. **Download the latest NixOS ISO**  
   Go to: https://nixos.org/download.html  
   Choose the **Graphical Live CD** (Plasma or GNOME — easier for beginners).  
   Verify the SHA256 checksum if you want to be extra safe.

2. **Create a bootable USB**  
   - Linux/Mac: `dd if=nixos-*.iso of=/dev/sdX bs=4M status=progress oflag=sync`
   - Windows: Use Rufus or Balena Etcher.

3. **Back up any important data** on the target machine! ⚠️

## Boot the NixOS Installer

- Insert USB, boot from it (enter BIOS/UEFI if needed).
- Choose **Install NixOS** or just use the live environment.
- Open a terminal (it's root by default).

## Partition & Format Disks

We'll do a simple EFI + root setup. Adjust if you need swap, LUKS encryption, etc.

```bash
# List disks (be VERY careful — don't wipe the wrong one!)
lsblk

# Example: partition /dev/sda (replace sda with your disk!)
cfdisk /dev/sda
# Create:
# - 512M EFI partition (type EF00)
# - Rest as Linux root (type 8300)

# Format
mkfs.fat -F32 /dev/sda1          # EFI
mkfs.ext4 -L nixos /dev/sda2      # Root (or use btrfs: mkfs.btrfs -L nixos /dev/sda2)

# If you want swap, add a swap partition and: mkswap /dev/sda3 && swapon /dev/sda3
```

> Tip: Want Btrfs? Use `mkfs.btrfs` and later enable snapshots — super cool for rollbacks!

## Mount Partitions

```bash
mount /dev/sda2 /mnt

mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
```

## Generate Hardware Config

```bash
nixos-generate-config --root /mnt
```

This creates:
- `/mnt/etc/nixos/hardware-configuration.nix`
- `/mnt/etc/nixos/configuration.nix` (basic template)

## Set Up Your Flake Config Repo

We'll create a simple flake-based config right on the installer.

```bash
# Install git
nix-shell -p git

# Go to the config directory
cd /mnt/etc/nixos

# Initialize a git repo (or clone your own from GitHub later)
git init

# Create the flake structure
mkdir -p hosts/myhost home/myuser
```

Now create the files (use `nano` or `vim` — both are available).

### 1. `flake.nix` (main entry point)

```nix
{
  description = "My beautiful NixOS + Home Manager setup";

  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
    home-manager = {
      url = "github:nix-community/home-manager";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, home-manager, ... }: {
    nixosConfigurations.myhost = nixpkgs.lib.nixosSystem {
      system = "x86_64-linux";
      modules = [
        ./hosts/myhost/configuration.nix

        home-manager.nixosModules.home-manager
        {
          home-manager.useGlobalPkgs = true;
          home-manager.useUserPackages = true;
          home-manager.users.myuser = import ./home/myuser/home.nix;
        }
      ];
    };
  };
}
```

> Replace `myhost` with your desired hostname and `myuser` with your username.

### 2. `hosts/myhost/configuration.nix`

```nix
{ config, pkgs, ... }:

{
  imports = [ ./hardware-configuration.nix ];

  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

  networking.hostName = "myhost";  # Change this!
  networking.wireless.enable = true;  # Or use NetworkManager

  time.timeZone = "Asia/Tashkent";  # Your timezone 💫

  users.users.myuser = {
    isNormalUser = true;
    extraGroups = [ "wheel" "networkmanager" ];
    initialPassword = "changeme";  # Change after first boot!
  };

  environment.systemPackages = with pkgs; [
    wget curl git vim nano
  ];

  system.stateVersion = "24.11";  # Match your NixOS version
}
```

### 3. `home/myuser/home.nix`

```nix
{ config, pkgs, ... }:

{
  home.username = "myuser";
  home.homeDirectory = "/home/myuser";

  home.packages = with pkgs; [
    firefox neovim htop
    # Add your favorites here!
  ];

  programs.git = {
    enable = true;
    userName = "Your Name";
    userEmail = "you@example.com";
  };

  programs.zsh = {
    enable = true;
    oh-my-zsh = {
      enable = true;
      theme = "robbyrussell";
    };
  };

  home.stateVersion = "24.11";
}
```

### 4. Move hardware config

```bash
mv hardware-configuration.nix hosts/myhost/
```

Edit `hosts/myhost/configuration.nix` to import it (already done above).

## Install NixOS with the Flake

```bash
nixos-install --flake /mnt/etc/nixos#myhost
```

- It will ask for root password — set a strong one.
- When finished: `reboot`
- Remove USB and boot into your new system! 🎉

## First Boot & Final Steps

1. Log in as your user (password = initialPassword you set).
2. **Immediately change passwords**:
   ```bash
   passwd          # root
   passwd          # your user
   ```

3. (Optional) Commit your config to Git:
   ```bash
   cd /etc/nixos
   git add .
   git commit -m "Initial install"
   # Push to GitHub for backup/reproducibility
   ```

## Applying Changes Later

Edit files in `/etc/nixos`, then:

```bash
sudo nixos-rebuild switch --flake /etc/nixos#myhost
```

Home Manager changes are applied automatically! No separate command needed. ✨

To update everything:

```bash
cd /etc/nixos
nix flake update
sudo nixos-rebuild switch --flake .#myhost
```

## Tips & Troubleshooting

- **Garbage collect old generations**: `sudo nix-collect-garbage -d`
- **Rollback**: `sudo nixos-rebuild switch --rollback`
- **Wifi issues?** Enable NetworkManager: add `networking.networkmanager.enable = true;` and group "networkmanager"
- **Want more packages?** Just add them to `environment.systemPackages` or `home.packages`
- **Stuck?** The NixOS Discord, Reddit r/NixOS, or Discourse are super friendly.
- **Advanced**: Modularize further (split files, add more hosts, secrets with sops-nix, etc.)

You're done! Enjoy your beautiful, reproducible NixOS system. 🐧❤️

Feel free to fork this setup, customize, and make it yours. Happy Nixing!
```
```
