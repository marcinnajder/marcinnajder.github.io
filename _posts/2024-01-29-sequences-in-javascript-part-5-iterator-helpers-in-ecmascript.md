---
layout: post
title: Sequences in JS part 5 - iterator helpers in ECMAScript [TS]
date: 2024-01-29
tags:
  - js
  - powerseq
published: true
series: sequences
---

{%- include /_posts/series-toc.md series-name="sequences" -%}

## Introduction

At this point, we should have a good understanding of iterators and generators in JS. Those fundamental building blocks allow us to write very elegant functional code using operators like `filter`, `map`, `reduce`, ... . [powerseq](https://github.com/marcinnajder/powerseq) library provides more than 70 general purpose operators. One of the key design decisions behind this library is to allow for introducing completely new operators and glue them together with existing ones. Operators are just simple functions with specified signatures, they take sequences and return new sequences or values. The `pipe` function composes operators into a single query expression. I cannot write code without those operator functions and it doesn't matter which language I am currently programming in ( TS, C#, F#, Kotlin, Clojure, ...). It's just the way I am looking at the coding problems.

Nowadays most of the technologies provide built-in implementations of such operators, so we could ask ourselves whether the community behind JS standard is thinking about this too. There is an official [proposal-iterator-helpers](https://github.com/tc39/proposal-iterator-helpers) at stage 3 at this moment (march of 2024). In this article, we will see in great detail how this feature has been designed and what is the difference between the approach taken by powerseq and this proposal. We even try to implement this proposal ourselves from scratch both on the JavaScript and TypeScript levels.

## `Iterable` and `Iterator` objects

First, let's make sure that we fully understand the `Iterable` and `Iterator` objects in JS. There are some details crucial to grasping the design decision behind the iterator helpers proposal. This is how the JS iterator pattern is defined using TS language constructs:

```typescript
interface Iterable<T> {
  [Symbol.iterator](): Iterator<T>;
}
interface Iterator<T> {
  next(): { done: boolean; value: T };
}
```

The `Iterable` interface introduced in ES2015 defines the common API used for iterating over the elements of collections like `Array`, `Set`, or `Map`. Even `String` implements this interface so we can iterate over single characters. Thanks to the new kind of loop called `for/of`, we can walk through all items in a very elegant way.

```typescript
const iterable: Iterable<number> = [1, 2, 3];
for (const item of iterable) {
  console.log(item);
}
```

We can think about `for/of` loop like a `while` loop using the **iterator** object for the execution of iteration.

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
// creating a new iterator, it's like calling "iterable.getIterator()" function
const iterator: Iterator<number> = iterable[Symbol.iterator]();

console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: 3, done: false }
console.log(iterator.next()); // { value: undefined, done: true }
```

The `Iterable` interface allows us to represent "lazy sequences". `Array` type represents "collection", which means that we have some values stored in memory and also the `length` property getting the count of elements. The sequence type is just an algorithm for generating values. There is no `length` property in the sequence so it can be infinite. Laziness of the sequence means that the values are generated on demand, so even if the sequence produces values from 1 to 1000, the consuming code can decide to process only a few first items. Laziness is a very powerful feature, the question is which interface (**iterable** or **iterator**) is directly responsible for that trait? Let's take a look at the following code:

```typescript
const infiniteIterator: Iterator<number> = {
  next() {
    return { value: 666, done: false };
  },
};
console.log(infiniteIterator.next()); // { value: 666, done: false }
console.log(infiniteIterator.next()); // { value: 666, done: false }
```

The code above proves that the **iterator** object itself can represent a lazy sequence. So why do we even need a second object like **iterable** ? Let's imagine that we have an array of values `[1, 2, 3]` and the `Array` type would implement only the `Iterator` interface. That allows us to iterate over all values but unfortunately only once. There is no way to reset the **iterator** object to be set at the beginning of the sequence. The `Iterable<T>` interface works like a factory for new `Iterator` objects. `Array` type implements `Iterable` so we can potentially create many different **iterator** objects at the same time. One of them could be located at the beginning of a sequence, the other one could be in the middle. We can iterate over `Array` values many times because each new **iterator** object starts from the beginning. It's very natural for developers to iterate over the same instance of `Array` many times using the `for/of` loop. But this does not have to be this way, because this is another possible behavior that will be discussed next.

## Generator functions create iterable iterators

ES2015 introduced two connected but still distinct features, iterators and generators. In the case of iterators, it's all about `Iterable` interfejs and `for/of` loop. Generators are special functions that allow to creation objects implementing the `Iterable` interface in a very convenient way.

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

But there are two interesting facts about objects returned from the generator function. The first one is that those objects not only implement the `Iterable` interface but also the `Iterator` interface. We can write the following code:

```typescript
const iterator: Iterator<number> = return123();

console.log(iterator.next()); // { value: 1, done: false }
console.log(iterator.next()); // { value: 2, done: false }
console.log(iterator.next()); // { value: 3, done: false }
console.log(iterator.next()); // { value: undefined, done: true }
```

The second fact is even more interesting. Because the object returned from the generator function implements an `Iterable` interface, we can potentially create many different **iterator** objects. But it turns out that every time we want to create a new **iterator** object, the same instance in memory is returned. This is the same instance as the object returned from the generator function.

```typescript
const generatorResult = return123();
const iterable: Iterable<number> = generatorResult;
const iterator: Iterator<number> = generatorResult;

console.log(iterable[Symbol.iterator]() === iterable[Symbol.iterator]()); // true
console.log(iterable[Symbol.iterator]() === iterator); // true
```

It was quite shocking for me when I encountered this behavior for the first time implementing [powerseq](https://github.com/marcinnajder/powerseq) library. Especially since other technologies I knew back then (like C#, F#, Dart, ...) worked differently. As far as I know, at least Python works the same as JavaScript. Let's take a look at a very simple example code:

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

The function `return123` was implemented as a generator function. Each `for/of` loop asks for a new **iterator** object but the same instance is returned. Writing code like this I would expect that two independent iterations over sequence would be executed but in reality, only the first loop is executed. **iterator** object is located at the end of the sequence after the first loop, the second loop reuses its instance. For me, it was quite unexpected and unnatural. That's why all operators from powerseq library returning sequences don't behave this way. Returned sequences always create new instances of **iterator** so each iteration starts the execution from the beginning.

## Iterator helpers proposal

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

The generator function `naturals` returns a sequence of natural numbers starting from `0`, then the `map` method is called. Where does `map` come from and what we can tell about its signature? As we have just discussed, generator functions return objects that are **iterable** and **iterator** simultaneously. Just after looking at the code above, we are not able to guess whether the `map` method is a member of the `Iterable` or `Iterator` interface. One thing we can say about the result of the `map` method is that it implements the `Iterator` interface. Maybe the result also implements `Itereable`, but the code says nothing about it.

The first difference between operators from powerseq library (and many other similar solutions from different technologies) and JavaScript iterator helpers proposal is the type representing the sequence. In the first case the signature of the operator looks like this `(source: Iterable<...>, ...): Iterable<...>`, it takes and returns `Iterable`. In the second case it looks like this `(source: Iterator<...>, ...): Iterator<...>`, it takes and returns `Iterator`. This proposal describes many useful operators, some of them return lazy sequences of values (`map, filter, take, drop, flatMap` ), and others return scalar values immediately (`reduce, toArray, forEach, some, every, find`). Unfortunately, the API of the `Iterator` interface allows us to iterate a sequence of values only once.

The second difference is that operators in powerseq are just regular functions. The library provides many built-in operators but the final user can always decide to write their operator or reimplement the existing one. Function `pipe` combines any operators (existing or new) in the same way. In the case of iterator helpers proposal operators are instance methods of the `Iterator` interface so there is no elegant way of extending the list of standard operators. We can change dynamically JS prototype objects at runtime (we will be talking about it further), but this is more like an unrecommended hack. I am aware that this problem comes from the OOP design (Object Oriented Programming) of this feature but for me, the FP (Functional Programming) approach (separating data and behavior) would be better here. Especially if we think about another [pipeline operator proposal](https://github.com/tc39/proposal-pipeline-operator) that would allow us to write something like this `[1, 2, 3] |> filter(x => x % 2 === 0) |> map(x => x.toString())`.

There is also one interesting aspect of using `Iterator` instead of `Iterable`. When the `Iterable` interface was introduced in ES2015 many other useful features were added too. TypeScript language provides special declaration files describing JavaScript API. If we look at `node_modules/typescript/lib/lib.es2015.iterable.d.ts` file we can find all places where the `Iterable` interface is used:

- `String` type, collections (`Array`, `Map`, `Set`, `WeakMap`, `WeakSet`) and `arguments` object implement `Iterable`interface
- `Promise.all` and `Promise.race` take `Iterable` interface as arguments
- `for/of` loop iterates over `Iterable` interface
- spread operator works with with `Iterable` interface, so we can write `["a", ..."bcd", "e"]` returning `["a", "b"," c", "d", "e"]`

Because powerseq library uses `Iterable` interface everything seems very easy and natural.

```typescript
for (const digit of filter("ab5c10d15ef", (c) => c >= "0" && c <= "9")) {
  console.log(digit); // 5, 1, 0, 1, 5
}
```

The same code using a new iterator helper proposal can be written like this.

```typescript
for (const digit of Iterator.from("ab5c10d15ef")
  .filter((c) => c >= "0" && c <= "9")
  .toArray()) {
  console.log(digit); // 5, 1, 0, 1, 5
}
```

First, we have to wrap `String` type into `Iterator` interface, then we can call any of the proposed methods. If the final result represents a sequence of values like in the code above, the returned type is `Iterator`. Maybe it will also implement the `Iterable` interface but so far we just don't know. We have to convert a sequence to a new `Array` object just to iterate over its items using the `for/of` loop. The alternative option is to iterate over items using the provided `forEach` method as follows `... .forEach(digit => console.log(digit))`. Fortunately the built-in collections like `Array`, `Maps`, `Set` contain methods `entries`, `keys`, and `values` returning a sequence of items are represented as objects implementing both `Iterable` and `Iterator` interfaces. Thanks to this API, we will be able to write the following code:

```typescript
console.log(
  [1, 2, 3, 4, 5]
    .values()
    .filter((n) => n % 2 === 0)
    .toArray()
); // 2, 4
```

## Custom implementation of iterator helpers proposal

In the last part of the article, we will implement the iterators helpers proposal from scratch. This helps us better understand the details behind this proposal. There are two aspects of the implementation, JavaScript and TypeScript aspect.

First, let's recap the main interfaces representing **iterable** object defined in `node_modules/typescript/lib/lib.es2015.iterable.d.ts` file. The code below is simplified comparing the original one leaving only elements important in our context.

```typescript
interface IteratorResult<T> {
  done: boolean;
  value: T;
}
interface Iterator<T> {
  next(): IteratorResult<T>;
}
interface Iterable<T> {
  [Symbol.iterator](): Iterator<T>;
}

interface IterableIterator<T> extends Iterator<T> {
  [Symbol.iterator](): IterableIterator<T>;
}
interface Array<T> {
  [Symbol.iterator](): IterableIterator<T>;
  entries(): IterableIterator<[number, T]>;
  keys(): IterableIterator<number>;
  values(): IterableIterator<T>;
}
```

Here we are using TS interfaces for describing the API of JS objects. I don't want to go too much into detail about how types work in JS but I would like to point out one interesting thing. Interface `Array` describes a built-in type representing an array of items so we can write in JS `new Array(1, 2, 3)`. But there is no such thing as `Iterable` or `Iterator` in JS world. It's only the description of API, there is no function or variable named that way that it could be used. The new proposal introduces the `Iterator` type so we can use it in our JS code. For this article, I will call this type Iterator\_` to avoid naming conflict with an existing TS interface. This is the first part of the implementation:

```typescript
export type Iter<T> = Iterable<T> | Iterator<T>;

function isIterable<T>(iter: Iter<T>): iter is Iterable<T> {
  return (iter as any)[Symbol.iterator] !== undefined;
}

export class Iterator_<T> extends IteratorHelpers<T> {
  private constructor(private iterator: Iterator<T>) {
    super();
  }

  next() {
    return this.iterator.next();
  }

  static from<TR>(iter: Iter<TR>) {
    return new Iterator_<TR>(isIterable(iter) ? iter[Symbol.iterator]() : iter);
  }
}
```

Following to the proposal description, `Iterator` represents a type that can be instantiated using the constructor or static method like`Iterator.from("abcd")`. This object implements the `Iterator` interface so we have to provide an implementation of the `next` method. The base class `IteratorHelpers` provides the implementation of all crucial methods like `map`, `filter`, `reduce`, and so on. I will explain later why the implementation is split into two separate classes. Let's try to implement a simple `map` method.

```typescript
abstract class IteratorHelpers<T> implements Iterator<T> {
  abstract next(): IteratorResult<T, any>;

  map<R>(f: (item: T, index: number) => R) {
    let index = 0;
    return Iterator_.from<R>({
      next: () => {
        const result = this.next();
        return result.done
          ? { done: true }
          : { done: false, value: f(result.value, index++) };
      },
    } as Iterator<R, R>);
  }
}
```

`IteratorHelpers` is an abstract class with one abstract method `next` required by the `Iterator` interface. The `map` method
returns a new instance of `Iterator`, a wrapper around JS object with the `next` method. The implementation is quite simple, every time someone calls `next` on the returned object, also `next` will be called on the object we are currently in, and then mapping function `f` is called.

We can use generator functions instead of direct implementation of `Iterator` interface. Let's look at the implementation of
`map` and `filter` methods.

```typescript
abstract class IteratorHelpers<T> implements Iterator<T> {
  // ...

  protected getIterable(): Iterable<T> {
    return { [Symbol.iterator]: () => this };
  }

  map<R>(f: (item: T, index: number) => R) {
    return Iterator_.from(go(this.getIterable()));

    function* go(items: Iterable<T>) {
      let index = 0;
      for (const item of items) {
        yield f(item, index++);
      }
    }
  }

  filter(f: (item: T, index: number) => boolean) {
    return Iterator_.from(go(this.getIterable()));

    function* go(items: Iterable<T>) {
      let index = 0;
      for (const item of items) {
        if (f(item, index++)) {
          yield item;
        }
      }
    }
  }
}
```

Inner functions are implemented as generators taking an `Iterable` interface. Helper method `this.getIterable()` wraps `Iterator` into `Iterable`. Because operators coming from the powerseq library take arguments of `Iterable` interface, we can use them to implement all methods described in the iter iterator helpers proposal.

```typescript
import {
  every,
  find,
  flatmap,
  foreach,
  reduce,
  skip,
  some,
  take,
} from "powerseq";

abstract class IteratorHelpers<T> implements Iterator<T> {
  // ...
  toArray(): T[] {
    return [...this.getIterable()];
  }

  drop(limit: number) {
    return Iterator_.from(skip(this.getIterable(), limit));
  }

  every(f: (item: T, index: number) => boolean) {
    return every(this.getIterable(), f);
  }

  find(f: (item: T, index: number) => boolean) {
    return find(this.getIterable(), f);
  }

  foreach(f: (item: T, index: number) => void) {
    foreach(this.getIterable(), f);
  }

  some(f: (item: T, index: number) => boolean) {
    return some(this.getIterable(), f);
  }

  take(limit: number) {
    return Iterator_.from(take(this.getIterable(), limit));
  }

  reduce<A>(f: (total: A, value: T) => A, initialValue: A) {
    return reduce(this.getIterable(), f, initialValue); // +index
  }

  flatMap<R>(f: (value: T, index: number) => Iterable<R> | Iterator<R>) {
    return Iterator_.from(
      flatmap(this.getIterable(), (item, index) => {
        const iter = f(item, index);
        return isIterable(iter) ? iter : { [Symbol.iterator]: () => iter };
      })
    );
  }
}
```

Now it's time to explain why the implementation is split into two separate classes `Iterable_` and `IteratorHelpers`. We can think about the `Iterator` type as a wrapper around `Iterable` type represented JS object with a fluent programming interface `Iterator.from("abcd").filter(...).map(...).toArray()` where each operator returns `Iterator` type.

But there is a second way of looking at the `Iterator` type. Even today when the proposal is not part of the JS standard yet, there are many places in JS API where objects implement the `Iterator` interface. We can find those places tracking all usages of the `IterableIterator<T>` interface (type implementing both `Iterable` and `Iterator` interfaces). Collections data types like `Array`, `Map`, `Set` contain methods `entries`, `keys`, `values` returning `IterableIterator<T>`. Another place is a generator function. Each generator function returns an object implementing `IterableIterator<T>` and that's why previously we could have written the following code `naturals().map(value => value * value)`. In all those cases we don't have to create an instance of the `Iterable` class manually.

We can think of the `IteratorHelpers<T>` type as a set of concrete functions like `map`, `filter`, .. that depends on one (abstract) `next` function. This set of functions can be copied to any existing JS object as long as that object provides the `next` function. In dynamic languages, such a programming trick is called [mixin](https://en.wikipedia.org/wiki/Mixin). In all places where our code is not responsible for creating `Iterator` objects, we have to extend those objects at runtime injecting a set of new functions.

```typescript
/** Object.assign copies only enumerable properties, prototype object has non-enumerable properties */
function assign(target: any, source: any) {
  for (const property of Object.getOwnPropertyNames(source)) {
    if (property !== "constructor") {
      target[property] = source[property];
    }
  }
}

const iteratorHelpersProto = Object.getPrototypeOf(
  Object.getPrototypeOf(Iterator_.from([]))
);

assign(Object.getPrototypeOf([1, 2, 3].values()), iteratorHelpersProto); // Array
assign(
  Object.getPrototypeOf(new Set([1, 2, 3]).values()),
  iteratorHelpersProto
); // Set
assign(Object.getPrototypeOf(new Map([[1, 1]]).values()), iteratorHelpersProto); // Map
assign(
  Object.getPrototypeOf(Object.getPrototypeOf((function* () {})())),
  iteratorHelpersProto
); // generator function
```

This code looks cryptic at first glance. I don't want to go too much into detail about how the prototype mechanism works in JS, but I will try to explain some basics. Every time we execute `[1, 2, 3].values()` or `['a','b'].values()`a new instance of `Iterator` is created. In the future, when the iterator helpers proposal will be part of the JS standard, all browsers will support it natively. It means that functions like `map` and `filter` will be available directly inside `Iterator` object. For now, we have to use a little bit of magic. Instead of extending each new instance of `Iterator` object, we can use the prototype mechanism to extend only one instance per "type of iterator". Calling `Object.getPrototypeOf([1, 2, 3].values())` returns a regular JS object called prototype. The concrete instance of an array doesn't matter, the prototype is exactly the same object in memory. The prototype object is like a base class in the OOP world. If we add something to it, all distinct instances will have it. The helper function called `assign` copies all properties (methods in this case) one by one to prototypes of different "types of iterators".

In the end, we have to somehow extend the definition of existing types on the TypeScript level.

```typescript
declare global {
  interface IterableIterator<T> extends Omit<IteratorHelpers<T>, "next"> {}
  interface Generator<T = unknown, TReturn = any, TNext = unknown>
    extends IteratorHelpers<T> {}
}
```

Believe it or not, the following code works correctly on the JavaScript and TypeScript levels.

```typescript
Iterator_.from({
  next() {
    return { done: false, value: 666 };
  },
})
  .take(3)
  .toArray(); // [666, 666, 666]

function* range(start: number, count: number) {
  const end = start + count;
  for (let i = start; i < end; i++) {
    yield i;
  }
}
range(0, Number.MAX_VALUE)
  .drop(100)
  .filter((x) => x % 2 === 0)
  .take(5)
  .toArray();
// [ 100, 102, 104, 106, 108 ]
```

## Summary

I hope this article helped you better understand the iterator helpers proposal and all aspects around **iterable** objects in general. On the one hand, it's good to have lazy sequence operators baked-in JS standard. But on the other hand, the current design doesn't convince me. It has some serious drawbacks, it's mainly about using the `Iterator` interface instead of `Iterable`, and instance methods instead of regular functions. But at the same time, I am aware of the importance of consistent JS API design which is mostly OOP. Maybe it's too late to fix the proposal or maybe the designers of JS standard will find some simple, elegant, and extensible solution.

[Source code](https://github.com/marcinnajder/misc/tree/master/2024_01_29_iterator_helpers_in_javascript_standard)
