---
layout: post
title: Lisp for JavaScript developer part 2 - built-in functions [JavaScript, Lisp]
date: 2023-03-07
tags: js lisp
series: jslisp
---

{%- include /_posts/series-toc.md series-name="jslisp" -%}

## Introduction 

We already know how to define functions using basic types like `string`, `boolean`, or `nil` in Lisp. The language's syntax is very different from other programming languages, but commonly used building blocks like variables, conditions, and loops are the same. One of the main features of functional programming is treating functions as values. We can pass them as arguments to the function or return them from the function. Functions that do that are also called "higher order functions", and in this part of the series, we will learn a few such functions. But first, let's start with the primitive built-in functions.

## Built-in functions

In a "typical" programming language like JavaScript, we think in terms of built-in operators like `+, -, >, ==, &&, ||` and existing functions like `parseInt, console.log, JSON.parse, Math.max` included in the standard  library. In Lisp there is no difference between keywords, operators, and functions. They are all used in exactly the same way.

Below you can find the list of standard builtin Clojure specific functions also implemented in JavaScript grouped into the sample categories like:
- predicates `nil?`, `string?`, ..
- arithmetic and logic operators  `+`, `*`, ...
- math functions `inc`, `dec`, ... 
- functions working with functions `identity`, `constantly`, ...

```scheme
(nil? nil) ;; => true
(string? "") ;; => true
(boolean? true) ;; => true
(int? 1) ;; => true
(fn? +) ;; => true
(vector? [1 2 3]) ;; => true

(+ 1 2 3) ;; => 6
(* 1 2 3) ;; => 6
(< 1 2) ;; => true

(inc 10) ;; => 11
(dec 10) ;; => 9
(mod 10 3) ;; => 1
(abs -10) ;; => 10
(max 5 10 15) ;; => 15

(zero? 0) ;; => true
(pos? 1) ;; => true
(even? 10) ;; => true
(odd? 11) ;; => true

(identity 10) ;; => 10
(identity "mama") ;; => "mama"

((constantly 666) 1) ;; => 666
((constantly 666) 1 2 3) ;; => 666

((complement empty?) [])   ;; => false

(and (> 2 1) (string/ends-with? "tata" "ta") false) ;; => false
(or (> 2 1) (string/ends-with? "tata" "ta") false) ;; => true

;; falshy - only false oraz nil
(and "" 0) ;; => 0
(and "" 0 false) ;; => false
(and "" 0 nil) ;; => nil

(or "" 0) ;; => ""
```

```js
var nilp = (x) => typeof x === "undefined" || x === null;
var stringp = (x) => typeof x === "string";
var booleanp = (x) => typeof x === "boolean";
var numberp = (x) => typeof x === "number";
var functionp = (x) => typeof x === "function";

var plus = (...args) => args.reduce((p, c) => p + c, 0);
var minus = (...args) =>
  args.length === 1 ? -args[0] : args.reduce((p, c) => p - c);
var multiply = (...args) => args.reduce((p, c) => p * c, 1);
var mod = (...args) => args.reduce((p, c) => p * c, 1);
var inc = (x) => x + 1;
var dec = (x) => x - 1;
var mod = (x, div) => x % div;
var abs = (x) => Math.abs(x);
var max = (...xs) => Math.max(...xs);
var min = (...xs) => Math.min(...xs);
var evenp = (x) => x % 2 === 0;
var oddp = (x) => x % 2 === 1;
var zerop = (x) => x === 0;
var posp = (x) => x > 0;
var str = (...args) => args.join("");

var identity = (x) => x;
var constantly = (x) => () => x;
var complement = (f) => (...args) => !f(...args);

2 > 1 && "tata".endsWith("ta") && false; // => false
2 > 1 || "tata".endsWith("ta") || false; // => true

// ... in JS "" and 0 are also falsy
"" && 0; // => ""
"" && 0 && false; // => ""
"" && 0 && null; // => ""

"" || 0; // => 0
```

Unlike in most programming languages, in Lisp, characters `?` or `-` can be used as part of identifiers, variable or function names. There is also a naming convention for predicates (functions returning `boolean` values) with `?` at the end, for instance `(nil? ..)` or `(even? ...)`.

Lisp uses `nil` values, the same as `null` in other languages. But there is also an additional concept called "falsy and truthy values", which I have encountered only in JavaScript. Boolean values are mostly used inside conditions, for instance in `if` or `while` statements. The idea behind "falsy and truthy values" is that any expression in the language can be treated as a condition, it can always be converted into a `boolean` type. For instance, in JavaScript values like `false`, `null`, `undefined`, `0`, `""` are treated as "true", all other expressions are "false". That's why we often find code like `var city = (person && person.address && person.address.city) || ""`. It means "define a new variable `city` with value of `person.address.city`, but set empty string `""` only if `person` or `person.address` or `person.address.city` don't exist. In Clojure dialect of List "falsy and truthy values" work similar to JavaScript, but there are some small differences. According to [the documentation](https://clojure.org/guides/learn/flow#_truth): " In Clojure, all values are logically true or false. The only "false" values are `false` and `nil` - all other values are logically true".

The last interesting detail is that many of the functions above are variadic. In many cases passing any number of arguments is quite convenient, for instance the conditions can be written like this `(and ... (or ... ... ...) ...)`, or the arithmetic operations like this `(+ 5 10 15)`, `(min 5 10 15)`. The same function can be called with many argument values and one value representing the collection of items. We can calculate the sum of all numbers stored in the variable `my-numbers` by calling the helper function `(apply + my-numbers)`.

## 'apply' function

The `apply` function takes two arguments: a function and a collection of values. It calls the function passing values as arguments and returns the result.

```scheme
(apply add [1 2]) ;; => 3
(apply * [2 3]) ;; => 6
```

```js
add.apply(null, [1, 2]);
```

The JavaScript function is a regular object containing members like properties or methods. Method `apply` works almost the same as the Clojure `apply` function presented above. The difference is an additional argument `null`. In the object-oriented programming style (OOP), a method is just a function called in the context of some additional object; for instance, `firstName.substring(1)` is just a function `substring` of type `string`. In functional or procedural programming, we pass all data as function arguments, like `substring(firstName, 1)`. Both JavaScript and Lisp treat functions as values. We can create a variable referencing a method `const substr = firstName.substring` and execute it later, execution of `substr.apply("JavaScript", [1])`  returns `"avaScrip"`. The first argument of the `apply` function represents "this" object, the `substring` function was called in the context of the `"JavaScript"` value instead of the `firstName` variable.

### 'partial' function

`partial` is another builtin function working with function values. The term "partial function application" in functional languages means "calling" a function without passing all argument values. In such a case, the function's body can not be executed, instead a new function is created. All passed arguments are bound and the returned function expects only the missing arguments. Let's say we have `add` function that adds two numbers, we can easily create a new function `increment` that increments by one the passed argument. 

```scheme
(def increment
  (partial add 1))
(increment 10) ;; => 11
((partial add 1) 10) ;; => 11
```

```js
var inc = add.bind(null, 1);
inc(10); // => 11
add.bind(null, 1)(10); // => 11
```
Analogously to the `apply` function, a JavaScript object representing the function contains a method `bind`.

## 'comp' function (function composition)

The next `comp` function implements the math term function composition. Let's say we have an `increment` function and a `str` function that converts any number into a string. Now we would like to create a new function taking a number and returning a string working like this `arg => str(increment(increment(arg)))`. A new composed function increments the number by two and converts it to a string.


```scheme
(def increment-twice-then-to-string
  (comp str increment increment))

(increment-twice-then-to-string 10) ;; => "12"
```

```js
var comp = (...funcs) => arg => funcs.reduceRight((a, f) => f(a), arg);

var incrementTwiceThenToString = comp(str, increment, increment);
incrementTwiceThenToString(10); // => "12"
```

This time there is no built-in function in JavaScript, but such a function can be easily implemented using the `reduceRight` array method. Many programming languages provide built-in functions or operators supporting function composition. For instance, in Haskell language `.` operator is used and that corresponds to [the math dot notation](https://en.wikipedia.org/wiki/Function_composition). Our previous example would look like this `str . increment . increment` in Haskell. The order of functions seems unnatural, because it tries to simulate the math notation. 

## Summary

In this article, we have implemented some basic built-in Clojure functions and operators in JavaScript. Next, we will discuss data structures like immutable linked lists and maps. 
