---
layout: post
title: Incremental search with Rx [C#]
date: 2009-09-14
tags:
  - csharp
series: incremenal-search
---
This time we will implement "incremental search" using the Reactive Framework. This is a very good example showing what the Rx is all about.

Let's say we have a Windows Forms application with a single TextBox. When the user stops typing, the application immediately sends the request to the remote web service to find all words that contain the text from the TextBox. We use Rx to handle two essential issues:

- How to find the moment when the user has just stopped typing?
- How to ensure that the correct results will be displayed when calling a web service that is implemented as an asynchronous operation (results for the most recent input should discard responses for all previous requests)?

For the sake of simplicity, we simulate calling a web service by a simple asynchronous operation that returns results after 3 or 6 seconds. This allows us to easily check whether the correct results are displayed every time. Just type 'iobserv', then wait for about 2 seconds (user stopped writing) and append the letter 'e'. After about 3 seconds, the results for 'ibserve' text should be displayed and then never changed.

```csharp
const string RxOverview =
@"
http://channel9.msdn.com/shows/Going+Deep/Expert-to-Expert-Brian-Beckman-and-Erik-Meijer-Inside-the-NET-Reactive-Framework-Rx/

Now, what is Rx?

The .NET Reactive Framework (Rx) is the mathematical dual of LINQ to Objects. It consists of a pair 
of interfaces IObserver / IObservable that represent push-based, or observable, collections, plus a 
library of extension methods that implement the LINQ Standard Query Operators and other useful 
stream transformation functions.
... 
Observable collections capture the essence of the well-known subject/observer design pattern, and 
are tremendously useful for dealing with event-based and asynchronous programming, i.e. AJAX-style 
applications. For example, here is the prototypical Dictionary Suggest written using LINQ query 
comprehensions over observable collections:

IObservable<Html> q = from fragment in textBox
               from definitions in Dictionary.Lookup(fragment, 10).Until(textBox)
               select definitions.FormatAsHtml();

q.Subscribe(suggestions => { div.InnerHtml = suggestions; })
";

var textBox = new TextBox();
var label = new Label
                {
                    Text = "results...",
                    Location = new Point(0, 40),
                    Size = new Size(300, 500),
                    BorderStyle = BorderStyle.FixedSingle,                                
                };
var form = new Form { Controls = { textBox, label } };

Func<string, IObservable<string[]>> search = (s) =>
{
    var subject = new Subject<string[]>();

    ThreadPool.QueueUserWorkItem((w) =>
    {
        Thread.Sleep(s.Length % 2 == 0 ? 3000 : 6000);

        var result = RxOverview.
            Split(new[] { " ", "\n", "\t", "\r" }, StringSplitOptions.RemoveEmptyEntries).
            Where(t => t.ToLower().Contains(s.ToLower())).
            ToArray();

        subject.OnNext(result);
        subject.OnCompleted();
    }, null);

    return subject;
};

IObservable<Event<EventArgs>> textChanged = textBox.GetObservableTextChanged();

var q =
    from e in textChanged
    let text = (e.Sender as TextBox).Text
    from x in Observable.Return(new Unit()).Delay(1000).Until(textChanged)  // first issue
    from results in search(text).Until(textChanged)                         // second issue
    select new { text, results };

var a1 = q.Send(SynchronizationContext.Current).Subscribe(r =>
{
    label.Text = string.Format(" Text: {0}\n Found: {1}\n Results:\n{2}",
        r.text,
        r.results.Length,
        string.Join("\n", r.results));
});

form.ShowDialog();
```

Now, try to implement the same functionality without the Rx Framework.