---
layout: post
title: RxSandbox [C#]
date: 2009-11-26
tags:
  - csharp
series: rxsandbox
---
Rx is awesome :)

I have just started a new project called RxSandbox. It will be a very simple application that helps to understand how Rx works, how each operator is implemented. But let's start from the beginning. A few days ago, I was playing with Rx operators such as Zip, Merge, Wait, etc., writing a very short chunk of code for each operator. I wondered how they work in detail, if they cache the values from sources, when exactly is OnCompleted executed on the observers, and so on. The code could look like this:

```csharp
a.Zip(b, (x, y) => x + " - " + y)
```

I wanted `a` and `b` to implement as `IObservable<string>`. But what is the easiest way to create the implementation for them? I wanted to be focused on writing code using operators, so I needed some infrastructure for creating sample observable data sources. The RxSandbox is the solution! When you write something like this:

```csharp
Expression<Func<ManualObservable<string>, ManualObservable<string>,
    IObservable<string>>> zipExpression
        = (a, b) => a.Zip(b, (x, y) => x + " - " + y);
        
Control control = RxExpressionVisualizer.CreateControl(zipExpression);
```

RxSandbox will automatically generate a testing UI control:

![rxsandbox](/assets/images/rxsandbox.png)

In the code above, the `ManualObservable` means the user can manually define the values sent to the expression by writing them in a text box. Other possibilities could be random observation, interval observation, and many more. The UI control will be different for each type of Observable source. The only reason the `Expression<Func<...>>` type has been used here is that this allows us to display the body of the expression, but of course this is just an option. Generally, the test case is a method of taking some Observable sources and returning an observable collection. It can be some complicated code snippet with many statements as well.

Possible scenarios working with RxSanbox:

- Run RxSandbox application, create a new project in VS, write Rx expression, compile it, RxSandbox automatically finds a new version of dll file and loads it, you are ready to test it (it may be [MEF](http://msdn.microsoft.com/en-us/library/dd460648\(VS.100\).aspx), VS AddIn not just a standalone application)
- Run RxSandbox, write your Rx expression inside RxSandbox, start testing it (the easy way to do this is to use [ExpressionTextBox](http://msdn.microsoft.com/en-us/library/system.activities.design.view.expressiontextbox\(VS.100\).aspx) control from WF4.0, which allows us to write a single VB expression and expose it as an LINQ expression type)
- Drawing marble diagrams live during testing and also presenting the definitions of the operators as marble diagrams (of course with pac-mans and hearts :) - )
- Supporting Silverlight version
- Many, many more....

I hope you got the idea. [Here](http://code.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=mnajder&DownloadId=8410) you can find the prototype.