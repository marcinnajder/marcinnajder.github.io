---
layout: post
title: Linq to ICollectionView [C#]
date: 2009-11-12
tags:
  - csharp
series: linq-collectionview
---
Currently, I'm working on a project written in Silverlight, and I have encountered an interface called `ICollectionView` and its standard implementation , [PagedCollectionView](http://msdn.microsoft.com/en-us/library/system.windows.data.pagedcollectionview\(VS.95\).aspx). In short, this components allow us to create the view of collection of items with filtering, grouping and sorting functionality. The idea behind this interface is very similar to [DataView](http://msdn.microsoft.com/en-us/library/system.data.dataview.aspx)/DataTable mechanism. DataTable is responsible for storing data, and DataView is just an appropriately configured proxy (only filtering and sorting in this case), which can be bound to UI controls. Let's look at a very simple example:

```csharp
public class Number
{
    public int Value { get; set; }
    public int Random { get; set; }

    public static Number[] GetAll()
    {
        var random = new Random();
        return
            (from n in Enumerable.Range(1,5)
             from m in Enumerable.Repeat(n, n)
             select new Number {Value = n, Random = random.Next(10)}).ToArray();            
    }
}

// Silverlight
public class LinqToICollectionView : UserControl
{
    public LinqToICollectionView()
    {
        Number[] numbers = Number.GetAll();

        Content = new StackPanel
        {
            Orientation = Orientation.Horizontal,
            Children =
            {
                new DataGrid { ItemsSource = new PagedCollectionView(numbers).SetConfiguration()},
            }
        };
    }
}

public static class Configurator
{
    public static ICollectionView SetConfiguration(this ICollectionView view)
    {
        // filtering
        view.Filter = (object o) => ((Number)o).Value < 5;
        // grouping
        view.GroupDescriptions.Add(new PropertyGroupDescription("Value"));
        // sorting
        view.SortDescriptions.Add(new SortDescription("Value", ListSortDirection.Descending));
        view.SortDescriptions.Add(new SortDescription("Random", ListSortDirection.Ascending));
        return view;
    }     
}
```


![icollectionview](/assets/images/icollectionview.png)

But wait a minute, we said ... filtering, ordering, grouping ? Let's use LINQ query to configure ICollectionView:

```csharp
public static class Configurator
{
    public static ICollectionView SetConfigurationWithLinq(this ICollectionView view)
    {
        var q =
            from n in new View<Number>()
            where n.Value < 5
            orderby n.Value descending, n.Random
            group n by n.Value;
        q.Apply(view);
        return view;
    }        
}
```

The whole implementation consists of 3 simple classes: `View<T>`, `OrderedView<T>`, and `GroupedView<T>`.

```csharp
public class View<T>
{
    public IEnumerable<GroupDescription> GroupDescriptions { get { ... } }
    public IEnumerable<SortDescription> SortDescriptions { get { ... } }
    public Func<T,bool> Filter { get { ... } }
    
    public View<T> Where(Func<T, bool> func) { ... }
    public SortedView<T> OrderBy<T2>(Expression<Func<T, T2>> func) { ... }
    public SortedView<T> OrderByDescending<T2>(Expression<Func<T, T2>> func) { ... }
    public GroupedView<T> GroupBy<T2>(Expression<Func<T, T2>> func) { ... }
    
    public void Apply(ICollectionView collectionView) { ... }
}
public sealed class SortedView<T> : View<T>
{    
    public SortedView<T> ThenBy<T2>(Expression<Func<T, T2>> func) { ... }
    public SortedView<T> ThenByDescending<T2>(Expression<Func<T, T2>> func)  { ... }
}
public sealed class GroupedView<T> : View<T>
{
    public GroupedView<T> ThenBy<T2>(Expression<Func<T, T2>> func) { ... }
}
```

Methods for sorting and grouping, which take an expression tree as a parameter analyze the tree looking for indicated members (fields or properties) and collect appropriate SortDescription, and GroupDescription objects. The `Where` method takes a delegate type, which is combined via logical "and" operator with the existing filter delegate set previously (in case when the `Where` method is called many times). Of course, the same mechanism work also in WPF.

```csharp
Number[] numbers = Number.GetAll();

// WPF
new Window
{
    Content = new StackPanel
    {
        Orientation = Orientation.Horizontal,
        Children =
        {
            CreateListView(new CollectionViewSource { Source = numbers }.View.SetConfiguration()),
            CreateListView(new CollectionViewSource { Source = numbers }.View.SetConfigurationWithLinq())
        }
    }
}
.ShowDialog();

private static ListView CreateListView(ICollectionView view)
{
    return new ListView
    {
        GroupStyle = { GroupStyle.Default },
        View = new GridView
        {
            Columns =
            {
                new GridViewColumn { Header = "Value", DisplayMemberBinding = new Binding("Value") },
                new GridViewColumn { Header = "Random", DisplayMemberBinding = new Binding("Random") },
            }
        },
        ItemsSource = view
    };
}
```

At the end I'd like to mention one interesting thing. We only support filtering, sorting and grouping and don't support for instance projection, joining and so on. That's way this code should compile:

```csharp
var v1 = // only filtering specified
    from n in new View<Number>() 
    where n.Value < 5 
    select n;

var v2 = // grouping (last grouping definition overrides previous ones)
    from n in new View<Number>()
    group n by n.Random into s 
    group s by s.Value;
```

But these will not:

```csharp
var v3 = // joining is not supported
    from p in new View<Number>()
    join pp in new[] { 1, 2, 3, 4 } on p.Random equals pp
    select p;

var v4 = // projection is not supported
    from p in new View<Number>()
    where p.Value > 5
    select p.Random;

var v5 = // at least one filtering, grouping or sorting definition must be specified
    from p in new View<Number>()
    select p;

var v6 = // Numer type is the only valid type of grouped element
    from p in new View<Number>()
    group p.Random by p.Value;
```

As homework, I leave you a question: why does it work this way? :)

