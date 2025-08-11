---
layout: post
title: Drag and drop with Rx [C#]
date: 2009-09-07
tags:
  - csharp
series: drag-n-drop
---
Look how easily we can implement "drag and drop" functionality using the Reactive Framework.

```csharp
var form = new Form
{
    Controls =
       {
           new Label {Text = "label1", BorderStyle = BorderStyle.FixedSingle},                       
           new Button {Text = "button1"},
           new Label {Text = "label2", BorderStyle = BorderStyle.FixedSingle},
       }
};

Func<Control, IObservable<Event<MouseEventArgs>>> mouseDown = c => c.GetObservableMouseDown();
Func<Control, IObservable<Event<MouseEventArgs>>> mouseUp = c => c.GetObservableMouseUp();
Func<Control, IObservable<Event<MouseEventArgs>>> mouseMove = c => c.GetObservableMouseMove();

var q =
    from Control con in form.Controls
    select
    (
        from d in mouseDown(con)
        from u in mouseMove(con).Until(mouseUp(con))
        select u
    );

q.Merge().Subscribe(args =>
{
    var control = args.Sender as Control;
    control.Location = Point.Add(control.Location,
        new Size(args.EventArgs.X, args.EventArgs.Y));
});

form.ShowDialog();
```

Now all controls on the form can be moved from one place to another :)

![rxdragdrop](/assets/images/rxdragdrop.png)