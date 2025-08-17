---
layout: post
title: Immutable list using modern C# part 1 - minimal [C#]
date: 2021-10-10
tags: 
series: immutable-list-csharp
---
{%- include /_posts/series-toc.md series-name="immutable-list-csharp" -%}

## LList

An immutable linked list is a fundamental data structure in all functional programming languages. List implementation is straightforward yet very powerful. Over the years, non-functional languages like C#, Java, and others have copied many features from functional languages, for instance: lambdas, tuples, records, pattern matching, immutability, expression-based style, and so on. Many developers are not aware of how powerful an immutable list is. This article will present the main features of that data structure by implementing it from scratch using modern C#.

Let's start by defining an immutable list in the simplest possible way using a C# positional record.

```csharp
public record LList<T>(T Head, LList<T>? Tail);


var list1 = new LList<int>(1, new LList<int>(2, new LList<int>(3, null)));
LList<int> list2 = new(1, new(2, new(3, null)));

Console.WriteLine(list1.Head); // 1

Console.WriteLine(list1.Tail!.ToString());
// LList { Head = 2, Tail = LList { Head = 3, Tail =  } }

Console.WriteLine(list1 == list2); // True
```

A record in C# is just a class with useful functionalities presented in detail later. Each node of a singly linked list stores some value in the `Head` property and the reference to the next node in the `Tail` property. Those properties have only getters, so the node object and the whole list are immutable. They can not be changed after creation. We can add a new item to the beginning of the list by creating a new node with `Tail` property set to an existing list. We can remove the first element just by returning `Tail` property of the first node. An immutable list is called a "persistent data structure" because each "modification" creates a new instance of the list, where the structure in memory is shared.

Records in C# give us some methods that the compiler generates for free. The `GetHashCode` and `Equals` methods use all fields defined in the record type. In our case, those fields are backing fields for properties `Head` and `Tail`, so we can compare two lists by value.

The `ToString` method returns values of all properties. It's not the perfect representation of a list, but it's still better than the default implementation of `ToString`. We will provide our implementation of `ToString` later.

C# 9.0 (2020) introduced records and a new way of creating object instances, "target-typed new expressions". We can omit the name of the type in places where the compiler can infer it from the context, like here `LList<int> list2 = new(1, new(2, new(3, null)))`. We will implement some helper functions later to simplify the list creation even more. I did one strange trick: a `null` value represents an empty list. Once again, our first implementation is supposed to be as simple as possible; we will improve it later.

In functional programming, we separate the definition of data types from behavior. Since our record represents data, let's implement the first few practical methods. 

```csharp
public static class LListM
{
    public static IEnumerable<T> ToEnumerable<T>(this LList<T>? llist)
    {
        if (llist == null)
        {
            yield break;
        }

        var node = llist;
        while (node != null)
        {
            yield return node.Head;
            node = node.Tail;
        }
    }
    
    public static LList<T>? ToLList<T>(this IEnumerable<T> llist)
    {
        return NextValue(llist.GetEnumerator());
        
        static LList<T>? NextValue(IEnumerator<T> e) =>
	        e.MoveNext() ? new(e.Current, NextValue(e)) : null;
    }

    public static LList<T>? LListFrom<T>(params T[] items) =>
	    items.Length == 0 ? null : items.ToLList();    
}
```

I created a new static class, `LListM`, from the "linked list module." All methods will be static, and some will be defined as extension methods where it makes sense. The `ToEnumerable` method converts `IEnumerable<T>` into a list `LList<T>`, and the function `ToLList` does the opposite operation. So, the following expression `new []{1,2,3}.ToLList()` is another way to create our immutable linked list. 

The `LListFrom` method is a variadic function; it can take a variable number of arguments. Now, instead of writing `new(1, new(2, new(3, null)))`, we can execute `LListM.LListFrom(1,2,3)` methods. Unfortunately, for an empty list, an explicit generic type argument is necessary, `LListM.LListFrom<int>()`. `LListFrom` is like a static factory method. Because it's not an extension method (but still static), it could be imported in a new way using C# 6.0 (2015) "static imports". This feature allows us to simulate importing functions from modules in functional languages. One line of code `using static LList;` must be placed at the top of the C# file, then the class name can be omitted once its members are used, like `LListFrom(1,2,3)`. 

`List <T>` type is defined as a recursive data structure; in such cases, often code manipulating them is also recursive. Alternative recursive implementation of `ToEnumerable` could look like this:

```csharp
public static class LListM
{
    public static IEnumerable<T> ToEnumerableRec<T>(this LList<T>? llist)
    {
        if (llist == null)
        {
            yield break;
        }

        yield return llist.Head;

        foreach (var item in ToEnumerableRec(llist.Tail))
        {
            yield return item;
        }
    }
}
```

To this day, I still remember the feeling of implementing my own immutable linked list in OCaml during one of my studies courses. The whole idea was to build memory muscle for recursive code. Let's implement the simplest function counting the length of the list.

```csharp
public static class LListM
{
    public static int Count<T>(this LList<T>? llist) =>
        llist switch
        {
            null => 0,
            (_, var Tail) => 1 + Count(Tail)
        };
}
```

The pattern-matching switch expression introduced in C# 8.0 (2019) allows the function to be implemented as a single expression. There are two cases. When the list is empty, stop the recursion, returning 0. Otherwise, add one to the length of the tail. Let's implement two legendary LINQ operators, `Where` and `Select`. 


```csharp
public static class LListM
{
    public static LList<R>? Select<T, R>(this LList<T>? llist, Func<T, R> f) =>
        llist switch
        {
            null => null,
            (var Head, var Tail) => new(f(Head), Select(Tail, f))
        };

    public static LList<T>? Where<T>(this LList<T>? llist, Func<T, bool> f) =>
        llist switch
        {
            null => null,
            (var Head, var Tail) =>
	            f(Head) ? llist with { Tail = Where(Tail, f) } : Where(Tail, f)
        };
}
```

The general rule is the same: we need at least two branches in recursive code, one of which stops the recursion. Now we can write a beautiful LINQ query like this:

```csharp
var ints = LListFrom(5, 10, 15, 20, 25);
var q =
    from i in ints
    where i % 10 == 0
    select $"{i:.00}";

Console.WriteLine(q);
// LList { Head = 10, Tail = LList { Head = 20,00, Tail =  } }
```

LINQ was introduced in C# 3.0 (2008), and the goal was to introduce key features from functional languages to object-oriented languages like C#. The other nice features were added too, like lambda expressions, anonymous types, collections, and object initializers. The whole set of LINQ operators based on lazy sequences ( `IEnumerable<T>` interface) changed forever how C# code is written. It all comes from functional ancestors like Lisp, ML, or Haskell.  An immutable list is the most fundamental data structure in all those languages.

As homework, please implement the following LINQ operators: `Aggregate, Take, Skip, Concat, Zip, SelectMany, Reverse, All, Any, ElementAt, SequenceEqual`.

```csharp
public static class LListM
{
    public static A Aggregate<T, A>(this LList<T>? llist, A seed, Func<A, T, A> f) =>
        llist switch
        {
            null => seed,
            (var Head, var Tail) => Aggregate(Tail, f(seed, Head), f)
        };

    public static T Aggregate<T>(this LList<T>? llist, Func<T, T, T> f) =>
        llist switch
        {
            null => throw new Exception("List contains no elements"),
            { Tail: null } => llist.Head,
            _ => Aggregate(llist.Tail, llist.Head, f)
        };

    public static LList<T>? Take<T>(this LList<T>? llist, int count) =>
        llist switch
        {
            null => null,
            (var Head, var Tail) => count <= 0 ? null : new(Head, Take(Tail, count - 1))
        };

    public static LList<T>? Skip<T>(this LList<T>? llist, int count) =>
        llist switch
        {
            null => null,
            (var Head, var Tail) => count <= 0 ? llist : Skip(Tail, count - 1)
        };

    public static LList<T>? Concat<T>(this LList<T>? llist1, LList<T>? llist2) =>
        (llist1, llist2) switch
        {
            (null, _) => llist2,
            ((var Head, var Tail), _) => new(Head, Concat(Tail, llist2))
        };

    public static LList<R>? Zip<T1, T2, R>(this LList<T1>? llist1, LList<T2>? llist2, Func<T1, T2, R> f) =>
        (llist1, llist2) switch
        {
            (null, _) or (_, null) => null,
            ((var Head1, var Tail1), (var Head2, var Tail2)) => new(f(Head1, Head2), Zip(Tail1, Tail2, f))
        };

    public static LList<TT>? SelectMany<T, TT>(this LList<T>? llist, Func<T, LList<TT>?> f) =>
        llist switch
        {
            null => null,
            (var Head, var Tail) => f(Head).Concat(SelectMany(Tail, f))
        };

    public static LList<R>? SelectMany<T, TT, R>(this LList<T>? llist, Func<T, LList<TT>?> f, Func<T, TT, R> r) =>
        llist switch
        {
            null => null,
            (var Head, var Tail) => f(Head).Select(tt => r(Head, tt)).Concat(SelectMany(Tail, f, r))
        };

    public static LList<T>? Reverse<T>(this LList<T>? llist)
    {
        return Reverse2(llist, null);

        static LList<T>? Reverse2(LList<T>? llist, LList<T>? result) =>
            llist switch
            {
                null => result,
                (var Head, var Tail) => Reverse2(Tail, new(Head, result)),
            };
    }

    public static bool All<T>(this LList<T>? llist, Func<T, bool> f) =>
        llist switch
        {
            null => true,
            (var Head, var Tail) => f(Head) && All(Tail, f)
        };

    public static bool Any<T>(this LList<T>? llist, Func<T, bool> f) =>
        llist switch
        {
            null => false,
            (var Head, var Tail) => f(Head) || Any(Tail, f)
        };

    public static T ElementAt<T>(this LList<T>? llist, int index) =>
        (llist, index) switch
        {
            (_, < 0) => throw new Exception("Index cannot be less than zero"),
            (null, _) => throw new Exception("Index out of bounds"),
            ((var Head, _), 0) => Head,
            ((var Head, var Tail), _) => ElementAt(Tail, index - 1)
        };

    public static bool SequenceEqual<T>(this LList<T>? llist1, LList<T>? llist2, Func<T, T, bool> equals) =>
        (llist1, llist2) switch
        {
            (null, null) => true,
            (null, _) or (_, null) => false,
            ((var Head1, var Tail1), (var Head2, var Tail2)) => equals(Head1, Head2) && SequenceEqual(Tail1, Tail2, equals)
        };
}

```

Complete source code can be downloaded from [here](https://github.com/marcinnajder/misc/blob/master/2021_04_18_make_a_lisp_in_csharp/Mal/Mal/PowerFP/LList.cs)