---
layout: post
title: Immutable list using modern C# part 5 - optimization [C#]
date: 2023-11-14
tags: 
series: immutable-list-csharp
---
{%- include /_posts/series-toc.md series-name="immutable-list-csharp" -%}

## LList

Let's look at the implementation of the function `ToLList`, which converts a sequence of values into an immutable list.

```csharp
public static class LList
{
    private static LList<T> FromSeqUsingRecursion<T>(this IEnumerable<T> items)
    {
        using var iterator = items.GetEnumerator();
        return Next(iterator);
        
        static LList<T> Next(IEnumerator<T> iter) =>
	        iter.MoveNext() ? new(iter.Current, Next(iter)) : LList<T>.Empty;
    }
}
```


The main issue is the list's immutability. The list consists of simple nodes. Each node has a property `Head` and a reference to the following list stored in the `Tail` property. Neither of them can be changed after node creation. If we want to create the first node in the list, we must first process all nodes after the first element. Source sequence implements `IEnumerable<T>`, so we don't know the length and can only move forward one item at a time.

For very long sequences, we can encounter the stack overflow problem. We call the `Next` helper function recursively for each element in the sequence, and we have to wait for the result. Only then can we create the next node in the list. We could implement this function differently to avoid the potential stack overflow problem.

```csharp
public static class LList
{
    private static LList<T> FromSeqUsingListReversion<T>(IEnumerable<T> items)
    {
        return ToReversedList(ToReversedList(items));

        static LList<T> ToReversedList(IEnumerable<T> xs) => 
			xs.Aggregate(LList<T>.Empty, (list, item) => new(item, list));
    }
}
```

But this time, we had to iterate over all items twice. We walked through the source `IEnumerable<T>` sequence to build an immutable list. But the list was in the reverse order and had to be reversed again. 

We can use one trick to avoid this problem. Our list can be immutable for the outside world, but mutable for internal code. We could change the values of some private fields after the node object is created. Property `Tail` has only a getter and points to the next element on the list. So it's immutable for the outside world. But we can change the value of the `tail` private backing field.

The algorithm of building a list from a sequence could work as follows:
- Take the first value from the sequence and create the first node of the list. `Head` property will be set to that value, `Tail` property will be set to `null`
- Read the second value from the sequence, create the second node holding that value, and set the private field `tail` of the first node to the second node.
- Do the same for all values from the sequence.

This way, after scanning all values once, the whole would be created. Unfortunately, we have one serious problem. We wanted constant access to the list's length. Each node has `Length` property. We don't know the number of items during the value scanning phase. Again, in the worst case, we would have to iterate all items twice. The first time to create a linked list and count the number of items, and the second time to set the `Length` for each node in the list. After a while, I have found a solution. It is not the prettiest, but it works.

```csharp
public record LList<T>
{
    private readonly LengthValue lengthValue;
    private readonly T head;
    private LList<T> tail; // mutable

    public int Length => lengthValue.Value;
    public T Head => EnsureNonEmpty(head);
    public LList<T> Tail => EnsureNonEmpty(tail);

    public LList(T head, LList<T> tail) =>
	    (this.head, this.tail, lengthValue) = (head, tail, new LengthValue(tail.Length + 1));
    
    public LList() =>
	    (head, tail, lengthValue) = (default!, default!, new LengthValue(0));

    private LList(T head, LList<T> tail, LengthValue lengthValue) =>
	    (this.head, this.tail, this.lengthValue) = (head, tail, lengthValue);

    private R EnsureNonEmpty<R>(R result) => 
	    IsEmpty ? throw new InvalidOperationException("List is empty") : result;


    internal static LList<T> ToLList(IEnumerable<T> items)
    {
        using var iterator = items.GetEnumerator();

        if (!iterator.MoveNext())
        {
            return Empty;
        }

        var valueRef = new ValueRef<int>();
        var i = 0;
        var first = new LList<T>(iterator.Current, Empty, new LengthValue(i, valueRef));
        var last = first;

        while (iterator.MoveNext())
        {
            last = last.tail = new LList<T>(iterator.Current, Empty, new LengthValue(++i, valueRef));
        }

        valueRef.Value = i + 1;

        return first;
    }

    private record class ValueRef<V>
    {
        public V? Value;
    }

    private class LengthValue
    {
        private int indexOrLength;
        private ValueRef<int>? lengthRef;

        public int Value =>
	        lengthRef == null ? indexOrLength : lengthRef.Value - indexOrLength;

        public LengthValue(int length) => indexOrLength = length;

        public LengthValue(int index, ValueRef<int> lengthRef) =>
	        (indexOrLength, this.lengthRef) = (index, lengthRef);

        public override bool Equals(object? obj) =>
	        obj is LengthValue l ? Value.Equals(l.Value) : false;

        public override int GetHashCode() => Value.GetHashCode();
    }
}
```

The property `Length` is an integer from the outside world, but internally, it is stored as a private type `LengthValue`. It is like a wrapper for an integer value exposed as a `Value` property representing one of two scenarios.

When the list is created using a collection expression `[1,2,3]` or a helper method `LListFrom(1,2,3)`, the constructor `LengthValue(int length)` is used. The `Length` property reads the value of the `length` argument directly.

When the list is created from an `IEnumerable<T>` type using the internal method `ToLList`, the constructor `LengthValue(int index, ValueRef<int> lengthRef)` is used. The `index` argument represents the node's index in the list, counting from 0. The `lengthRef` points to the number of all nodes in the list wrapped into `ValueRef<int>` type. All nodes will have a reference to the same instance of `ValueRef<int>`; its public property `Value` will be set at the end of iteration when the number of elements is known. Once an immutable list is created, the length of that list is read by calling the `Length` property of the first node with index 0. The correct value is calculated by subtracting `lengthRef.Value - indexOrLength;` 

Interestingly, a similar solution has been used in [F#](https://github.com/dotnet/fsharp/blob/main/src/FSharp.Core/local.fs#L531) and [Scala](https://youtu.be/7mTNZeiIK7E?t=1439) immutable lists.

Complete source code can be downloaded from here, [minimal](https://github.com/marcinnajder/misc/blob/master/2021_04_18_make_a_lisp_in_csharp/Mal/Mal/PowerFP/LList.cs) or [full](https://github.com/marcinnajder/misc/blob/master/2023_10_29_algorithms_and_data_structures_in_csharp/AlgorithmsAndDataStructures/LList.cs) .




