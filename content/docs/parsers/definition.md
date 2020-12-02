---
title: Defining Complex Parsers
weight: 34
---

# Defining Complex Parsers

We will now use parser combinators to define a more realistic example of a parser.

[Kate's Grammar Tool](https://github.com/katef/kgt)
includes a famous grammar for arithmetic expressions 
as well as a corresponding railroad diagram:

![Railroad Diagram for Language of Arithmetic Expressions](../expr.svg)

## Tests for our parser

The diagram contains several named parts which together describe
the syntax of arithmetic expressions.
An interesting property of this description is that
the structure of the rules follows the usual precedence rules
of addition and multiplication:
when parsing an expression based on this diagram
the rules will be nested correctly even without corresponding parentheses.
The following tests demonstrate how a parser corresponding to the above railroad diagram
parses its input.

```java
@Test
void testRightNested() {
  assertEquals("(3 + (4 * 5))", EXP.parse("3+4*5").toString());
}

@Test
void testLeftNested() {
  assertEquals("((3 * 4) + 5)", EXP.parse("3*4+5").toString());
}

@Test
void testExplicitlyRightNested() {
  assertEquals("(3 * (4 + 5))", EXP.parse("3*(4+5)").toString());
}

@Test
void testExplicitlyLeftNested() {
  assertEquals("((3 + 4) * 5)", EXP.parse("(3+4)*5").toString());
}
```

Here, `EXP` is an instance of the class `ExpParser` defined below.
The method `parse` returns an instance of the interface `Exp`
defined previously for representing arithmetic expressions.
The method `toString` for `Exp` instances
returns a fully parenthezised representation of the expression it is called on.

## Parser definitions

To implement a parser corresponding to the above railroad diagram
we define the following class.

```java
public class ExpParser {
  public Exp parse(final String input) {
    return expr().results(input).findFirst()
        .orElseThrow(() -> new RuntimeException("parse error for: " + input));
  }
  // to be continued
}
```

The method `expr` defined below returns a parser of type `Parser<Exp>`.
On the resulting parser, the `results` method is used to compute
a stream of produced results for the given input,
and `findFirst` is called on the resulting stream
to retrieve the first result as an optional value.
If the parser fails and no result is produced
then the `parse` method throws a parse error
in the form of a runtime exception.

### Integer parsers

The railroad diagram refers to the name `integer` without
explaining further how integers are represented.
Before we implement the diagram using parser combinators,
we first define parsers for a simple representation of integers
to be used later.
The parser `DIGITS` can be used to parse positive integers
represented as a non-empty string of digits.

```java
private static final Parser<Integer> DIGITS =
    Parser.forString(Character::isDigit)
        .filter(digits -> !digits.isEmpty())
        .map(Integer::parseInt);
```

A parser for possibly negative integers
can be defined in terms of `DIGITS` as follow.

```java
// <integer> ::= "-" <digits> | <digits>
private static final Parser<Integer> INTEGER =                  // <integer> ::=
    Parser.forChar().filter(chr -> chr == '-').flatMap(_sign -> // "-"
    DIGITS.map(n -> -n)                                         // <digits>
    ).or(DIGITS);                                               // | <digits>
```

The `INTEGER` parser is defined using `or` as an alternative between
a sequential parser for a minus sign followed by digits as one alternative
and digits without a sign as second alternative.
The `map` combinator is used to negate the result of the first occurrence of `DIGITS`
following the minus sign.

### Expression parsers

We are now ready to translate the railroad diagram into parsers.
Each rule will be translated into a parser that produces and `Exp` instance.
The simplest case is the rule for `const` which we can express
by adjusting the result of the `INTEGER` parser as follows.

```java
// <const> ::= <integer>
private static final Parser<Exp> CONST = INTEGER.map(Num::new);
```

The remaining rules all have a similar structure:
they consist of alternative branches
where the first branch is a sequential parser.
We can expect to use `flatMap` for the sequential parsing and `or` for the alternatives.

As all of the remaining rules refer to themselves or each other recursively,
we define methods instead of constants to implement them,
because for constants such cyclic references are not allowed in Java.
The first rule for `expr` can be translated into a parser as follows.

```java
// <expr> ::= <term> "+" <expr> | <term>
private Parser<Exp> expr() {                                    // <expr> ::=
  final Parser<Exp> opExpr =
      term().flatMap(fst ->                                     // <term>
      Parser.forChar().filter(chr -> chr == '+').flatMap(chr -> // "+"
      expr().map(snd ->                                         // <expr>
      new Bin(chr, fst, snd))));
  return opExpr.or(term());                                     // | <term>
}
```

Here, we define a parser `opExpr` for the seequential part of the rule
and return an alternative parser as result.
The sequential part is expressed with two calls to `flatMap`.
Finally, `map` is used to adjust the result of the last parser in the sequence
and combine the results of all parsers in a `Bin` instance.

The rule for `term` can be translated similarly.

```java
// <term> ::= <factor> "*" <term> | <factor>
private Parser<Exp> term() {                                    // <term> ::=
  final Parser<Exp> opTerm =
      factor().flatMap(fst ->                                   // <factor>
      Parser.forChar().filter(chr -> chr == '*').flatMap(chr -> // "*"
      term().map(snd ->                                         // <term>
      new Bin(chr, fst, snd))));
  return opTerm.or(factor());                                   // | <factor>
}
```

The definition of `term` has the same structure as the definition of `expr`,
but the parsers filling in the holes of this structure are all different.
Finally, we translate the rule for `factor` into a parser
to complete our implementation.

```java
// <factor> ::= "(" <expr> ")" | <const>
private Parser<Exp> factor() {                                  // <factor> ::=
  final Parser<Exp> parenthezised =
      Parser.forChar().filter(chr -> chr == '(').flatMap(_lp -> // "("
      expr().flatMap(exp ->                                     // <expr>
      Parser.forChar().filter(chr -> chr == ')').map(_rp ->     // ")"
      exp)));
  return parenthezised.or(CONST);                               // | <const>
}
```

This parser has a slightly different structure than the previous ones
because two of the results in the sequential parser
(the left and right parentheses)
are ignored in the produced result.

## Tasks

### Generalized operator parsing

Refactor the parser definitions to generalize the parsing of operator symbols.

 1. First, extend the `ExpParser` class by a method with the following signature.

    ```java
    private Parser<Character> opCharIn(String chars);
    ```

    This method should return a parser for characters that are contained in the given string.
    Modify the `expr` and `term` parsers to use `opCharIn`.
    Check that the tests for the expression parser still pass.

 2. Change the calls to `opCharIn` to allow for more arithmetic operators,
    at least adding operator symbols for subtraction and division.
    Make sure to add new operator symbols in such a way that
    their usual precedence rules are respected.
    Add tests to verify that your extension is correct.

 3. Change the definition of `opCharIn` to allow an arbitrary number of 
    whitespace characters before and after the operator symbol.
    Add tests to verify that expressions can be parsed with an arbitrary amount of
    whitespace before and after binary operator symbols.

### Interactive evaluation

Implement an interactive evaluator of arithmetic expressions as command line program.
Here is an example interaction:

```
Enter arithmetic expressions to evaluate them, press ^C to quit.
eval> 6*3 + 4
((6 * 3) + 4) = 22
eval> 6 * (3+4)
(6 * (3 + 4)) = 42
eval> 5 % 2
parse error for: 5 % 2
eval> 1/0
(1 / 0) = undefined
eval> 50 - 50/6
(50 - (50 / 6)) = 42
eval> ^C
```

Use the `lines` method provided by `BufferedReader`
to process inputs in a stream pipeline.
Implement error handling for parse errors and division by zero
as shown in the example as well as for possible I/O exceptions.
