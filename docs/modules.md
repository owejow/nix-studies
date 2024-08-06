# Nix Module System

A Nix module system is a language for handling configuration, that is
implemented in the nix library. The module system adds type checking,
composition and extensibility.

The entry point for evaluating a nix module is the "lib.evalModules" function.

The type declarations for nix modules are defined in "lib/types.nix" file.
Types such as: unspecified, bool, booByOr, int, ints.unsigned, ints.positive,
ints.u8, ints.u16, ints.32 and ints.u64.

## Structure of a NixOS module

A nix module is a function that takes an attribute set and returns an attribute
set. A module can declare an option, which defines the attributes that are
allowed in the final attribute set. A module can define values for options it
defines or options defined in other modules.

The simplest module is a function that takes any attribute set and returns an
empty attribute set:

```nix
    {...}: { }
```

The structure of a full NixOS module is provided below:

```nix
    {config, pkgs, ...}: {
        imports = [
            # paths to other modules
        ];
        options = {
            # option declarations
        };
        config = {
            # option definitions
        };

    }
```

The variable pkgs contains nixpkgs by default. Its path is derived from the
NIX_PATH. Not clean on the best mechanism to provide an alternative nixpkgs.
The config contains the full system configuration of all the imported modules.
The config and pkgs parameters can be omitted if the function does not
reference pkgs or config.

The imports section enumerates paths to other modules taht will be included in
the evaluation. The default set of options imported by nixos is defined in the
module/module-list.nix file. These files do not need to be explicitly added
to the import list.

The options contains a nested set of option declarations. An option declaration
specifies the name, type and descripton of a configuration option. It is not
legal to declare an option that has note been declared inside any module.

A sample option declaration is provided below:

```nix
    {
        options = {
            name = mkOption {
                type = types.bool;        # type of option. If not set unspecified behaviour is used
                default = 0;        # default value to use if no value is provided. If not provided users
                                    #       must provide a value.
                defaultText = 0;    # Textual representation of default value to be printed in the manual
                example = 20;       # example value to appear in the manual
                description = 20;   # description for use in the NixOS manual. Can be in markdown format
            };
        };
    }

```

The config contains a nested set of option definitions

## Option Types

Option types define constraints the values that a module option can take. The
simplest available builtin types in the module system are Basic types. The
values will be merged if there are multiple definitions for an option.

### Basic Types

- types.bool : can take on values of true or false.
- types.boolByOr : can be true or false. Definitions are merged with logical OR
  operator.
- types.path : a filesystem path. In case of derivations it is preferred to use
  the more specific types.packages.
- types.pathInStore : a path that is contained in the Nix store.
- types.package : a top-level store path. It can be an attribute set that points to the
  store path. A derivation or a flake input will work.
- types.enum : one element of a list. as an example ["north" "west" "south" "east"]. This option
  does not support merging.
- types.anything : accepts any value and recursively merges attribute sets together.
- types.raw : a type that performs no checking, merging or nested evaluation. Use
  this type when checking, merging and nested valuations are not desirable.
- types.optionType : the type of an options type. It is used as the type for \_module.freeformType.
- types.pkgs : a type for the top-level nixpkgs set.

### Numeric types

- types.int : a signed integer
- types.ints.{s8, s16, s32} : signed integers with legnth of 8, 16, and 32-bits.
  The integers will go from -2^(n/2) to 2^(n/2)-1 respectively. For 8-bits the
  range would span : -128 to 127.
- types.ints.unsiged : an unsigned integer that is >=0
- types.ints.{u8, u16, u32}: unsigned integers with fixed length (8, 16, 32 bits). They
  go from 0 to 2^(n-1). An 8-bit integer will go from 0 to 255 for 8-bits.
- types.ints.between <lowest> <highest>: an integer between the lowest and highest values.
- types.port : a port number. This type is an alias to types.ints.u16
- types.float : a floating point number
- types.number : a signed integer or floating point number. No conversion is done between
  the two types. Multiple equal definitions can only be merged if they have the same type.
- types.numbers.between lowest highest: an integer floating point number between lowest and
  highest (both inclusive).
- types.numbers.nonnegative : a nonnegative integer or floating point number. The number should
  be >= 0.
- types.numbers.postitive : a positive integer or floating point number that is >= 0.

### String Types

- types.str : a string. multiple defintions can not be merged.
- types.sepratedString <sep> : a string. multiple defintions are concatenated with <sep>
- types.lines : a string. multiple definitions are concatenated with a newline "\n"
- types.commas : a string. multiple definitons concatenated with a comma
- types.envVar : a string. multiple defintions are concatenated with a colon ":"
- types.strMatching : a string matching a specific regular expression. Multiple defintions
  can not be merged.

### Submodule types

- types.submodule <o> : a set of submodules <o>. <o> can be an attribute set, a function
  returning an attribute set, or apath to a file containing a value. This option is
  equivalent to:

  ```nix
      types.submoduleWith { modules = toList o; shorthandOnlyDefinesConfig = true; }
  ```

- types.submoduleWith { modules, specialArgs ? {}, shorthandOnlyDefinesConfig ? false} :
  This type offers more flexibilty than the type.submodule type.
  - The argument "modules" is the list of modules to be used by default for
    this submodule type. This gets combined with all option defintions to build a
    final list of modules.
  - The "specialArgs" attribute
    sets extra arguments that are passed to the module function. The \_module.args option
    should be used instead for most arguments since it allowes overrideing. The specialArgs
    should only be used for arguments that can't go through the module fixed-point. For example
    overriding the lib argument is a good use case for using specialArgs. The lib argument
    is itself used to define_module.args and therefore can not be overriden inside \_modue.args.
  - shorthandOnlyDefinesConfig : determines whether the definitions of this type should
    default to the config section of a module if this attribute is true. Enabling this
    option only has a benefit if the submodule defines an option named config or options.
    When the option is true, the option can be set with the-submodule.config = "value" instead
    of the-submodule.config.config. When a module does not set the config or options keys,
    all keys are interpreted as option definitions in the config section. Enabling this option
    puts all attributes in the config section.
- types.deferredModule : represens a mdule value, such as a module file or a configuration.
  It can be set multiple times. Deferred modules can be imported into "other" fixedpoints,
  such as submodules.

### Union types

Union type is a type that is valid if the value is valid for at least one of the specified types.

- type.either <t1> <t2> : The value can be either of type <t1> or type <t2>. Multiple values
  can not be merged
- types.oneOf [ <t1> <t2> ...] : value is one of the specified types
- types.nullOr <t> : value is null or of type <t>

### Sum types

A sum type is conceptually a types.enum, where each valid item is paired with at least a type.

- types.attrTag {<attr1> = <option1>; <attr2> = <option2>; ... } : an attribute set containing
  one attribute, whose name must be picked from the attribute set (<attr1> etc.) and whose
  value consistes of definitions that are valid for the corresponding option (<option1> etc.).
  These types appear in the documentation as attribute-tagged union.

### Composed Types

Composed types are types that take a type as a parameter.

- types.listOf <t> : a list of <t> type. (e.g. listOf int). Multiple definitions are merged
  with list concatenation.
- types.attrsOf <t> : an attribute set of where all the values of of type <t>.
  Multiple definitions result in the joined attribute set. The values for this
  type are strict, which means that values of this type can not depend on other
  attributes.
- types.lazyAttrsOf <t> : an attribute set where all the values are of type
  <t>. Multiple definitions result in the joined attribute set. This is the lazy version of
  types.attrsOf.
- types.unique <t> : ensures that type <t> cannot be merged. It ensures that
  the option definition is only provided once.
- types.unique {message = <m> } <t> : ensures that type <t> cannot be merged.
  prints message <m> after the line: "the option <option path> is defined
  multiple times. and before the list definition locations."
- types.coercedTo <from> <f> <to> : type <from> will be coerced to type <to>
  using the function <f>. The function <f> should take an argument of type <from>
  and return an argument of type <to>. Can be used to preserve backward
  compatibility if an option's type is changed.

## Sources

- [Writing Nixos Modules](https://nixos.org/manual/nixos/unstable/index.html#sec-writing-modules)
- [Nix Module System (nixos.asia)](https://nixos.asia/en/nix-modules)
- [Types File](https://github.com/NixOS/nixpkgs/blob/master/lib/types.nix)
- [A Basic Module (nix.dev)](https://nix.dev/tutorials/module-system/a-basic-module/)
- [Module System Deep Dive (nix.dev)](https://nix.dev/tutorials/module-system/deep-dive)
