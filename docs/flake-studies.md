---
title: "Flake Studies"
date: 2024-07-31
---

# Flake Studies

## Overview

NOTE: the following are excerpts taken from studying various sources in
[Nix](../index.md). This document uses [Zero-to-nix
Flakes](https://zero-to-nix.com/concepts/flakes) as a starting point.

Flakes process nix code. They take inputs and generate a set of outputs that nix can use.
Flakes can output various things such as:

- packages: Nix packages are self-contained bundles that are built using
  derivations and provide some kind of software or dependencies of software on
  your computer. A package can have dependencies on other packages, which are
  encoded in its closure. Each package lives at a unique path in the Nix store
  indexed by its name and its input hash.
- run programs
- development environments
- formatters
- overlays
- hydra jobs

## Flake references

A flake reference is a string that identifies where the flake is located. Flake references
can be used inside input declarations as well shell environments when running commands like
nix run or nix develop.

Sample flake references can be found at:

| Reference                                                                      | Description                                                                                             |
| ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| path:/home/nix-stuff/my-flake                                                  | The /home/nix-stuff/my-flake directory on the current host                                              |
| github:DeterminateSystems/zero-to-nix                                          | The DeterminateSystems/zero-to-nix GitHub repository                                                    |
| github:DeterminateSystems/zero-to-nix/other                                    | The other branch of the DeterminateSystems/zero-to-nix GitHub repository                                |
| github:DeterminateSystems/zero-to-nix/d51c83a5d206e882a6f15a282e32b7079f5b6d76 | Commit d51c83a5d206e882a6f15a282e32b7079f5b6d76 on the DeterminateSystems/zero-to-nix GitHub repository |
| nixpkgs                                                                        | The most recent revision of the nixpkgs-unstable branch of Nixpkgs (an alias for github:NixOS/nixpkgs)  |
| nixpkgs/release-22.11                                                          | The release-22.11 branch of Nixpkgs                                                                     |
| <https://flakehub.com/f/NixOS/nixpkgs/0.2405.*.tar.gz>                         | The most recent revision of the nixos-24.05 branch of Nixpkgs hosted on FlakeHub                        |

Flake references are defined formally in: [Flake References](https://nix.dev/manual/nix/2.18/command-ref/new-cli/nix3-flake#flake-references)

## Flake inputs

The inputs describe the dependencies that a flake needs to be built. A input
schema is defined in the [Nix Reference
Manual](https://nix.dev/manual/nix/2.18/command-ref/new-cli/nix3-flake#flake-inputs)

## Flake Lock File

The flake.lock file is used to specify the revisions that are used in the inputs section of a flake.
The flake.lock file is automatically generated when executing nix build, nix develop or nix flake show.

The combination of a flake.nix and flake.lock fully specify the flake. The flake.lock pins the definitions to a specific
revision even though it is not contained in the nix code.

Sample excerpt from a flake.lock file:

```nix
{
  "nodes": {
    "nixpkgs": {
      "locked": {
        "lastModified": 1668703332,
        // A SHA of the contents of the flake
        "narHash": "sha256-PW3vz3ODXaInogvp2IQyDG9lnwmGlf07A6OEeA1Q7sM=",
        // The GitHub org
        "owner": "NixOS",
        // The GitHub repo
        "repo": "nixpkgs",
        // The specific revision
        "rev": "de60d387a0e5737375ee61848872b1c8353f945e",
        // The type of input
        "type": "github"
      }
    },
    // Other inputs
  }
}
```

In the above example the nixpkgs is pinned to the revision: "de60d387a0e5737375ee61848872b1c8353f945e"

## Flake Output schema

The output schema for a flake is described in the nix package manager
src/nix/flake.cc in CmdFlakeCheck. The output schema represents the various
outputs for the following commands:

```bash

    # the output is specified as a comment
    nix flake check # checks

    nix build # packages

    nix run   # apps

    nixfmt or nixpkgs.fmt   # formatter

    nixos-rebuild switch  # nixosConfigurations

    nix develop # devshells

    nix flake  init # templates

```

The output "overlays" and "nixosModules" are consumed by other flakes. The "hydraJobs" output is
used to run hydra jobs.

This following excerpt is taken from [NixOS wiki](https://nixos.wiki/wiki/Flakes)

```nix
    # <system> is something like "x86_64-linux", "aarch64-linux", "i686-linux", "x86_64-darwin"
    # <name> is an attribute name like "hello".
    # <flake> is a flake name like "nixpkgs".
    # <store-path> is a /nix/store.. path
        { self, ... }@inputs:
        {
          # Executed by `nix flake check`
          checks."<system>"."<name>" = derivation;
          # Executed by `nix build .#<name>`
          packages."<system>"."<name>" = derivation;
          # Executed by `nix build .`
          packages."<system>".default = derivation;
          # Executed by `nix run .#<name>`
          apps."<system>"."<name>" = {
            type = "app";
            program = "<store-path>";
          };
          # Executed by `nix run . -- <args?>`
          apps."<system>".default = { type = "app"; program = "..."; };

          # Formatter (alejandra, nixfmt or nixpkgs-fmt)
          formatter."<system>" = derivation;
          # Used for nixpkgs packages, also accessible via `nix build .#<name>`
          legacyPackages."<system>"."<name>" = derivation;
          # Overlay, consumed by other flakes
          overlays."<name>" = final: prev: { };
          # Default overlay
          overlays.default = final: prev: { };
          # Nixos module, consumed by other flakes
          nixosModules."<name>" = { config, ... }: { options = {}; config = {}; };
          # Default module
          nixosModules.default = { config, ... }: { options = {}; config = {}; };
          # Used with `nixos-rebuild switch --flake .#<hostname>`
          # nixosConfigurations."<hostname>".config.system.build.toplevel must be a derivation
          nixosConfigurations."<hostname>" = {};
          # Used by `nix develop .#<name>`
          devShells."<system>"."<name>" = derivation;
          # Used by `nix develop`
          devShells."<system>".default = derivation;
          # Hydra build jobs
          hydraJobs."<attr>"."<system>" = derivation;
          # Used by `nix flake init -t <flake>#<name>`
          templates."<name>" = {
            path = "<store-path>";
            description = "template description goes here?";
          };
          # Used by `nix flake init -t <flake>`
          templates.default = { path = "<store-path>"; description = ""; };
        }
```

## Flake Registries

Flake registries enable users to refer to flakes using a symbolic identifier. As an example,
**nixpkgs** is a symbol that refeers to: github:NixOS/nixpkgs/nixpkgs-unstable.

Sample symbolic references are provided below:

| Symbolic Identifier | Full Flake Reference                  |
| ------------------- | ------------------------------------- |
| Symbolic identifier | Full flake reference                  |
| nixpkgs             | github:NixOS/nixpkgs/nixpkgs-unstable |
| flake-utils         | github:numtide/flake-utils            |
| home-manager        | github:nix-community/home-manager     |

The example below uses flake registries and omits using an input section to the flake.
The default registries are defined in a JSON file: [flake registry](https://github.com/NixOS/flake-registry/blob/master/flake-registry.json)

Registries can be managed through the "nix registry" command. Further details
can be found at:
[registry](https://nix.dev/manual/nix/2.18/command-ref/new-cli/nix3-registry)

```nix
{
  outputs = { self, nixpkgs, flake-utils }:

    let
      system = "x86_64-linux";
      pkgs = import nixpkgs { inherit system; };
    in flake-utils.lib.eachDefaultSystem (system: {
      devShells.default =
        pkgs.mkShell { packages = with pkgs; [ curl git jq wget ]; };
    });
}

```
