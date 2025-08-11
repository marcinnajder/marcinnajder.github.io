---
layout: post
title: Expression tree instead of strings part 1 - ReflectionHelper [C#]
date: 2008-11-08
tags:
  - csharp
series: expression-trees
---
{%- include /_posts/series-toc.md series-name="expression-trees" -%}

Reflection is a handy mechanism that lets us build ingenious and flexible solutions. It is used in many places in the .NET framework, for instance, in Windows Forms to identify bound members, in Workflow Foundation to pass some parameters to a workflow instance, and many others. Those two samples have one common thing; we must use string literals in code to identify type members such as properties. And of course it's not a problem if the types we use are not changing, it means once specified, the member won't be changed in the future. Like which shouldn't change, like .NET Framework types (it shouldn't be changed cause of backward compatibility).

Let's imagine that we bind one property of our business object to a TextBox's property Text, and one day during refactoring, someone else changes the name or the type of the property, or even worse - deletes it. The code will compile correctly, but we can't be sure how it will work at runtime because it depends on the sort of change. Maybe some exception will be thrown, everything will look fine, but the property won't be set after modifying text in the TextBox, or perhaps all will work fine. Such bugs are very unpredictable and very hard to find. It would be nice if we could know after the change about possible problems (compilation failed). Sometimes we know that the property is used in many places in the code, so we rename it via our IDE (for instance, in Visual Studio, we can choose the option 'Refactor -> Rename...'), and we want to be sure that everything works fine. Is this possible?

Today, I'll show you a utility class called ReflectionHelper, which allows us to realize the mentioned scenario. Currently, when we extract information about a property via reflection, we write something like this:

```csharp
public class SomeClass
{
    public int InstanceProperty { get; set; }
    public static int StaticProperty { get; set; }
}

PropertyInfo property1 = typeof(SomeClass).GetProperty("InstanceProperty");
PropertyInfo property2 = typeof(SomeClass).GetProperty("StaticProperty");
```

With the `ReflectionHelper` the same code can be written like this:

```csharp
PropertyInfo property3 = ReflectionHelper.GetProperty((SomeClass o) => o.InstanceProperty );            
PropertyInfo property4 = ReflectionHelper.GetProperty(() => SomeClass.StaticProperty);
```

This is precisely what we wanted to achieve. We created an instance of the `PropertyInfo` class without using any string literals. Now, when we manually change the name of a property in one place, the code won't compile. If you rename it using the 'Rename...' option in VS, the code inside the lambda expression will be changed, too.

Now, let's see how `ReflectionHelper` works. `ReflectionHelper` is a static class with two methods that extract information about a given property. One for static properties and one for instance. The implementation is relatively short and looks like this:

```csharp
public static PropertyInfo GetProperty<TObject,T>(Expression<Func<TObject,T>> p) 
{ 
    return GetPropertyImpl(p); 
}
public static PropertyInfo GetProperty<T>(Expression<Func<T>> p) 
{ 
    return GetPropertyImpl(p); 
}
private static PropertyInfo GetPropertyImpl(LambdaExpression p)
{
    return (PropertyInfo)((MemberExpression)(p.Body)).Member;
}
```

The key point to understand how the code works is a new feature of C#3.0 called expression trees. In the latest version of C#, we can use a lambda expression in code. There are two kinds of lambda expression, where the body of the method is a single expression (`x => x.ToString()`) or one or many statements (`x => {return x.ToString(); }`).

```csharp
Func<int, int> expressionBody = (a) => a + 1;
Func<int, int> statementBody = (a) => { return a + 1; };
```

In many places, we use lambda with a single expression body just because it's shorter, but there is a scenario when we must use it. Generally, when a compiler sees a lambda expression in code, it's treated as an anonymous method, but a lambda expression can also be used to build expression trees. Let's look at the example:

```csharp
Expression<Func<int, int>> exp1 = (a) => a + 1;
Expression<Func<int, int>> exp2 = (a) => { return a + 1; }; 
// Error: A lambda expression with a statement body cannot be converted to an expression tree
```

Ok, but what is an expression tree? When the compiler sees a lambda with an expression body not in the context of a delegate type but in the context of a special generic type `System.Linq.Expressions.Expression<TDelegate>`, a lot of code is generated behind the scenes.

```csharp
ParameterExpression p;
Expression<Func<int, int>> exp1 = Expression.Lambda<Func<int, int>>
(
    Expression.Add // body
    (
        p = Expression.Parameter(typeof(int), "a"),
        Expression.Constant(1, typeof(int))
    ),
    new ParameterExpression[] { p } // parameters
);
```

The code of a lambda expression is interpreted and translated into code that builds its structure. Use '[Expreesion Tree Visualizer](http://www.talentgrouplabs.com/blog/archive/2007/11/27/do-not-miss-the-expression-tree-visualizer.aspx)' to better visualize the structure of expression tree.

When we look back at the implementation of the `GetProperty` method, we can see that it takes an expression tree as a parameter. Inside the method, we make some assumptions about the structure of the tree so we know exactly where information about the property is stored. Of course, if the tree's structure were different, some exception could be thrown. The same technique can extract information about fields, methods, and constructors (`ReflectionHelper` has appropriate functionality). There are different ways of getting information about the methods.

```csharp
public class SomeClass
{
    public void InstanceMethod(int i) { }
}
```

Instead of writing a literal call

```csharp
var method = typeof(SomeClass).GetMethod("InstanceMethod");
```

use ReflectionHelper like this:

```csharp
var method = ReflectionHelper.GetMethod<SomeClass,int>(o => o.InstanceMethod);
```

or like this:

```csharp
var method = ReflectionHelper.GetMethodByCall<SomeClass>(o => o.InstanceMethod(1));
```

The implementation looks like this:

```csharp
public class ReflectionHelper
{    
    public static MethodInfo GetMethodByCall<TObject>(Expression<Action<TObject>> expression) 
    { 
        return GetMethodByCallImpl(expression); 
    }
    public static MethodInfo GetMethodByCall(Expression<Action> expression) // for static methods 
    { 
        return GetMethodByCallImpl(expression); 
    }
    private static MethodInfo GetMethodByCallImpl(LambdaExpression expression)
    {
        return ((MethodCallExpression)expression.Body).Method;
    }    
    
    public static MethodInfo GetMethod<TObject, T>(Expression<Func<TObject, Action<T>>> expression) 
    { 
        return GetMethodImpl(expression); 
    }
    public static MethodInfo GetMethod<T>(Expression<Func<Action<T>>> expression) // for static methods 
    { 
        return GetMethodImpl(expression); 
    }
    private static MethodInfo GetMethodImpl(LambdaExpression expression)
    {
        return (MethodInfo)((ConstantExpression)((MethodCallExpression)((UnaryExpression)expression.
            Body).Operand).Arguments.Last()).Value;
    }
}
```

When we use the GetMethod method, we don't need to write method parameters inside brackets. Still, we need to provide their types as generic method arguments (if parameters exist or the method returns something). In the second approach, we write code that executes the method inside a lambda expression. The code is never executed, so we don't have to pass any correct arguments. These two approaches have one common disadvantage. When we change the method's signature by changing parameters or return type, we need to change the code reflecting that method, too. Maybe there is some solution, but I couldn't find it :(

Full implementation of `ReflectionHelper` class with code presenting many different scenarios of using it can be found [here](http://code.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=mnajder&DownloadId=3716). In the next post, I'll show you how the `ReflectionHelper` is used in real life :) Stay tuned!