---
title: Combinators in Isolation
weight: 32
---

# Combinators in Isolation

Parsers are instances of the 
[generic type](https://docs.oracle.com/javase/tutorial/java/generics/types.html)
`Parser<A>` which has a type parameter `A` for the type of produced results.
Like streams and optionals,
parsers also provide implementations for the combinators
`map`, `filter` and `flatMap`.
Before we observe how to combine parsers, hoewever,
we first look at how to create basic parsers.

## Basic parsers

Basic parsers differ in how much input they consume
as well as what result they produce.

### Single characters

The static method `Parser.forChar` creates a parser for a single character
of type `Parser<Character>`.
The following test asserts than
the created parser consumes exactly one character from the input
which is also the result it produces.

```java
@Test
void testThatParserForCharYieldsFirstInputChar() {
  final Character character = 'X';
  final Parser<Character> parser = Parser.forChar();
  final String input = character.toString();
  final Stream<Character> results = parser.results(input);
  assertStreamEquals(Stream.of(character), results);
}
```

This test uses the method `results`,
which returns a stream of possible results
the parser can produce when consuming the given input.
Often the stream will contain exactly one result.
It may contain more if the parser is ambiguous
or no result at all if parsing fails (see below.)

### Multiple characters

The static method `Parser.forString` creates a parser of type `Parser<String>`.
It takes a predicate of type `Predicate<Character>` as argument
and consumes characters from the input as long as they satifsy the given predicate.
The following test asserts that the resulting parser produces the consumed string.

```java
@Test
void testThatParserForStringYieldsMatchingInput() {
  final Parser<String> parser = Parser.forString(Character::isLetter);
  final String input = "hello";
  final Stream<String> results = parser.results(input);
  assertStreamEquals(Stream.of(input), results);
}
```

Like in the previous test the input matches what the parser expects.

### Parsing failure

Parsers can fail if there is insufficient input or
if there is additional unrecognized input.
The following test asserts that 
a parsers for letters cannot parse digits.

```java
@Test
void testThatParserForLettersFailsOnDigits() {
  final Parser<String> parser = Parser.forString(Character::isLetter);
  final String input = "0815";
  final Stream<String> results = parser.results(input);
  assertStreamEquals(Stream.empty(), results);
}
```

The static method `Parser.failure` creates a parser
that does not consume any input
and also does not produces a result.
The following tests assert that the created pasers fail
regardless of the given input.

```java
@Test
void testThatFailingParserFailsForEmptyInput() {
  final Parser<Object> parser = Parser.failure();
  final String input = "";
  final Stream<Object> results = parser.results(input);
  assertStreamEquals(Stream.empty(), results);
}

@Test
void testThatFailingParserFailsForNonEmptyInput() {
  final Parser<Object> parser = Parser.failure();
  final String input = "X";
  final Stream<Object> results = parser.results(input);
  assertStreamEquals(Stream.empty(), results);
}
```

Parsers that always fail do not seem very useful on their own.
However, they are sometimes handy to create complex parsers from simpler ones.

### Explicit results

The static method `Parser.of` also creates a parser that does not consume any input.
It takes an argument which is produced by the resulting parser.
The following test asserts that the result is produced without reading any input.

```java
@Test
void testThatParserOfGivenResultSucceedsOnEmptyInput() {
  final Integer number = 42;
  final Parser<Integer> parser = Parser.of(number);
  final String input = "";
  final Stream<Integer> results = parser.results(input);
  assertStreamEquals(Stream.of(number), results);
}
```

Again, such parsers are not useful on their own
but will be handy to create complex parsers from simpler ones.

## Complex parsers

The combinators `map`, `fiter`, and `flatMap` can be used
to create complex parsers from simpler ones.
We will also see the `or` combinator that we have not discussed before.

### Adjusting results with `map`

The signature of the `map` combinator for parsers
resembles the `map` signatures we have seen for other types.

```java
<B> Parser<B> map(Function<A, B> function);
```

The method returns a parser of type `Parser<B>` 
when called on a parser of type `Parser<A>`
with an approriate function as argument.
The following test asserts that `map` can be used to adjust the result of a parser.

```java
@Test
void testThatMapAdjustsResultOfParser() {
  final Parser<String> parser = Parser.forString(Character::isLetter);
  final Parser<Integer> adjusted = parser.map(String::length);
  assertStreamEquals(Stream.of(5), adjusted.results("hello"));
}
```

### Restricting results with `filter`

The signature of the `filter` combinator for parsers
resembles the `filter` signatures we have seen for other types.

```java
Parser<A> filter(Predicate<A> predicate);
```

The method returns a new parser with the same type as the one it is called on.
However, the resulting parser produces only those results of the original parser
that satisfy the given predicate and fails for other results.

```java
@Test
void testThatParserForDigitYieldsInputDigit() {
  final Character digit = '7';
  final Parser<Character> p = Parser.forChar().filter(Character::isDigit);
  assertStreamEquals(Stream.of(digit), p.results(digit.toString()));
}

@Test
void testThatParserForDigitFailsForNonDigitInput() {
  final Parser<Character> p = Parser.forChar().filter(Character::isDigit);
  assertStreamEquals(Stream.empty(), p.results("X"));
}
```

### Sequencial parsers with `flatMap`

The signature of the `flatMap` combinator for parsers
resembles the `flatMap` signatures we have seen for other types.

```java
<B> Parser<B> flatMap(Function<A, Parser<B>> function);
```

The resulting parser first consumes input according to the parser it is called on
and then consumes more input according the parser returned by the given function.
The result produced by the second parser is also the result produced by the
parser returned by `flatMap`.
The following test asserts that parsers can be sequenced using `flatMap`
and results can be combined by adjusting the result of the second parser.

```java
@Test
void testThatSequentialParserCanCombineResults() {
  final Parser<String> digits = Parser.forString(Character::isDigit);
  final Parser<String> letters = Parser.forString(Character::isLetter);
  final Parser<String> digitsAndLetters = //
      digits.flatMap(ds -> letters.map(ls -> ds + ls));
  final String input = "12abc";
  assertStreamEquals(Stream.of(input), digitsAndLetters.results(input));
}
```

### Alternative parsers with `or`

The `or` combinator has the following signature.

```java
Parser<A> or(Parser<A> parser);
```

The method can be called on a parser of type `Parser<A>`
and returns a parser of the same type.
The resulting parser consumes input according to the parser the method is called on
or according to the parser provided as argument.
The produced result is the result produced by the successful parser.

```java
@Test
void testThatParserForDigitsOrLettersParsesEitherButFailsOnBoth() {
  final Parser<String> digits = Parser.forString(Character::isDigit);
  final Parser<String> letters = Parser.forString(Character::isLetter);
  final Parser<String> digitsOrLetters = digits.or(letters);

  assertStreamEquals(Stream.of("12"), digitsOrLetters.results("12"));
  assertStreamEquals(Stream.of("abc"), digitsOrLetters.results("abc"));
  assertStreamEquals(Stream.empty(), digitsOrLetters.results("12abc"));
}
```

## Tasks

### Ambiguous parsers

Some parsers can consume their inputs in more than one way
and as a consequence produce more than one result.
A simple example would be a parser that is combined with itself using `or`.
Another, more subtle, example would be a sequential parser
whose parts can be combined in different ways to parse given input.

Write at least two more tests for parsers
documenting the behaviour of alternative and sequential parsers
with more than one result.

### Reasoning

Can you spot a connection between the `or` combinator for parsers
and the corresponding logic operation on boolean values?
Write another test to document such a connection.

