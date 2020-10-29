---
title: Streams
weight: 10
---

# Streams

Java provides a 
[Stream API](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/stream/Stream.html)
for data processing with higher-order functions.
Stream processing consists of three parts:

  * creating a stream of data elements,
  * transforming the elements of the stream uniformly, and finally
  * collecting all elements into a processed result.

The `map`, `filter`, and `flatMap` combinators are used in the second part
and allow to implement complex transformation algorithms in a compositional way.

In the third part, the stream can be processed one-element-at-a-time using
[Iterators](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/Iterator.html)
or in bulk using
[Collectors](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/stream/Collector.html).
Bulk processing has advantages both conceptually
(because we can think about operations more abstractly)
and for performance reasons
(because data can be processed in parallel automatically.)

In this tutorial we will discuss creating and collecting streams only to the extend
necessary to follow our main topic, which is how to use `map`, `filter`, and `flatMap`.

We will start by writing [unit tests](tests) for these methods 
to demonstrate how they can be used in isolation.
Then we will write a small command line application 
to demonstrate [stream programming in practice](practice).
The practical example will be an opportunity to discuss how functional programming patterns
for streams interact with exception handling and resource management.
