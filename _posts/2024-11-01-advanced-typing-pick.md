---
layout: post
title: Advanced typing - pick [TS]
date: 2024-11-01
tags:
  - ts
series: advanced-typing
---

{%- include /_posts/series-toc.md series-name="advanced-typing" -%}

## Pick

Let's say we have a simple TS type describing an object, like `Person`.

```typescript
type Person = { id: number; name: string; age: number; }
```

It is typical to map this object or an array of objects into something similar but slightly different. For instance, we would like to choose only some properties, or change the name or types of properties.

```typescript
const people: Person[] = [
	{ id: 1, name: "marcin", age: 40 },
	{ id: 2, name: "gosia", age: 30 },
];

const names1 = people.map(p => ({ id: p.id, name: p.name })); 
const names2 = people.map(({ id, name }) => ({ id, name })); 
// -> // { id: number; name: string; }[]
```

 The code above annoys me every time I write it. It's because mapping logic cannot be written without repeating the property names twice, like the `name` property in each lambda above was repeated. At least, I don't know how to do this. In C#, for example, the following code is correct `p => new { p.Id, p.Name }`. It uses an anonymous type where the property names are inferred from the context. We could specify the names explicitly if we wanted, like `p => new { p.Id, PersonName = p.Name }`.
 
 I had been thinking about this problem for a while, and finally, I found a simple solution: a helper function called `pick`.

```typescript
function pick<T, P extends keyof T>(obj: T, ...props: P[]): Pick<T, P> {
    var result = {} as any;
    for (const p of props) {
        const value = obj[p];
        if (typeof value !== "undefined") {
            result[p] = value;
        }
    }
    return result;
}

const names3 = people.map(p => pick(p, "id", "name"));
```

TypeScript is a compelling language. Our `pick` function is fully statically typed. The name of `name3` variable was inferred as `Pick<Person, "id" | "name">[]`, which is equivalent to  `{id: number; name: string;}[]`. If we misspell the name of the properties, `"id"` or `"name"`, we will get an error from the TS compiler. The editor also suggests those names via Intellisense. The JS code is simple, but most of the TS job was done by the built-in `Pick<T, P>` defined in the following way:

```typescript
type Pick<T, K extends keyof T> = {
    [P in K]: T[P];
};
```

There are situations when we would like to change the names of some properties. Our `Person` object has only three properties, but let's imagine we have an object with 20 properties. We want to take 10 of them with the same names and three others with names changed. Still, our `pick` function would allow us to avoid duplications for the first 10 properties. I found one solution that does not look too bad.

```typescript
// const names4: { personAge: number; id: number; name: string; }[]
const names4 = people.map(p => ({ ...pick(p, "id", "name"), personAge: p.age, }));
```

I hear what you are saying. My colleagues told me the same: "It's too much. I would rather use standard JS features than a custom `pick` function in such scenarios.". So I started thinking again :), and I implemented a new `pickm` function that offers richer mapping functionality. 

```typescript
// const names4: { personAge: number; id: number; name: string; }[]
let names4 = people.map(p => ({ ...pick(p, "id", "name"), personAge: p.age, }));

// const names5: PickMC<Person, "id" | "name" | ["personAge", "age"]>[]
let names5 = people.map(p => pickm(p, "id", "name", ["personAge", "age"]));

// those are exactly the same types :)
names4 = names5;
names5 = names4;
```

After reading this article, please remember that almost anything you want to express in TS, can be accomplished. A new function `pickm` takes any number of strings representing property names or a mapping tuple/pair, like `["personAge", "age"]`. Still, as before, Intellisense works, and everything is type-safe. The JS implementation of the function does not look much more complicated than the previous `pick` function.

```typescript
export function pickm<T, const Es extends Entry<T>[]>(obj: T, ...entries: Es): PickM<T, Duplicate<Es>> {
  const result = {} as any;
  for (const entry of entries) {
    if (Array.isArray(entry)) {
      const [newKey, key] = entry;
      const value = obj[key];
      if (typeof value !== "undefined") {
        result[newKey] = value;
      }
    } else {
      const value = obj[entry];
      if (typeof value !== "undefined") {
        result[entry] = value;
      }
    }
  }
  return result;
}
```

The TS side of the solution is much harder to understand.

```typescript
type EntryPair<T> = [string, keyof T];

type Entry<T> = keyof T | EntryPair<T>;

type Duplicate<T> =
  T extends [infer Item, ...infer Rest]
    ? (Item extends [infer Item1, infer Item2] ? [[Item1, Item2], ...Duplicate<Rest>] : [[Item, Item], ...Duplicate<Rest>])
    : [];

type PickM<T, Es extends EntryPair<T>[] | unknown> =
  Es extends [[infer NewKey, infer Key], ...infer Rest]
    ? (Key extends keyof T ? { [key in (string & NewKey)]: T[Key] } & PickM<T, Rest> : unknown) : unknown;

```

Let's explain what is happening. Types like `EntryPair<T>` and `Entry<T>` are primarily used as constraints of a generic parameter; they appear after `... extends Entry<T>[]`. 

The method signature looks like this `pickm<T, const Es extends Entry<T>[]>(obj: T, ...entries: Es)`. For our previous call `pickm(p, "id", "name", ["personAge", "age"])`, a generic type argument `Es` will be inferred as `["id", "name", ["personAge", "age"]]`. The helper type `Duplicate<Es>` will transform actual `Es` into `[["id", "id"], ["name", "name"], ["personAge", "age"]]`. We introduced the `Duplicate` type because the final type `PickM<T, Es extends EntryPair<T>[] | unknown>` expects `Es` to be an array of `EntryPair<T>` (which is always a pair). `Duplicate` type ensures pairs.

The purpose of `PickM<T, Es>` should be similar to built-in`Pick<T, K>`; it creates a new object type from the input object type `T`  that contains only specified (and mapped) properties.

Recursion and a ternary operator (`... ? ...: ... )` are two fundamental mechanisms in TypeScript that allow us to build more complicated types. They are used by `Duplicate<T>`and `PickM<T, ES>`.

The simplified definition of  `Duplicate<T>` looks like this.

```typescript
type Duplicate<T> = T extends [infer Item, ...infer Rest] ? ... : [];`
```

If the `T` is an empty array,  `[]` will be returned. Otherwise, `...` part will be executed and we can be sure that a new variable `Item` is set to the first item from the array. `...` part is defined like this.

```typescript
Item extends [infer Item1, infer Item2]
	? [[Item1, Item2], ...Duplicate<Rest>] 
	: [[Item, Item], ...Duplicate<Rest>]
```
 
 It is a simple check whether the `Item` is an array. If it is, it will be returned as it is. Otherwise, it will be duplicated as `[Item, Item]`. In both cases, recursive `Duplicate<Rest>` calls are made, too.

Unfortunately, the type `PickM` would be more complicated to explain. Thus, I don't even try to go into details. The simplified type definition looks like this.

```typescript
type PickM<T, Es> = ... ? { [key in NewKey : T[Key] } & PickM<T, Rest> : unknown`
```

I hope you can see the recursive call of `PickM`. The pair `[NewKey, Key]` describes the mapping of property names. The final result is represented as an intersection type of `{ [key in NewKey : T[Key] }` and the recursive call `PickM<T, Rest>` for mapping pairs left.

Maybe we should stop here, because I have yet another idea ... ;) What about a scenario like `pickm(p, "id", "name", ["personAge", "age"], ["personCounty", "address.country"])`? Check out the next article about the `ensureProperties` function.