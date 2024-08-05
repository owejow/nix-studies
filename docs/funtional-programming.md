---
title: "Functional Programming"
---

# Overview

General notes on functional programming. Need to better understand concepts
such as lazy evaluation to make sense of constructs used in nixpkgs.

At first sight it is quite mind-boggling how such an expression evaluates until you
understand lazy evaluation.

```nix
      # code excerpt taken from lib/fixed-points.nix
      fix = f: let x = f x; in x;
```

A funtional programming language consists entirely of functions. The functions
are like ordinary mathematical functions. The special characteristics of
functional languages includes:

1. no assignment statements: Once a value is given it never changes
2. no side effects: functions only compute results based on the input
3. order of execution is irrelevant
4. programs are referentially transparent: variables can be freely replace variables by their value

## Structured Programming

Advantages of structured programming:

1. contain no goto statements
2. blocks in structured programs only have a single entry and exit
3. more tractable mathematically than their unstructured counterparts

In reality the most important of structured vs unstructured is that structured
are designed in a modular way. Small modules can be coded easily. General purpose
modules can be reused. Each module can be tested independently.

Functional programming provides mechanism to create smaller, simpler and more
generalized modules, glued together.

## Glueing Functions Together

Glueing simple functions together to make more complex ones. Functional composition
allows the creation of more complex constructs based on base functions.

## Glueing Programs Together

The program "g (f input)" will conventionally compute (f input) and then
provide the entirely result to function g. This can lead to inefficiencies if
(f input) generates too much data that is not required for function g.

In functional programming the functions f and g are glued together in strict
synchronization. The function "f" will only started once g tries to read its
value. Function f only runs long enough until g is provided the value it is
trying to read. The function f then gets suspended and waites for g to request
additional input. If g terminates without reading all of function f's output,
function f is terminated.

This method of evaluation is called "Lazy Evaluation" because function f is run
as little as possible. Lazy evaluation is uniformly applicable for every function
call. This allows any part of the program to be modularized.

The opposite of lazy evaluation is eager evaluation. A lazy algorithm is one
that does not calculate the arguments to a function until it is actually
needed.

## Lazy evaluation flavors

Two forms of lazy evaluation exist: call-by-name and call-by-need.

Call-by-name is simply delaying the computation of function arguments until the
argument needs to be inspected or evaluated. In call-by-value, the opposite of
call-by-name, arguments are evaluated before being passed to the function.

In call-by-need all thunks (or delayed computations) get associated with
labels. The results of a thunk are stored in a table that associates labels
with values. Function invocations of a lable will lookup the vlaue rather than
evaluate the thunk. The results of thunks are memoized. All expressions are
essentially thunks.

## Examples

Program to calculate primes:

```Haskell

    primes = sieve [2..]
    sieve (p:xs) = p : sieve [x | x <- xs, x `mod` p  /= 0]

```

Sample execution:

```Haskell
    take 10 primes
    # [2, 3, 4, 5, 7, 11, 13, 17, 19, 23, 29]

    takeWhile (< 10) primes
    # [2, 3, 5, 7]

```

## functions in lib.fixed-point.nix

Understanding these functions is critical to understanding the overall structure and behavior of nixpkgs:

```nix
  fix = f: let x = f x; in x;
```

```nix
  fix' = f: let x = f x // { __unfix__ = f; }; in x;
```

```nix
  converge = f: x:
    let
      x' = f x;
    in
      if x' == x
      then x
      else converge f x';
```

```nix
  extends =
    overlay:
    f:
    # The result should be thought of as a function, the argument of that function is not an argument to `extends` itself
    (
      final:
      let
        prev = f final;
      in
      prev // overlay final prev
    );


```

```nix
  composeExtensions =
    f: g: final: prev:
      let fApplied = f final prev;
          prev' = prev // fApplied;
      in fApplied // g final prev';

```

```nix
  composeManyExtensions =
    lib.foldr (x: y: composeExtensions x y) (final: prev: {});

```

```nix
  makeExtensible = makeExtensibleWithCustomName "extend";
```

```nix
  makeExtensibleWithCustomName = extenderName: rattrs:
    fix' (self: (rattrs self) // {
      ${extenderName} = f: makeExtensibleWithCustomName extenderName (extends f rattrs);
    });
```

## Sources

- [Why Functional Programming Matters](https://www.cs.tufts.edu/~nr/cs257/archive/john-hughes/whyfp.pdf)
- [Programming in Haskell](https://www.cs.nott.ac.uk/~pszgmh/pih.html)
- [Lazy Evaluation Quora](https://www.quora.com/How-is-lazy-evaluation-implemented-in-functional-programming-languages)
- [Purely Functional Data Structures](https://www.amazon.com/Purely-Functional-Data-Structures-Okasaki/dp/0521663504/ref=sr_1_1?sr=8-1)
- [Solutions to Purely Functional Data Structures](https://github.com/qnikst/okasaki)
