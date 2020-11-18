---
title: Using Optionals
weight: 24
---

# Using Optionals

In order to discuss how to replace `null` checks with idiomatic use of optional combinators,
we introduce types representing arithmetic expressions
and implement different methods to evaluate them.

## Arithmetic expressions

Arithmetic expressions come in different variants.
We restrict ourselves to representing constants and
applications of binary operators for basic arithmetic operations.
The different kinds of expressions are represented as different classes
`Num` and `Bin` implementing the interface `Exp`.

```java
public interface Exp {
  <R> R transform(ExpTransform<R> toResult);
}
```

As we want to implement different methods for evaluating expressions we use
[double dispatch](https://en.wikipedia.org/wiki/Double_dispatch)
and implement different evaluators as instances of the interface `ExpTransform<R>`.
This interface has overloaded methods to transform each possible type of expressions.

```java
public interface ExpTransform<R> {
  default R fromExp(Exp exp) {
    return exp.transform(this);
  }

  R fromExp(Num num);
  R fromExp(Bin bin);
}
```

The 
[default implementation](https://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html)
for the `Exp` type
calls the `transform` method of the given expression
which needs to be implemented in concrete classes
by dispatching to one of the other overloaded `fromExp` methods.

The class `Num` represents integer constants.

```java
public class Num implements Exp {
  private final int value;

  public Num(int value) {
    this.value = value;
  }
  
  @Override
  public <R> R transform(ExpTransform<R> toResult) {
    return toResult.fromExp(this);
  }

  public int getValue() {
    return value;
  }
}
```

It wraps an internal value providing read-access via `getValue`.
More complex expressions can be constructed using the class `Bin`
representing applications of binary operators.

```java
public class Bin implements Exp {
  private final BinOp op;
  private final Exp leftArg;
  private final Exp rightArg;

  public Bin(final char op, final Exp leftArg, final Exp rightArg) {
    this.op = new BinOp(op);
    this.leftArg = Objects.requireNonNull(leftArg);
    this.rightArg = Objects.requireNonNull(rightArg);
  }

  @Override
  public <R> R transform(ExpTransform<R> toResult) {
    return toResult.fromExp(this);
  }
  
  // getters for private members omitted
}
```

It wraps values representing the operator as well as its arguments
providing read-access via corresponding getter methods.
The operator is represented by the class `BinOp`.

```java
public class BinOp {
  private final char chr;

  public BinOp(final char chr) {
    this.chr = chr;
  }

  public char getChar() {
    return chr;
  }

  public int apply(final int left, final int right) {
    // implementation omitted
  }
}
```

The implementation of `apply` uses corresponding Java operators
for basic arithmetic operations.
With these definitions, the arithmetic expression `1 + 2 / 3` can be represented
using the following Java expression.

```java
new Bin('+', new Num(1), new Bin('/', new Num(2), new Num(3)))
```

Here is a first version of an evaluator for arithmetic expressions[^boxing].

[^boxing]: We use the boxed type `Integer` instead of the primitive type `int` as result type
because parameters of generic types like `ExpTransform<R>`
cannot be instantiated with primitive types.
If we would experience the performance penalty of boxing
we could specialize used generic types with versions using `int` instead of a type parameter.

```java
public class Evaluator implements ExpTransform<Integer> {
  @Override
  public Integer fromExp(Num num) {
    return num.getValue();
  }

  @Override
  public Integer fromExp(Bin bin) {
    final Integer left = fromExp(bin.getLeftArg());
    final Integer right = fromExp(bin.getRightArg());
    return bin.getOp().apply(left, right);
  }
}
```

The version of `fromExp` for constants simply returns the wrapped value.
For applications of binary operators, 
`fromExp` is called recursively to evaluate both arguments
before applying the operator to compute the result.

## Explicit `null` checks

The shown evaluator will throw an arithmetic exception
when division by zero occurs during the evaluation.
Our next evaluator avoids runtime exceptions and returns `null` instead.

```java { lineos=table, hl_lines=[9] }
public class NullEvaluator implements ExpTransform<Integer> {
  public Integer fromExp(Num num) {
    return num.getValue();
  }

  public Integer fromExp(Bin bin) {
    final Integer left = fromExp(bin.getLeftArg());
    final Integer right = fromExp(bin.getRightArg());
    return applyOp(bin.getOp(), left, right);
  }

  private Integer applyOp(BinOp op, Integer left, Integer right) {
    if (left == null || right == null //
        || op.getChar() == '/' && right == 0) {
      return null;
    } else {
      return op.apply(left, right);
    }
  }
}
```

The method for `Num` instances is unchanged.
The version for `Bin` uses `applyOp` to check for division by zero
instead of applying the operator directly.
As recursive calls might return `null`,
it also implements explicit `null` checks for their results.

Unfortunately, returning `null` when evaluating expressions
is just as likely to cause a runtime exception down the road
if callers are not aware of possibly missing results.

## Naive use of optionals

Our next evaluator replaces nullable integers with optional ones.
Initially, the implementation mimics the previous version,
replacing `null` checks with corresponding checks on optionals.

```java
public class OptEvaluator implements ExpTransform<Optional<Integer>> {
  @Override
  public Optional<Integer> fromExp(Num num) {
    return Optional.of(num.getValue());
  }

  @Override
  public Optional<Integer> fromExp(Bin bin) {
    final Optional<Integer> left = fromExp(bin.getLeftArg());
    final Optional<Integer> right = fromExp(bin.getRightArg());
    return applyOp(bin, left, right);
  }

  private Optional<Integer>
      applyOp(Bin bin, Optional<Integer> left, Optional<Integer> right) {
    final BinOp op = bin.getOp();
    if (left.isEmpty() || right.isEmpty()
        || op.getChar() == '/' && right.get() == 0) {
      return Optional.empty();
    } else {
      return Optional.of(op.apply(left.get(), right.get()));
    }
  }
}
```

As mentioned previously, optionals are primarily intended to be used
in the result type of methods.
However, the new version of `applyOp` uses `Optional` not only in the result type
but also in the type of two arguments.
Moreover, first checking if an optional value is present
and then using `get` to access it
is a common anti-pattern when programming with optionals.

## Idiomatic use of optionals

We will now transform this implementation
making more idiomatic use of the optional combinators in every step.
As it turns out, we do not need the method `applyOp`
to implement error handling
but can use `map`, `filter`, and `flatMap` instead.

### `flatMap` specializes sequences

Here is an alternative implementation of `fromExp` for `Bin` arguments
that replaces the sequence of recursive calls with nested calls to `flatMap`.

```java
@Override
public Optional<Integer> fromExp(Bin bin) {
  final BinOp op = bin.getOp();
  return
  fromExp(bin.getLeftArg()).flatMap(left ->
  fromExp(bin.getRightArg()).flatMap(right -> {
    if (op.getChar() == '/' && right == 0) {
      return Optional.empty();
    } else {
      return Optional.of(op.apply(left, right));
    }
  }));
}
```

Here, `left` and `right` are `Integer` values guaranteed to be not `null`,
so we do not need to check them any more or use `get` to access their values.

### `filter` specializes conditionals

Next, we can use `filter` to sort out values that would lead to division by zero
before calling `flatMap` for the `right` argument.

```java
@Override
public Optional<Integer> fromExp(Bin bin) {
  final BinOp op = bin.getOp();
  return
  fromExp(bin.getLeftArg()).flatMap(left ->
  fromExp(bin.getRightArg())
      .filter(right -> op.getChar() != '/' || right != 0)
      .flatMap(right -> Optional.of(op.apply(left, right))));
}
```

We negate the condition used before
to describe those elements that will *not* lead to division by zero
and use it in the argument passed to `filter`.

### `map` specializes `flatMap`

Finally, we can emply a useful property for reasoning about programs
involving optional combinators.
Calling `flatMap` and immediately creating an optional value in the passed function
is equivalent to calling `map` with a corresponding function
that does not create an optional value.

```java
@Override
public Optional<Integer> fromExp(Bin bin) {
  final BinOp op = bin.getOp();
  return
  fromExp(bin.getLeftArg()).flatMap(left ->
  fromExp(bin.getRightArg())
      .filter(right -> op.getChar() != '/' || right != 0)
      .map(right -> op.apply(left, right)));
}
```

Here, the second call to `flatMap` has been replaced with a corresponding call to `map`.
