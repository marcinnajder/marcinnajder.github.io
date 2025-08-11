---
layout: post
title: Generating observable events with T4 template for Rx [C#]
date: 2009-08-15
tags:
  - csharp
series: t4-rx
---

Reactive Framework (Rx) is what you need to play with. Firstly, watch [this](http://langnetsymposium.com/2009/talks/23-ErikMeijer-LiveLabsReactiveFramework.html), [this](http://channel9.msdn.com/shows/Going+Deep/Expert-to-Expert-Brian-Beckman-and-Erik-Meijer-Inside-the-NET-Reactive-Framework-Rx/) and [this](http://channel9.msdn.com/shows/Going+Deep/Kim-Hamilton-and-Wes-Dyer-Inside-NET-Rx-and-IObservableIObserver-in-the-BCL-VS-2010/), then download [Silverlight Toolkit](http://silverlight.codeplex.com/SourceControl/ListDownloadableCommits.aspx), where you can find System.Reactive.dll. If you don't want to use Silverlight's version of Rx, convert it to the full .NET framework's version [this way](http://evain.net/blog/articles/2009/07/30/rebasing-system-reactive-to-the-net-clr). Next read [this](http://themechanicalbride.blogspot.com/2009/07/introducing-rx-linq-to-events.html), [this](http://themechanicalbride.blogspot.com/2009/07/developing-with-rx-part-1-extension.html), [this](http://themechanicalbride.blogspot.com/2009/07/developing-with-rx-part-2-converting.html) and [this](http://themechanicalbride.blogspot.com/2009/08/joy-of-rx-building-asynchronous-service.html).

Now, you are ready to write your own LINQ queries over events. :) Let's see a very simple query listening to 3 events of the System.Windows.Forms.Form class.

```csharp
var form = new Form();

IObservable<Event<KeyEventArgs>> keyDown =
    Observable.FromEvent<KeyEventArgs>(form, "KeyDown");
IObservable<Event<KeyEventArgs>> keyUp =
    Observable.FromEvent<KeyEventArgs>(form, "KeyUp");
IObservable<Event<MouseEventArgs>> mouseMove =
    Observable.FromEvent<MouseEventArgs>(form, "MouseMove");

var downMoveUp =
    from key in keyDown
    from mouse in mouseMove.Until(keyUp)
    select " X = " + mouse.EventArgs.X + " Y = " + mouse.EventArgs.Y;

downMoveUp.Subscribe(x => Console.WriteLine(x));

Console.WriteLine("Press any key and move mouse until key up ... ");

form.ShowDialog();
```

When creating an event observer, you must put a hardcoded event name and an appropriate event argument type. This is quite uncomfortable, especially when you want to use this event in many different places, so let's wrap this code into an extension method.

```csharp
public static class FormExtenions
{
    public static IObservable<Event<KeyEventArgs>> GetObservableKeyDown(this Form form)
    {
        return Observable.FromEvent<KeyEventArgs>(form, "KeyDown");
    }
    public static IObservable<Event<KeyEventArgs>> GetObservableKeyUp(this Form form)
    {
        return Observable.FromEvent<KeyEventArgs>(form, "KeyUp");
    }
    public static IObservable<Event<MouseEventArgs>> GetObservableMouseMove(this Form form)
    {
        return Observable.FromEvent<MouseEventArgs>(form, "MouseMove");
    }
}

var keyDown = form.GetObservableKeyDown();
var keyUp = form.GetObservableKeyUp();
var mouseMove = form.GetObservableMouseMove();
```

Writing extension methods for all events of all controls is obviously a very boring and error-prone task, so let's use [T4](http://www.olegsych.com/2007/12/text-template-transformation-toolkit/) templates to generate that code.

![T4](/assets/images/ObservableEventsT4.png)


This produces:

```csharp
using System.Linq;
using System.Collections.Generic;

namespace ObservableEvents
{
    public static class ExtensionMethods
    {
        public static IObservable<Event<System.Windows.Forms.ControlEventArgs>> GetObservableControlAdded(this System.Windows.Forms.Control source)
        {
            return Observable.FromEvent<System.Windows.Forms.ControlEventArgs>(source,"ControlAdded" );
        }
        public static IObservable<Event<System.Windows.Forms.ControlEventArgs>> GetObservableControlRemoved(this System.Windows.Forms.Control source)
        {
            return Observable.FromEvent<System.Windows.Forms.ControlEventArgs>(source,"ControlRemoved" );
        }
        public static IObservable<Event<System.Windows.Forms.DragEventArgs>> GetObservableDragDrop(this System.Windows.Forms.Control source)
        {
            return Observable.FromEvent<System.Windows.Forms.DragEventArgs>(source,"DragDrop" );
        }
        public static IObservable<Event<System.Windows.Forms.DragEventArgs>> GetObservableDragEnter(this System.Windows.Forms.Control source)
        {
            return Observable.FromEvent<System.Windows.Forms.DragEventArgs>(source,"DragEnter" );
        }
        public static IObservable<Event<System.Windows.Forms.DragEventArgs>> GetObservableDragOver(this System.Windows.Forms.Control source)
        {
            return Observable.FromEvent<System.Windows.Forms.DragEventArgs>(source,"DragOver" );
        }
```

Templates also work in Silverlight projects. Download the source code for this post from [here](http://code.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=mnajder&DownloadId=6952) and enjoy the Rx!.