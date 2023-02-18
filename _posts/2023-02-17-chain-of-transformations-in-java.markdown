---
layout: post
title:  "Chain transformations in Java"
date:   2023-02-17 10:00:00 +0200
categories: java functional
---

## What

It happened that I had the need to optimize a piece of product's business logic on which I was working.
This specific part of code had to manipulate a string and return in output a "sanitized" value.

Here a snippet:
```java
String sanitize(final String input) {
    String a1 = input.toLowerCase();
    String a2 = a1.trim();
    String a3 = a2.replaceAll("world", "");
    String result = a3.toUpperCase();

    return result;
}
```
> It's not the actual code but it makes the point

This part was working great. No bugs, I had just to add some new logic on it.
Let's say that, in this case, the easiest path for me I would have been to just add an additional line in this method, in the right point, that did what I need, and my job it would have been done.

But.. looking at it with the eyes of a "Functional Java" fan, there it was something that it didn't convinced me at 100% in this code.
So I tough that it was worth invest a small percentage of the time dedicated to this task in order to review and improve this method.

Again: it wasn't mandatory, but since I had eyes and focus on this part of code, why do not invest a small amount of time in it?

As said, I love to approach to problems with a functional point of view, but this doesn't means that a "more classic Java" approach is wrong.
Even because, looking at this snippet, this is a pure function.
So, just because this code does not use something that comes from: `java.util.function.*` or it's not using `flatMap` means that it's wrong? Absolutely not! But.. again.. there it was something that made me though that it can be improved.

So, at this point it's sure that I will review this part of code. Let's resume the situation: 

**What I want?** 

> A function that applies a predefined **list of operations** that takes a value as input and provides a **transformed** value **of the same type** as output.

Let's focus on the highlights now: 

1. Conceptually there is a list of operations. It means that there it could be 1 or N transformations to apply and the order matters;
2. Each transformation aims to produce a new value. Not to modify the existing once;
3. Input and output types are the same.


## First approach

Let's put hands on it.

Above, in point 1. and 3. it was specified that it's about a list of transformations. For transformation it means an operation that takes an input of type X and returns an output of type X.

Since in this example it's focused on `strings`, let's refer to transformations as `List<Function<String, String>>`, so something like that can be produced:

```java
private static final Function<String, String> toLowerCase = String::toLowerCase;
private static final Function<String, String> toUpperCase = String::toUpperCase;
private static final Function<String, String> trim = String::trim;
private static final Function<String, String> dropWordHello = s -> s.replaceAll("hello", "");
private static final List<Function<String, String>> transformations = List.of(
        toLowerCase,
        trim,
        dropWordHello,
        toUpperCase
);
```

*The `transformations` object carries the list of operations that we want to apply to our input.*
If in the future will come the need to add/remove an operation, it's the matter to modify this list.

At this point it just about understand how to let it land,
and the desiderata is to map that list in something like: 

```java
//Pseudo-code for simplicity
toUpperCase(dropWordHello(trim(toLowerCase("Hello World!"))))
```

How to obtain it by starting from a list of transformations?
As usual `Streams` comes in help of us.

Since the need here is to combine values into one Streams provide the [reduce](https://docs.oracle.com/en/java/javase/18/docs/api/java.base/java/util/stream/Stream.html#reduce(T,java.util.function.BinaryOperator)) method for this purpose.
The javadoc for this method is exhaustive: 

> A reduction operation (also called a fold) **takes a sequence of input elements and combines them into a single summary result** by repeated application of a combining operation, such as finding the sum or maximum of a set of numbers, or accumulating elements into a list. The streams classes have multiple forms of general reduction operations, called reduce() and collect(), as well as multiple specialized reduction forms such as sum(), max(), or count().

The right combination of `Stream#reduce(..)` method and `Function` interface allow to smoothly reach the point here.

The reduce method has the following signature: `T reduce(T identity, BinaryOperator<T> accumulator)` with `T == Function<String, String>` in this example.

First parameter `T identity` identifies the identity/beginning value for the accumulating function. In a sum example it will be the neutral value, so 0. In a `Function<String, String>` example like this one the neutral value is identified by a function that always returns it input value: `a -> a`.
The Function interface provides the `identity()` method as utility for this purpose: 

```java
/**
    * Returns a function that always returns its input argument.
    *
    * @param <T> the type of the input and output objects to the function
    * @return a function that always returns its input argument
    */
static <T> Function<T, T> identity() {
    return t -> t;
}
```

Second parameter `BinaryOperator<T> accumulator` allows to drive how the accumulator should behave by defining how an element A and his next element B should interact in order to return a single element of the same type.

Probably code this sentence is easier than explain it: What this example wants to achieve is: *run function B with as input the output of A.* the easiest way to achieve it is by: `(a, b) -> a.andThen(b)`.
So even in this case the `Function` interface has a method that comes as help.

```java
/**
    * Returns a composed function that first applies this function to
    * its input, and then applies the {@code after} function to the result.
    * ...
    * @return a composed function that first applies this function and then
    * applies the {@code after} function
    */
default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
    Objects.requireNonNull(after);
    return (T t) -> after.apply(apply(t));
}
```

The job is done, let's glue stuff together and obtain a pointer to that chain of transformations.

```java
final var chain = transformations
    .stream()
    .reduce(Function.identity(), Function::andThen);
```

Now it's only the matter of trigger the computation and the whole chain will be evaluated: 

```java
System.out.println(chain.apply("Hi, I'm Jacopo!"));
System.out.println(chain.apply("Welcome to my"));
System.out.println(chain.apply("Hello World"));

/*
Output:
HI, I'M JACOPO!
WELCOME TO MY
 WORLD
*/
```

## Handling transformations' order

The first point in above analysis says: 

> 1. Conceptually there is a list of operations. It means that there it could be 1 or N transformations to apply and the order matters;

The order matters. Indeed looking in example's transformations the third output line still present a whitespace at the begin even if one of the functions is a `trim`.
This happens because the chain is firstly run the trim function, and then is manipulating the string removing a word. So `"Hello world"` becomes `" world"`.

At this point it's enough review the order of the transformations, but chain's order now is hardcoded. 
Can it be improved in order to easier integrate it by and external parties? (like a system property, DB record, configuration file)
> (aim of this document is not about to explain how to integrate with an external system, but how to adapt what it's described above in order to integrate in the easiest way possible)

There are tons of way to do it but the easiest that, at the time, it came in my mind was to assign at each transformation an identifier (key) and let the order be driven by a list that reference to these keys.
Let's write it down that it's easier show it than explain it.

First, transformations becomes a Map: 

```java
private static final Function<String, String> toLowerCase = String::toLowerCase;
private static final Function<String, String> toUpperCase = String::toUpperCase;
private static final Function<String, String> trim = String::trim;
private static final Function<String, String> dropWordHello = s -> s.replaceAll("hello", "");

private static final Map<String,Function<String, String>> transformations = Map
        .ofEntries(
                Map.entry("toLowerCase", toLowerCase),
                Map.entry("toUpperCase", toUpperCase),
                Map.entry("trim", trim),
                Map.entry("dropWordHello", dropWordHello)
        );
```

Each entry of the Map is identified by a key that in that case it's a string.
Now let's define the order with a list of these keys: (let's remember that the aim is to trim the string after all the other operations)

```java
final var transformationsOrder = List.of("toLowerCase", "dropWordHello", "toUpperCase", "trim");
```

When the order is defined it's only the matter of translate it in a chain of transformations that respects that order.
Nothing of really complicated since the usefulness of `Stream#reduce(..)` function was already address before.

```java
final var chain = transformationsOrder // Let's start from the transformation's order
        .stream()
        /* Let's try to map each value in the list to an actual transformation operation.
            If we don't found it (due to a mistake, typo, etc..) let's return null.
            They will be handled in next operations */
        .map(e -> transformations.getOrDefault(e, null))
        /* Filtering away null mean that configured some transformation that it will actually not executed.
            So it can be actually risky this way to handle this scenario. But let's proceed like that for simplicity
            At the end there is not a right way to do it. It's a trade-off that depends on the scenario in which we are. */
        .filter(Objects::nonNull)
        .reduce(Function.identity(), Function::andThen);
```

Calling now the example code will produce the desiderate output (*with no whitespace at start/end of lines*):
```java
System.out.println(chain.apply("Hi, I'm Jacopo!"));
System.out.println(chain.apply("Welcome to my"));
System.out.println(chain.apply("Hello World"));

/*
Output:
HI, I'M JACOPO!
WELCOME TO MY
WORLD
*/
```


## Conclusions

There is nothing special in this document. 
What is special is what it comes since Java 8 and how well it operates together. 

Both `Streams` and `Function` allows Java developers to write better, clever and simpler code. This example aims to highlight how meaningfully they operates together.

Indeed the core point of this example is behind the `Stream#reduce(..)` method and how it behave with, in this case: `Function`; but it could be anything else.

Once that it's address it can be integrated with higher-level logic in complex systems.

Full example [here](https://github.com/Jacopo47/chain-of-transformations-demo)