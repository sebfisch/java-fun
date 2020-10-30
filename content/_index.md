---
title: "Functional Patterns in Java"
---

# Functional Patterns in Java

> Conventional programming languages are growing *ever more enormous, but not stronger*.

In his Turing-Award lecture in 1978,
[John Backus](https://en.wikipedia.org/wiki/John_Backus)
asked:
[Can programming be liberated from the von Neumann style?](https://www.thocp.net/biographies/papers/backus_turingaward_lecture.pdf)
He gave a damning report on the state of conventional programming languages at the time.

> *Inherent defects* at the most basic level cause them to be both *fat and weak*:
> their primitive word-at-a-time style of programming
> inherited from their common ancestor—the von Neumann computer,
> their close coupling of semantics to state transitions,
> their division of programming into a world of expressions and a world of statements,
> their inability to effectively use powerful combining forms
> for building new programs from existing ones,
> and their lack of useful mathematical properties for reasoning about programs.

This assessment is decades old.
The functional programming language
[Lisp](https://en.wikipedia.org/wiki/Lisp_(programming_language))
is two decades older but was not conventional at the time, and still isn't today.
Functional programming languages focus on *expressions rather than statements*,
which enables them to provide *powerful combining forms* for programs
with *useful properties for reasoning*.
As a result, conventional programming languages are incorporating patterns
from functional programming.
Java is no exception.

The word __pattern__ is used here in the sense of *design pattern* 
(and not as in pattern matching.)
In functional programming, design patterns often take the form of so called 
higher-order functions, which are functions that take other functions as arguments
or return other functions as results.
In modern Java, functions can be represented using 
[functional interfaces](https://docs.oracle.com/javase/specs/jls/se14/html/jls-9.html#jls-9.8)
and corresponding instances can be written concisely using
[lambda expressions](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)
or 
[method references](https://docs.oracle.com/javase/tutorial/java/javaOO/methodreferences.html).
Representing functions as values (or objects in Java) allows them to be used in expressions
without executing them right away as in statements including procedure (or method) calls.
In expressions, functions can be more flexibly combined 
for building new programs from existing ones.
Algebraic properties, similar to primary school arithmetic, 
can be used to manipulate combined expressions, including those containing functions.

In this tutorial, we will look at combining forms, or *combinators*,
that are particularly prevalent: `map`, `filter`, and `flatMap`.
All of them are provided for different types by Java's standard library.
We will discuss how to use them for two of those types and then 
implement our own types providing these combinators.

The underlying source code is available online, 
and this tutorial includes tasks to extend it.
If you want to follow along, you can use your own Java development environment
or install
[Docker](https://docs.docker.com/get-docker/)
and
[VS Code](https://code.visualstudio.com/download)
with the
[Remote Development Extension Pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)
to use a predefined environment without creating (or adjusting) your own.
To use the predefined environment in VS Code
click on the Remote Containers icon in the bottom-left corner,
select *Clone Repository in Container Volume*,
and provide the URL of the accompanying source code repository.
Alternatively, you can use `git` to clone the repository,
open the corresponding folder in VS Code,
and then select *Reopen Folder in Container*.

The most eye-catching use of functional programming patterns in Java is probably with
[streams](docs/streams/), so let's start with them.
