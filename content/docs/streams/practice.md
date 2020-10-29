---
title: Streams in Practice
weight: 14
---

# Streams in Practice

We now discuss a small command line application
demonstrating how streams help solving problems in practice.

## Source File Search

Our application will search through files in a given folder
and print all lines containing a match for a given regular expression.
We will only search in Java source files in the given folder.
When searching for lines matching the regular expression `/public static[^=]*\(/`
in our own `src` folder our application might produce output like this.

```
/home/me/java-fun-code/src/main/java/sebfisch/SrcFileSearch.java
  public static void main(final String[] args) {
/home/me/java-fun-code/src/main/java/sebfisch/parser/Parser.java
  public static <R> Parser<R> failure() {
  public static <R> Parser<R> of(R result) {
  public static Parser<Character> forChar() {
  public static Parser<String> forStringOf(final Parser<Character> charParser) {
```

File names are printed (using their absolute path)
before all matching lines from the corresponding Java files.

To compute a stream of Java source files in a given folder
we define the following method.

```java
static Stream<Path> getJavaFiles(final Path root) throws IOException {
  return Files.walk(root) //
      .filter(Files::isReadable) //
      .filter(path -> path.toString().endsWith(".java"));
}
```

The predefined method `Files.walk` expects a `Path` as argument
and returns a stream of paths representing files or directories
contained inside the given root path.
It might throw an `IOException` which we pass on to the caller of `getJavaFiles`.

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

When `javaFiles` contains a stream of paths returned by `getJavaFiles`,
we can use the predicate `containsMatch` to print matching lines as follows.

```java
javaFiles //
    .flatMap(SrcFileSearch::getLines) //
    .filter(containsMatch) //
    .forEach(System.out::println);
```

Here, the method reference `SrcFileSearch::getLines` is equivalent to the
lambda expression `path -> SrcFileSearch.getLines(path)`
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
