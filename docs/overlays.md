---
title: "Nix Overlays"
---

# Nix Overlays

An overlay is a basic function that describes the changes that we want
to overlay on top of an existing package collection.

## Overriding Packages for All Consumers

Simple example to override library-a in nixpkgs:

```nix
    final: prev: {
        library-a = prev.library-a.overrideAttrs (...);
    }
```

The function accepts two parameters and returns an attribute set. These
attributes are laid over the initial pkgs attribute. If the attribute library-a
did not exist, it would be added to pkgs. By performing an overlay of a library
all packages that rely on the library need to be recompiled, as it likely would
not exist in the nixpkgs cache.

## Overriding Packages for A Single Consumers

The below example only app-c will use a variant of library-b, whose library-c
dependency has been changed.

```nix
    final: prev: {
        app-c = prev.app-c.override {
            library-b = prev.library-b.override {
                library-a = prev.library-a.overrideAttrs (...);
            };
        };
    }
```

## Meanings of Final and prev

Two parameters are required to define an overlay function: final and prev. The
**prev** argument refers to the pkgs attribute before the overlay is applied.
The **final** argument is the result of the overlay after all overlays have
been applied.

the returned attribute set returned by the function is a modified version
of the original pkgs.

## When to Use **prev** or **final**?

Two general rules for when to choose between **prev** and **final**:

### Graft-Preserving Rule Flavor

1. Use **prev** by default
2. Use **final** when referencing a package/derivation from some other package

This implementation would enable the efficient implementation of the grafting
feature. Generally an overlay that contains a critical security update would
trigger a rebuild on the patched package and all depending packages. grafting
allows the rebuild of only the patched packages. All dependent packages are
dynamically linked to the patched package. This approach has not taken off.

## Everything is Overridable

1. Use **final** by default
2. use **prev** if you override a symbol to avoid infinite recursion

Everything is overridable by this rule.

## Adding a New Package to Nixpkgs

This overlay adds a new package called **hello-script**

```nix
    # file: overlay-hello.nix

    final: prev: {
        hello-script = prev.writeShellScriptBin "hello" "echo Hello World!";
    }

```

How to use the overlay _overlay-hello.nix_

```shell
    nix repl

    nix-repl> overlayFunction = import ./overlay-hello.nix

    nix-repl> pkgs = import <nixpkgs> {
                overlays = [ overlayFunction ];
              }

    nix-repl> :b pkg.shello-script

```

Using an overlay that utilizes callPackage:

```nix
    final: prev: {
        myPackage = prev.callPackage ./path/to/default.nix { };
    }

```

## Overlay an Existing Package

```nix
final: prev: {
  hello = prev.hello.overrideAttrs (oldAttrs: {
    postPatch = ''
      substituteInPlace src/hello.c --replace "world!" "Nixcademy!"
    '';
    # to disable the unit tests from running
    doCheck = false;
  });
}
```

In the above example we would get an infinite recursion if we used
final.hello instead of prev.hello. When overriding an existing
symbol without renaming it you should use prev instead of final.

## Overriding Nested Attribute Sets

Adding or overriding existing symbols like packages or nix library
functions is more complicated.

The three main ways to do this are:

- overriding plain nested Sets
- overriding extensible nested sets
- overriding scopes

### Overriding plain nested sets

Add pkgs.lib.plusOne function:

```nix
    final: prev {
        # need the use prev.lib or { } to prevent dropping all pre-existing
        # symbols if it was not run on a pre-existing lib set
        lib = prev.lib or { } // {
            plusOne = x: x + 1;
        }
    }

```

Placing a function on the pkgs.lib.subCategory.plusOne

```nix
    final: prev: {
      lib = prev.lib or { } // {
        subCategory = prev.lib.subCategory or { } // { plusOne = x: x + 1; };
      };
    }
```

## Overriding Extensible Nested Sets

The **extend** function inside of lib provides another overlay mechanism
to extend the lib subattribute set.

```nix
    final: prev: {
        lib = prev.lib.extend (libFinal: libPrev: {
            plusOne = x: x + 1;
        });
    }

```

The [_makeExtensible_](https://github.com/NixOS/nixpkgs/blob/master/lib/fixed-points.nix#L376) function allows us to make overridable attribute sets.

## Overriding scopes

Nested sets can contain main related packages that refer to each other. The
[makeScope](https://nixos.org/manual/nixpkgs/stable/#function-library-lib.customisation.makeScope)
function can be used to accomplish this. The makeScope provides a specialized
callPackage function that makes it easier to obtain dependencies. Otherwise
users for pkgs.callPackages would have to write out nested paths.

A scope can be overriden using the following technique:

```nix
final: prev: {
  gnome = prev.gnome.overrideScope (gnomeFinal: gnomePrev: {
    updateScript = gnomePrev.updateScript.override {
      versionPolicy = "odd_unstable";
    };
  });
}
```

## Don't use **rec** Attribute Sets

In the following example pkg-b explicity consumes pkg-a. If
someone adds an overlay to pkg-a then pkg-b would not notice it.jjj

```nix
    final: prev: rec {
       pkg-a = prev.callPackage ./a { };
       pkg-b = prev.callPackage ./b { dependency-a = pkg-a; }
    }

```

To remedy is to use the overlay mechanism:

```nix
    final: prev: {
       pkg-a = prev.callPackage ./a { };
       pkg-b = prev.callPackage ./b { dependency-a = final.pkg-a; }
    }

```

## Don't Reference other packages via prev

The following example does not add global patches or changes the version
of the symbols clangStdenv or boost185:

```nix
final: prev: {
  myCppProject = prev.callPackage ./my-project {
    stdenv = prev.clangStdenv;
    boost = prev.boost185;
  };
}
```

Solution is to use final:

```nix
final: prev: {
  myCppProject = prev.callPackage ./my-project {
    stdenv = final.clangStdenv;
    boost = final.boost185;
  };
}
```

## Don’t Use External Parameters

```nix
# file: overlay.nix

{ boost }:
final: prev: {
    myPackage1 = prev.callPackage ./my-package2 { inherit boost; };
    myPackage2 = prev.callPackage ./my-package2 { inherit boost; };
    myPackage3 = prev.callPackage ./my-package3 { inherit boost; };
    # ...
}
```

The above overlay could be composed like so:

```nix
pkgs = import ./path/to/nixpkgs {
  overlays = [
    (import ./overlay.nix { boost = pkgs.boost185; })
  ];
}
```

The issue is that if many packages consume this parameter each one is locked
into this value. A better approach would be to create another package symbol
and reuse it:

```nix
final: prev: {
  myBoostVersion = final.boost185;
  myPackage1 = prev.callPackage ./my-package2 { boost = final.myBoostVersion; };
  myPackage2 = prev.callPackage ./my-package2 { boost = final.myBoostVersion; };
  myPackage3 = prev.callPackage ./my-package3 { boost = final.myBoostVersion; };
  # ...
}
```

The value of myBoostVersion can be changed in a later overlay like so:

```nix
pkgs = import ./path/to/nixpkgs {
  overlays = [
    (import ./overlay.nix)
    (final: prev: { myBoostVersion = final.boost180; })
  ];
}
```

## Beware of Importing Extra nixpkgs to an Overlay

## Sources

- [Nixpkgs Overlays Techniques and Best Practices](https://nixcademy.com/posts/mastering-nixpkgs-overlays-techniques-and-best-practice/)
- [NixOs Wiki Overlays](https://nixos.wiki/wiki/Overlays)
- [Do's and Dont's of Overlays](https://flyingcircus.io/news/detailsansicht/nixos-the-dos-and-donts-of-nixpkgs-overlays)
- [Shipping Security Updates](https://github.com/NixOS/nixpkgs/pull/10851)
- [NixOs Security Update-Update](https://www.theshortlog.com/posts/nix-security-updates-update/)
