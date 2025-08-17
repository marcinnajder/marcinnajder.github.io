---
layout: post
title: Immutable list using modern C# part 4 - LINQ [C#]
date: 2023-11-07
tags: 
series: immutable-list-csharp
---
{%- include /_posts/series-toc.md series-name="immutable-list-csharp" -%}

## LList

In the first part of the series, we have implemented many LINQ operators like `Where, Select, SelectMany, Count, ....` . They were extension methods returning our immutable list `LList<T>`, and returning `LList<T>` or a scalar value. Thanks to these methods, we could write regular LINQ queries using query expression syntax, such as `from x in list where x > 0 selects x * 10`.  That query is translated into a chain of method execution, `list.Where(x => x > 0).Select(x => x * 10)`. After executing the `Where` method, a temporary list is created in memory and passed into the `Select` method, creating another list. 

Our list is a perfect candidate for a sequence represented as an `IEnumerable<T>` interface in  .NET. We can easily convert between `LList` and the `IEnumerable<T>` interface in both directions. This way, we could use more than 50 built-in LINQ operations. However, the internal behavior would differ from our custom implementation. We will discuss the pros and cons of both approaches. 

Our `LList<T>` could implement an `IEnumerable<T>` the following way:

```csharp
public record LList<T> : IEnumerable<T>
{
    // ...
    public IEnumerator<T> GetEnumerator()
    {
        var node = this;
        while (!node.IsEmpty)
        {
            yield return node.Head;
            node = node.Tail;
        }
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();

    public override string ToString() => $"[{string.Join(", ", this)}]";
}

public static class LList
{
    public static LList<T> ToLList<T>(this IEnumerable<T> items)
        => items switch
        {
            LList<T> llist => llist,
            T[] array => Of(array),
            IList<T> ilist => Enumerable.Range(1, ilist.Count)
	            .Aggregate(LList<T>.Empty, (list, i) => new(ilist[^i], list)),
            var ienumerable => FromSeqUsingRecursion(ienumerable)
        };

    private static LList<T> FromSeqUsingRecursion<T>(this IEnumerable<T> items)
    {
        using var iterator = items.GetEnumerator();
        return Next(iterator);
        
        static LList<T> Next(IEnumerator<T> iter) =>
	        iter.MoveNext() ? new(iter.Current, Next(iter)) : LList<T>.Empty;
    }
}

LList<int> ints = [5, 10, 15, 20, 25];
var q =
    from i in ints
    where i % 10 == 0
    select $"{i:.00}";
Console.WriteLine(q.ToLList()); // [10,00, 20,00]
```

Since `LList<T>` type implements the `IEnumerable<T>` interface, the overridden method `ToString` takes one line of code. The `ToLList` converts `IEnumerable<T>` to `LList<T>`, trying to be intelligent about that. It checks the real type of the `items` argument, applying a dedicated and most efficient strategy. We will discuss this topic in detail in the next part. Now, let's come back to LINQ.

LINQ provides around 50 extension methods for the basic `IEnumerable<T>` interface. Knowing that the data structure is of type `LList<T>`, we can offer better implementations for some of them, for instance `Count`, `First`, `Single`, ... . We can even implement all LINQ operators like `Where, Select, ...` for type `LList<T>`, when this makes sense. There is a big difference between `IEnumerable<T>` and `LList<T>` in .NET.  The first type is lazy, the second type is eager. In most cases where we execute a few operators together in a chain, the lazy approach is preferable. However, there are operators for which dedicated implementations could be handy. Let's look at a few of them.

```csharp
public static class LList
{
    public static T First<T>(this LList<T> list) => list.Head;
    
    public static T? FirstOrDefault<T>(this LList<T> list) =>
	    list.IsEmpty ? default : list.Head;
    
    public static T Single<T>(this LList<T> list) =>
        list switch
        {
            [var item] => item,
            _ => throw new InvalidOperationException("Sequence contains no elements or more than one element")
        };

    public static LList<T> ConcatL<T>(this LList<T> list1, LList<T> list2) => 
	    list1 switch
        {
            ([]) => list2,
            [var head, .. var tail] => new(head, ConcatL(tail, list2))
        };

    public static LList<T> SkipL<T>(this LList<T> list, int n) =>
	    (list, n) switch
        {
            ([], _) => [],
            (_, 0) => list,
            ([_, .. var tail], _) => tail.SkipL(n - 1)
        };

    public static LList<T> SkipWhileL<T>(this LList<T> list, Func<T, bool> f) => 
	    list switch
        {
            ([]) => [],
            [var head, .. var tail] => f(head) ? SkipWhileL(tail, f) : list
        };
}

```

Functions above return `LList<T>` instead of `IEnumerable<T>`; they are eager, so running them returns the results immediately. We can take advantage of the linked list structure; the list is just a pair of a `Head` and a `Tail`. We can use the `Tail` property directly once we reach some point while iterating the items.

The following expression `list1.ConcatL(list2)` will work better than calling two LINQ operators like `list1.Concat(list2).ToLList()`. In the first case, the second list `list2` is not iterated; `list2` is used as a tail. Similarly, the same benefits come from the custom implementation of `SkipL` and `SkipWhileL`.

In general, dedicated operators implemented for the `LList<T>` type are useful when executing a single operator, instead of the query combining many operators. In cases such as `list.Where(...).ToLList()`, going forth and back between `IEnumerable<T>` and `LList<T>` types unnecessarily complicates the code and slows down the execution. This is one of the reasons why languages like F# provide dedicated modules for `Seq`, `List`, `Array`, ... types containing the same operators `map`, `filter`, `reduce`, ... .

In the next part, we will try to optimize converting `IEnumerable<T>` into `LList<T>`.

Complete source code can be downloaded from here, [minimal](https://github.com/marcinnajder/misc/blob/master/2021_04_18_make_a_lisp_in_csharp/Mal/Mal/PowerFP/LList.cs) or [full](https://github.com/marcinnajder/misc/blob/master/2023_10_29_algorithms_and_data_structures_in_csharp/AlgorithmsAndDataStructures/LList.cs) .