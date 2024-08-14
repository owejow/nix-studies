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

The fix program is a fixed point combinator. Additional information regarding
this topic can be found at: (Fixed Point
Combinator)[https://en.wikipedia.org/wiki/Fixed-point_combinator]

A funtional programming language consists entirely of functions. The functions
are like ordinary mathematical functions. The special characteristics of
functional languages includes:

1. no assignment statements: Once a value is given it never changes
2. no side effects: functions only compute results based on the input
3. order of execution is irrelevant
4. programs are referentially transparent: variables can be freely replace variables by their value

## Laziness

An excellent explanation of Laziness in Nix can be found at:
[What You Need to Know About Laziness](https://nixcademy.com/posts/what-you-need-to-know-about-laziness/).

The paper that describes NixOs functional aspects written by the original
creator of Nix: [A Purely Functional Linux Distribution](https://edolstra.github.io/pubs/nixos-jfp-final.pdf)

A decription of laziness algorithm in Nix: [Maximal Laziness](https://edolstra.github.io/pubs/laziness-ldta2008-final.pdf)

Methodology used to unwrap the outermost part of data-structures in Nix's lazy
evaluation is called Weak Head Normal Form. It is described in this [wikibooks
entry](https://en.wikibooks.org/wiki/Haskell/Graph_reduction#Weak_Head_Normal_Form)

The resolution of lazy values can lead to infinite recursion in certain cases.
The evaluator can detect these "infinite" recursions using the
[blackholing](https://www.microsoft.com/en-us/research/wp-content/uploads/1992/04/spineless-tagless-gmachine.pdf)

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

Each new variable gets assigned a parameterless function that calculates its
result. When the variable’s value computation is referenced, its parameterless
function is executed. Such a parameterless function is also called thunk.

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

## Critical lib.fixed-point.nix functions

Understanding these functions is critical to understanding the overall structure and behavior of nixpkgs:

### lib.fix

The lib.fix or fixed-point-combinator function is critical for many of the
nixpkgs features. The function is defined in a single line and seems quite
innocuous at first glance:

```nix
  fix = f: let x = f x; in x;
```

An example provided in the source code is the following:

```nix
    nix-repl> f = self: {
                    foo = "foo";
                    bar = "bar";
                    foobar = self.foo + self.bar;
                  }
    nix-repl> fix f
        # =>   { bar = "bar"; foo = "foo"; foobar = "foobar"; }
```

How does the fix function manage to resolve the "self" variables in the
function declaration g? The tricky part is how is it possible that "x" is
resolved to anything? The initial declaration of "x" is equal to "f(x)". This
seemingly circular definition is resolvable because Nix is a lazy language, it
evaluates its arguments as little as required to return a result.

Arguments in nix are passed by name inside the function. The expressions remain
as thunks, or unexecuted expressions. The nix interpreter would evaluate the above
call in the following manner:

1. Initial call to evaluate:

   ```nix
       let
           g = self: {
               foo = "foo";
               bar = "bar";
               foobar = self.foo + self.bar;
           }
       in
       lib.fix g;
   ```

2. The function fix definition gets expanded:

   ```nix
       (f: let x = f x; in x;) g
   ```

3. The function is called with argument g. In the first step
   the argument f is substituted with the variable definition for
   g:

   ```nix
       let x = (self: { foo = "foo"; bar = "bar"; foobar = self.foo + self.bar;}}) x
   ```

4. The next step is where the variable x is substtituted for the argument "self"
   for the function call:

   ```nix
       let x = {foo = "foo"; bar = "bar"; foobar = x.foo + x.bar};
   ```

5. Next we substitute the value of x into the right hand side:

   ```nix
       let x = {
                foo = "foo";
                bar = "bar";
                foobar = {foo = "foo"; bar = "bar"; foobar = x.foo + x.bar}.foo +
                         {foo = "foo"; bar = "bar"; foobar = x.foo + x.bar}.bar;
           }
   ```

6. The value of x.foo is resolved

   ```nix
       let x = {
                foo = "foo";
                bar = "bar";
                foobar = "foo" +
                         {foo = "foo"; bar = "bar"; foobar = x.foo + x.bar}.bar;
           }
   ```

7. The value of x.bar is resolved:

   ```nix
       let x = {
                foo = "foo";
                bar = "bar";
                foobar = "foo" +
                         "bar";
           }
   ```

8. The foobar expression is evaluated:

   ```nix
       let x = {
                foo = "foo";
                bar = "bar";
                foobar = "foobar";
           }
   ```

9. Amazingly the expression x = f x was evaluated without actually knowing what the
   value of x was originally. Next we return "f x" or the final attribute set
   calculated above:

   ```nix
       {
         foo = "foo";
         bar = "bar";
         foobar = "foobar";
       }
   ```

## functions inside of lib.fixed-point.nix

2. fix'

   ```nix
     fix' = f: let x = f x // { __unfix__ = f; }; in x;
   ```

3. converge

   ```nix
     converge = f: x:
       let
         x' = f x;
       in
         if x' == x
         then x
         else converge f x';
   ```

4. extends

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

5. composeExtensions

   ```nix
     composeExtensions =
       f: g: final: prev:
         let fApplied = f final prev;
             prev' = prev // fApplied;
         in fApplied // g final prev';

   ```

6. composeManyExtensions

   ```nix
     composeManyExtensions =
       lib.foldr (x: y: composeExtensions x y) (final: prev: {});

   ```

7. makeExtensible

   ```nix
     makeExtensible = makeExtensibleWithCustomName "extend";
   ```

8. makeExtensibleWithCustomName

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
- [What You Need to Know About Laziness](https://nixcademy.com/posts/what-you-need-to-know-about-laziness/)
