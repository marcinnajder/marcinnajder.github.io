---
layout: post
title: Immutable list using modern C# part 2 -  AoC puzzle [C#]
date: 2022-12-01
tags: 
series: immutable-list-csharp
---
{%- include /_posts/series-toc.md series-name="immutable-list-csharp" -%}

## Advent Of Code 2022 Day 1

In this article, we will implement the whole working solution for [Advent Of Code 2022 Day 1](https://adventofcode.com/2022/day/1) puzzle using a custom implementation of an immutable linked list, `LList<T>`. Our program takes a list of numbers divided into groups using blank lines.

```
1000
2000
3000

4000

5000
6000

7000
8000
9000

10000
```

In the first part of the puzzle we have to sum up numbers inside groups to find the biggest sum. For our sample data the correct answer is `24000` (`7000+8000+9000`). In the second part, we must find the sum of the top three groups. This time the correct answer is `45000` (`24000+11000+1000`).

```csharp
static class Day1
{
    public static LList<LList<int>?>? LoadData(string input)
    {
        return input.Split(Environment.NewLine)
            .Concat(new[] { "" })
            .Aggregate(
                ((LList<LList<int>?>? Lists, LList<int>? List))(null, null),
                (s, line) =>
                    line switch
                    {
                        "" => (new(s.List, s.Lists), null),
                        _ => (s.Lists, new(int.Parse(line), s.List))
                    })
            .Item1;
    }

    static LList<T>? InsertSorted<T>(LList<T>? xs, T x) where T : IComparable<T>
        => xs switch
        {
            null => new(x, null),
            (var Head, var Tail) => x.CompareTo(Head) < 0 ? new(x, xs) : new(Head, InsertSorted(Tail, x))
        };

    static LList<T>? InsertSortedPreservingLength<T>(LList<T>? xs, T x) where T : IComparable<T>
        => xs switch
        {
            null => null,
            (var Head, var Tail) => x.CompareTo(Head) < 0 ? xs : InsertSorted(xs, x)!.Tail
        };

    public static string Puzzle(string input, int topN)
        => LoadData(input)
            .ToEnumerable()
            .Select(l => l.ToEnumerable().Sum())
            .Aggregate(Repeat(0, topN).ToLList(), InsertSortedPreservingLength)
            .ToEnumerable()
            .Sum()
            .ToString();

    public static string Puzzle1(string input) => Puzzle(input, 1);

    public static string Puzzle2(string input) => Puzzle(input, 3);
}
```

The `LoadData` function takes a text representation of input and parses it, returning lists of numbers like `((1000 2000 3000) (4000) (5000 6000) ...)`. Both parts of the puzzle were implemented using the same common function, `Puzzle,` parameterized with a `topN` argument. For the first part of the puzzle, the value was `1`, and for the second, the value was `3`.

We need to find the biggest `n` values in the collection, so the natural solution would be sorting and then taking the first `n` elements. But sorting is quite expensive, and frankly speaking, we don't need to store all elements at all. We need to hold the top `n` numbers throughout the whole computation. It's worth mentioning that a sample text file contains only 14 lines, but the final text file contains more than 2000 lines.

An immutable linked list is the best data structure for the job. Function `InsertSorted` returns a new list with an element `x` inserted into the appropriate location so that the returned list remains sorted. Let's say we have a list of `5, 10, 15` and want to add a new value `2`. We check the head of the list, which is `5`, and because the inserted value is smaller than the current smallest, it becomes a new head pointing to an existing list. Inserting a new element at the beginning is a highly cheap operation. Inserting a new element in the middle of the list rewrites previous elements and leaves further elements unchanged. Function `InsertSortedPreservingLength` executes `InsertSorted` internally, preserving the length of the list. Inserting `8` into `5, 10, 15` returns `8, 10, 15`.
