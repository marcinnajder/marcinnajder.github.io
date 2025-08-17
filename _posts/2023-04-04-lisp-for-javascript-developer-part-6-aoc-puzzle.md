---
layout: post
title: Lisp for JS developers part 6 - AoC puzzle [JS, Lisp]
date: 2023-04-04
tags:
  - js
  - lisp
series: jslisp
---

{%- include /_posts/series-toc.md series-name="jslisp" -%}

## Advent Of Code 2022 Day 1

In the last article of the series, we will implement the whole working solution for [Advent Of Code 2022 Day 1](https://adventofcode.com/2022/day/1) puzzle. Our program takes a list of numbers divided into groups using blank lines.

```
1000
2000
3000

4000

5000
6000

7000
8000
9000

10000
```

In the first part of the puzzle we have to sum up numbers inside groups to find the biggest sum. For our sample data the correct answer is `24000` (`7000+8000+9000`). In the second part, we must find the sum of the top three groups. This time the correct answer is `45000` (`24000+11000+1000`).

```scheme
(defn load-data [text]
 (->> text
 string/split-lines
 (reduce (fn  [lists line]
 (if (string/blank? line)
 (cons '() lists)
 (let [[inner-list & tail] lists]
 (cons (cons (Integer/valueOf line) inner-list) tail))))
 '(()))))

(defn insert-sorted [xs x]
 (if (empty? xs)
 (list x)
 (let [[head & tail] xs]
 (if (< x head)
 (cons x xs)
 (cons head (insert-sorted tail x))))))

(defn insert-sorted-preserving-length [xs x]
 (if (<= x (first xs))
 xs
 (rest (insert-sorted xs x))))

(defn puzzle [text top-n]
 (->> text
 load-data
 (map #(apply + %))
 (reduce insert-sorted-preserving-length (into '() (repeat top-n 0)))
 (apply +)))

(defn puzzle-1 [text]
 (puzzle text 1))

(defn puzzle-2 [text]
 (puzzle text 3))
```

The `load-data` function takes a text representation of input and parses it, returning lists of numbers like `'((1000 2000 3000) (4000) (5000 6000) ...)`. Both parts of the puzzle were implemented using the same common function, `puzzle,` parameterized with a `top-n` argument. For the first part of the puzzle, the value was `1`, and for the second, the value was `3`.

We need to find the biggest `n` values in the collection, so the natural solution would be sorting and then taking the first `n` elements. But sorting is quite expensive, and frankly speaking, we don't need to store all elements at all. We need to hold the top `n` numbers throughout the whole computation. It's worth mentioning that a sample text file contains only 14 lines, but the final text file contains more than 2000 lines.

An immutable linked list is the best data structure for the job. Function `insert-sorted` returns a new list with an element `x` inserted into the appropriate location so that the returned list remains sorted. Let's say we have a list of `5, 10, 15` and want to add a new value `2`. We check the head of the list, which is `5`, and because the inserted value is smaller than the current smallest, it becomes a new head pointing to an existing list. Inserting a new element at the beginning is a highly cheap operation. Inserting a new element in the middle of the list rewrites previous elements and leaves further elements unchanged. Function `insert-sorted-preserving-length` executes `insert-sorted` internally, preserving the length of the list. Inserting `8` into `5, 10, 15` returns `8, 10, 15`.

Now, let's implement exactly the same algorithm in JavaScript. Because all necessary Clojure API have already been implemented, the code can be transformed almost directly line by line.

```js
var integer_valueOf = (text) => parseInt(text);
var string_splitLines = (text) => text.split(os.EOL);
var string_blankp = (text) => text.trim().length === 0;
var slurp = (filePath) => fs.readFileSync(filePath, "utf-8");

function loadData(text) {
  return pipe(text, string_splitLines, (o) =>
    reduce(
      (lists, line) =>
        string_blankp(line)
          ? cons(list(), lists)
          : cons(cons(integer_valueOf(line), first(lists)), rest(lists)),
      list(list()),
      o
    )
  );
}

function insertSorted(xs, x) {
  return emptyp(xs)
    ? list(x)
    : x < first(xs)
    ? cons(x, xs)
    : cons(first(xs), insertSorted(rest(xs), x));
}

function insertSortedPreservingLength(xs, x) {
  return x <= first(xs) ? xs : rest(insertSorted(xs, x));
}

function puzzle(text, topN) {
  return pipe(
    text,
    loadData,
    (o) => map((xs) => apply(plus, xs), o),
    (o) => into(vector(), o),
    (o) =>
      reduce(insertSortedPreservingLength, into(list(), repeat(topN, 0)), o),
    (o) => apply(plus, o)
  );
}

function puzzle1(text) {
  return puzzle(text, 1);
}
function puzzle2(text) {
  return puzzle(text, 3);
}
```

## Summary

We have reached the end of the series. My main goal was to encourage developers to learn new programming languages, even strange-looking ones like Lisp. Lisp and functional programming show how to write simple, readable, and reliable programs. This introduction to List was intentionally written in a very lightweight way, without going too deeply into the details. A few lines of code explain some concept better than a whole paragraph of text. JavaScript is the most often used programming language in the world, and there are many similarities between Lisp and JavaScript. I hope you have noticed a few of them.
