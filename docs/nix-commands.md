---
title: "Nix Commands"
---

Since Nix 2.4+ a set of new commands have been introduced to collect most of
the most commonly used commands into a a cohesive interface. The _nix-shell_
command has been broken into multiple comamnds with different semantics:

1. _nix shell_
2. _nix develop_
3. _nix run_

Previously, invoking nix-shell without any other flags would setup a build environment
of a derivation. This is not the behaviour for _nix shell_.

## Development Shells

The nix develop command is used to create development shells. The command is by
default meant to be used with flakes. The -f flag can be used with standalone
derivations.

The nix develop command is intended to recreate build environments for single
packages. As an example, when you call the command _nix develop
nixpkgs#hello_, you are dropped into a build environment for the hello
command. This helps to debug or develop the build process for a package.

The _nix develop_ command can run the build phases directly. The _nix develop_
command creates a shell with all buildInputs and environmetn variable of a
derivation loaded and shellHooks executed.

The _nix develop_ command can run individual run phases for a shell using the
--<PHASE> o r--phase PHASE (for non standard phase options). Arbitrary commands
can be executed in the shell using:

```bash
    nix develop --command COMMAND [ARGS...]
```

This command is most useful for locally developing packages.

## Development environments

The nix develop commnd can be used to create a development environment for
your project. All buildinputs and environment variables can be gathered together
to enable development. The setupHooks are also run when the shell is opened.
The direnv tool can be used in conjunction with nix develop to create an
elegant workflow for development.

The _mkShell_ function provides a convenient mechanism to collect packages for
your environment:

```nix
    pkgs.mkShell = {
      # a list of packages to add to the shell environment
      packages ? [ ]
    , # propagate all the inputs from the given derivations
      inputsFrom ? [ ]
    , buildInputs ? [ ]
    , nativeBuildInputs ? [ ]
    , propagatedBuildInputs ? [ ]
    , propagatedNativeBuildInputs ? [ ]
    , ...
    }: ...

```

The _shellHook_ commands are run whenever the shell is entered. Attributes that
are unknown to mkDerivation are applied as environment variables

The _nix develop_ command loads a flake's development shell directly, if the flake
defines a devShell output:

```nix
    {
      description = "Flake utils demo";

      inputs.nixpkgs.url = "github:nixos/nixpkgs";
      inputs.flake-utils.url = "github:numtide/flake-utils";

      outputs = { self, nixpkgs, flake-utils }:
        let
        in
          flake-utils.lib.eachDefaultSystem (
            system:
              let
                pkgs' = import nixpkgs { inherit system; };
              in
                rec {
                  packages = { myPackage = ... }; # packages defined here
                  devShell  = pkgs'.mkShell = {
                    # a list of packages to add to the shell environment
                    packages ? [ jq ]
                    , # propagate all the inputs from the given derivations
                      # this adds all the tools that can build myPackage to the environment
                    inputsFrom ? [ pacakges.myPackage ]
                  };
                }
          );
    }
```

## Temporary Programs

Software can be added to a user's profile using the _nix profile install_
command. Please note that this command is incompatible with _nix-env- and also
with_home-manager_. The command _nix-shell -p <package+>_ will retrieve the
desired packages and open a shell with these packages mixed in.

The _nix-shell -p <package+>_ command constructs a derivation with the specified
programs as buildInputs:

```nix
    with import <nixpkgs> { };
    (pkgs.runCommandCC or pkgs.runCommand) "shell" { buildInputs = [ (asciidoc) ]; } ""

```

In nix 2.4 the _nix shell_ command does the same thing but focused on flakes. Any
output of a flake can be added to an environment. As an example: _nix shell nixpkgs#ascii_.

Unfortunately, _nix shell_ cannot be used to load an environment with packages
directly as python modules using the shellHook. The following two commands do
not evaluate to the same thing:

```bash
    nix shell nixpkgs#{python,python3Packages.numpy}

    nix-shell -p python python3Packages.numpy
```

As an alternative you can run one of the following two commands:

```bash
    # create an ad-hoc derivation
    nix shell --impure \
        --expr "with import <nixpkgs> {}; python.withPackages (pkgs: with pkgs; [ prettytable ])"
```

Use nix develop and passs in a devShell:

```bash
nix develop --impure \
        --expr "with import <nixpkgs> {}; pkgs.mkShell { packages = [python3 python3Packages.numpy ];}"
```

## Run scripts

The nix-shell command can be used to run commands from a derivation that is not currenty installed
in the user's profile.

The _nix-shell --command COMMAND ARGS_ runs the command within the build
environment. The nix-shell with the --run argument would simply run the
argument. The _--command_ argument is interactive, if a command fails the user
is dropped into the shell with the build environment.

The _--run_ command runs non-interactively and closes the shell after the
command returns.

The _nix-shell --run_ command in the new syntax would look like:

```nix
    nix shell nixpkgs#asciidoc -c asciidoc
```

An alternative way of invoking the above command in the new syntax is provided below:

```nix
    nix run nixpkgs#asciidoc
```

The _nix run_ command can be used to run _app_ outputs of flakes.

```nix
   outputs = {self}: {
      ...
      apps.x86_64-linux.watch = {
        type = "app";
        program = "${generator-watch}";
      };
    }
```

If an attribute of _nix run_ is not found as an app, nix will look up a program using the
following key:

    ```nix
        programs.${<program>}/bin/<program>
    ```

## Shebang interpreter

The nix-shell commadn can be used as a she-bang interpreter for shell scripts:

    ```nix
        #! /usr/bin/env nix-shell
        #! nix-shell -i python -p python3
        print("hello")
    ```

This mechanism is currently not available through the new command interface.

## New Command Non-flake

Traditional _\*.nix_ files can be used with _--expr_ argument. The default flake
input requires greater purity strictness. The _--impure_ flage can be used
to circumvent the purity requirements on file imports:

```bash
   nix shell --impure --expr "import my.nix {}"
```

## Summary of Command Comparison

|                      | nix-shell                                               | nix develop                                       | nix shell                   | nix run                     |
| -------------------- | ------------------------------------------------------- | ------------------------------------------------- | --------------------------- | --------------------------- |
| runs shellHooks      | yes                                                     | yes                                               | no                          | no                          |
| use as interpreter   | yes                                                     | no                                                | no                          | no                          |
| supports flakes      | no                                                      | yes                                               | yes                         | only                        |
| evaluate nix file    | yes                                                     | with --impure, -f or --expr                       | with --impure, -f or --expr | with --impure, -f or --expr |
| modifies environment | PATH, attributes mkDerivation and changes by shellHooks | attributes mkDerivation and changes by shellHooks | PATH                        | nothing                     |

## Sources

- [New Nix Command](https://blog.ysndr.de/posts/guides/2021-12-01-nix-shells/)
