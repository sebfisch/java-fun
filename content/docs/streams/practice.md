---
title: Streams in Practice
weight: 14
---

# Streams in Practice

We now discuss a small command line application
demonstrating how streams help solving problems in practice.

## Implementing source file search

Our application will search through files in a given folder
and print all lines containing a match for a given regular expression.
We will only search in Java source files in the given folder.
When searching for lines matching the regular expression `/public static[^=]*\(/`
in our own `src` folder, our application might produce output like this.

```
/home/me/java-functional-patterns/src/main/java/sebfisch/SrcFileSearch.java
  public static void main(final String[] args) {
/home/me/java-functional-patterns/src/main/java/sebfisch/parser/Parser.java
  public static <R> Parser<R> failure() {
  public static <R> Parser<R> of(R result) {
  public static Parser<Character> forChar() {
  public static Parser<String> forStringOf(final Parser<Character> charParser) {
```

File names are printed (using their absolute path)
before all matching lines from the corresponding Java files.

To compute a stream of Java source file names in a given folder
we define the following method.

```java
static Stream<Path> walkJavaFiles(final Path root) throws IOException {
  return Files.walk(root)
      .filter(Files::isReadable)
      .filter(path -> path.toString().endsWith(".java"));
}
```

The predefined method `Files.walk` expects a `Path` as argument
and returns a stream of paths representing files or directories
contained inside the given root path.
It might throw an `IOException` which we pass on to the caller of `walkJavaFiles`.

We use `filter` twice to keep only those paths that represent
readable files with a `.java` extension.
The method reference `Files::isReadable` is, in this case,
equivalent to the lambda expression `path -> Files.isReadable(path)`.

To simplify our `main` method, we hard-code the directory and regular expressions
to the values used above.
Replacing those values with command-line arguments would be straight forward.

```java
final Path srcPath = Path.of("src");
final String regExp = "public static[^=]*\\(";
final Predicate<String> containsMatch = Pattern.compile(regExp).asPredicate();
```

When `javaFiles` contains a stream of paths returned by `walkJavaFiles`,
we can use the predicate `containsMatch` defined here to print matching lines as follows.

```java
javaFiles
    .flatMap(SrcFileSearch::readLines)
    .filter(containsMatch)
    .forEach(System.out::println);
```

Here, the method reference `SrcFileSearch::readLines` is equivalent to the
lambda expression `path -> SrcFileSearch.readLines(path)`
calling a static method that we will discuss below.

The `forEach` method expects a
[Consumer](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/util/function/Consumer.html)
as argument which is a function that does not return a result.
It applies the given consumer to each element of the stream it is called on.
The method reference `System.out::println` is equivalent to the following
instance of an anonymous class.

```java
new Consumer<Path>() {
  public void accept(Path path) {
    System.out.println(path);
  }
}
```

## Extending the application

We managed to write our application using a stream pipeline,
but as it stands it prints only matching lines and no file names.
We can extend the pipeline to print file names as follows.

```java {lineos=table,hl_lines=[2]}
javaFiles
    .peek(System.out::println); // ONLY THIS LINE IS NEW
    .flatMap(SrcFileSearch::readLines)
    .filter(containsMatch)
    .forEach(System.out::println);
```

The `peek` method expects a consumer as argument, applies it to every element,
and returns a new stream containing the same elements as the stream it was called on.
However, the evaluation order is different than the previous sentence might suggest.
The `peek` method does *not* apply the consumer to every element of the stream first,
and only then create a new stream as result.

Java streams are evaluated on demand.
The consumer passed to `peek` (in our case the method reference `System.out::println`)
is only called when an element is traversed in the resulting stream.
Elements are traversed by the terminal operation of the pipeline (in our case `forEach`),
and as a consequence the different invocations of `System.out::println`
in `peek` and `forEach` are interleaved when executing the pipeline.
As a result, the output of filenames appears directly before the output of
matching lines from the corresponding file.

As it stands, our application prints the name of every searched file.
We can extend it as follows to print only files that include at least one matching line.

```java {lineos=table,hl_lines=[2]}
javaFiles
    .filter(file -> readLines(file).anyMatch(containsMatch)) // ADDED THIS LINE
    .peek(System.out::println);
    .flatMap(SrcFileSearch::readLines)
    .filter(containsMatch)
    .forEach(System.out::println);
```

The lambda expression passed to `filter` in the new line uses `readLines`
to compute a stream of lines in the given file.
The terminal operation `anyMatch` expects a predicate as argument
and returns a boolean result that is `true` if and only if
the stream contains at least one element satisfying the given predicate.

As it stands, our application prints file names relative to the `src` folder
used as search root.
We can extend it as follows to print absolute file names instead.

```java {lineos=table,hl_lines=[2]}
javaFiles
    .map(Path::toAbsolutePath) // THE ONLY NEW LINE
    .filter(file -> readLines(file).anyMatch(containsMatch))
    .peek(System.out::println);
    .flatMap(SrcFileSearch::readLines)
    .filter(containsMatch)
    .forEach(System.out::println);
```

We use a method reference to the predefined method `Path::toAbsolutePath`
as argument of the `map` combinator, to convert every searched file name.

By using stream combinators we were able to extend a basic version of our program
(that only printed matching lines) in a modular way.
Each of the extensions

  * printing file names,
  * printing only relevant file names,
  * and printing absolute instead of relative paths

required one new line of code and no changes in existing code.

## Handling exceptions

Several of the underlying operations in our application like searching through directories
as well as opening and closing files can throw exceptions.
We will now discuss how the application handles them.
The stream pipeline developed before does not need to be changed.

The pipeline starts with a stream of file names returned by the method `walkJavaFiles`
defined above.
That method can throw an `IOException` which we handle in our `main` method.

```java {lineos=table,hl_lines=[1,"9-11"]}
try (final Stream<Path> javaFiles = walkJavaFiles(srcPath)) {
  javaFiles
      .map(Path::toAbsolutePath)
      .filter(file -> readLines(file).anyMatch(containsMatch))
      .peek(System.out::println)
      .flatMap(SrcFileSearch::readLines)
      .filter(containsMatch)
      .forEach(System.out::println);
} catch (IOException e) {
  System.err.println(e.getMessage());
}
```

We use a
[try-with-resources](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html)
statement to make sure that the stream is closed eventually in addition to handling
exceptions.
We will discuss [managing resources](#managing-resources) in the next section.

The method `readLines` is used twice in the pipeline,
and so far we have not discussed its implementation.
In fact, there is a method with almost the same signature
in the predefined `Files` class.

```java
public static Stream<String> lines(Path path) throws IOException;
```

This is almost the signature we need except for the possible `IOException`.
We use `readLines` in arguments of the `filter` and `flatMap` combinators.
The corresponding functional interfaces `Predicate` and `Function`
define methods `test` and `apply`, and those methods do not declare
any exceptions in their signature.
As a consequence we cannot pass predicates or functions that may throw exceptions
to the stream combinators.

The method `readLines` is defined as follows.[^preliminary]

[^preliminary]: We will define a different version of this method below.

```java
private static Stream<String> readLines(final Path file) {
  try {
    return Files.lines(file);
  } catch (IOException e) {
    throw new UncheckedIOException(e);
  }
}
```

This method wraps the `Files.lines` method and "handles" the `IOException`
by re-throwing it wrapped in an `UncheckedIOException`.
Unchecked exceptions allow us to hide underlying exceptions because they don't need
to be handled or declared in method signatures.
As a consequence, we can use `readLines` in predicates and functions
passed to stream combinators.
To make sure we actually handle the wrapped exception,
we need to add a `catch` clause for `UncheckedIOException` in our `main` method.

```java {lineos=table,hl_lines=["11-13"]}
try (final Stream<Path> javaFiles = walkJavaFiles(srcPath)) {
  javaFiles
      .map(Path::toAbsolutePath)
      .filter(file -> readLines(file).anyMatch(containsMatch))
      .peek(System.out::println)
      .flatMap(SrcFileSearch::readLines)
      .filter(containsMatch)
      .forEach(System.out::println);
} catch (IOException e) {
  System.err.println(e.getMessage());
} catch (UncheckedIOException e) {
  System.err.println(e.getMessage());
}
```

The necessity to wrap exceptions in unchecked exception types is an indication
that the integration of functional programming patterns in conventional languages
is not always seamless because functional features are added to existing features
like exception handling after the fact.
Having both feature sets in mind from the start
could potentially help language designers find a more seamless integration.

## Managing resources

The implementation developed so far has a problem.
The API documentation for `Files.lines` includes the following note.

> This method must be used within a try-with-resources statement or similar control structure
> to ensure that the stream's open file is closed promptly
> after the stream's operations have completed.

In this case, using a try-with-resources statement in the definition of `readLines`
would not have the desired effect
because the stream is returned from the function and consumed outside of it.
Instead we make sure that the stream returned by `readLines` is closed
in a different way.

The first call to `readLines` in the argument to `filter` is problematic.
In general, terminal operations do not close the streams they consume,
and `anyMatch` is no exception.
As a consequence, files are left open after checking if they contain a matching line.

Interestingly, the second use of `readLines` as argument of `flatMap` 
does not have this problem.
The API documentation for `flatMap` contains the following remark.

> Each mapped stream is closed after its contents have been placed into this stream.

As a consequence, files are closed correctly when opened for printing matching lines
by the second use of `readLines`.
We can change the definition of `readLines` as follows to make sure
files are always closed after consuming the returned stream.

```java {lineos=table,hl_lines=[3]}
private static Stream<String> readLines(final Path file) {
  try {
    return Stream.of(Files.lines(file)).flatMap(s -> s); // CHANGED
  } catch (IOException e) {
    throw new UncheckedIOException(e);
  }
}
```

We create a one-element stream containing the stream we want to return
and call `flatMap` with the identity function to immediately flatten
the created one-element stream.
The mentioned property of `flatMap` ensures that the underlying file is closed
when the returned stream is consumed completely.

In general, wrapping a stream inside a one-element stream and then flattening
the resulting stream using `flatMap` with the identity function
will return a new stream that contains the same elements as the original stream.
Our discussion shows that, in Java, this wrapping-and-flattening operation is not insignificant
regarding resource consumtion
which is another indication that the integration of functional programming patterns
into conventional languages is not always seamless.

## Task: Avoid reopening files

The presented implementation opens some files twice.
Modify the stream pipeline in such a way that
no files are opened more than once
and only matching lines are held in memory.

