---
layout: post
title: Advanced typing - ensurePaths [TS]
date: 2025-05-30
tags:
  - ts
series: advanced-typing
---

{%- include /_posts/series-toc.md series-name="advanced-typing" -%}

## Introduction 

The TypeScript compiler has a special option called [strictNullChecks](https://www.typescriptlang.org/tsconfig/#strictNullChecks), which works the following way:

> When `strictNullChecks` is `false`, `null` and `undefined` are effectively ignored by the language. This can lead to unexpected errors at runtime.
> 
> When `strictNullChecks` is `true`, `null` and `undefined` have their own distinct types and you’ll get a type error if you try to use them where a concrete value is expected.

Let's take a look at a simple example. 

```typescript
type Person = {
    name?: string;
    age?: number;
    address?: {
        city?: string;
        country?: boolean,
        no?: number
    };
}

function formatPersonSummary(p: Person) {
    if (p.address.country === "Poland") {
        return `${p.name.trim()} lives in Poland`;
    }
    return `${p.name.trim()} lives in ${p.address.country.toUpperCase()}, city ${p.address.city}`;
}
```

When the `strictNullChecks` option is disabled, the code compiles without any errors. After enabling the options, the compiler generates six errors, such as 
- `'p.address' is possibly 'undefined'.`
- `'p.name' is possibly 'undefined'.`
- `'p.address.country' is possibly 'undefined'.`

All `Person` properties are optional, and the function's author assumed that all properties would always be set. Moreover, this code may work fine because this function was never called without the required data. Potential problems are hidden until the `strictNullChecks` option is enabled.

I had a similar problem in one of the projects I have been working on. After many years of work on the project, we decided to turn on the `strictNullChecks` option. We got a few thousand TS errors. In general, there are two possible solutions. The type definition of the data may be wrong and has to be changed. For instance, all `Person` properties were made optional by mistake, and in reality, they are required. The second solution is introducing additional checks everywhere the data model is used. Current implementation of `formatPersonSummary` assumes that all properties are set, so throwing an exception could be a proper strategy.

```typescript
function formatPersonSummary(p: Person) {
    if(!p.name){
        throw new Error(`name is required`);
    }
    if(!p.address || !p.address.country || !p.address.city){
        throw new Error(`full address is required`);
    }

    if (p.address.country === "Poland") {
        return `${p.name.trim()} lives in Poland`;
    }
    return `${p.name.trim()} lives in ${p.address.country.toUpperCase()}, city ${p.address.city}`;
}
```

There are other options, too. An empty string or a constant text message could be returned depending on the missing data. 

```typescript
function formatPersonSummary(p: Person) {
    if(!p.name){
        return "Unknown person";
    }
    if (p.address?.country === "Poland") {
        return `${p.name.trim()} lives in Poland`;
    }

    return `${p.name.trim()} lives in ${p.address?.country?.toUpperCase() ?? "another country like Poland"}, city ${p.address?.city ?? "unknown"}`;
}
```

TypeScript uses a feature called control flow analysis. It tracks each condition in the code and changes the types on the fly. The property `p.name` can be potentially `undefined`, but after the check `if(!p.name){ ... }`, we can be sure that it is not `undefined`, and expressions like `p.name.trim()` don't cause any errors. 

I started thinking about those issues. There are many ways of handling a missing value. We can throw an exception or use some default value. We can use operators like `.?`, `.!`, or`??`. What I don't like about those operators is that the same property often has to be checked several times inside the function. Checking the existence of one or many properties, frequently, the whole long property paths must be checked, then throwing an exception with a helpful message. It all seems repetitive. I had an idea, we could provide a couple of functions that would simplify it and reduce the amount of code. They could work like a guard clause.

## Ensuring value

Let's start with two simple functions, `ensureValue` and `requiredValue`.

```typescript 
function ensureValue<T>(value: T | null | undefined): asserts value is NonNullable<T> {
    if (value === null || typeof value === "undefined") {
        throw new Error(`value '${value}' is null or undefined`);
    }
}

function requiredValue<T>(value: T | null | undefined): NonNullable<T> {
    ensureValue(value)
    return value;
}
```

The first function is magical. It does not return any value, but uses some strangely looking syntax in place where the result type is typically put, `... asserts value is NonNullable<T>`. This feature is called [assertion functions](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#assertion-functions). 

```typescript
let pp: Person | undefined;

console.log(pp.name); // error: 'pp' is possibly 'undefined'
console.log(requiredValue(pp).name); // no errors

ensureValue(pp);
console.log(pp.name); // no error s

ensureValue(pp.address);
ensureValue(pp.address.city); // no errors
```

The TypeScript compiler knows that after the `ensureValue` function is called, the passed `value` argument will be of the type specified after the `asserts` keyword. In our case, the built-in type `NonNullable<T>` was used. This type is defined the following way `type NonNullable<T> = T & {};`, it removes `null` and `undefined` values from `T`.  Without calling `ensureValue(pp)` above, the compiler would generate an error `'pp' is possibly 'undefined'` once any object property is read. The `requiredValue` function internally uses `ensureValue` to check whether the `value` exists, and then returns it. 
## Ensuring properties

The first two functions focus on any values passed into the functions. The following functions simplify how object properties are handled. 

```typescript
type RequiredProperties<T extends object, P extends keyof T> = Omit<T, P> & Required<Pick<T, P>>;

function formatMessage(obj: object, propertyName: string): string {
    return `property '${propertyName}' of '{${Object.keys(obj)}}' object is null or undefined`
}

function ensureProperties<T extends object, P extends keyof T>(obj: T | null | undefined | RequiredProperties<T, P>, ...propertyNames: P[]): asserts obj is RequiredProperties<T, P> {
    ensureValue(obj);
    for (const propertyName of propertyNames) {
        if (obj[propertyName] === null || typeof obj[propertyName] === "undefined") {
            throw new Error(formatMessage(obj, String(propertyName)));
        }
    }
}

function requiredProperty<T extends object, P extends keyof T>(obj: T | null | undefined, propertyName: P): NonNullable<T[P]> {
    ensureValue(obj);
    if (obj[propertyName] === null || typeof obj[propertyName] === "undefined") {
        throw new Error(formatMessage(obj, String(propertyName)));
    }
    return obj[propertyName];
}
```

The `ensureProperties` function takes any number of properties and checks whether all exist. The second function `requiredProperty` checks and returns the value of the specified property. Additionally, the type of the property is changed by removing optionality. 


```typescript
let p: Person | undefined;

console.log(p.name.toUpperCase()); // error: 'p.name' is possibly 'undefined'
console.log(requiredProperty(p, "name").toUpperCase());

ensureProperties(p, "name", "age");
console.log(p.name.toUpperCase(), p.age >= 18 ? "adult" : "child") // no errors
```

On the type level, the `RequiredProperties` does the whole work. After the execution of `ensureProperties(p, "name", "age")`, the type of the `p` variable is changed. The property `name` and `age` are no longer optional. Of course, this code is type-safe. Intellisense suggests the correct property names, and the compiler checks them during compilation. 

The `RequiredProperties` type is implemented based on three built-in types: `Omit`, `Pick`, and `Required`. Their definitions are simple.

```typescript
type Pick<T, K extends keyof T> = {
    [P in K]: T[P];
};

type Exclude<T, U> = T extends U ? never : T;

type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

type Required<T> = {
    [P in keyof T]-?: T[P];
};
```

## Ensuring paths

It's time for the final part of our article, the hardest one. The `ensureProperties` function allows for the specification of many properties, but only one level deep. Let's say we want to ensure the `city` property is set. 

```typescript
let p: Person | undefined;

console.log(p.address.city.toUpperCase());
// 'p' is possibly 'undefined'.
// 'p.address' is possibly 'undefined'.
// 'p.address.city' is possibly 'undefined'.
```

The TypeScript compiler generated three errors, as displayed above. We can use previous functions to resolve those issues.

```typescript
let p: Person | undefined;

ensureValue(p);
ensureValue(p.address);
ensureValue(p.address.city);
// or 
ensureProperties(p, "address");
ensureProperties(p.address, "city");

console.log(p.address.city.toUpperCase()); // no errors
```

It works, but it's very verbose. The problem is that the long, nested path has to be checked on each level. The last function, `ensurePaths`, supports such a complicated scenario. 

```typescript
let p: Person | undefined;

ensurePaths(p, "address.city", "name");
console.log(p.address.city.toUpperCase(), p.name.trim()); // no errors
```

Yep, it is TypeScript. It has done it again. I don't know any other programming language that can do such things. We can pass an object `p` and the path like `"address.city."` If `p` or `p.address` or `p.address.city` are not set, an exception with detailed information will be thrown. Otherwise, the type definition of the `p` object will be changed recursively, such that all segments along the `"p.address.city"` path will be required. Next, I will paste the whole code responsible for this magic, but please don't read it too deeply.

```typescript
export type Paths<O> = PathsAux<O, [], "">[number];

type PathsAux<O, Acc extends string[], Current extends string> =
    O extends object ?
    keyof O extends infer Keys
    ? Keys extends string
    ? Keys extends keyof O
    ? PathsAux<O[Keys], [...Acc, `${Current}${Keys}`], `${Current}${Keys}.`>
    : Acc : [...Acc, Keys] : Acc : Acc;

type PathsOrString<T> = string & Paths<T>

type SplitPath<T extends string> = T extends `${infer Head}.${infer Tail}` ? [Head, Tail] : (T extends `${infer Head}` ? [Head, unknown] : never);

type RequiredSinglePath<T extends {}, P extends PathsOrString<T>> =
    SplitPath<P> extends [infer Head extends keyof T, infer Tail]
    ? Omit<T, Head> & { [key in Head]-?: Tail extends string ? RequiredSinglePath<NonNullable<T[Head]>, Tail> : T[Head] }
    : never;

type RequiredPaths<T extends {}, P extends Array<Paths<T>>> =
    P extends [infer Head extends PathsOrString<T>, ...infer Rest extends Array<PathsOrString<T>>]
    ? (RequiredSinglePath<T, Head> & RequiredPaths<T, Rest>) : unknown;


function ensurePaths<T extends object, P extends Array<PathsOrString<T>>>(obj: T | null | undefined | RequiredPaths<T, P>, ...propertyPaths: P): asserts obj is RequiredPaths<T, P> {
    ensureValue(obj);
    for (const propertyPath of propertyPaths) {
        let currentObj: any = obj;
        for (const propertyName of propertyPath.split(".")) {
            ensureProperties(currentObj, propertyName);
            currentObj = currentObj[propertyName];
        }
    }
}

type RequiredSinglePathValue<T extends {}, P extends PathsOrString<T>> =
    SplitPath<P> extends [infer Head extends keyof T, infer Tail]
    ? Tail extends string ? RequiredSinglePathValue<NonNullable<T[Head]>, Tail> : NonNullable<T[Head]>
    : never;

function requiredPath<T extends object, P extends PathsOrString<T>>(obj: T | null | undefined, propertyPath: P): RequiredSinglePathValue<T, P> {
    ensurePaths(obj, propertyPath);
    return propertyPath.split(".").reduce((o, properyName) => o[properyName], obj as any);
}
```

Relax. I won't explain all those types in detail. We want to finish reading the article this week. Just look at the examples below, try to understand what those types do, not necessarily how they were implemented. 

```typescript
type Type1 = RequiredProperties<Person, "age" | "name">;
// -> Omit<Person, "name" | "age"> & Required<Pick<Person, "name" | "age">>

type Type2 = Paths<Person>
// -> "name" | "age" | "address" | "address.city" | "address.country" | "address.street" | "address.no"

type Type3 = SplitPath<"person.address.city">
// -> ["person", "address.city"]

type Type4 = RequiredSinglePath<Person, "address.city">
// -> Omit<Person, "address"> & { address: Omit<{ country?: string; city?: string; street?: string; no?: number;}, "city"> & { city: string; }; }

type Type5 = RequiredPaths<Person, ["address.city", "address.country", "name"]>
// -> Omit<Person, "address"> & { address: Omit<{ country?: string; ... }, "city"> & { city: string; }; } & { address: Omit<{ country?: string; ... }, "country"> & {...; }; } & Omit<...> & { ...; }

type Type6 = RequiredSinglePathValue<Person, "address.no">
// -> number
```