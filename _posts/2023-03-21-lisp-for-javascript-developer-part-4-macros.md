---
layout: post
title: Lisp for JavaScript developer part 4 - macros [JavaScript, Lisp]
date: 2023-03-21
tags: js lisp
series: jslisp
---

{%- include /_posts/series-toc.md series-name="jslisp" -%}

## Introduction 

At the end of the previous article, we saw the simple query using the `filter` and `map` functions.


```scheme
(map
 (fn [x] (* x 10))
 (filter odd? [1 2 3 4 5 6])) ; => (10 30 50)
```

Of course, there are many more useful functions working with collections. The order in which we write them in a single expression is unnatural. First, the `filter` function is executed, and only then the `map`, but we write them in the opposite order. This article teaches a unique concept of Lisp called a macro. It's like a function with special capabilities thanks to the feature of Lisp called "homoiconicity'. In Lisp, the code can be treated as data, and the data can be treated as code. This allows us to generate and modify the code "on the fly" at runtime. It is sometimes called meta-programming or dynamic programming. We can implement our macros or use existing ones, for example `->>` macro simplifies the way we write queries.

## Macros

The following code snippet `(/ (+ 10 5) 3)` executes and returns `5`. We can always precede some Clojure code with a single quote like `'(/ (+ 10 5) 3)`. This means that the code should not be executed immediately, but we want to treat it as a data structure. We can manipulate the data freely to finally turn it back into code. Such a dynamically built code can be also executed.

```scheme
(def forms-1 '(/ (+ 10 5) 3))

(let [[op-1 [op-2 a b] c] forms-1]
  (println "operators:" op-1 op-2) ;; operators: / +
  (println "values: " a b c) ;; values:  10 5 3
  `(/ ~a ~b)) ;; => 2
```

```js
var a = 10;
var b = 5;
eval(`${a} / ${b}`);
```

In the example above, the destructuring feature of Clojure is used to extract parts of our expression `'(/ (+ 10 5) 3)` into separate variables `op-1, op-2, a, b, c`. Then we print them into the console, and at the end, a new expression `(/ 10 5)` is created and executed, returning `2`. No analogous mechanism in JavaScript gives us access to the parsed representation of code, but there is a special function `eval`. This function allows the execution of any string representing the correct JavaScript code. It should never be used in production code because it is insecure and inefficient. Here, only for demonstration purposes, the string `"10 / 5"` is created using JavaScript template literals feature and executed, returning the same value `2`.

#### ->> "thread-last" macro

In the code below `->>` symbol looks like an execution of a regular function in Clojure, but it is a macro. The macro is like a function, but all parameters are put in quotes automatically. Instead of passing the evaluated value of a parameter known before starting the function execution, the whole data representation of the parameter is passed into the macro. Inside the macro, we have access to all parameters in their original shapes, and we can manipulate them by building a new Clojure expression. `->>` symbol is called "thread-last" macro, because it takes the first parameter and places it as the last parameter of the function call.  The best way to explain its behavior is to use a simple example, this piece of code `(->> 5 (mul 100) (inc 1)))` will be transformed into this `(inc 1 (mul 100 5))`.  Thanks to this macro, queries over collections of items using functions like `filter, map, ...` look very natural. The execution order is directly represented in code; first `filter` will be executed, then `map`. 

```scheme
(->>
 [1 2 3 4 5 6]
 (filter odd?)
 (map #(* % 10)))

(macroexpand
 '(->>
   [1 2 3 4 5 6]
   (filter odd?)
   (map #(* % 10))))

;; => (map (fn* [p1__8212#] (* p1__8212# 10)) (filter odd? [1 2 3 4 5 6]))
```

```js
into(vector(), map(x => x * 10, filter(oddp, list(1, 2, 3, 4, 5, 6))));
pipe(
    list(1, 2, 3, 4, 5, 6),
    o => filter(oddp, o),
    o => map(x => x * 10, o),
    o => into(vector(), o)
); // => [ 10, 30, 50 ]
```

The built-in `macroexpand`function shows what is happening behind any macro by returning the final representation after transformation. We can use a simple helper function `pipe` imported from the powerseq library to simulate macro behavior in JavaScript. This function takes some value as the first parameter and applies it to the function passed as a second argument, then the result is applied to the next function, and so on. We will come back to this function in the following article.

#### -> "thread-first" macro

In collection functions like `filter, map, ...`, the last argument is a collection. That was the reason we were using the "thread-last" macro. There is also the "thread-first" macro which sets the first argument. For instance, functions like `assoc` or `dissoc` take a map data structure as the first argument. 

```scheme
(assoc (assoc {} :name "marcin") :age 123) ;; => {:name "marcin", :age 123}

(->
 {}
 (assoc :name "marcin")
 (assoc :age 123))
;; => {:name "marcin", :age 123}

(macroexpand
 `(->
   {}
   (assoc :name "marcin")
   (assoc :age 123)))
;; => (clojure.core/assoc (clojure.core/assoc {} :name "marcin") :age 123)
```

```js
assoc(assoc({}, "name", "marcin"), "age", 123);
pipe(
    {},
    o => assoc(o, "name", "marcin"),
    o => assoc(o, "age", 123)
); // => { age: 123, name: 'marcin' }
```

#### Summary

Quotations with macros are very powerful features heavily used in languages from the Lisp family. Other languages inspired by the same idea introduced similar elements. In 2008, C# 3.0 introduced LINQ (language integrated query) using [expression trees](https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/expression-trees) internally, F# language has [code quotations](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/code-quotations). However, because of the Lisp syntax, the macro feature is natural and straightforward to implement.

