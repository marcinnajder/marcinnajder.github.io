---
layout: post
title: Iterable vs Observable [TS]
date: 2024-08-29
tags:
  - ts
series: iter-vs-obs
---
## Introduction


Around 2009 [Erik Meijer](https://en.wikipedia.org/wiki/Erik_Meijer_(computer_scientist)) invented [Rx](https://reactivex.io/). From the beginning, he wanted to explain how closely the two design patterns are related - Iterator and Observable. He even once said that he felt cheated by the [Gang of Four](https://wiki.c2.com/?GangOfFour) team; no one told him that those two design patterns are almost identical. In this article, we will discuss the process of transforming one design pattern into another, the same way the author did. We use the flexibility of the TypeScript type system to express the [duality](https://csl.stanford.edu/~christos/pldi2010.fit/meijer.duality.pdf) aspect of the transformation. Finally, we will implement a few standard operators like `map` and `filter` for both interfaces.

## From Iterable to Observable


The simple representation of the two interfaces in TypeScript, `Iterable` and `Observable`, could look as follows:

```typescript
interface Iterable<T> {
  iterator(): Iterator<T>;
}
interface Iterator<T> {
  next(): { done: boolean; value: T };
}

interface Observable<T> {
  subscribe(observer: Observator<T>): () => void;
}
interface Observator<T> {
  onNext(value: T): void;
  onError(error: Error): void;
  onCompleted(): void;
}
```

Both interfaces, `Iterable<T>` and `Iterator<T>`, contain only one method. The `Iterable<T>` interface works like a factory method responsible for creating new objects implementing `Iterator<T>`. Every call of the `next` function moves the current position, returning the JavaScript object `{ done: false; value: ... }`, or stops the iteration, returning `{ done: true }`. An interface with one method can be replaced with a type alias to the function description. So, the whole definition of those interfaces can be shortened to something like this.

```ts
type Iter<T> = () => () => null | T;
```

It is a function returning a function. Returning `null` means the end of the iteration process, and this simplification is used only for a moment. Later, different types of results will be expressed more accurately.

Erik's "aha moment" was the realization that an `Observable` is just a "reversed" `Iterable`. Let's say we have a function of a particular signature, for example, `type ToString = (n: number) => string;`. The reversed signature would look as follows `type ReversedToString = (n: string) => number;`. The types of arguments and results switched places. We can even extend this idea to objects, because objects are nothing more than sets of functions. 

```ts
type Reverse<F> = F extends (...args: infer Args) => infer Result
  ? (args: Reverse<Result>) => Reverse<Args>
  : F;
```

The TypeScript language is powerful enough to express such relations between types. Generic type`Reverse<F>` transforms some function type `F` into its reversed form. The alternative version of the previously defined function could look like this `type ReversedToString2 = Reverse<ToString>`. It even works for functions taking more than one argument, for example, function `Reverse<(a: number, b:number) => number>` is transformed into `(args: number) => [a: number, b: number]`.

Let's analyze what the following type definition could mean.

```ts
type Obs<T> = Reverse<Iter<T>>;
```

If we hover over the TypeScript type definition in an editor like VS Code, it shows something like this `type Obs<T> = (args: (args: Reverse<T> | null) => []) => []`. `Obs<T>` is a function taking one argument, a function itself. That argument function takes `null` or `Reverse<T>`, and `Reverse<T>` is `T` for non-function arguments. Can we already spot the similarities between `Observable<T>` and `Obs<T>` types? 

## Discriminated unions

Now, we will discuss the concept of discriminated unions, which will help us describe the types of operations more accurately. Most functional languages support discriminated unions (aka choice or sum types). Let's take a look at an example written in F#.

```fsharp
type Res<'a> =
    | Value of 'a
    | Completed
    | Error of Exception

let results = [ 5; 10; 15 ] |> Seq.map Value // seq<Res<int>>
let firstResult = results |> Seq.head // Res<int>
let text =
    match firstResult with
    | Error err -> "error: " + err.Message
    | Completed -> "completed"
    | Value value -> "value: " + value.ToString()
```

Generic type `Res<'a>`  represents some value in one of three possible states:
- Value - wraps any value of generic type `'a`
- Completed - means some operation is completed, and there is no additional data
- Error - means some operation has failed with additional data holding details information as an `Exception` type 

Discriminated unions have some special features:
- They work like an enumeration type supported in many other programming languages, but any case can hold some additional data
- There is a factory function for each of the cases, like calling `Value 1` returns `Res<int>` 
- Programming language provides a related feature called pattern matching, with extended support for discriminated unions. It's like if-then-else or switch/case expressions, but the compiler checks whether all cases are handled in code

The TypeScript language provides some building blocks that allow for the expression of discriminated unions. 

```ts
type Res<T> =
  | { type: "error"; err: Error }
  | { type: "completed" }
  | { type: "value"; value: T };
```

This is only a type definition. What about the other aspects, like constructor functions or support for pattern matching? I have created a small library [powerfp](https://github.com/marcinnajder/powerfp), to help address those issues. There are two possible scenarios for handling constructor functions. The first is when the code for the constructor functions is generated using a script provided by the library. The second one is to write the code for them manually, and the definition of the discriminated union type will be inferred automatically. Let's use the second approach.

```ts
const value = <T>(value: T) => ({ type: "value", value } as const);
const completed = { type: "completed" } as const;
const error = (error: Error) => ({ type: "error", err: error } as const);

type Res<T> = SumType<typeof value<T> | typeof completed | typeof error>;

const results: Iterable<Res<number>> = pipe([5, 10, 15], map(value));
const firstResult = pipe(results, find())!; // Res<number>
const text = matchUnion(firstResult, {
  error: ({ err }) => "blad: " + err.message,
  completed: () => "koniec",
  value: ({ value }) => "wartosc: " + value,
});
```

Thanks to `SumType` type imported from the powerfp library, a new `Res<T>` type was inferred as follows:

```ts
type Res<T> =
  | {
      readonly type: "completed";
    }
  | {
      readonly type: "value";
      readonly value: T;
    }
  | {
      readonly type: "error";
      readonly err: Error;
    };
```

Other functions like `pipe` or `matchUnion` were also imported from powerfp. The `pipe` is the equivalent of the pipe operator `|>` known from languages like F# or Elm. Instead of calling `func2(func1(val))`,  we can use helper function `pipe(val, func1, func2)`. With pipe operator, it would be `val |> func1 |> func2`. The `matchUnion` simulates a feature called pattern matching. This function takes an instance of a discriminated union and a JavaScript object representing some logic that must be executed for each case. The names of the object's properties are required, and must correspond to the values of the `type` property of `Res<T>`. The values of properties are functions with signatures taking appropriate unions and returning the same type. The TypeScript compiler returns errors when those conditions are not met. 

## Iterable operators

`Inter<T>` interface implements the iterator pattern, function `() => null | T` returns the next element in the sequence. `null` value is returned when there is no element left. There is another possible result not expressed in the function signature, which is an exception. Let's change the definition of our interfaces by introducing `Res<T>` type, which describes all three possible states. 

```ts
type Iter<T> = () => () => Res<T>;

type Dis = () => void;
type Obs<T> = (sub: (res: Res<T>) => void) => Dis;
```

We have added a new interface `Dis`, for disposing of resources. It was added only to the `Obs<T>` to simplify the code. In reality, stopping the iterator and cleaning up used resources should also be supported in case of the `Iter<T>` interface.

Now, we will implement a few functions using the `Iter<T>` type. Function `rangeIter` takes two numbers,  `start` and `count`, and returns a sequence of the `count` values starting from the `start`. Unlike a typical collection stored in memory,  the `Iter<T>` represents a lazy sequence. The consumer code pulls as many values as it needs. The JavaScript language has built-in support for the iterator pattern since ES2015,  and standard types like string, Array, and Maps implement this API. `fromSeqToIter` and `fromIterToSeq` functions convert values between the Iter<T> and built-in types in both directions. 

```ts
function rangeIter(start: number, count: number): Iter<number> {
  return () => {
    const max = start + count;
    let current = start;
    return () => (current < max ? value(current++) : completed);
  };
}

function fromSeqToIter<T>(items: Iterable<T>): Iter<T> {
  return () => {
    const iterator = items[Symbol.iterator]();
    return next;

    function next(): Res<T> {
      try {
        const res = iterator.next();
        return res.done ? completed : value(res.value);
      } catch (err: any) {
        return error(err);
      }
    }
  };
}

function* fromIterToSeq<T>(items: Iter<T>) {
  const iterator = items();
  let result = iterator();

  while (result.type === "value") {
    yield result.value;
    result = iterator();
  }

  if (result.type === "error") {
    throw result.err;
  }
}
```

The following function `filterIter` executes filtering logic based on the provided predicate function `f`. The `mapIter` function converts element by element, calling the mapping function `f`. We can create the whole query using the `pipe` function. 

```ts
function filterIter<T>(items: Iter<T>, f: (item: T) => boolean): Iter<T> {
  return () => {
    const iterator = items();
    return next;

    function next(): Res<T> {
      try {
        return matchUnion(iterator(), {
          error: (res) => res,
          completed: (res) => res,
          value: (res) => (f(res.value) ? res : next()),
        });
      } catch (err: any) {
        return error(err);
      }
    }
  };
}

function mapIter<T, R>(items: Iter<T>, f: (item: T) => R): Iter<R> {
  return () => {
    const iterator = items();
    return next;

    function next(): Res<R> {
      try {
        const x: Res<R> = matchUnion(iterator(), {
          error: (res) => res,
          completed: (res) => res,
          value: (res) => value(f(res.value)),
        });
        return x;
      } catch (err: any) {
        return error(err);
      }
    }
  };
}

const ys = pipe(
  [1, 2, 3, 4, 5],
  fromSeqToIter,
  (xs) => filterIter(xs, (x) => x % 2 === 0),
  (xs) => mapIter(xs, (x) => x.toString()),
  fromIterToSeq,
  toarray()
);
assert.deepStrictEqual(ys, ["2", "4"]);
```

## Observable operators

Let's implement some functions for `Obs<T>` instead of `Iter<T>`. 

```ts
function intervalObs(ms: number): Obs<number> {
  return (sub) => {
    let index = 0;
    const id = setInterval(() => {
      sub(value(index++));
    }, ms);
    return () => {
      sub(completed);
      clearInterval(id);
    };
  };
}

function fromSeqToObs<T>(items: Iterable<T>): Obs<T> {
  return (sub) => {
    try {
      for (const item of items) {
        sub(value(item));
      }
      sub(completed);
    } catch (err: any) {
      sub(err);
    } finally {
      return () => {};
    }
  };
}

function fromObsToIter<T>(obs: Obs<T>): Promise<T[]> {
  return new Promise(function (resolve, reject) {
    let buffer: T[] = [];
    const _ = obs((res) => {
      matchUnion(res, {
        error: (res) => reject(res.err),
        completed: () => resolve(buffer),
        value: (res) => buffer.push(res.value),
      });
    });
  });
}
```

The `intervalObs` function is similar to the `rangeIter` function; it generates numbers `0, 1, 2, 3, ...` separated by a time interval of specified `ms` milliseconds. Iterable represents synchronous communication, often called "pull" because the consumer decides when the producer generates the next value. Observable represents asynchronous communication called "push" because the producer is in charge of generating values. Conversion from a pull data source into a push data source is straightforward; it's implemented as the `fromSeqToObs` function. The opposite direction is not so obvious. We could block the execution thread or use a type like `Promise<T>` representing one asynchronous result instead of many asynchronous results like `Obs<T>`. JavaScript is a single-thread environment, so the `fromObsToIter` function realizes the second approach. We subscribe to the observable source and store all generated values inside a buffer; the promise object is resolved once the source is closed. Of course, the implementations of all functions above are simplified; many problematic corner cases exist, like buffer overflow. The code is mainly for educational purposes. Let's implement collection operators like `filter`, `map` and `take`. 

```ts
function filterObs<T>(obs: Obs<T>, f: (item: T) => boolean): Obs<T> {
  return (sub) => {
    const unsub = obs((res) => {
      matchUnion(res, {
        error: sub,
        completed: sub,
        value: (r) => {
          if (f(r.value)) {
            sub(r);
          }
        },
      });
    });

    function unsubscribe() {
      sub(completed);
      unsub();
    }
    return unsubscribe;
  };
}

function mapObs<T, R>(obs: Obs<T>, f: (item: T) => R): Obs<R> {
  return (sub) => {
    const unsub = obs((res) => {
      matchUnion(res, {
        error: sub,
        completed: sub,
        value: (r) => {
          sub(value(f(r.value)));
        },
      });
    });

    function unsubscribe() {
      sub(completed);
      unsub();
    }
    return unsubscribe;
  };
}

function takeObs<T>(obs: Obs<T>, count: number): Obs<T> {
  if (count === 0) {
    return (sub) => {
      sub(completed);
      return () => {};
    };
  }
  
  return (sub) => {
    let index = 0;
    const unsub = obs((res) => {
      matchUnion(res, {
        error: sub,
        completed: sub,
        value: (r) => {
          sub(r);
          if (++index === count) {
            unsubscribe();
          }
        },
      });
    });

    function unsubscribe() {
      sub(completed);
      unsub();
    }
    return unsubscribe;
  };
}

pipe(
  intervalObs(10),
  (xs) => takeObs(xs, 3),
  (xs) => filterObs(xs, (x) => x % 2 === 0),
  (xs) => mapObs(xs, (x) => x.toString()),
  fromObsToIter
).then((items) => assert.deepEqual(items, ["0", "2"]));
```

## Summary

This article aimed to show how the Observables API was born directly from the Iterables API. First, we reduced the type definition based on object interfaces into simple functions. Then, thanks to TypeScript's expressiveness, we applied a simple transformation at the type level swapping results and parameters. Finally, we implemented typical collection operators like `map` and `filter` for both APIs, as well as conversions between them. 

The idea was to get the essence of both interfaces to show how they work and how similar they are. I wasn't the first person to do this. Many years ago, Bart de Smet, a member of the Rx team, wrote a series of excellent articles called "MinLINQ - The Essence of LINQ" about the same topic. Unfortunately, they are probably [no longer available](https://www.developerfusion.com/media/92222/bart-de-smet-minlinq-the-essence-of-linq/).