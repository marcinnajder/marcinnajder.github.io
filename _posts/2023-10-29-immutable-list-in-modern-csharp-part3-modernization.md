---
layout: post
title: Immutable list using modern C# part 3 - modernization [C#]
date: 2023-10-29
tags: 
series: immutable-list-csharp
---
{%- include /_posts/series-toc.md series-name="immutable-list-csharp" -%}
## LList

The C# language still evolves, and a new version of .NET is released each year. Our first minimalistic implementation of immutable linked lists used modern C# features, like records, pattern matching, static imports, etc. But it was 2021, and many interesting features have been added to it since then. Especially in the context of collections, C# 11 (2022) added list patterns, and C# 12 (2023) added collection expressions. In this article, we will modernize our list implementation and improve performance. 

`LList<T>` is a positional record with two properties, `Head` , and `Tail`. In addition to that, we have implemented a static class `LListM` with helper static methods like `LListFrom, ToEnumerable, ToLList`, and extension methods corresponding to the LINQ operators `Where, Select, Count, Aggregate, ...`

```csharp
public record LList<T>(T Head, LList<T>? Tail);

var ints = LListFrom(5, 10, 15, 20, 25);
// new(5, new(10, new(15, new(20, new(25, null)))));

var q =
    from i in ints
    where i % 10 == 0
    select $"{i:.00}";

Console.WriteLine(q);
// LList { Head = 10, Tail = LList { Head = 20,00, Tail =  } }
```

Our implementation caused one minor inconvenience. The empty list was represented as a null value, so we could not distinguish between not having a list (`null` value) and an empty list. Let's change this.

```csharp
public record LList<T>
{
    private readonly int length;
    private readonly T head;
    private readonly LList<T> tail;

    public int Length => length;
    public T Head => EnsureNonEmpty(head);
    public LList<T> Tail => EnsureNonEmpty(tail);

    public LList(T head, LList<T> tail)
	    => (this.head, this.tail, length) = (head, tail, tail.Length + 1);
    
    public LList() => (head, tail, length) = (default!, default!, 0);

    public bool IsEmpty => Length == 0;
    public static LList<T> Empty { get; } = new LList<T>();

    private R EnsureNonEmpty<R>(R result) =>
	    IsEmpty ? throw new InvalidOperationException("List is empty") : result;
}

LList<int> list3 = new(1, new(2, new(3, new())));
LList<int> list4 = new(1, new(2, new(3, LList<int>.Empty)));
```

The new implementation is longer, but the public API has not changed much. Now, we can create an empty list using a new parameterless constructor or the static `Empty` property. Thanks to the `Empty` singleton property, we can avoid allocating new memory for each instance of an empty list.

The second critical new feature is access to the `Length` property. Previously, we had to iterate over all items in the list using the `Count` extension method. Because our list is immutable, we can calculate the `Length` property only once during the creation phase. The constructor increments the length of the `Tail` by one and stores its value in an additional read-only field `length`. Thanks to this trick, access to the length of the list is constant. This will be crucial once we start working with indexes, ranges, and pattern matching.

C# 12 introduces a new feature called "collection expression" that provides a unified way of creating collections of any type, including custom types like `LList<T>`. Because our list is immutable, we must implement a special factory method and use an attribute called `[CollectionBuilder]`.

```csharp
[CollectionBuilder(typeof(LList), nameof(LList.Create))]
public record LList<T> { /* ... */ }

public static class LList
{
    public static LList<T> Create<T>(ReadOnlySpan<T> items)
    {
        var result = LList<T>.Empty;
        for (int i = items.Length - 1; i >= 0; i--)
        {
            result = new(items[i], result);
        }
        return result;
    }

    public static LList<T> Of<T>(params T[] items)
	    => Create(new ReadOnlySpan<T>(items));
}

var list5 = LList.Of(1, 2, 3);
var list6 = LList.Of<int>(); // empty list

// C#12 collection expression
LList<int> list7 = [1, 2, 3];
LList<int> list8 = []; // empty list
LList<int> list9 = [0, ..list7, 4, 5];  // spread operator
```

Previously, static helper methods were defined in the `LListM` class, and now, all new functionality will be implemented in the `LList` class. Code inside the `Create` method is quite tricky. Because type `ReadOnlySpan<T>` gives us the number of elements and the ability to read the nth item of the collection via the indexer, we iterate backwards, building subsequent list nodes. Our list is immutable; the node can not be changed after creation. We'll explain in detail later why the reverse order was so important. 

We will implement the C# indexer and the special `Slice` method in the next step.

```csharp
public record LList<T>
{
    // ...

    public T this[int i] => i >= 0 && i < Length
		? (i == 0 ? Head : Tail[i - 1])
		: throw new ArgumentOutOfRangeException();

    public LList<T> Slice(int start, int length) =>
        length == 0 || (length > 0 && start >= 0 && start < Length && start + length - 1 < Length)
	         ? SliceSafe(start, length)
	         : throw new ArgumentOutOfRangeException();

    private LList<T> SliceSafe(int start, int length) =>
        (start, length) switch
        {
            (_, 0) => Empty,
            (0, _) => length == Length ? this : new(Head, Tail.SliceSafe(0, length - 1)),
            _ => Tail.SliceSafe(start - 1, length)
        };
}

LList<int> ints = [5, 10, 15, 20, 25];

Console.WriteLine($"{ints[0]}, {ints[1]}, ..., {ints[^2]}, {ints[^1]}");
// 5, 10, ..., 20, 25

Console.WriteLine(ints[1..3]); // [10, 15]
Console.WriteLine(ints[1..]); // [10, 15, 20, 25]
Console.WriteLine(ints[..^2]); // [5, 10, 15]

Console.WriteLine(Object.ReferenceEquals(ints[1..], ints.Tail)); // True !!
```

Because now type `LList<T>` has a `Length` property, we can specify indexes "from the end" using the `^` operator. Thanks to the `Slice` method, the range syntax with `..` can be used. The `SliceSafe` was implemented in a particular way. If the passed range arguments include all items up to the end of the list, the original list stored in the `Tail` property is returned. Now we have all the pieces necessary to support pattern matching working with collection data types.

```csharp
static class LList
{
    public static int CountL<T>(this LList<T> list)
        => list switch
	    {
	        [] => 0,
	        [_, .. var tail] => 1 + CountL(tail),
	    };
}
```

This code looks like a canonical implementation of a recursive function calculating the list length presented in the first part of the series. The best thing about this pattern matching is that it's exhaustive and works efficiently. If we remove any of those cases, the code will not compile. Extracting a new variable `var tail` of type `LList<T>` is like calling `list.Slice(1)` and that executes `list.Tail` so there is no copying of memory at all. Such an efficiency was possible only because the rest operator `[ , .. var tail]` was the last in the list pattern. That function is equivalent to the following code. 

```csharp
static class LList
{
    public static int CountL<T>(this LList<T> list) =>
	    list.Length == 0 ? 0 : 1 + CountL(list.Tail);
}
```

In the next part, we will discuss two approaches to LINQ support.