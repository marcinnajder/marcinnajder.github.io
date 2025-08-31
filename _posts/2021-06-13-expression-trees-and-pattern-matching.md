---
layout: post
title: Expression trees and pattern matching [C#]
date: 2021-06-13
tags: 
series: expr-trees-pattern-matching
---
## Introduction

During my last .NET workshop, I received a question about expression trees. The participant had reported a magic bug in Jira that needed to be resolved. They used the [OData](https://www.odata.org/) protocol, and during one of the HTTP requests, a strange exception was thrown by the Entity Framework. The OData protocol is based on REST and provides the whole query language encoded directly in the URL address of the HTTP request. [Here](https://www.odata.org/documentation/odata-version-2-0/uri-conventions/) you can find sample queries. The following sample URL `https://services.odata.org/Northwind.svc/Products?$filter=Price gt 3 and Price le 200` expresses a query condition `... Where Price> 3 && Price <= 200`. .NET provides official libraries supporting the OData protocol. URL query is automatically translated to an expression tree representing a LINQ query and then sent to the configured LINQ provider, like Entity Framework. So there is a lot of magic involved in the process.

The error message said only something like `... query '(int p) => ((p.CompareTo(3) > 0) AndAlso (p.CompareTo(200) <= 0))' is not supported by Entity Framework`.  The problem was the translation from the URL into the expression tree. Logically, the query is correct. `p` is a price of type int that contains the method `CompareTo`, comparing two numbers. It returns a number greater than, less than, or equal to zero. Entity Framework probably didn't support such a condition, expecting something like this `(int p) => p > 3 && p <= 200`. To fix the problem, we must somehow transform an expression tree in flight. But first, let's introduce some basic information about expression trees in C#. 

## Expression trees

Expression trees were introduced in C#3.0 in 2009 as a part of LINQ. I often explain this C# feature by comparing it to Lego bricks. Let's say we have a box of Lego bricks in different sizes and colors. Using them, we can build any piece of C# code; it is like dynamic SQL. For example, the red block is a method parameter, a bigger one, blue, taking other blocks is a binary operator like `+`, and the other green is a lambda expression. This is how we could build the following lambda expression `(int a, int b) => a + b`

```csharp
ParameterExpression aParamExpr = Expression.Parameter(typeof(int), "a");
ParameterExpression bParamExpr = Expression.Parameter(typeof(int), "b");
BinaryExpression addExpr = Expression.Add(aParamExpr, bParamExpr);
Expression<Func<int, int, int>> lambdaExpr =
    Expression.Lambda<Func<int, int, int>>(addExpr, aParamExpr, bParamExpr);    
```

Expression trees provide a vast hierarchy of types like `ParameterExpression`, `BinaryExpression`, ... inheriting from the base type `Expression`. There are two primary purposes of an expression tree:
- We can read the expression tree, analyzing what's inside.
- We can compile such a tree into a regular delegate pointing to the method in memory.

Let's compile our expression tree using the `Compile` method and execute it as a regular method built from C# source code.

```csharp
Func<int, int, int> add = lambdaExpr.Compile();
Console.WriteLine("10+5=" + add(10, 5)); // 10+5=15
```

Analyzing expression trees is a fundamental mechanism behind LINQ providers. Every LINQ query is transformed into a chain of method calls.

```csharp
var q = from el in elements where el > 10 select el * 10;
// code above is translated into code below
var q = elements.Where(el => el > 10).Select(el => el * 10);
```

Then, in the second step, the C# compiler searches for the appropriate `Where` and `Select` methods. If the `elements` variable implements the `IEnumerable<T>` interface, the following extension methods will be used.

```csharp
public static class Enumerable
{
	public static IEnumerable<T> Where<T>(
		this IEnumerable<T> source, Func<T, bool> predicate) { ...}

	public static IEnumerable<R> Select<T, R>(
		this IEnumerable<T> source, Func<T, R> selector) { ... }	
}
```

If, however, the `elements` variable implements the `IQueryable<T>` interface, different extension methods will be used.

```csharp
public static class Queryable
{
	public static IQueryable<T> Where<T>(
		this IQueryable<T> source, Expression<Func<T, bool>> predicate) { ... }

	public static IQueryable<R> Select<T, R>(
		this IQueryable<T> source, Expression<Func<T, R>> selector) { ... }
}
```

Can you see the differences? The second case, `IQueryable` is used instead of `IEnumerable`, and `Expression<Func<T, bool>>` instead of `Func<T, bool>`. 

This is the magic behind the LINQ provider. A LINQ provider is a library responsible for analysing the expression tree of the LINQ query and performing some operations. Entity Framework is a LINQ provider that generates and executes SQL. 

There is one more important aspect of expression trees. Let's say that the variable `elements` used in the LINQ query above is of type `IEnumerable<T>`. We could add an extension method `AsQueryable` in the middle of the query, which takes `IEnumerable<T>` and returns `IQueryable<T>`.

```csharp
var q = elements.AsQueryable().Where(el => el > 10).Select(el => el * 10);
```

Now, `Queryable.Where` method is used instead of `Enumerable`. We can still pass a lambda expression where `Expression<...>` type is expected, and the code compiles correctly.

Let's take a look at another example using our `add` method.

```csharp
Func<int, int, int> addDelegate = (a, b) => a + b;

Expression<Func<int, int, int>> addExpression = (a, b) => a + b;

// the C# compiler generates the code below

// ParameterExpression aParamExpr = Expression.Parameter(typeof(int), "a");
// ParameterExpression bParamExpr = Expression.Parameter(typeof(int), "b");
// BinaryExpression addExpr = Expression.Add(aParamExpr, bParamExpr);
// Expression<Func<int, int, int>> addExpression =
//    Expression.Lambda<Func<int, int, int>>(addExpr, aParamExpr, bParamExpr); 
```

## Visitor pattern

Once we know how expression trees work, we can fix the reported bug. We must change a condition like `p.CompareTo(3) > 0` into `p > 3`. But there is one problem. Expression trees are immutable; they cannot be changed after creation. We can build a new expression tree reusing existing parts. Fortunately, .NET provides a built-in mechanism for analyzing or modifying expression trees. It is a class called `ExpressionVisitor` that implements the visitor design pattern. We define our class that inherits from `ExpressionVisitor` and overrides selected base methods. In our case, code like `... > ...` is represented as a `BinaryExpression`. 

```csharp
using System.Linq.Expressions;
using static System.Linq.Expressions.ExpressionType; // using static enum

class MyVisitor : ExpressionVisitor
{
    protected override Expression VisitBinary(BinaryExpression node)
    {
        var type = node.NodeType;
        if (type == GreaterThan || type == GreaterThanOrEqual || type == LessThan || type == LessThanOrEqual)
        {
            var call = node.Left as MethodCallExpression;
            if (call != null)
            {
                if (call.Method.Name == "CompareTo")
                {
                    return Expression.MakeBinary(node.NodeType, call.Object!, call.Arguments[0]);
                }
            }
        }

        return base.VisitBinary(node);
    }
}
```

The typical code working with expression trees is very special. First, we check the subexpression's actual type, then we cast it to a specific class, and only then can we read properties. Now, let's test whether our visitor works correctly. 

```csharp
Expression<Func<int, bool>> expr = p => p.CompareTo(3) > 0 && p.CompareTo(200) <= 0;

var vistor = new MyVisitor();
expr = (Expression<Func<int, bool>>)vistor.Visit(expr2);

Console.WriteLine(expr); // p => ((p > 3) AndAlso (p <= 200))
```

## Pattern matching

Pattern matching is a language feature primarily used in functional languages. Initial support was added in C#7.0 (2017), and then new capabilities were extended every year. Let's rewrite our visitor class to use pattern matching.

```csharp
class MyVisitor : ExpressionVisitor
{
    protected override Expression VisitBinary(BinaryExpression node)
    {
        if (node is
            {
                NodeType: GreaterThan or GreaterThanOrEqual or LessThan or LessThanOrEqual,
                Left: MethodCallExpression { Method: { Name: "CompareTo" }, Object: { } obj, Arguments: [var arg] }
            })
        {
            return Expression.MakeBinary(node.NodeType, obj, arg);
        }

        return base.VisitBinary(node);
    }
}
```

There are two main goals of pattern marching:
- It works like a predicate checking whether a specific variable has a defined shape.
- Optionally, if the predicate is `true`, we can extract new data from the expression.
 
That code is much cleaner and easier to understand. There is no type checking and casting, and the expression is very declarative. New variables, `obj` and `arg`, are assigned to correct data and can be used only inside an if statement.

## Summary

There have been many discussions about the direction of the C# language. The language is big, and any other new feature added must be justifiable. Some developers use pattern matching writing `p is null` instead of `p == null`, or `p is { Name: "marcin" }` instead of `p != null && p.Name == "marcin"`. That does not bother me, but these are not the main scenarios why pattern matching was introduced. I hope I showed you an example where pattern matching really makes a difference. 