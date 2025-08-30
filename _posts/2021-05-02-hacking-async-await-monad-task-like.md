---
layout: post
title: Hacking async/await - Monad task-like [C#]
date: 2021-05-02
tags: 
series: hacking-async-await
---

{%- include /_posts/series-toc.md series-name="hacking-async-await" -%}

## Introduction

First, we implemented a custom task-like object like `Optional<T>` using an async method builder. Then, we introduced the concept of a Monad implementing its API for a couple of types, such as `Optional<T>`, `Task<T>`, `IEnumerable<T>`, and `IO<T>`.  Async methods and `async/await` keywords can simplify code interacting with the Monad type in the same way the `foreach` keyword simplifies interaction with the `IEnumerable<T>` type. It's time to reveal how an async method builder based purely on the Monad API could be implemented. 

## Expressing Monad abstraction in the language

Before we present any code, I would like to answer the following question: if Monads are so helpful, why have we never heard about them? The main reason is that most programming languages do not support the language features necessary for expressing abstraction behind Monads. Almost any language supports interfaces, abstract classes, generics, and the ability to describe the type of the function. But it is not enough.

Interface `IEnumerable<T>` represents an abstraction of the Iterator design pattern, where the generic parameter `T` parameterizes the type of iterated items. This simple abstraction allowed programmers to implement over 50 LINQ operators. We would like to have a similar abstraction for Monads. Unfortunately, it is impossible in the current state of C#.

Let's remind that every Monad type `Monad<T>` must implement two functions:
- `Monad<T> Return<T>(T value)`
- `Monad<R> SelectMany<T, R>(Monad<T> m, Func<T, Monad<R>> f)`

C#11 extended the capabilities of interfaces such that static abstract methods can be members of an interface. At least we can express the `Return` static method.

```csharp
interface IMonad<TSelf, T>
    where TSelf : IMonad<TSelf, T>
{
    static abstract TSelf Return(T value);
    // abstract TSelf<R> SelectMany<R>(Func<T, TSelf<R>> f); // incorrect C#
}

class Optional<T> : IMonad<Optional<T>, T>
{
    ... 
    public static Optional<T> Return(T value) => new(value);
}

static M[] CreateMonadsFromValues<M, T>(params T[] values)
    where M : IMonad<M, T>
{
    return Array.ConvertAll(values, v => M.Return(v));
}

var monads = CreateMonadsFromValues<Optional<int>, int>(1, 2, 3);

Console.WriteLine(monads); // [Optiona(1),Optiona(2),Optiona(3)]
```

The definition of methods like `SelectMany` or `Select` is impossible. Haskell has a feature called typeclasses, where the Monad type could be defined in the following way:

```haskell
class Monad m where
    bind   :: m a -> (a -> m b) -> m b
    return :: a -> m a
```

We can think of Haskell typeclasses as more powerful interfaces. In the future, it may be possible to represent the Monad type in C# code. But so far, let's try to use some tricks.

## Monad async method builder

Our implementation of `OptionalMethodBuilder<T>` presented in the first article of the series was specific to the `Optional<T>` type. The hardest part was figuring out the implementation using only Monad methods, `Return`, and `SelectMany`. After a few hours, I finally found the solution. Please, do not try to analyze it too deeply. 

```csharp
public static partial class AwaiterExtensions
{
    public static OptionalAwaiter<T> GetAwaiter<T>(this Optional<T> monad) => new OptionalAwaiter<T>(monad);
}

internal interface IOptionalAwaiter
{
    Optional<object> MoveNext(Action moveNext, Func<Optional<object>> getNextMonad);
}

public class OptionalAwaiter<T> : INotifyCompletion, IOptionalAwaiter
{
    public bool IsCompleted => false;
    public void OnCompleted(Action continuation) { }
    protected T result = default!;
    public T GetResult() => result;

    // ***********************************

    private Optional<T> monad;
    public OptionalAwaiter(Optional<T> monad) => this.monad = monad;

    Optional<object> IOptionalAwaiter.MoveNext(Action moveNext, Func<Optional<object>> getNextMonad)
    {
        return monad.SelectMany(t =>
        {
            result = t;
            moveNext();
            return getNextMonad();
        });
    }
}

public class OptionalMethodBuilder<T>
{
    public void Start<TSM>(ref TSM stateMachine) where TSM : IAsyncStateMachine => stateMachine.MoveNext();

    public void SetStateMachine(IAsyncStateMachine stateMachine) { }
    public void SetException(Exception exception) { }
    public void AwaitUnsafeOnCompleted<TA, TSM>(ref TA awaiter, ref TSM stateMachine)
        where TA : ICriticalNotifyCompletion where TSM : IAsyncStateMachine
    { }

    // ***********************************

    private bool isFirstCall = true;
    private Optional<object>? currentTask;
    public Optional<T>? Task { get; private set; }

    public static OptionalMethodBuilder<T> Create() => new OptionalMethodBuilder<T>();

    public void AwaitOnCompleted<TA, TSM>(ref TA awaiter, ref TSM stateMachine)
        where TA : INotifyCompletion where TSM : IAsyncStateMachine
    {
        if (awaiter is IOptionalAwaiter awaiterO)
        {
            var wasFirstCall = isFirstCall;
            isFirstCall = false;

            currentTask = awaiterO.MoveNext(stateMachine.MoveNext, () => currentTask!);

            if (wasFirstCall)
            {
                Task = currentTask.Select(v => (T)v);
            }
        }
    }

    public void SetResult(T result)
    {
        if (isFirstCall)
        {
            Task = Monad.ReturnO(result);
        }
        else
        {
            currentTask = Monad.ReturnO<object>(result!);
        }
    }
}
```

The key point is that we can now replace the `Optional` type with any other Monad type, and the code should be compiled and run successfully. Of course, we don't want to write manually such code for types like `Task<T>`, `IEnumerable<T>`, and `IO<T>`. This is why I created a T4 template generating C# code. 

```t4
<#@ output extension=".cs" #>
<#@ import namespace="System.Linq" #>
// this code is generated automatically from 'AsyncMethodBuilders.generated.tt', do not change it manually
#nullable enable

using System.Runtime.CompilerServices;

namespace MonadsInCSharp;

<#
    // - restore T4 dotnet global tool
    // dotnet tool restore

    // - execute T4 tamplate
    // dotnet tool run t4 ./MonadsInCSharp/AsyncMethodBuilder/AsyncMethodBuilders.generated.tt 

    var monadTypes = new [] { ("Optional","O"), ("TTask","TT"), ("IO","IO"), ("IEnumerable","E"), ("Task", "T") };

    foreach(var (name, shortcut) in monadTypes)
    {
#>
// ** <#= name #> **
public static partial class AwaiterExtensions
{
    public static <#= name #>Awaiter<T> GetAwaiter<T>(this <#= name #><T> monad) => new <#= name #>Awaiter<T>(monad);
}

internal interface I<#= name #>Awaiter
{
    <#= name #><object> MoveNext(Action moveNext, Func<<#= name #><object>> getNextMonad);
}

public class <#= name #>Awaiter<T> : INotifyCompletion, I<#= name #>Awaiter
{
    public bool IsCompleted => false;
    public void OnCompleted(Action continuation) { }
    protected T result = default!;
    public T GetResult() => result;

    // ***********************************

    private <#= name #><T> monad;
    public <#= name #>Awaiter(<#= name #><T> monad) => this.monad = monad;

    <#= name #><object> I<#= name #>Awaiter.MoveNext(Action moveNext, Func<<#= name #><object>> getNextMonad)
    {
        return monad.SelectMany(t =>
        {
            result = t;
            moveNext();
            return getNextMonad();
        });
    }
}

public class <#= name #>MethodBuilder<T>
{
    public void Start<TSM>(ref TSM stateMachine) where TSM : IAsyncStateMachine => stateMachine.MoveNext();

    public void SetStateMachine(IAsyncStateMachine stateMachine) { }
    public void SetException(Exception exception) { }
    public void AwaitUnsafeOnCompleted<TA, TSM>(ref TA awaiter, ref TSM stateMachine)
        where TA : ICriticalNotifyCompletion where TSM : IAsyncStateMachine
    { }

    // ***********************************

    private bool isFirstCall = true;
    private <#= name #><object>? currentTask;
    public <#= name #><T>? Task { get; private set; }

    public static <#= name #>MethodBuilder<T> Create() => new <#= name #>MethodBuilder<T>();

    public void AwaitOnCompleted<TA, TSM>(ref TA awaiter, ref TSM stateMachine)
        where TA : INotifyCompletion where TSM : IAsyncStateMachine
    {
        if (awaiter is I<#= name #>Awaiter awaiter<#= shortcut #>)
        {
            var wasFirstCall = isFirstCall;
            isFirstCall = false;

            currentTask = awaiter<#= shortcut #>.MoveNext(stateMachine.MoveNext, () => currentTask!);

            if (wasFirstCall)
            {
                Task = currentTask.Select(v => (T)v);
            }
        }
    }

    public void SetResult(T result)
    {
        if (isFirstCall)
        {
            Task = Monad.Return<#= shortcut #>(result);
        }
        else
        {
            currentTask = Monad.Return<#= shortcut #><object>(result!);
        }
    }
}
<#
    }
#>
```

Now we can choose any type, for instance, `IObservable<T>`. First, we implement Monad functions. Then, we change the T4 template at the top where types are defined by adding a new entry, such as `var monadTypes = new [] { ("Optional","O"), ..., ("IObservable", "OO") }`. After saving the template file, a new version of the C# file will be generated automatically. 

## Summary

In the end, let's discuss the constraints of my solution. I already mentioned a strange behaviour I encountered while testing async methods with the `IEnumerable` type. I got a stack overflow exception once I started iterating over the result of such an async method. My solution does not work with Monad, which the `SelectMany` method calls the passed function more than once. `IEnumerable<T>`, `Observable<T>`, and `T[]` are examples of such Monad types. 

The second problem occurred once I tried to use a custom async method builder for the type that already provides one, like `Task<T>` or `ValueTask<T>`. The complete working solution requires two classes, builder and awaiter. Even if we provide an extension method `GetAwaiter()`, the C# compiler will not use it inside the generated state machine because types like  `Task<T>` or `ValueTask<T>` provide built-in instance methods. We have led to inconsistency, where our builder class is used, but our awaiter class is not used. Not knowing what was happening, I started debugging the builder class code. I noticed that the empty method `AwaitUnsafeOnCompleted` was called instead of `AwaitOnCompleted`. After googling for a while, I finally understood what was really happening. C# compiler generates [different code](https://github.com/dotnet/roslyn/blob/d4dab355b96955aca5b4b0ebf6282575fad78ba8/src/Compilers/CSharp/Portable/Lowering/AsyncRewriter/AsyncMethodToStateMachineRewriter.cs#L423) for the state machine depending on the type of awaiter returned from the `GetAwaiter()` method. If the type implements the `ICriticalNotifyCompletion` interface, the `AwaitUnsafeOnCompleted` method is called. Otherwise, the `AwaitOnCompleted` method is called.

Theoretically, we could change the implementation of the async method builder to support such scenarios, but I decided it was not worth the effort. Ultimately, all the work that was done served only an educational purpose. It was one massive hack, but I hope you learned something interesting.
