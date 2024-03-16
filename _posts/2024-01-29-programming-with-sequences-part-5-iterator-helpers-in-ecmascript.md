---
layout: post
title: Programming with sequences part 4 - iterator helpers in ECMAScript
date: 2024-01-29
tags: js powerseq
published: false
series: programming-with-sequences
---

{%- include /_posts/series-toc.md series-name="programming-with-sequences" -%}

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

## `Iterable` and `Iterator` objects

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

## Generators create iterable iterators

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

Generator function `naturals` returns sequence of natural numbers starting from `0`, then `map` method is called. Where does `map` come from and what we can tell about its signature ? As we have just discussed, generator functions return objects that are **iterable** and **iterator** simultaneously. Just after looking at the code above, we are not able to guess whether `map` method is a member of `Iterable` or `Iterator` interface. One thing we can definitely say about the result of `map` method is that it implements `Iterator` interface. Maybe the result also implements `Itereable`, but the code says about about it.

The first difference between operators from powerseq library (and many other similar solutions from different technologies) and JavaScript iterator helpers proposal. In the first case the signature of operator looks like this `(source: Iterable<...>, ...): Iterable<...>`, it takes and returns `Iterable`. In the second case it looks like this `(source: Iterator<...>, ...): Iterator<...>`, it takes and returns `Iterator`. This proposal describes many useful operators, some of them return lazy sequences of values (`map, filter, take, drop, flatMap` ), other return scalar values immediately (`reduce, toArray, forEach, some, every, find`). Unfortunately the API of `Iterator` interface allows us to iterate sequence of value only once.

The second difference is that operators in powerseq are just regular functions. Library provides many builtin operators but final user can always decide to write its own operator or reimplement existing one. Function `pipe` combines any operators the same way. In case of iterator helpers proposal operators are instance methods of `Iterator` interface so there is no elegant way of extending the list of standard operators. We can change dynamically prototype JS object in runtime (we will be talking about it further), but this is more like a unrecommended hack. I am aware know this problem comes from OOP (Object Oriented Programming) design of this feature but for me personally FP (Functional Programming) approach (separating data and behavior) would be better here. Especially if we think about another [pipeline operator proposal](https://github.com/tc39/proposal-pipeline-operator) that would allow us to write something like this `[1, 2, 3] |> filter(x => x % 2 === 0) |> map(x => x.toString())`.

There is also one interesting aspect of using `Iterator` instead of `Iterable`. When `Iterable` interface was introduced in ES2015 many other useful features were added too. TypeScript language provides special declaration files describing JavaScript API. If we look at `node_modules/typescript/lib/lib.es2015.iterable.d.ts` file we can find all places where `Iterable` interface is used:

- `String` type, collections (`Array`, `Map`, `Set`, `WeakMap`, `WeakSet`) and `arguments` object implement `Iterable`interface
- `Promise.all` and `Promise.race` take `Iterable` interface as arguments
- `for/of` loop iterates over `Iterable` interface
- spread operator works with with `Iterable` interface, so we can write `["a", ..."bcd", "e"]`

Because powerseq library uses `Iterable` interface everything seems very easy and natural.

```typescript
for (const digit of filter("ab5c10d15ef", (c) => c >= "0" && c <= "9")) {
  console.log(digit); // 5, 1, 0, 1, 5
}
```

The same code using new iterator helper proposal can be written like this.

```typescript
for (const digit of Iterator.from("ab5c10d15ef")
  .filter((c) => c >= "0" && c <= "9")
  .toArray()) {
  console.log(digit); // 5, 1, 0, 1, 5
}
```

First we have to wrap `String` type into `Iterator` interface, then we can call any of proposed methods. If final result represent a sequence of values like in the code above, the returned type is `Iterator`. Maybe it will also implement `Iterable`iterface but so for we just don't know. We have to convert sequence to new `Array` object just to iterate over its items using `for/of` loop. The alternative option is to iterate over items using `forEach` method like this `... .forEach(digit => console.log(digit))`. Builtin collections like `Array`, `Maps`, `Set` contain methods `entries`, `keys`, `values` returning sequence of items represented object implementing both `Iterable` and `Iterator` interfaces. We will be able to write such a code

```typescript
console.log(
  [1, 2, 3, 4, 5]
    .values()
    .filter((n) => n % 2 === 0)
    .toArray()
); // 2, 4
```

## Custom implementation of Iterator helpers proposal

In the last part of article we will implement iterators helps proposal from scratch. This helps us better understand the details behind this proposal. There are two aspect of the implementation, JavaScript and TypeScript aspect.

First let's recap the main interfaces representing **iterable** object defined in `node_modules/typescript/lib/lib.es2015.iterable.d.ts` file. The code below is simplified compering the original one leaving only elements important in our context.

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

Here we are using TS interfaces for describing API of JS objects. I don't want to go too much into details how types work in JS but I would like to point out one interesting thing. Interface `Array` describes builtin type representing array of items so we can write in JS `new Array(1, 2, 3)`. But there is not such think like `Iterable` or `Iterator` in JS world. It's only the description of API, there is no function or variable named that way that we could use. New proposal introduces `Iterator` type that be used in JS code. For the purpose of this article I will call it `Iterator_` type to avoid naming conflict with an existing TS interface. This is the fist part of the implementation:

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

Following to the proposal description, `Iterator` represents a type that can be instantiated using constructor or static method `Iterator.from("abcd")`. This object implements `Iterator` interface so we have to provide implementation of `next` method. The base class `IteratorHelpers` provides the implementation of all crucial methods like `map`, `filter`, `reduce` and so on. I will explain later why the implementation is splitted into two separate classes. Let's try to implement simple `map` method.

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

`IteratorHelpers` is an abstract class with one abstract metod `next` required by `Iterator` interface. `map` method
returns a new instance of `Iterator` a wrapper around JS object with `next` method. The implementation is quite simple, every time someone calls `next` on the returned object, also `next` will be called on the object we are currently in and then mapping function `f` is called.

We can use generator functions instead of direct implementation of `Iterator` interface. Let's look at implementation of
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

Inner functions are implemented as generators taking `Iterable` interface. Helper method `this.getIterable()` wraps `Iterator` into `Iterable`. Operators coming from powerseq library take `Iterable` interface so we can use them to implement all methods described in iterator helpers proposal.

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

Now it's time to explain why the implementation is splitted into two separate classes `Iterable_` and `IteratorHelpers`. We can think about `Iterator` type as a wrapper around `Iterable` represented JS object with fluent interface like `Iterator.from("abcd").filter(...).map(...).toArray()` where each operator returns `Iterator` type.

But there is second way of looking at the `Iterator` type. Even today when the proposal is not part of the JS standard yet, there are many places in JS API where objects implement `Iterator` interface. We can find those places tracking the usage of `IterableIterator<T>` interface. Collections data types like `Array`, `Map`, `Set` contain methods `entries`, `keys`, `values` returning `IterableIterator<T>`. Another place is generator function. Each generator function returns object implementing `Iterator` and that's why previously we could have written the following code `naturals().map(value => value * value)`. In all those cases we don't have to create instance of `Iterable` class manually.

We can think of `IterableIterator<T>` type like a set of concrete functions like `map`, `filter`, .. that depends on one `next` function. This set of functions can be copied to any existing JS object as long as that object provides `next` function. Such a programming trick is called [mixin](https://en.wikipedia.org/wiki/Mixin) in dynamic languages. In all places where our code is not responsible for creating `Iterator` objects, we have to extend those objects at runtime injecting a set of new functions.

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

This code looks cryptic at first glance. I don't want go too much into great details how prototype mechanism works in JS, but I will try to explain some basics. Every time we execute `[1, 2, 3].values()` or `['a','b'].values()`a new instance of `Iterator` is created. In the future when the iterator helpers proposal will be part of JS standard, all browser will support it natively. It means that functions like `map`, `filter` will be available directly inside `Iterator` object. For now we have use some magic. Instead of extending each new instance of `Iterator` object, we can use prototyp mechanism to extend only one instance per "type of iterator". Calling `Object.getPrototypeOf([1, 2, 3].values())` returns a regular JS object called prototype. The concrete instance of array doesn't matter, the prototype is exactly the same object in memory. The prototype object is like a base class in OOP languages. If we add something to it, all distinct instances will have it. Helper function called `assign` copies all properties (methods in this case) one by one to prototypes of different "types of iterators".

At the end we have to somehow extend the definition of exiting types on the TypeScript level.

```typescript
declare global {
  interface IterableIterator<T> extends Omit<IteratorHelpers<T>, "next"> {}
  interface Generator<T = unknown, TReturn = any, TNext = unknown>
    extends IteratorHelpers<T> {}
}
```

Believe me or not but the following code works corretly on the JavaScript and TypeScript levels.

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

I hope this article helped you better understand iterator helpers proposal and all aspect around **iterable** objects in general. On the one hand, it's good to have lazy sequence operators in baked in JS standard. But on the other hand, the current design doesn't convince me. It has some serious drawbacks, it's mainly about using `Iterator` interface instead of `Iterable` and instance methods instead of regular functions. But the same time, I am aware of importance of consistent JS API design which is mostly OOP. Maybe it's just too late to fix it or maybe the designers of JS standard will find some elegant and extensible solution.

[Source code](https://github.com/marcinnajder/misc/tree/master/2024_01_29_iterator_helpers_in_javascript_standard)
