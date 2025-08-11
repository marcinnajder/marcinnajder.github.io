---
layout: post
title: RxSandbox V1 [C#]
date: 2010-03-08
tags:
  - csharp
series: rxsandbox
---
- [[New version](http://mnajder.blogspot.com/2011/05/rx-projects-update.html) (2011.05.06)]
- RxSandbox download has been upgraded to the newest version of Rx (Build 1.0.2677.0)08/27/2010 [download](http://code.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=mnajder&DownloadId=13718)
- RxSandbox download has been upgraded to the newest version of Rx (Build 1.0.2617.0 07/15/2010). [download](http://code.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=mnajder&DownloadId=13676)  
	-(Changes: solution converted to VS2010, Net 4.0; 56 operators, grouping operators on the tree control; zoom)
- RxSandbox download has been upgraded to the newest version of Rx (Build 1.0.2441.0 04/14/2010). [download](http://code.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=mnajder&DownloadId=10232)
- RxSandbox download has been upgraded to the newest version of Rx (Build 1.0.2350.0 03/15/2010.] [download](http://code.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=mnajder&DownloadId=9324)

I am really happy to announce that the new version of RxSandbox has just been released. Last time , I wrote about the concept and the main ideas behind the project. That was just a small prototype, the proof of concept. This release is much more mature. The user can test more standard Rx operators, and the API has been changed a little bit, but the concept remains the same.

These are the main new features:
- Marble diagrams
- New powerful API
- Extensibility mechanism
- Description of Rx operators taken from Rx documentation

Let's start from the end-user, which is not necessarily interested in writing any code. He wants to experiment with Rx operators and check their behavior in specific scenarios. When we start the RxSandbox application, we will see the list of standard operators on the left-hand side. Let's assume that we don’t know how the Merge operator works. After double-clicking on the Merge node, the new tab will be displayed. We can find the short description of the Merge operator taken from the documentation provided by the Rx installer, and we can also find the code sample that can be tested interactively through the UI. Each input argument is presented as a simple automatically generated UI with one textbox where we can write the input value, three buttons, and the history of the source. The output of the expression can be presented in two ways: by a simple list displaying the results (“Output” tab) or the marble diagrams drawn live during testing (‘Marble diagram – Live” tab). There is also one tab called “Marble diagram - Definition” showing the operator definition.

![rxsandboxv1](/assets/images/rxsandboxv1.png)

Now, let's see what the developer can do with RxSandbox. The most important class in Rx API is the ExpressionDefinition. It holds all necessary information describing a tested expression, such as name, description, and sample code.

![rxsandboxdiagram](/assets/images/rxsandboxdiagram.png)

When we look inside RxSandbox project, we will see that the definitions of all standard operators are very similar, for example, the Merge operator is defined like this:

```csharp
public static ExpressionDefinition Merge()
{
    Expression<Func<IObservable<string>, IObservable<string>, IObservable<string>,
        IObservable<string>>> expression
            = (a, b, c) => Observable.Merge(a, b, c);
    
    return ExpressionDefinition.Create(expression);
}
```

This is all we need to write. Other things like the operator’s name, description, and the text of the expression can be inferred from the LINQ expression. Of course, all this information can be set manually using the appropriate Create method overload and the ExpressionSettings class. All .NET types are supported as observable types, not only the System.String type like in this example. The only requirement is there must exist a [TypeConverter](http://msdn.microsoft.com/en-us/library/system.componentmodel.typeconverter\(VS.71\).aspx) for that type. Later in this post, I’ll show how to implement TypeConverter for custom type and how to create ExpressionDefinition without using LINQ expression but using the whole method with many statements. The second very important class in RxSandbox API is an ExpressionInstance class which is very useful in scenarios where we want to use some RxSandbox functionalities directly from code without any UI experience (for example, during writing Unit Tests or recording a marble diagram).

```csharp
Expression<Func<IObservable<string>, IObservable<string>, IObservable<string>,
   IObservable<string>>> expression
       = (a, b, c) => Observable.Merge(a, b, c);

ExpressionDefinition definition = ExpressionDefinition.Create(expression);

using (var instance = ExpressionInstance.Create(definition))
{
    ExpressionInstance instance = ExpressionInstance.Create(definition);

    // using non-generic type 'ObservableSource'
    ObservableSource output1 = instance.Output;
    output1.ObservableStr.Subscribe(Console.WriteLine);

    // using generic type 'ObservableSource<T>'
    ObservableOutput<string> output2 = instance.Output as ObservableOutput<string>;
    output2.Observable.Subscribe(Console.WriteLine);

    instance["a"].OnNext("one");    // using non-generic type 'ObservableInput'
    (instance["a"] as ObservableInput<string>).OnNext("two"); // using generic type
    instance["b"].OnNext("tree");
    instance["a"].OnCompleted();
    instance["b"].OnCompleted();
    instance["c"].OnCompleted();
}
```

When we add delay before sending each input signal we can very easily record sample marble diagram.

```csharp
internal static class Extensions
{
    internal static void OnNext2(this ObservableInput input, string value)
    {
        input.OnNext(value);
        Thread.Sleep(100);
    }
    internal static void OnError2(this ObservableInput input)
    {
        input.OnError(new Exception());
        Thread.Sleep(100);
    }
    internal static void OnCompleted2(this ObservableInput input)
    {
        input.OnCompleted();
        Thread.Sleep(100);
    }
}
```

Marble diagrams are described in a very simple object model.

![rxsandboxdiagram2](/assets/images/rxsandboxdiagram2.png)

This object model can be serialized to Xml format, for instance the code above creates fallowing marble diagram:

```xml
<Diagram>
  <Input Name="a">
    <Marble Value="one" Order="0" />
    <Marble Value="two" Order="1" />
    <Marble Kind="OnCompleted" Order="3" />
  </Input>
  <Input Name="b">
    <Marble Value="tree" Order="2" />
    <Marble Kind="OnCompleted" Order="4" />
  </Input>
  <Input Name="c">
    <Marble Kind="OnCompleted" Order="5" />
  </Input>
  <Output>
    <Marble Value="one" Order="0" />
    <Marble Value="two" Order="1" />
    <Marble Value="tree" Order="2" />
    <Marble Kind="OnCompleted" Order="5" />
  </Output>
</Diagram>
```

![rxsandboxmarblediagram](/assets/images/rxsandboxmarblediagram.png)


As we can see, a single marble diagram has a very simple XML representation, and diagrams for all standard operators from RxSandbox are stored in the Diagrams.xml file (the file path can be changed in the configuration files).

Short description added to all standard operators extracted from Rx documentation XML file is the next new feature of the current release. Of course, not all tested reactive expressions are related to one particular operator, so the description can be set manually (ExpressionSettings.Description property). When we want to write a very complicated LINQ expression, or the expression is passed as a delegate type to the ExpressionDefinition. Create a method and we also want to provide a description from a particular Rx operator. At the same time, we can do it by indicating the operator's MethodInfo type (ExpressionSettings.Operator property).

The last but not least new feature is the extensibility mechanism. When we want to write our custom reactive expression without changing anything inside the RxSadbox project, we can do it by implementing the IExpressionProvider interface directly or by inheriting from an abstract class ExpressionAttributeBasedProvider and setting our assembly name in the configuration file (ExtensionsAssembly element). RxSandbox loads that assembly during the startup process, analyzes it, and finds all expression providers. [Few weeks ago](http://mnajder.blogspot.com/2009/08/incremental-find-with-reactive.html) I have been writing about Incremental Find implemented using Rx, lets see how such a query can tested via RxSandbox.

```csharp
[AttributeUsage(AttributeTargets.Method,AllowMultiple = false, Inherited = true)]
public class ExpressionAttribute : Attribute { }

public interface IExpressionProvider
{
    IEnumerable<ExpressionDefinition> GetExpressions();
}

public abstract class ExpressionAttributeBasedProvider : IExpressionProvider
{
    public IEnumerable<ExpressionDefinition> GetExpressions()
    {
        var q =
            from m in this.GetType().GetMethods()
            let attr = Attribute.GetCustomAttribute(m, typeof(ExpressionAttribute)) 
                as ExpressionAttribute
            where attr != null
            select m.Invoke(null, null) as ExpressionDefinition;
        return q.ToList();
    }
}

public class CustomExpressions : ExpressionAttributeBasedProvider
{
    [Expression]
    public static ExpressionDefinition IncrementalSearch()
    {
        Func<IObservable<string>, IObservable<Person>, IObservable<Person>> expression
                = (codeChanged, webServiceCall) =>
                      {
                        var q =
                            from code in codeChanged
                            from x in Observable.Return(new Unit())
                                .Delay(TimeSpan.FromSeconds(4)).TakeUntil(codeChanged)
                            from result in webServiceCall.TakeUntil(codeChanged)
                            select result;

                          return q;
                      };

        return ExpressionDefinition.Create(expression, new ExpressionSettings
           {
               Name = "Incremental find",
               Description = @"Send the code of the person you are looking for, "
                    + "after four seconds (if you don't send new code again) web service "
                    + "will be called. The result won't be returned if new code is provided "
                    + "in the meantime.",                   
           });
    }
}

[TypeConverter(typeof(PersonConverter))]
public class Person
{
    public string Code { get; set; }
    public string Name { get; set; }
}

public class PersonConverter : TypeConverter
{
    public override bool CanConvertFrom(ITypeDescriptorContext context, Type sourceType)
    {
        if (sourceType == typeof (string))
            return true;
        return base.CanConvertFrom(context, sourceType);
    }
    
    public override object ConvertFrom(ITypeDescriptorContext context, CultureInfo culture, object value)
    {
        if (value is string)
        {
            string[] v = ((string)value).Split(new[] { ',' });
            return new Person {Code = v[0], Name = v[1]};
        }
        return base.ConvertFrom(context, culture, value);
    }
    
    public override object ConvertTo(ITypeDescriptorContext context,
       CultureInfo culture, object value, Type destinationType)
    {
        if (destinationType == typeof (string))
        {
            var person = value as Person;
            return person.Code + "," + person.Name;
        }                
        return base.ConvertTo(context, culture, value, destinationType);
    }
}
```

So that’s it for this release of RxSandbox. I encourage you to play with it and let me know what you think.

And that’s not the end of the project. In upcoming releases, I plan to:
- Create a Silverlight version with the ability to write a reactive expression directly in the web browser
- Add integration with MEF
- Add better look and feel experience

Here you can [download](http://code.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=mnajder&DownloadId=9324) sources and binaries