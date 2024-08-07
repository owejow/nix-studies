---
title: "Library Functions in Nixpkgs"
---

# Overview

There are 400+ library functions that are defined inside of nixpkgs. These functions are
used extensively throughout the nix ecosystem. The nix repl can be started by invoking the
following command on the terminal:

```bash
    nix repl
```

The text "repl>" signifies that the command is run inside the nix repl shell. To explore
the nixpkgs library functions inside the nix reply, execute the following sequence inside

```bash
    nix repl
    repl> :l <nixpkgs>   # load nixpkgs to your current session
    repl> lib.length     # if you type lib.<Tab> you will get a complete list

```

The nix language, nixpkgs.lib and builtins.lib make up the core foundation on which the
nix ecosystem is built.

The :doc command on the nix repl can be used to view documentation regarding
the various functions.

```bash
    repl> :doc lib.length
          # command returns documentation:
          #
          #        Return the length of the list e.

    repl> :doc lib.getAttr

          # command return :
          #
          #       Synopsis: builtins.getAttr s set
          #       getAttr returns the attribute named s from set. Evaluation
          #           aborts if the attribute doesn’t exist. This is a dynamic version of the .
          #           operator, since s is an expression rather than an identifier.
```

Unfortunately, many of the library and builtin functions lack documentation.

```bash
    repl> :doc lib.getOutput
        # error: value does not have documentation
```

Fortunately, the repl can be used to find the declaration for library functions.

```bash
    repl> :e lib.getOutput

```

The ":e" will open the declaration of lib.getOutput with the default editor. By default
this editor is nano. The code snippet displayed inside the editor is the following:

```nix
  getOutput = output: pkg:
    if ! pkg ? outputSpecified || ! pkg.outputSpecified
      then pkg.${output} or pkg.out or pkg
      else pkg;
```

The source code for the library functions for nixpkgs can be found at:
[github.com/NixOS/nixpkgs](https://github.com/NixOS/nixpkgs)

```bash
    git clone --depth 1 https://github.com/NixOS/nixpkgs
```

The "--depth 1" option is used to create a shallow clone with a history
truncated to a single commit. This is done to reduce the size of the download.

The library functions are downloaded through the file: nixpkgs/lib/default.nix.
