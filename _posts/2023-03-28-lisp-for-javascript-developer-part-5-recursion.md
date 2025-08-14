---
layout: post
title: Lisp for JS developers part 5 - recursion [JS, Lisp]
date: 2023-03-28
tags:
  - js
  - lisp
series: jslisp
---

{%- include /_posts/series-toc.md series-name="jslisp" -%}

## Recursion

In this short article, we will discuss recursion. We were using only basic types, different types of collections, functions with local variables, and if-then-else expressions—no loops at all. In function programming, recursion is often used instead of typical loops like `for`, `while`, or `foreach`. Let's write a function counting the sum of all numbers stored in a linked list.

```scheme
(defn sum-numbers [numbers]
  (if
   (empty? numbers)
    0
    (let [[head & tail] numbers]
      (+ head (sum-numbers tail)))))

(sum-numbers '()) ; => 0
(sum-numbers '(1 2 3)) ; => 6
```

If the list is empty, the value `0` is returned. Otherwise, the first number is added to the sum of all numbers without the first number.

Let's analyze what is really happening during the execution. Any function call puts all arguments and local variables onto the stack; they stay there until the function returns the result. We run the following expression `(sum-numbers '(1 2 3))`, next the same function is executed recursively `(+ 1 (sum-numbers '(2 3)))`. One function needs to wait for the result of the other function because it has some additional task to do (adding value `1` to the result). In case of recursively executed functions, there are long chains of calls, depending on the scenario, that can even cause "a stack overflow exception". We can change this function a little bit to show what "tail-call optimization" is.

```scheme
(defn sum-numbers-with-tail-call [numbers total]
  (if
   (empty? numbers)
    total
    (let [[head & tail] numbers]
      (sum-numbers-with-tail-call tail (+ head total)))))

(sum-numbers-with-tail-call '(1 2 3) 0) ; => 6
```

The crucial difference lies in the last expression inside the function body, `(sum-numbers-with-tail-call tail (+ head total))`. First function with the following values is called `(sum-numbers-with-tail-call '(1 2 3) 0)`, then `(sum-numbers-with-tail-call '(2 3) 1)`. Because there is no additional operation besides calling the same function recursively, the calling function does not need to stay on the stack waiting for the result. Tail-call optimization can be potentially used by the compiler or the runtime.

In an imperative style of programming loops are everywhere, in most cases they are replaced by recursion in a function style. But code using recursion can be repetitive and harder to understand. That's why functions like `map, filter, reduce` were invented. The internals of collection iteration are hidden from the caller, focusing only on essential logic. Clojure language provides few helper functions working like a loop, one of them is called `loop`.

```scheme
(defn sum-numbers-with-tail-loop [numbers]
  (loop [lst numbers
         total 0]
    (if
     (empty? lst)
      total
      (let [[head & tail] lst]
        (recur tail (+ head total))))))

(sum-numbers-with-tail-loop '(1 2 3))
```

The `loop` function is a generic representation of a loop in code. It allows us to define a state initialized with default values represented as any number of local variables. The special symbol `recur` starts the next iteration of the loop, passing a new state. The loop can be stopped by returning the final result instead of calling `recur`.

It's interesting how easily the same functionality of `loop` can be implemented in JavaScript :)

```javascript
function loop(state, body) {
  return body(state);
}

function sumNumbersWithTailLoop(numbers) {
  return loop({ lst: numbers, total: 0 }, function recur({ lst, total }) {
    return emptyp(lst)
      ? total
      : recur({ lst: rest(lst), total: total + first(lst) });
  });
}

console.log(sumNumbersWithTailLoop(list(1, 2, 3))); // => 6
```

To summarize, any loop inside our code can also be written using recursion. However, such code can be very tricky, and this is why Lisp programmers often use higher-order functions like `map, filtr, reduce`.
Recursion requires introducing a new function that includes at least two branches of execution, one of which has to be a stop condition. Thanks to the `loop` macro, boilerplate code can be avoided in most cases.
