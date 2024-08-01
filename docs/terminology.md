--- title: "Nix Terminology" date: 2024-07-31 ---

# Flake Terms

Terms taken from: [Concepts](https://zero-to-nix.com/concepts)

**Caching**: Nix uses caching to make building packages faster and more
efficient. This in turn makes other Nix operations that involve building
packages, like creating development environments and standing up NixOS
environments, faster and more efficient as well.

**Channels**: A Nix channel is a mechanism used by the legacy Nix CLI to keep
Nixpkgs up to date. In the new Nix CLI, channels have been replaced by flakes.

**Closures**: a package's closure encapsulates all of the packages required to
build or run it (as well as the packages required to build those packages, and
so on). A package can have zero packages in its closure, because no other
packages are required to build it, or it can have many. Whenever you build a
package, Nix always realises its entire closure in a sandboxed environment.

**CLI**: The unified CLI is a still-experimental way of using Nix that involves
just one executable called nix. All Nix functionality is wrapped into this
tool, including commands like nix build for building packages instead of the
old nix-build tool, nix develop for activating Nix development environments
instead of the old nix-shell tool, nix store for managing the Nix store instead
of the old nix-store tool, and more. Because the unified CLI isn't yet
official, it needs to be explicitly enabled in your Nix configuration by adding
nix-command to your experimental-features list.

**Declarative Programming**: A programming paradigm used by Nix that emphasizes
what you want to build rather than how things are built

**Derivation**: A derivation is an instruction that Nix uses to realise a Nix
package. They're created using a special derivation function in the Nix
language. They can depend on any number of other derivations and produce one or
more final outputs. A derivation and all of the dependencies required to build
it—direct dependencies, and all dependencies of those dependencies, etc—is
called a closure.

**Determinate Nix Installer**: A fast, stable Nix installer created by
[Determinate Systems](https://determinate.systems/).

**Development Environment**: is a derivation which builds a shell system
environment. This environment is hermetically sealed off from the host, which
means that no other dev environment can make changes or override tooling in it.

**Ecosystem**: The Nix ecosystem is a software ecosystem that has the Nix
package manager and language at its core but also includes a vast network of
related tools and systems built on top of Nix. On the official side, the Nix
ecosystem includes NixOS, NixOps, Hydra, Home Manager, and other projects. On
the unofficial side, the Nix ecosystem includes everything from language
libraries and helpers to package repositories to developer tools and far
beyond.

**Flake Hub**: FlakeHub is a platform for discovering and publishing Nix flakes
built by Determinate Systems. [FlakeHub](https://flakehub.com/)

**Flakes**: Nix flake is a directory with a flake.nix and flake.lock at the
root that outputs Nix expressions that others can use to do things like build
packages, run programs, use development environments, or stand up NixOS
systems. If necessary, flakes can use the outputs of other flakes as inputs.

**Hermeticity** is a property of Nix builds, which isolates them from the host
system via various mechanisms. This results in a system where the same set of
source inputs will always map to the same build outputs, because changes on the
host can not affect a build.

**Incremental builds** are build processes that don't need to build the entire
dependency tree of a software artifact every time because they use mechanisms
like intelligent caching to avoid rebuilding artifacts that are already
available. Nix is one of several available build systems that offers
incremental builds.

**Nix**: Nix refers to a few different things: The pure and functional
programming language, The Nix CLI, The overall Nix package management system

**Nix Language**: The language you use to define Nix package builds,
development environments, NixOS configurations, and more.

**NixOS**: NixOS is a Linux distribution based on Nix. It's unique amongst
distributions because it enables you to use the Nix language to declaratively
configure your operating system in a configuration.nix file.

**Nixpkgs**: Nixpkgs (pronounced "Nix packages") is a big collection of Nix
expressions, packages (who'd have thought...), and build utilities for the Nix
ecosystem. It is sometimes also called a "Nix standard library" (or Nix
stdlib).

**Packages**: Nix packages are self-contained bundles that are built using
derivations and provide some kind of software or dependencies of software on
your computer. A package can have dependencies on other packages, which are
encoded in its closure. Each package lives at a unique path in the Nix store
indexed by its name and its input hash.

**Package Management**: is the art of building and distributing software
packages.

**Pinning a dependency**: refers to the act of specifying an exact revision for
Nix to use. This is particularly interesting in relation to Nix's
reproducibility guarantees.

**Provenance**: Provenance is a term that's basically synonymous for the
origins of a thing. In software, provenance usually refers to the build process
that created an artifact (a program, a file, a smartphone app, and so on).

**Realisation**: is the process whereby a Nix derivation is transformed into a
package. While a derivation is essentially a plan for a package, realisation is
the build process turns that plan into an actual output directory full of
content.

**Reproducibility**: is a software design concept, where an operation for the
same inputs yields the same output. This is an important property for Nix,
because hashes are based on the inputs of a build, and thus it must be
guaranteed that the same inputs yield in the same build output.

**Sandboxing**: Isolating the Nix build process from everything else on your
system

**Store**: The Nix store is a storage system in your filesystem with its root
at /nix/store by default. The Nix daemon stores several things in the Nix
store:

- Derivations: stand-alone files suffixed with .drv such as
  "/nix/store/<hash>-my-package.drv"
  - The products of derivations: a folder containing build outputs such as
    "/nix/store/<hash>-my-package/"
  - Patches: stand-alone files suffixed with .patch such as
    "/nix/store/<hash>-my-package.patch"

**System Specificity**: In Nix, the term system specificity expresses the fact
that all Nix derivations have a system attribute that specifies which system
the derivation is built on. In order to realize a Nix derivation into a
package, each derivation in the package's closure needs to be supported on the
target system. So if you want to build the package foo on an x86_64-linux
system, any dependency of the foo derivation needs to be supported on
x86_64-linux.
