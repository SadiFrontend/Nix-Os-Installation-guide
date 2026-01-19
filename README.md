NixOS Configuration (Flakes + Home Manager)

This repository contains a declarative NixOS system configuration using Nix Flakes and Home Manager.
The goal is to make the system reproducible, rollback-safe, and easy to maintain.

📌 Features

✅ NixOS with Flakes enabled

🏠 Home Manager integrated (user-level configuration)

🔁 Rollback support via Nix generations

🧩 Modular configuration structure

📦 Declarative system & user packages

📂 Repository Structure
.
├── flake.nix
├── flake.lock
├── README.md
├── hosts
│   └── default
│       ├── configuration.nix
│       └── hardware-configuration.nix
├── home
│   └── username
│       ├── home.nix
│       └── programs.nix


Replace username with your actual Linux username.

🧠 Key Concepts
NixOS

Entire OS defined in configuration files

Changes applied with nixos-rebuild

Rollback available from GRUB or systemd-boot

Flakes

Pin dependencies (nixpkgs, home-manager)

Provide reproducible builds

Standardize project structure

Home Manager

Declarative user environment

Manages dotfiles, shell, editor, git, etc.

Works alongside system configuration

🧩 Requirements

NixOS 22.11+

Internet connection

User with sudo access

🚀 Enable Flakes on a Fresh NixOS Install

Edit /etc/nixos/configuration.nix:

nix.settings.experimental-features = [ "nix-command" "flakes" ];


Apply:

sudo nixos-rebuild switch

🧱 flake.nix

This is the entry point of the system.

{
  description = "NixOS configuration with Flakes and Home Manager";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";

    home-manager = {
      url = "github:nix-community/home-manager";
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { self, nixpkgs, home-manager, ... }:
  let
    system = "x86_64-linux";
  in {
    nixosConfigurations.default = nixpkgs.lib.nixosSystem {
      inherit system;

      modules = [
        ./hosts/default/configuration.nix

        home-manager.nixosModules.home-manager
        {
          home-manager.useGlobalPkgs = true;
          home-manager.useUserPackages = true;
          home-manager.users.username = import ./home/username/home.nix;
        }
      ];
    };
  };
}

🖥 System Configuration
hosts/default/configuration.nix
{ config, pkgs, ... }:

{
  imports = [
    ./hardware-configuration.nix
  ];

  nix.settings.experimental-features = [ "nix-command" "flakes" ];

  boot.loader.systemd-boot.enable = true;
  boot.loader.efi.canTouchEfiVariables = true;

  networking.hostName = "nixos";
  networking.networkmanager.enable = true;

  time.timeZone = "UTC";

  i18n.defaultLocale = "en_US.UTF-8";

  users.users.username = {
    isNormalUser = true;
    extraGroups = [ "wheel" "networkmanager" ];
  };

  environment.systemPackages = with pkgs; [
    git
    vim
    wget
    curl
  ];

  services.openssh.enable = true;

  system.stateVersion = "23.11";
}

🏠 Home Manager Configuration
home/username/home.nix
{ config, pkgs, ... }:

{
  home.username = "username";
  home.homeDirectory = "/home/username";

  home.stateVersion = "23.11";

  imports = [
    ./programs.nix
  ];

  home.packages = with pkgs; [
    neovim
    htop
    tree
  ];

  programs.home-manager.enable = true;
}

home/username/programs.nix
{ pkgs, ... }:

{
  programs.git = {
    enable = true;
    userName = "Your Name";
    userEmail = "you@example.com";
  };

  programs.bash.enable = true;

  programs.starship = {
    enable = true;
    settings = {
      add_newline = false;
    };
  };
}

🔨 Applying the Configuration

From the root of the repo:

sudo nixos-rebuild switch --flake .#default

🔁 Rollbacks
Temporary (boot menu)

Reboot

Select a previous generation

Permanent
sudo nixos-rebuild switch --rollback

📦 Updating Inputs
nix flake update
sudo nixos-rebuild switch --flake .#default

🧪 Useful Commands
Command	Description
nix flake show	Inspect flake outputs
nix flake check	Validate configuration
nixos-rebuild build	Build without switching
home-manager switch	(Standalone HM only)
🧠 Tips & Best Practices

Commit flake.lock to Git

Use nixos-rebuild build before switching

Split configs into modules as setup grows

Keep system.stateVersion unchanged

📚 Learning Resources

NixOS Manual

Home Manager Manual

Nix Pills

NixOS Wiki

✅ Status

This configuration is:

✔ Reproducible

✔ Declarative

✔ Flake-based

✔ Home Manager integrated
