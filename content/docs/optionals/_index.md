---
title: Optionals
weight: 20
---

# Optionals

Java provides an 
[Optional API](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/Optional.html)
for handling missing values with functional combinators.

> `Optional` is primarily intended for use as a method return type where there is a clear need to represent "no result," and where using `null` is likely to cause errors.

The combinators `map`, `filter` and `flatMap` are useful for handling optional values,
but their use does not come naturally to programmers who are used to 
conventional programming patterns involving `null` checks.
After introducing the `Optional` interface via [unit tests](tests/) 
demonstrating the behaviour of selected combinators,
we show how to transform traditional `null` checks into idiomatic use of optionals.
We will also discuss anti-patterns when programming with optionals 
and how to avoid them.

At the same time, we will see another example of how the functional combinators can be used
as *powerful combining forms for building new programs from existing ones*
while employing *useful properties for reasoning about programs*
as mentioned in the introduction to this tutorial.
More specifically, we will see how programs that would conventionally use
sequencing and conditional statements 
can be expressed using expressions involving functional interfaces with optionals.
Our discussion of this combinatorial style of programming
by means of the relatively simple type of optionals
will serve as a bridge to combinatorial parsing
and can help consolidate and extend intuitions assiciated with `map`, `filter` and `flatMap`
that are independent of concrete types implementing these combinators.

