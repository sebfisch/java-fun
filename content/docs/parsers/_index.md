---
title: Parsers
weight: 30
---

# Parsers

John Backus received the 
[Turing Award](https://en.wikipedia.org/wiki/Turing_Award)
in 1977:

> For profound, influential, and lasting contributions 
> to the design of practical high-level programming systems,
> notably through his work on FORTRAN,
> and for seminal publication of *formal procedures
> for the specification of programming languages*.

The mentioned formal procedures include the
[Backus-Naur form (BNF)](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form)
which is a notation for grammars of formal languages
initially developed to formalize the syntax of
the programming language ALGOL.

Parsers transform text into structured data.
They are a prime example for the combinatorial style of programming
because more complicated parsers can be defined in terms of simpler ones.
We will see that the definition of a parser in combinatorial style
reflects a grammar for its parsed language
and its graphical representation as so called
[railroad diagram](https://en.wikipedia.org/wiki/Syntax_diagram).

Like in the previous sections,
we will first look at parser combinators 
[in isolation](tests)
before discussing a more realistic example
of [using them](definition) to define a parser for arithmetic expressions
with precedence rules for binary operators.
Unlike in the previous sections,
we will also discuss how to [implement parser combinators](implementation),
which are not predefined in Java.

