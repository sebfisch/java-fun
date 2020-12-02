---
title: Implementing Combinators
weight: 36
---

# Implementing Combinators

Until now we have seen three examples of types implementing
`map`, `filter`, and `flatMap` combinators
that can be used to define complex programs based on simpler ones.
We have also seen useful properties of these combinators
that can be used to reason about the meaning of programs.
Some of these properties were valid regardless of the concrete type they are implemented for.
As such, they help form intuitions about the meaning of those combinators
that is independent of concrete types.

We will now implement the parser combinators with such properties in mind.
More specifically, we have observed that both `map` and `filter` are special cases of `flatMap`
for different types.
We will make use of this insight by implementing `map` and `filter` in terms of `flatMap`
making sure that the mentioned properties hold for parsers by definition.

In order to focus on the essentials,
our implementation does not concern itself with orthogonal issues
such as information hiding.
This tutorial is meant to convey the essence of the implementation
and explicitly not meant to provide an implementation ready for development in production.
A production ready implementation for parser combinators in Java
would be an interersting project, which is, however, out of scope for this tutorial.

## Parsers are functions

A parser is a function that transforms text into structured data.
In order to implement sequential parsing,
we need to be able to compute intermediate results
along with remaining input that is still to be processed.
In order to implement alternative parsing,
we need to be able to handle different possibilities
for combinations of intermediate results with remaining input.

We implement parsers as a 
[functional interface](https://docs.oracle.com/javase/specs/jls/se14/html/jls-9.html#jls-9.8)
as follows.
The advantage of this representation is that we can write parsers concisely as
[lambda expressions](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)
instead of anonymous classes when implementing the combinators.
A disadvantage of this representation is that we cannot hide 
the method used to compute intermediate results.

```java
@FunctionalInterface
public interface Parser<A> {
  class Result<R> {
    final R result;
    final String remainingInput;
    Result(R result, String remainingInput) {
      this.result = Objects.requireNonNull(result);
      this.remainingInput = Objects.requireNonNull(remainingInput);
    }
  }

  Stream<Result<A>> intermediateResults(String input);
  
  // to be continued
}
```

The class `Parser.Result` is a generic type
combining an intermediate result of type `R` with remaining input.

By returning a stream of intermediate results including remaining input,
the method `intermediateResults` allows us to implement combinators
for sequential as well as alternative parsing.
When using parsers, we have used a simpler method `results`
which is implemented in terms of `intermediateResults` as follows.

```java
default Stream<A> results(String input) {
  return intermediateResults(input)
      .filter(res -> res.remainingInput.isEmpty())
      .map(res -> res.result);
}
```

For the sake of simplicity, we provide a
[default implementation](https://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html)
directly inside the `Parser` interface
instead of defining a class implementing the interface.

## Basic parsers

Basic parsers can be implemented as static methods in the `Parser` interface.
The simplest example is the `failure` method
which returns a parser that produces no results.

```java
static <R> Parser<R> failure() {
  return input -> Stream.empty();
}
```

The lambda expression used in the `return` statement
matches the method `intermediateResults`
which is the only (non-default) method of the functional interface `Parser`.
It is translated by the Java compiler into the following instance of an anonymous class.

```java
new Parser<R>() {
  @Override
  Stream<Result<R>> intermediateResults(String input) {
    return Stream.empty();
  }
}
```

We will continue to use lambda expressions to define parsers
but stop to mention anonymous classes they translate to.

The next basic parser does not consume any input
and produces the result given as argument.

```java
static <R> Parser<R> of(R result) {
  return input -> Stream.of(new Result<R>(result, input));
}
```

The created stream contains a single intermediate result
along with the complete input provided to the parser
meaning that the complete input will be provided to subsequent parsers
if there are any.

The next basic parser is the first to consume some input.
It consumes exactly one character
which is also produced as result
and passes on the rest of the input to subsequent parsers.

```java
static Parser<Character> forChar() {
  return input -> input.isEmpty()
      ? Stream.empty()
      : Stream.of(new Result<Character>(input.charAt(0), input.substring(1)));
}
```

If the provided input is empty the created parser fails 
by returning an empty stream of results.

Our final basic parser can be defined in terms of other combinators and basic parsers.

```java
static Parser<String> forString(Predicate<Character> predicate) {
  return
  Parser.forChar().filter(predicate).flatMap(chr ->
  forString(predicate).map(str -> chr + str)
  ).or(Parser.of(""));
}
```

It is implemented as alternative between

  * a parser consuming non-empty strings of characters satisfying the given predicate\
    and producing them as result
  * and a parser producing the empty string without consuming any input.

The definition is recursive:
the non-empty case is a sequence of a parser for a single character
followed by a parser for the remaining string created with a recursive call.

## Parser combinators

Parser combinators create more complex parsers based on simpler ones.
We implement them as methods with default implementations for the `Parser` interface
starting with the implementation of the `or` combinator for alternative parsing.

```java
default Parser<A> or(Parser<A> parser) {
  return input -> Stream.concat(
      intermediateResults(input),
      parser.intermediateResults(input));
}
```

The input provided to the returned parser
is passed on to the original parser as well as to the parser passed as argument.
The results of both parsers are collected into a single stream.
If one parser fails, the results of the other are produced.
If both parsers succeed, the resulting parser produces multiple results.

Next we implement `flatMap` for sequential parsing.

```java
default <B> Parser<B> flatMap(Function<A, Parser<B>> function) {
  return input -> intermediateResults(input).flatMap(parsed -> //
  function.apply(parsed.result).intermediateResults(parsed.remainingInput));
}
```

This implementation computes intermediate results based on the original parser,
applies the given function to each parsed result,
passes the remaining input to the resulting parsers,
and collects the results of those calls into a single stream.

The remaining combinators `map` and filter are implemented
in terms of `flatMap` and basic combinators
based on properties of those combinators observed previously.

```java
default <B> Parser<B> map(Function<A, B> function) {
  return flatMap(result -> Parser.of(function.apply(result)));
}

default Parser<A> filter(Predicate<A> predicate) {
  return flatMap(result ->
  predicate.test(result) ? Parser.of(result) : Parser.failure());
}
```

## Task: Add combinators

Add combinators with the following signatures to the `Parser` interface.

```java
Parser<Optional<A>> optional()
Parser<List<A>> list()
```

Define the combinators in terms of existing ones
without using `intermediateResults` directly.
Write tests documenting the behaviour of the new combinators.
