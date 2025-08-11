---
layout: post
title: Async CTP on WP7 [C#]
date: 2011-01-21
tags:
  - csharp
series: asyncctp
---
In this post I will show you how to use [Async CTP](http://msdn.microsoft.com/en-us/vstudio/gg316360) released during last PDC conference inside your Windows Phone application. Async CTP extends two .Net platform languages C# and VB giving us a new way of writing asynchronous code. There are two main aspects of this project: new C#/VB compiler (two new keywords: async, await) and library AsyncCtpLibrary.dll. When the compiler sees asynchronous method it generates IL code (which we will see in details further) and that code is using some types from the mentioned library. The problem is that so far we have only two versions of library: .Net and Silverlight. So in this post I will show you how we can use  Async CTP on Windows Phone 7 despite these limitations.

The first thing we need to do before we write our first asynchronous method in the WP7 project is we need to add the reference to Silverlight version of the library AsyncCtpLibrary_Silverlight.dll. Just after doing it Visual Studio shows  a warning: “Adding a reference to a Silverlight assembly may lead to unexpected application behavior. Do you want to continue?”. The problem is that the assembly is compiled under full version of Silverlight and the Windows Phone does not contain the full version but some subset of it running on the top of .Net compact framework. So if we execute some code from such a assembly which is using types that are not available on the phone we will get the runtime exception. The key element of the AsyncCtpLibrary are Tasks and unfortunately they are not running on the phone throwing exception at runtime. The question is: can we use Async CTP without Tasks ? Of course we can! The whole mechanism behind asynchronous methods doesn’t force us to use the instance of type Task followed by await keyword. That type needs to contain method (instance or extension method) called GetAwaiter returning any type that contains appropriate pair of methods (instance or extension) BeginAwait and EndAwait. And all I did is I implemented such a matching type.

```csharp
public enum AwaiterResultType
{
    Completed,
    Failed,
    Cancelled
}

public class AwaiterResult<T>
{
    public AwaiterResultType ResultType { get; private set; }
    public Exception Exception { get; private set; }
    public T Value { get; private set; }

    private AwaiterResult() { }
    public static AwaiterResult<T> Completed(T result)
    {
        return new AwaiterResult<T> { ResultType = AwaiterResultType.Completed, Value = result };
    }
    public static AwaiterResult<T> Failed(Exception exception)
    {
        return new AwaiterResult<T> { ResultType = AwaiterResultType.Failed, Exception = exception };
    }
    public static AwaiterResult<T> Cancelled()
    {
        return new AwaiterResult<T> { ResultType = AwaiterResultType.Cancelled };
    }
}

public class Awaiter<T>
{
    private SynchronizationContext Context { get; set; }
    private bool IsSynchronized { get; set; }
    private Action _continuation;

    public AwaiterResult<T> Result { get; private set; }

    private Awaiter() { }

    public Awaiter<T> GetAwaiter()
    {
        return this;
    }

    public bool BeginAwait(Action continuation)
    {
        if (Result != null)
            return false;

        _continuation += continuation;
        return true;
    }

    public T EndAwait()
    {
        if (Result == null)
            throw new InvalidOperationException("Awaiter does not contain the result yet.");

        if (Result.ResultType == AwaiterResultType.Completed)
            return Result.Value;

        if (Result.ResultType == AwaiterResultType.Failed)
            throw Result.Exception;

        return default(T); // if cancelled
    }

    private void OnResult(AwaiterResult<T> result)
    {
        if (result == null) throw new ArgumentNullException("result");
        if (Result != null) throw new InvalidOperationException("The result can be provided only once.");

        Result = result;
        if (IsSynchronized && Context != null)
        {
            Context.Post(o =>
                             {
                                 if (_continuation != null)
                                     _continuation();
                                 _continuation = null;
                             }, null);
            Context = null;
        }
        else
        {
            if (_continuation != null)
                _continuation();
            _continuation = null;
        }
    }

    public static Awaiter<T> Create(Action<Action<AwaiterResult<T>>> resultProvider, bool synchronizeWithCurrentContext = true)
    {
        if (resultProvider == null) throw new ArgumentNullException("resultProvider");

        var awaiter = new Awaiter<T> { IsSynchronized = synchronizeWithCurrentContext };
        if (synchronizeWithCurrentContext)
            awaiter.Context = SynchronizationContext.Current;
        resultProvider(awaiter.OnResult);
        return awaiter;
    }
}
```

The key class here is Awaiter class containing appropriate three methods: GetAwaiter, BeginAwait and EndAwait. This is an abstraction of asynchronous work (very similar to Task type) which at some point in time is finishing its work in one of three possible states: an exception could be thrown, maybe someone cancelled the execution or the work has been completed correctly returning some result. The constructor is private so the factory method called Create is the only way to create an instance of Awaiter type. So let’s see how we can use it:

```csharp
public static class AwaiterEx
{
    public static Awaiter<T> Run<T>(Func<T> action)
    {
        return Awaiter<T>.Create(resultProvider =>
        {
            ThreadPool.QueueUserWorkItem(o =>
            {
                try
                {
                    var result = action();
                    resultProvider(AwaiterResult<T>.Completed(result));
                }
                catch (Exception exception)
                {
                    resultProvider(AwaiterResult<T>.Failed(exception));
                }
            },null);
        });         
    }

    public static Awaiter<object> Delay(TimeSpan timeSpan)
    {
        return Awaiter<object>.Create(resultProvider =>
        {
            var timer = new Timer(o => resultProvider(AwaiterResult<object>.Completed(null)));
            timer.Change((int)timeSpan.TotalMilliseconds, -1);
        });
    }
}

async private static void Sample()
{
    int intResult = await AwaiterEx.Run(() =>
    {
        Thread.Sleep(3000);
        return 123;
    });
    MessageBox.Show("Completed: " + intResult);

    await AwaiterEx.Delay(TimeSpan.FromSeconds(3));
    MessageBox.Show("After 3 seconds...");
}
```

Now let’s look at simplified version of what compiler is generating underneath to allow us to write synchronously looking code running asynchronously.

```csharp
private static void SampleInternals()
{
    Awaiter<int> awaiter1 = null;
    Awaiter<object> awaiter2 = null;
    int state = 0;
    
    Action action = null;
    action = () =>
    {
        if (state == 1) goto JUMP_LABEL_1;
        if (state == 2) goto JUMP_LABEL_2;
        
        awaiter1 = AwaiterEx.Run(() =>
        {
            Thread.Sleep(3000);
            return 123;
        }).GetAwaiter();
        state = 1;
        
        if (awaiter1.BeginAwait(action))
            return;
        
        JUMP_LABEL_1:
        var intResult = awaiter1.EndAwait();
        MessageBox.Show("Completed: " + intResult);


        awaiter2 = AwaiterEx.Delay(TimeSpan.FromSeconds(3)).GetAwaiter();
        state = 2;

        if (awaiter2.BeginAwait(action))
            return;

        JUMP_LABEL_2:
        awaiter2.EndAwait();
        
        MessageBox.Show("After 3 seconds...");
    };

    action();         
}
```

The last thing worth mentioning is how to write asynchronous method returning some value. Asynchronous methods have two limitations: void, Task and `Task<T>` are the only valid return types and out parameters are not allowed. As we said before Tasks under WP7 don’t work so we cannot return Task object. In fact the solution is very simple:

```csharp
public static class AwaiterEx
{
    public static Awaiter<object> BeginWriteAwaiter(this Stream stream, byte[] buffer, int offset, int numBytes)
    {
        return Awaiter<object>.Create(resultProvider =>
        {
            try
            {
                stream.BeginWrite(buffer, offset, numBytes, o =>
                {
                    try
                    {
                        stream.EndWrite(o);
                        resultProvider(AwaiterResult<object>.Completed(null));
                    }
                    catch (Exception exception)
                    {
                        resultProvider(AwaiterResult<object>.Failed(exception));
                    }
                }, null);
            }
            catch (Exception exception)
            {
                resultProvider(AwaiterResult<object>.Failed(exception));
            }
        });
    }
}

public Awaiter<object> WriteFileAwaiter(string path, string text)
{
    return Awaiter<object>.Create(async resultProvider =>
    {
        try
        {
            using (var stream = IsolatedStorageFile.OpenFile(path, FileMode.Create))  
                // caution: in fact the stream is running synchronously on WP7
            {
                var data = Encoding.Unicode.GetBytes(text);
                await stream.BeginWriteAwaiter(data, 0, data.Length);
                resultProvider(AwaiterResult<object>.Completed(null));
            }
        }
        catch (Exception exception)
        {
            resultProvider(AwaiterResult<object>.Failed(exception));
        }
    });
}
```

I encourage you to [download sources](http://code.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=mnajder&DownloadId=14873) with attached samples and play with Async CTP on WP7 because it changes a lot in terms of writing asynchronous code. AsyncCtpLibrary library provides TaskEx class with few very useful methods so you can find counterpart in my library called AwaiterEx with following API:

```csharp
public static class AwaiterEx
{
    public static Awaiter<string> DownloadStringAwaiter(this WebClient webClient, Uri uri) { ... }
    public static Awaiter<object> BeginWriteAwaiter(this Stream stream, byte[] buffer, int offset, int numBytes) { ... }
  
    public static Awaiter<T> ToAwaiter<T>(this IObservable<T> observable) { ... }
    public static Awaiter<T[]> ToAwaiterAll<T>(this IObservable<T> observable) { ... }
    public static IObservable<T> ToObservable<T>(this Awaiter<T> awaiter) { ... }
    
    public static Awaiter<T[]> WhenAll<T>(this IEnumerable<Awaiter<T>> awaiters) { ... }
    public static Awaiter<T[]> WhenAll<T>(params Awaiter<T>[] awaiters) { ... }
    public static Awaiter<T> Run<T>(Func<T> action) { ... }
    public static Awaiter<object> Delay(TimeSpan timeSpan) { ... }
}
```

Have fun and let me know if you liked it or not. [Download](http://code.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=mnajder&DownloadId=14873)
