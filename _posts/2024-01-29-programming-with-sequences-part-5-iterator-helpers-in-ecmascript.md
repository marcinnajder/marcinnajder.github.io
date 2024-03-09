---
layout: post
title: Programming with sequences part 4 - iterator helpers in ECMAScript
date: 2024-01-29
tags: js powerseq
published: false
series: programming-with-sequences
---

{%- include /_posts/series-toc.md series-name="programming-with-sequences" -%}

## Introduction

At this point we should have a good understanding of iterators and generators in JS. Those fundamental building blocks allows us to write very elegant functional code using operators like `filter`, `map`, `reduce`, ... . [powerseq](https://github.com/marcinnajder/powerseq) library provides more than 70 general purpose operators. One of the key design decision behind this library is to allow for introducing a completely new operators and glue them together with existing ones. Operators are just a simple functions with a specified signatures (they take sequences and return a new sequences or values), then `pipe` function compose them into single query expression. Personally, I cannot write code without those operator functions and it doesn't matter which programming language I am currently programming in ( TS, C#, F#, Kotlin, Clojure, ...). It's just my way of looking at programing problems.

Nowadays most of the technologies provide builtin implementations of such operators, so we could ask ourself whether the community behind the JS standard is thinking about this too. There is an official [proposal-iterator-helpers](https://github.com/tc39/proposal-iterator-helpers) at stage 3 at this moment (march of 2024). In this article we will see in great details how this feature has been designed and what is the difference between powerseq approach and this proposal. We even try to implement this proposal ourself from scratch both on the JavaScript level and TypeScript level.

#### `Iterable` and `Iterator` objects

First let's make sure that we fully understand `Iterable` and `Iterator` objects in JS. There are some details crucial to grasp the design decision behind iterator helpers proposal. This is how iterator pattern is defined in TypeScript language:

```typescript
interface Iterable<T> {
  [Symbol.iterator](): Iterator<T>;
}
interface Iterator<T> {
  next(): { done: boolean; value: T };
}
```

`Iterable` interface introduced in ES2015 defines the common API used for iterating over the elements of collections like `Array`, `Set` or `Map`. Even `String` type implements this interface so we could iterate over single characters. Thanks to the new kind of loop`for/of`, we can walk through all items in a very elegant way.

```typescript
const iterable: Iterable<number> = [1, 2, 3];
for (const item of iterable) {
  console.log(item);
}
```

We can think about `for/of` loop like a `while` loop using the **iterator** object for proceeding the iteration.

```typescript
const iterator = iterable[Symbol.iterator]();
let iteratorResult: IteratorResult<number>;

while (!(iteratorResult = iterator.next()).done) {
  console.log(iteratorResult.value);
}
```

It's worth pointing out, there are two separate objects: **iterable** and **iterator**:

```typescript
const iterable: Iterable<number> = [1, 2, 3];
// creating new iterator, it's like calling "iterable.getIterator()"
const iterator: Iterator<number> = iterable[Symbol.iterator]();

console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: 3, done: false }
console.log(iterator.next()); // { value: undefined, done: true }
```

Every time we talk about `Iterable` interface want to emphasize that it allows us to represent "lazy sequences". `Array` type represents "collection", it means that we have some values stored in memory and also `length` property getting the count of elements. The sequence type is just an algorithm for generating values, because there is no `length`property the sequence can be infinite. Laziness of the sequence means that the values are generated on demand, so even if the sequence produces values from 1 to 1000, the consuming code can decide to process only few first items. Laziness is a very powerful feature and we can ask ourself which interface (**iterable** or **iterator**) is directly responsible for this trait? Let's take a look at the following code:

```typescript
const infiniteIterator: Iterator<number> = {
  next() {
    return { value: 666, done: false };
  },
};
console.log(infiniteIterator.next()); // { value: 666, done: false }
console.log(infiniteIterator.next()); // { value: 666, done: false }
```

This code proves that only **iterator** object itself can represent lazy sequence. So why do we even need a second object like **iterable** ? Let's image that we have an array of values `[1, 2, 3]` and `Array` type would implement only `Iterator` interface. That allows us to iterate over all values but unfortunately only once. There is no way to reset **iterator** object to be set the beginning of the sequence. `Iterable<T>` interface works like a factory for new `Iterator` objects. `Array` type implements `Iterable` so we can potentially create many different **iterator** object at the same time. One of them could be located at the beginning of sequence, the other one could be in the middle. We can iterate over `Array` values many times because each new **iterator** object start from the beginning. It's very natural for developers to iterate over the same instance of `Array` many times using many `for/of` loop. But this does not have to be this way, because this is another possible behaviour that we will talk next.

#### Generators create iterable iterators

ES2015 introduced two connected but still distinct features: iterators and generators. In case of iterators it's all about `Iterable` interfejs and `for/of` loop. Generators are special functions that allows to create objects implementing `Iterable` interface in a very convenient way .

```typescript
function* return123() {
  yield 1;
  yield 2;
  yield 3;
}

const iterable: Iterable<number> = return123();
for (const item of iterable) {
  console.log(item);
}
```

But there are two interesting facts about object returned from generator function. The first one is that those objects not only implement `Iterable` interface but also `Iterator` interface. We can write the following code:

```typescript
const iterator: Iterator<number> = return123();

console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: 3, done: false }
console.log(iterator.next()); // { value: undefined, done: true }
```

The second fact is even more interesting. Because object returned from generator function implements `Iterable` interface, we can potentially create many different **iterator** objects. But it turns out that every time we want to create a new **iterator** object, the same instance in memory is returned. And this is the same instance as the object returned from generator function.

```typescript
const generatorResult = return123();
const iterable: Iterable<number> = generatorResult;
const iterator: Iterator<number> = generatorResult;

console.log(iterable[Symbol.iterator]() === iterable[Symbol.iterator]()); // true
console.log(iterable[Symbol.iterator]() === iterator); // true
```

It was quite shocking for me when I encountered this behavior for the first time implementing [powerseq](https://github.com/marcinnajder/powerseq) library. Especially that other technologies I knew back then (like C#, F#, Dart, ...) worked differently. As far as I know at least Python works the same as JavaScript. Let's take a look at very simple example code:

```typescript
const iterable = return123();
for (const item of iterable) {
  console.log(item);
}
for (const item of iterable) {
  // this is loop is not executed at all
  console.log(item);
}
```

Function `return123` was implemented as generator function. Each`for/of` loop asks for new **iterator** object but the same instance is returned. Writing code like this I would expect that two independent iteration over sequence would be executed but in reality only first loop is executed. **iterator** object is located at the end of sequence after the first loop, the second loop reuse its instance. For me it was quit unexpected and unnatural. That's why all operators from powerseq library returning sequences don't behave this way. Returned sequences always create new instance of **iterator** so each iteration starts the execution from the beginning.

#### Iterator helpers proposal

Let's take a look at the official example from [proposal-iterator-helpers](https://github.com/tc39/proposal-iterator-helpers)

```javascript
function* naturals() {
  let i = 0;
  while (true) {
    yield i;
    i += 1;
  }
}

const result = naturals().map((value) => {
  return value * value;
});
result.next(); //  {value: 0, done: false};
result.next(); //  {value: 1, done: false};
result.next(); //  {value: 4, done: false};
```

Generator function `naturals` returns sequence of natural numbers starting from `0`, then `map` method is called. Where does `map` come from and what we can tell about its signature ? As we have just discussed, generator functions return objects that are **iterable** and **iterator** simultaneously. Just after looking at the code above, we are not able to guess whether `map` method is a member of `Iterable` or `Iterator` interface. One thing we can definitely say about the result of `map` method is that it implements `Iterator` interface. Maybe the result also implements `Itereable`, but the code says about about it.

The first difference between operators from powerseq library (and many other similar solutions from different technologies) and JavaScript iterator helpers proposal. In the first case the signature of operator looks like this `(source: Iterable<...>, ...): Iterable<...>`, it takes and returns `Iterable`. In the second case it looks like this `(source: Iterator<...>, ...): Iterator<...>`, it takes and returns `Iterator`. This proposal describes many useful operators, some of them return lazy sequences of values (`map, filter, take, drop, flatMap` ), other return scalar values immediately (`reduce, toArray, forEach, some, every, find`). Unfortunately the API of `Iterator` interface allows us to iterate sequence of value only once.

The second difference is that operators in powerseq are just regular functions. Library provides many builtin operators but final user can always decide to write its own operator or reimplement existing one. Function `pipe` combines any operators the same way. In case of iterator helpers proposal operators are instance methods of `Iterator` interface so there is no elegant way of extending the list of standard operators. We can change dynamically prototype JS object in runtime (we will be talking about it further), but this is more like a unrecommended hack. I am aware know this problem comes from OOP (Object Oriented Programming) design of this feature but for me personally FP (Functional Programming) approach (separating data and behaveor) would be better here. Especially if we think about another [pipeline operator proposal](https://github.com/tc39/proposal-pipeline-operator)
