---
title: Combinators in Isolation
weight: 22
---

# Combinators in Isolation

In Java, optionals are instances of the 
[generic type](https://docs.oracle.com/javase/tutorial/java/generics/types.html)
`Optional<T>` which has a type parameter `T` for the type of wrapped elements.
The combinators `map`, `filter` and `flatMap` discussed with streams
are also available for optionals. 

## The `map` combinator

The `map` combinator has the following signature[^bounds].

[^bounds]: Real signatures might be more general than what is shown here because we specialize
    [bounded type parameters](https://docs.oracle.com/javase/tutorial/java/generics/bounded.html).

```java
<R> Optional<R> map(Function<T,R> function);
```
Apart from the result type, the type is the same as the type of `map` for streams.
The following tests assert that `map` applies the given function to the wrapped element
if one is present.

```java
@Test
void testThatMapOnEmptyYieldsEmpty() {
  final Optional<String> empty = Optional.empty();
  final Optional<Integer> result = empty.map(w -> w.length());
  assertTrue(result.isEmpty());
}

@Test
void testThatMapAppliesGivenFunctionToPresentElement() {
  final Optional<String> word = Optional.of("Hello");
  final Optional<Integer> result = word.map(w -> w.length());
  assertEquals(Optional.of(5), result);
}
```

In the first test the lambda expression is not called.
In the second test it is applied to the wrapped string
and the result is wrapped in a new optional value.

The static methods `Optional.empty` and `Optional.of` can be used to create optional values.
`Optional.of` throws an exception when `null` is passed as argument.
There is also `Optional.ofNullable` which returns an empty optional value when passed `null`.
The methods `isEmpty` and `isPresent` can be used to check if a value is present.

## The `filter` combinator

The `filter` combinator has the following signature.

```java
Optional<T> filter(Predicate<T> predicate);
```

Again, the type resembles the type of `filter` we have already seen for streams.
The following tests assert that `filter` applies the given predicate to a wrapped element.

```java
@Test
void testThatFilterRemovesNonMatchingElement() {
  final Optional<String> word = Optional.of("Hello");
  final Optional<String> result = word.filter(w -> w.length() > 6);
  assertTrue(result.isEmpty());
}

@Test
void testThatFilterKeepsMatchingElement() {
  final Optional<String> word = Optional.of("Optionals");
  final Optional<String> result = word.filter(w -> w.length() > 6);
  assertEquals(word, result);
}
```

Only matching elements are kept in the result.

## The `flatMap` combinator

The `flatMap` combinator has the following signature, resembling `flatMap` for streams.

```java
<R> Optional<R> flatMap(Function<T, Optional<R>> function);
```

The following tests assert that `flatMap` applies the given function to the wrapped element
(if one is present) and returns its result.

```java
@Test
void testThatFlatMapOnEmptyYieldsEmpty() {
  final Optional<String> empty = Optional.empty();
  final Optional<Integer> result =
      empty.flatMap(w -> w.chars().boxed().findFirst());
  assertTrue(result.isEmpty());
}

@Test
void testThatFlatMapOnPresentReturnsEmptyResultOfGivenFunction() {
  final String value = "";
  final Optional<String> word = Optional.of(value);
  final Optional<Integer> result =
      word.flatMap(w -> w.chars().boxed().findFirst());
  assertTrue(result.isEmpty());
}

@Test
void testThatFlatMapOnPresentReturnsPresentResultOfGivenFunction() {
  final String value = "Hello";
  final Optional<String> word = Optional.of(value);
  final Optional<Integer> result =
      word.flatMap(w -> w.chars().boxed().findFirst());
  assertEquals(Character.codePointAt(value, 0), result.get());
}
```

The terminal operation `findFirst` on streams is an example of a method 
that may have "no result."
It returns the first element of a stream represented as optional value
that is empty if the stream is empty.
The method `get` on optional values used in the third test
returns the wrapped element if one is present.
If not, the call throws an exception.
The API documentation for `get` states that the method `orElseThrow` 
with the same behaviour should be used instead,
but we will stick with `get` in this tutorial.

## Task: Reasoning

Think about properties that can be used to manipulate or reason about
expressions involving the presented combinators
and write more tests to check these properties.
Can you express some of the presented combinators using others
in a way that would allow arbitrary applications of one combinator
to be replaced by a corresponding application of another?
