---
title: Combinators in Isolation
weight: 12
---

# Combinators in Isolation

In Java, streams are instances of the
[generic type](https://docs.oracle.com/javase/tutorial/java/generics/types.html)
`Stream<T>` 
which has a type parameter `T` for the type of stream elements.

## The `map` combinator

The `map` combinator has the following signature[^bounds].

[^bounds]: Real signatures might be more general than what is shown here because we specialize
    [bounded type parameters](https://docs.oracle.com/javase/tutorial/java/generics/bounded.html).

```java
<R> Stream<R> map(Function<T,R> function);
```

It expects a 
[Function](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/function/Function.html)
as argument, which is itself a generic type with type parameters 
for the argument and result types.
The argument type of the passed function needs to match the element type of the original stream.
The element type of the returned stream matches the result type of the passed function.
The following test asserts that `map` applies the given function to each element
of the stream it is called on and returns a new stream containing the results.

```java
@Test
void testThatMapAppliesGivenFunctionToEachElement() {
  final Stream<String> words = Stream.of("Hello", "Streams");
  final Stream<Integer> result = words.map(w -> w.length());
  assertStreamEquals(Stream.of(5, 7), result);
}
```

The static method `Stream.of` can be used to create a stream from given elements.
In this test we create a stream of strings and use `map` with a function
that computes the length of strings.
The lambda expression used as argument of `map` is equivalent to the following
instance of an anonymous class.

```java
new Function<String,Integer>() {
  public Integer apply(String w) {
    return w.length();
  }
}
```

The `apply` method is applied to each string in the stream `words`
and all results are collected in the new stream `result`.

Note that we do not see 
in which order a given functions is applied to the elements of a stream by `map`.
In fact it might be applied in parallel depending on how the stream was created.
All we can see is that the function is applied to each element uniformly
rather than one element at a time.

## The `filter` combinator

The `filter` combinator has the following signature.

```java
Stream<T> filter(Predicate<T> predicate);
```

It expects a
[Predicate](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/function/Predicate.html)
as argument, which is a generic type for functions that return a `boolean` result.
The argument type of the passed predicate needs to match the element type
of the original stream
which is also the element type of the returned stream.
The following test asserts that `filter` applies the given predicate to each element
of the stream it is called on and returns a new stream containing only those elements
which are accepted by the predicate, removing the others.


```java
@Test
void testThatFilterRemovesNonMatchingElements() {
  final Stream<String> words = Stream.of("Hello", "Streams");
  final Stream<String> result = words.filter(w -> w.length() > 6);
  assertStreamEquals(Stream.of("Streams"), result);
}
```

The lambda expression used as argument of `filter` is equivalent to the following
instance of an anonymous class.

```java
new Predicate<String>() {
  public boolean test(String w) {
    return w.length() > 6;
  }
}
```

The `test` method is applied to each string in the stream `words`
and those strings where `test` returns `true` are collected in the new stream `result`.

The given predicate is applied uniformly to the elements of a stream by `filter`,
not necessarily one at a time, and possibly in parallel.

## The `flatMap` combinator

The `flatMap` combinator has the following signature.

```java
<R> Stream<R> flatMap(Function<T,Stream<R>> function);
```

It expects a function as argument where the argument type needs to match
the element type of the original stream.
The result type of the function passed as argument to `flatMap`
needs to be a stream itself, and the corresponding element type
matches the element type of the returned stream.
The following test asserts that `flatMap` applies the given function to each element
of the stream it is called on and returns a new stream combining all elements of streams
returned by the given function.

```java
@Test
void testThatFlatMapCombinesStreamResults() {
  final Stream<String> words = Stream.of("Hello", "Streams");
  final Stream<Integer> result = words.flatMap(w -> w.chars().boxed());
  assertEquals(12, result.count());
}
```

The lambda expression used as argument of `flatMap` is, again, equivalent to
an instance of an anonymous class.
The method `chars` on strings returns an `IntStream`
which is a stream where the elements are primitive `int` values
(one for each character of the string.)
We use the `boxed` method to convert the `IntStream` 
into a stream of type `Stream<Integer>` 
to match the result type of the argument passed to `flatMap`.
This time we use the *terminal operation* `count` on the stream `result`
to collect all 12 elements into a single number.

The `flatMap` combinator can be used to flatten a nested string
as the following test demonstrates.

```java
@Test
void testThatFlatMapWithIdentityFunctionFlattensNestedStreams() {
  final Stream<Stream<Integer>> nested = Stream.of(Stream.of(2),Stream.of(3,4));
  final Stream<Integer> flat = nested.flatMap(s -> s);
  assertStreamEquals(Stream.of(2, 3, 4), flat);
}
```

The lambda expression passed as argument is a function that
takes a stream as argument and returns it as result.

## Task: Reasoning

Think about useful properties that can be used to manipulate or reason about
expressions involving the presented combinators
and write more tests to check these properties.
Can you express some of the presented combinators using others
in a way that would allow arbitrary applications of one combinator
to be replaced by a corresponding application of another?
