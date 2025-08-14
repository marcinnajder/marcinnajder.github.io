---
layout: post
title: Lisp for JS developers part 1 - fundamentals [JS, Lisp]
date: 2023-02-28
tags:
  - js
  - lisp
series: jslisp
---

{%- include /_posts/series-toc.md series-name="jslisp" -%}

## Introduction

> Lisp is a family of programming languages with a long history and a distinctive, fully parenthesized prefix notation. Originally specified in the late 1950s, it is the second-oldest high-level programming language still in common use, after Fortran.

This is an excerpt from [wiki page](<https://en.wikipedia.org/wiki/Lisp_(programming_language)>) about Lisp programming language. For many years, so many programmers have fallen in love with Lisp again and again. There is some magic in that specified language, it just looks completely different comparing to almost any other language. It's dynamic and functional, considered a scripting language, favoring immutable data structures and recursion. REPL (read-evaluate-print-loop) support is almost legendary.

Interestingly, many programmers (me too) see [Lisp in JavaScript](https://www.crockford.com/javascript/javascript.html).

> Lisp in C's Clothing - JavaScript's C-like syntax, including curly braces and the clunky for statement, makes it appear to be an ordinary procedural language. This is misleading because JavaScript has more in common with functional languages like Lisp or Scheme than with C or Java. It has arrays instead of lists and objects instead of property lists. Functions are first-class. It has closures. You get lambdas without having to balance all those parens.

In this series, I will explain Lisp fundamentals using JavaScript. We will see a piece of List code side by side with JavaScript code. Strictly speaking, it will be the [Clojure](https://en.wikipedia.org/wiki/Clojure) dialect of Lisp. Source code explains faster and better than a million words. I hope that at the end of the series, you will gain a new skill called "can read Lisp code" and you appreciate the beauty of this unique ancestor of your programming language of choice. Lisp introduced so many [features](https://www.paulgraham.com/diff.html) which are obvious today, but in a minimalistic form.

## JS helper functions

Let's start by defining a few helper JavaScript functions. Functions like `pipe, map_, filter_, skip_, ...` come from my [poweseq](https://github.com/marcinnajder/powerseq) library and work with lazy sequences (iterators API). You can read more about the library in this [series](https://marcinnajder.github.io/2022/11/02/sequences-in-javascript-part-1-introduction-to-powerseq.html) of articles.

```js
var fs = require("fs");
var os = require("os");
var {
  pipe,
  skip: skip_,
  elementat: elementat_,
  find: find_,
  some: some_,
  map: map_,
  filter: filter_,
  repeatvalue: repeatvalue_,
} = require("powerseq");

function wrongArgType(arg) {
  const argType = typeof arg !== "object" ? typeof arg : arg.constructor.name;
  throw new Error(
    `function '${wrongArgType.caller.name}' was called with argument of unexpected type '${argType}'`
  );
}

var throww = (message) => {
  throw new Error(message);
};
```

Functions like `map, filter, reduce, ...` working with collections were introduced by the Lisp standard library. We will implement them from scratch later. We imported similar functions from powerseq adding a `_` postfix to avoid naming collisions. The `throw` keyword in JavaScript is a statement (not an expression returning a value). That's why the function `throww` was introduced; calling any function in JavaScript is an expression. JavaScript and Lisp are dynamic; it's very easy to pass a function argument with an invalid data type. Special function`wrongArgType` helps track such mistakes. This function
a little magical, it always throws an error with a message text including the function name calling `wrongArgType`.

## Syntax, global variables, builtin types

Not many programming languages support something like "global state" or "environment". It's like a global map of variables available anywhere in your code. We can read, add, remove or change any variables at any time.

```scheme
(def first-name "marcin")
(println "my name is:" first-name)
```

```js
var firstName = "marcin";
console.log("my name is:", firstName);
```

That was the first code snippet of the Lisp programming language. The syntax looks more like a readable text data format, like JSON or XML, than a typical programming language. Any piece of Lisp code looks like this `(def first-name "marcin")`, a list of items where the first item specifies the type of operation being performed. In our example, `def` means "define global variable" with a name `first-name` and a value `"marcin"`. In JavaScript, we can also define a global variable directly inside a script instead of variables inside functions, which are treated as local variables. JavaScript has data types like `number`, `string`, `boolean`, ... objects and collections, the same as Lisp. In dynamic languages, we don't specify the types explicitly next to variables or function parameters. It's like specifying type `object` everywhere in statically typed languages like Java, C#, or Go. But the types exist, they define a set of correct values (`1, 2.2` are `number`, `"marcin"` is a `string`, ... ) and operations that can be performed with them (`+` operator can be used with `number` or `string`, but cannot be used for instance with `boolean`, ... ).

## First-class functions

JavaScript was created in 1995 and was one of the first programming languages officially not considered as "functional" that supported "first-class functions". In object-oriented programming, term "method" is often used when we think about "function". Method is just a function defined in the context of some type, and later it's executed in combination with some object (a concrete instance of that type). But we would like to treat a function as a regular type like `number` or `string`. For example, we would like to define a variable of function type, store it as an item inside some collection, pass it into or return it from another function. And all of that without connection to any specific type or object. Additionally, we would like to be able to define an anonymous function (aka "lambda expressions") by specifying a function signature with a body. Both JavaScript and Lisp support that.

```scheme
(def add
  (fn [a b] (+ a b)))

(add 1 2) ;; => 3
(add 1) ;; => Execution error (ArityException) at .. Wrong number of args (1) passed to ...
((fn [a b] (+ a b)) 10 2) ;; => 12
```

```js
var add = (a, b) => a + b;

add(1, 2);
((a, b) => a + b)(1, 2);
```

`fn` symbol defines a new function, it takes two arguments `a` and `b` and returns the sum of them. After defining the `add` function, we can place `add` as the first symbol in the list representing Lisp code. `(add 1 2)` executes our function passing values `1` and `2`. This is quite a unique feature of Lisp; we can extend the "set of keywords" used in programming languages. In some sense, there is no difference between `def`, `fn`, ... and our custom function named `add`. As we said before, the first item on the list represents the operation performed. `((fn [a b] (+ a b)) 10 2)` is a completely valid code, often called "Immediately Invoked Function Expression" (IIFE).

## defn, variadic functions, comments

It's too early to discuss macros in Lisp. Let's only say that we can also define a function in a special way using the `defn` symbol. It's like a shortcut, instead of `(def add (fn [a b] (+ a b)))` we can write `(defn add [a b] (+ a b))`. Similarly, JavaScript functions can also be defined using the `function` keyword.

```scheme
(defn add [a b]
  (+ a b))

(macroexpand '(defn add [a b] (+ a b))) ;; => (def add (clojure.core/fn ([a b] (+ a b))))
```

```js
function add(a, b) {
  return a + b;
}
```

Variadic functions allow handling a variable number of arguments. For instance, we can pass two or more arguments instead of passing exactly two arguments.

```scheme
(defn add-at-least-2
  "function takes at least 2 arguments"
  [a b & rest-args]
  (+ a b (apply + rest-args)))

(add-at-least-2 1 2 3 4) ;; => 10
```

```js
/** function takes at least 2 arguments */
function addAtLeast2(a, b, ...restArgs) {
  return a + b + restArgs.reduce((x, y) => x + y, 0);
}
```

Nowadays, this feature is supported in almost all programming languages. In Lisp `&` symbol means that the next `rest-args` parameter will be a collection type. You can probably guess that code `(apply + rest-args)` uses a standard plus operator to sum up all items in a collection. `apply` is a built-in function, and it will be explained in detail later. Optional function comments are defined just after the function's name as a string.

## Control flow (if/then/else, variables, code blocks, simple loops)

Let's talk about typical control flow constructs like local variables, conditions, and loops.

```scheme
(defn func-1 [a b]
  (if (> a b)
    (let [sum (+ a b)
          value (mod sum 10)]
      (println "value:" value)
      value)
    (do
      (dotimes [n 3]
        (println n "hej")
        (println n "yo"))
      (doseq [n '(3 4 5)]
        (println n "hej")
        (println n "yo"))
      666)))

(func-1 10 9) ;; => 9
(func-1 9 10) ;; => 666
```

```js
function func1(a, b) {
  if (a > b) {
    var sum = a + b;
    var value = sum % 10;
    console.log("value:", value);
    return value;
  } else {
    for (var n = 0; n < 3; n++) {
      console.log(n, "hej");
      console.log(n, "yo");
    }
    for (var n of [3, 4, 5]) {
      console.log(n, "hej");
      console.log(n, "yo");
    }
    return 666;
  }
}
```

It is hard to believe that real programmers would like to write the code this way, all those closing brackets `))))` at the end. Honestly, I wrote some code in Clojure, with tools like formatters, linters, and code editors like VS Code. You quickly get used to this special syntax. In most cases, programmer uses only a few built-in keywords like `let, if, do, loop`. Brackets let us nest some code inside the other code. For instance, a typical if-then-else requires three parts `(if (> a b) a (+ b 10))`. The important thing is that each part of the code like this `(...)` is an expression, so it always returns some value in contrast to a statement. We can very easily compose smaller expressions into the bigger ones. It's one of the key pillars of functional programming in general.

## Summary

At this point, you should have some basic understanding of the Lisp language. I hope you can spot many similarities with JavaScript, despite its special syntax, of course. Both are dynamic and support basic data types and first-call functions. They also support less commonly known features like global variables and "falsy and truthy values."
