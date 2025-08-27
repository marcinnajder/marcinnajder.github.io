---
layout: post
title: Hacking async/await - Optional task-like [C#]
date: 2021-04-18
tags: 
series: hacking-async-await
---

{%- include /_posts/series-toc.md series-name="hacking-async-await" -%}

## Introduction

The async/await C# feature fundamentally changed how we write asynchronous code. It changed not only C# and the .NET ecosystem, but also other technologies. Nowadays, async/await is supported in many programming languages, such as JS, Dart, Python, and Kotlin. 

I remember precisely when it started. Around 2010, we were building Silverlight applications and racking our brains to find the simplest way of writing asynchronous code. Silverlight introduced a problematic constraint: the main UI thread could execute IO operations only asynchronously. Of course, Silverlight is just a .NET running in browsers, so it supports the same approaches to writing asynchronous code. All of them can be summarized as "continuation passing style." Such a code is hard to write and maintain. And suddenly, Microsoft presented Async CTP (Community Technology Preview), the first beta release of the async/await feature. We started using it immediately without hesitation; the difference between our old and new code was significant.

Finally, async/await was officially released in 2012 as a main feature of C# 5.0 and .NET 4.5. In 2017, C# 7.0 introduced some additions to async/await. Initially, an async method could only return `void`, `Task`, or `Task<T>`. Currently, the async method can also return `ValueTask`, `ValueTask<T>`, `IAsyncEnumerable<T>`, or any [task-like](https://github.com/dotnet/csharplang/blob/main/proposals/csharp-7.0/task-types.md) type. Quoting a [great article](https://devblogs.microsoft.com/premier-developer/extending-the-async-methods-in-c/):

> There are 3 ways how you can control the async method’s machinery:
> 1. Provide your own async method builder in the `System.Runtime.CompilerServices` namespace.
> 2. Use custom task awaiters.
> 3. Define your own task-like types.

In this article, we will implement a custom task-like type, providing our version of the `AsyncTaskMethodBuilder` type.

## Optional type

In 2005, .NET 2.0 introduced generic types, including the `Nullable<T>`. Value types like `int`, `bool`, or `DateTime` are not nullable. We can wrap them into `Nullable<T>` using question mark, writing code like `int?`, `bool?`, or `DateTime?`. And now, the following C# code `int? a = null;` is entirely valid. `Nullable<T>` type definition has a type constraint for `T` saying that the `T` type can only be a value type, not a reference type. This makes sense because reference types permit `null` values by default. 

Functional languages handle a lack of value in a specific way; they introduce a dedicated type, often called `Optional`, or `Option`. The concept of `null` value does not exist. Let's implement its equivalent in C#.

```csharp
public partial class Optional<T>
{
    public bool HasValue { get; }
    public T Value { get; }
    public Optional() => (HasValue, Value) = (false, default);
    public Optional(T value) => (HasValue, Value) = (true, value);

    public static Optional<T> None { get; } = new Optional<T>();
}
```

It is defined almost the same as the built-in `Nullable<T>` type, but `Optional<T>` is a class instead of a struct and has no constraint for the `T` type. But two properties, `HasValue` and `Value`, are the same. Now, we can use it instead of `Nullable<T>` to express the optionality of any value or reference type.

```csharp
class OptionalSamples
{
    Optional<int> TryParseInt(string text) =>
        int.TryParse(text, out var result) ? new Optional<int>(result) : Optional<int>.None;

    public static Optional<string> ProcessText1(string text1, string text2)
    {
        Optional<int> number1 = TryParseInt(text1);
        if (!number1.HasValue)
        {
        	return new Optional<string>(); // or Optional<string>.None
    	}

        Optional<int> number2 = TryParseInt(text2);
        if (!number2.HasValue)
        {
        	return new Optional<string>();
    	}

        return new Optional<string>((number1.Value + number2.Value).ToString());
    }
}
```

The first method, `TryParseInt`, converts a `string` value into an `int` only if the passed text value represents a valid number. Otherwise, an object representing a lack of value is returned. Method `ProcessText1` takes two strings and tries to parse them. If both numbers are valid, their sum is returned as a string. The whole idea is to propagate information about the potential lack of value. In contrast to `Nullable<string>`, code using `Optional<string>` where `string` is a reference type will be successfully compiled.

## Optional type and async/await

Now, let's look at the alternative implementation.

```typescript
class OptionalSamples
{
    async Optional<string> ProcessText2(string text1, string text2)
    {
        int number1 = await TryParseInt(text1);
        int number2 = await TryParseInt(text2);

        return (number1 + number2).ToString();
    }
}
```

It may be hard to believe, but the method `ProcessText2` above builds without errors and runs like the first `ProcessText1` method. Method signatures are the same. We can use the `await` keyword instead of manually checking the presence of a value. We return a final result as a `string` type, which will be automatically wrapped into `Optional<string>`.

The question is, how is this possible? From the beginning of async/await, we could write custom awaiters. For instance, Rx.NET provided an appropriate [extension method](https://github.com/dotnet/reactive/blob/fa1629a1e12a8fc21c95aeff7863425c2485defd/Rx.NET/Source/src/System.Reactive/Linq/Observable.Awaiter.cs#L21) for `IObservable<T>` interface so any observable object can be awaited. According to the documentation:

> In order for a type to be “awaitable” (i.e. to be valid in the context of an `await` expression) the type should follow a special pattern:
> - Compiler should be able to find an instance or an extension method called `GetAwaiter`. The return type of this method should follow certain requirements:
> - The type should implement [`INotifyCompletion`](http://referencesource.microsoft.com/#mscorlib/system/runtime/compilerservices/INotifyCompletion.cs,23) interface.
> - The type should have `bool IsCompleted {get;}` property and `T GetResult()` method.

This extension point was not enough. We could have awaited types other than `Task`, but we couldn't return any custom type from an asynchronous method. It had to be a `Task` or `void`. Since C# 7.0, we can use a new `AsyncMethodBuilderAttribute` attribute to specify an implementation of a custom async method builder. We will implement two additional types for `Optional<T>`: `OptionalAwaiter`, `OptionalMethodBuilder`.

 The term [duck typing](https://en.wikipedia.org/wiki/Duck_typing) is often used in dynamic programming. If I want to execute some functionality from an object, its real type does not matter as long as this object provides the set of members I need. For instance, in JS, any object containing the `then` method can be treated as an object of type `Promise`. Another example would be array-like objects, where any object with a `length` property and an indexer allowing reading the nth element can be treated as an object of type `Array`. 

The `OptionalAwaiter` and `OptionalMethodBuilder` types must provide a specific set of properties and methods described in the C# specification. The compiler will use those members in the generated code. To understand the general idea behind writing a custom async builder, let's just look at the code below. We don't have to analyze this in detail right now. 

```csharp
using System;
using System.Runtime.CompilerServices;

[AsyncMethodBuilder(typeof(OptionalMethodBuilder<>))]
public partial class Optional<T> {  }

public static class OptionalExtensions
{
    public static OptionalAwaiter<T> GetAwaiter<T>(this Optional<T> optional)
	    => new OptionalAwaiter<T>(optional);
}

internal interface IOptionalAwaiter
{
    bool HasValue { get; }
}

public class OptionalAwaiter<T> : INotifyCompletion, IOptionalAwaiter
{
    private Optional<T> optional;
    public bool IsCompleted => this.optional.HasValue;
    bool IOptionalAwaiter.HasValue => optional.HasValue;

    public OptionalAwaiter(Optional<T> optional) => this.optional = optional;

    public T GetResult() => optional.Value;
    public void OnCompleted(Action continuation) { }
}


public class OptionalMethodBuilder<T>
{
    public static OptionalMethodBuilder<T> Create()
	    => new OptionalMethodBuilder<T>();

    public void Start<TSM>(ref TSM stateMachine) where TSM : IAsyncStateMachine
	    => stateMachine.MoveNext();

    public Optional<T> Task { get; private set; }
    public void SetResult(T result) => Task = new Optional<T>(result);

    public void AwaitOnCompleted<TA, TSM>(ref TA awaiter, ref TSM stateMachine)
        where TA : INotifyCompletion where TSM : IAsyncStateMachine
    {
        if (awaiter is IOptionalAwaiter { HasValue: false })
        {
            Task = new Optional<T>();
        }
        else
        {
            awaiter.OnCompleted(stateMachine.MoveNext);
        }
    }

	// empty methods
    public void SetStateMachine(IAsyncStateMachine stateMachine) { }
    public void SetException(Exception exception) { }
    public void AwaitUnsafeOnCompleted<TA, TSM>(ref TA awaiter, ref TSM stateMachine)
        where TA : ICriticalNotifyCompletion where TSM : IAsyncStateMachine { }
}
```

Type `Optional<T>` is decorated with an attribute `AsyncMethodBuilder` pointing to `OptionalMethodBuilder<T>`. This class implements some magic properties and methods, some of which are static or even empty. The simplest way to understand anything from this cryptic code is to copy the whole code above to the web side[ https://sharplab.io/](https://sharplab.io/), click the [link](https://sharplab.io/#v2:CYLg1APgAgTAjAWAFBQAwAIpwHQCUCuAdgC4CWAtgKbYDCA9uQA6kA2lATgMocBupAxpQDOAbmTIoADnQB5RmTqEAhiwA8WVAD50ABXZ1BQoQBVKAD2IwAFBvTFzxOABpMcDPYswAlMgDeydED0UhJ0QnxyACMOOHQAXkwATnRjdgBPHSV2IUoASRIrD0cvMSQg4NDwqI4YeKSU9MzsvIKi71KAoKgAdnQrKuj2WLAwiMHvbGM6TmJ2EIBzKxLkAF9xJDkFZTUQ4m1UjKyc/OIbNzsHL3jNTsDdycajykKHFzp8YnQeLPR2YXwWMQrgB+MKUADusnkpEUKlUu00Vj+QgBQPQIChWzhCOwADlFJQOkgJABmdCMLJkFSYWqbGHbVTGG5IfxlLpkyJ0OgsdAACSUQgAaip8JR0L50PNKMQROg1mzAlAycZ0MKWKLxZLpbL5eUlZj6Soltc+vyhSLKC41aKrgkrAAzFQ5FzASiO1HLBWYMl02EsKwq77qyi27RWM3Wy2qi22vqzUUuIM2ol6slYABsBr9jO0+MIYolUplcrq+chvoZTKWpXl6wA2gBBIRpQj8ACy0oAFnRgAAhfCsV3sQppRiUOj2qwVlQd4jdvsDlhD1SaLxeAC6pPJlNI1NgWcr2glcvW+ozNIPKgAohZKIQhPShH5bt7XJnpywG+ClKR7Owc+gADi0pfj+f45oUnakEIl5qEy6B0NCfo+EgACQ5RxNoZawaBv4cBBiFYiwnq1kguwcNsFR/o6gjoLkH64X+z5epy3J8gKkaakWOqrKeaa0kh2yMfh8EYrk+JkPaaT0EwbBbC49GCSownsMx5SMHM3z2LBAGEYaLClKm6CsTyuRCDJjByZQwAmnO0HYHpfrYBGFqGUEJl0Qx354ewzkcRaJqOdsfnmsGKbsjh3l/lOSlwdoQUqKGdhQUIDmxXUCUGesRkqsBxC4P8gLGphCGxdgkZuYqaYACyyIQFlWcAZy1PwihkOEShbFcEryqRW77h+s7zv2g4icyrJGeeg1dj2I1LmN6A0H8nXPCh6FBCV2HTXOs2LsuVaei+Z61TMlKMpwbaIn89opBd6BCMQK1tko/BQfmVzgp2HBisYd1iU2Lb8Kd9jPa9ISUMg62BCVD1PS9b3UG2dA8JQuIONW2URR+AFQJmhbauSmkrfdBO6hFUAndKBUokVKrIh6Jq46WEI6VW9OAodXrHegjEyPVDCWdK1mMg2Li/ZdSJuikDZJBwLjXbdbb3Y9IPw+DKHlOUn3fdL6J0RJpBSQ10r0ug2t/Ireu5ADrbA5QoMIy+E2a3cN02IkHDBDBilESpmoucGGKOiwORyhrLvii+EeYJmCRbbFEGetHZMu5QIcQ165TO9Hioe75fPG/YTWw6rYP5tgSMo2jFhJxHKcnsSaEAPRN+glBMMQaToFQO3AE+XM1eg3DEHbDvg1Y1vNrbKv22r+bK3DZchpq9fc8PN6CIJVgb5Qglt2Ym9dSvR2D4xACq95KPalAFwLjUi2LF1XVLxgy1Aefyy/d0l7PS/hy75sfoyzEktX8AgVAGyNnfE2igzZfQtuLK2NsgYzzHvPHqvFG5AA===) to see what will happen. This website compiles C# code into assembly, and then disassembles it back to the C# code.

When the compiler sees an asynchronous method like this:

```csharp
async Optional<string> ProcessText2(string text1, string text2)
{
	int number1 = await TryParseInt(text1);
	int number2 = await TryParseInt(text2);

	return (number1 + number2).ToString();
}
```

It automatically generates the following code:

```csharp
internal static Optional<string> ProcessText2(string text1, string text2)
{
    ProcessText2StateMachine stateMachine = new ProcessText2StateMachine();
    stateMachine.builder = OptionalMethodBuilder<string>.Create();
    stateMachine.text1 = text1;
    stateMachine.text2 = text2;
    stateMachine.state = -1;
    stateMachine.builder.Start(ref stateMachine);
    return stateMachine.builder.Task;
}

private sealed class ProcessText2StateMachine : IAsyncStateMachine
{
    public int state;
    public OptionalMethodBuilder<string> builder;
    public string text1, text2;

    private int number1, number2;
    private object awaiterObj;

    private void MoveNext()
    {
        int num = state;
        string result;
        try
        {
            OptionalAwaiter<int> awaiter;
            OptionalAwaiter<int> awaiter2;
            if (num != 0)
            {
                if (num == 1)
                {
                    awaiter = (OptionalAwaiter<int>)awaiterObj;
                    awaiterObj = null;
                    num = (state = -1);
                    goto IL_00eb;
                }
                awaiter2 = TryParseInt(text1).GetAwaiter();
                if (!awaiter2.IsCompleted)
                {
                    num = (state = 0);
                    awaiterObj = awaiter2;
                    ProcessText2StateMachine stateMachine = this;
                    builder.AwaitOnCompleted(ref awaiter2, ref stateMachine);
                    return;
                }
            }
            else
            {
                awaiter2 = (OptionalAwaiter<int>)awaiterObj;
                awaiterObj = null;
                num = (state = -1);
            }
            number1 = awaiter2.GetResult();
            awaiter = TryParseInt(text2).GetAwaiter();
            if (!awaiter.IsCompleted)
            {
                num = (state = 1);
                awaiterObj = awaiter;
                ProcessText2StateMachine stateMachine = this;
                builder.AwaitOnCompleted(ref awaiter, ref stateMachine);
                return;
            }
            goto IL_00eb;
        IL_00eb:
            number2 = awaiter.GetResult();
            result = (number1 + number2).ToString();
        }
        catch (Exception exception)
        {
            state = -2;
            builder.SetException(exception);
            return;
        }
        state = -2;
        builder.SetResult(result);
    }

    void IAsyncStateMachine.MoveNext()
    {
        this.MoveNext();
    }


    private void SetStateMachine(IAsyncStateMachine stateMachine) { }

    void IAsyncStateMachine.SetStateMachine(IAsyncStateMachine stateMachine)
    {
        this.SetStateMachine(stateMachine);
    }
}
```

Wow, this code is vast and strange. :) But again, we don't have to understand every line of code. We can see that each asynchronous method in C# is compiled to a class, where all parameters and local variables become fields inside the class. The whole logic from our async method is put into the `MoveNext` method. The `MoveNext` method implements a state machine, where the private field `state` is a number, representing the current state. This function will be called many times while executing the original method `ProcessText2`, performing transitions from one state to the next state. The number of `MoveNext` method calls corresponds to the number of `await` keywords inside an async method. We even have one `goto` statement. The generated code occasionally calls our code implemented in `OptionalAwaiter<T>` and `OptionalMethodBuilder<string>` classes. 

We can analyze the places around the `await` keyword in the code above.  The awaitable object, like `Optional<T>` in our case, is preceded by the `await` keyword. Such an object should provide an instance and extension method called `GetAwaiter` that returns an awaiter object. This object is checked where the job is done by calling the property `IsCompleted`. If `true`, the execution moves to the following line below the `await` keyword. If `false`, the async builder object is asked to handle the continuation by calling the `AwaitOnCompleted` method. In our scenario, no asynchronous work that needs to be completed; `Optional<T>` is always finished. The built-in `Task` object representing the background job implements both branches, synchronous and asynchronous.

## Summary

Conceptually, the async/await feature is much deeper than we could think. In the next article, I will explain what I mean. So far, we have seen that any custom type can be defined as a result of an async method, as long as we provide objects with specified properties and methods. We saw how an async function is translated to a class implementing a state machine during compilation. It's worth mentioning that a similar code is generated for C# iterators, methods returning `IEnumerable<T>` or `IEnumerator<T>` using the `yield` keyword. C# has had this feature since C# 2.0 (2005). It is not a coincidence that any programming language supporting async/await, also supports iterators. Our simple example with the `Optional<T>` type has shown that async/await can also be useful in broader scenarios. Next time, we will be talking about a sacred topic - Monads.

