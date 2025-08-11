---
layout: post
title: When the Reactive Framework meets F# 3.0 [F#]
date: 2011-09-20
tags: fsharp
series: rx-meets-fsharp
---
I was writing about asynchronous sequences in C# a few days ago. This time, we will see how easy it is to implement the same web crawler using the Reactive Framework and a new F# 3.0 feature called [Query Expressions](http://msdn.microsoft.com/en-us/library/hh225374%28v=vs.110%29.aspx). The original implementation of a web crawler created by Tomas Petricek can be found [here](http://www.fssnip.net/7f), and my implementation is surprisingly similar:

```fsharp
let rec randomCrawl url =   
  let visited = new System.Collections.Generic.HashSet<_>()  
    
  let rec loop url = obs {  
    if visited.Add(url) then  
      let! doc = (downloadDocument url) |> fromAsync  
      match doc with   
      | Some doc ->  
          yield url, getTitle doc  
          for link in extractLinks doc do  
            yield! loop link   
      | _ -> () }  
  loop url  
  
rxquery {  
  for (url, title) in randomCrawl "http://news.bing.com" do  
  where (url.Contains("bing.com") |> not)  
  select title  
  take 10 into gr  
  iter (printfn "%s" gr)  
  }  
|> ObservableExtensions.Subscribe |> ignore
```

There are two interesting things inside the code above. The first one is `obs { … }` code construction. This is a custom implementation of [Computation Expression](http://msdn.microsoft.com/en-us/library/dd233182.aspx) builder. The `obs` is just a standard variable of a type that contains a special set of methods like: Bind, Delay, For, Combine, … and so on. The F# compiler translates the code inside the curly brackets into code calling those methods. In my case, the whole expression returns the implementation of `IObservable<T>` type. This allows us to write imperative code representing an observable source where each `yield` call returns the next value (OnNext of subscribed observer is being called), and the `let!` keyword causes the program to wait for the values from the other observable source (`let!` keyword works like the `await` keyword in C#). See the implementation of the builder class below:

```fsharp
type ObsBuiler() =   
  member this.Zero () = Observable.Empty(Scheduler.CurrentThread)  
  member this.Yield v = Observable.Return(v, Scheduler.CurrentThread)   
  member this.Delay (f: _ -> IObservable<_>) = Observable.Defer(fun _ -> f())  
  member this.Combine (o1,o2) = Observable.Concat(o1,o2)  
  member this.For (s:seq<_>, body : _ -> IObservable<_>) = Observable.Concat(s.Select(body))  
  member this.YieldFrom a : IObservable<_> = a  
  member this.Bind ((o : IObservable<_>),(f : _ -> IObservable<_>)) = o.SelectMany(f)   
  member this.TryFinally((o : IObservable<_>),f : unit -> unit ) = Observable.Finally(o,fun _ -> f())  
  member this.TryWith((o : IObservable<_>),f : Exception -> IObservable<_> ) = Observable.Catch(o,fun ex -> f(ex))  
  member this.While (p,body) = Observable.While((fun () -> p()), body)  
  member this.Using (dis,body) = Observable.Using( (fun () -> dis), fun d -> body(d))  
  
let obs = new ObsBuiler()
```

The second interesting part is the `rxquery { … }` code construction. This time we are using a feature called [Query Expressions](http://msdn.microsoft.com/en-us/library/hh225374%28v=vs.110%29.aspx) introduced in F# 3.0, which was released a few days ago during the [Build](http://www.buildwindows.com/) conference, together with Visual Studio 11 and Windows 8. We can write queries very similar to LINQ queries, and the F# compiler translates them into code calling methods like where, select, groupBy, take, and so on. So it works like LINQ queries in C#, but here we can extend the set of available methods arbitrarily !!! Look at Zip and ForkJoin methods below, which are unavailable by default with [QueryBuilder](http://msdn.microsoft.com/en-us/library/hh323943%28v=VS.110%29.aspx) implementation working with `IEnumerable<T>` type. Let’s see the implementation of the query builder class:


```fsharp
type RxQueryBuiler() =    
  member this.For (s:IObservable<_>, body : _ -> IObservable<_>) = s.SelectMany(body)  
  [<CustomOperation("select", AllowIntoPattern=true)>]  
  member this.Select (s:IObservable<_>, [<ProjectionParameter>] selector : _ -> _) = s.Select(selector)  
  [<CustomOperation("where", MaintainsVariableSpace=true, AllowIntoPattern=true)>]  
  member this.Where (s:IObservable<_>, [<ProjectionParameter>] predicate : _ -> bool ) = s.Where(predicate)  
  [<CustomOperation("takeWhile", MaintainsVariableSpace=true, AllowIntoPattern=true)>]  
  member this.TakeWhile (s:IObservable<_>, [<ProjectionParameter>] predicate : _ -> bool ) = s.TakeWhile(predicate)  
  [<CustomOperation("take", MaintainsVariableSpace=true, AllowIntoPattern=true)>]  
  member this.Take (s:IObservable<_>, count) = s.Take(count)  
  [<CustomOperation("skipWhile", MaintainsVariableSpace=true, AllowIntoPattern=true)>]  
  member this.SkipWhile (s:IObservable<_>, [<ProjectionParameter>] predicate : _ -> bool ) = s.SkipWhile(predicate)  
  [<CustomOperation("skip", MaintainsVariableSpace=true, AllowIntoPattern=true)>]  
  member this.Skip (s:IObservable<_>, count) = s.Skip(count)  
  member this.Zero () = Observable.Empty(Scheduler.CurrentThread)  
  member this.Yield (value) = Observable.Return(value)  
  [<CustomOperation("count")>]  
  member this.Count (s:IObservable<_>) = Observable.Count(s)  
  [<CustomOperation("all")>]  
  member this.All (s:IObservable<_>, [<ProjectionParameter>] predicate : _ -> bool ) = s.All(new Func<_,bool>(predicate))  
  [<CustomOperation("contains")>]  
  member this.Contains (s:IObservable<_>, key) = s.Contains(key)  
  [<CustomOperation("distinct", MaintainsVariableSpace=true, AllowIntoPattern=true)>]  
  member this.Distinct (s:IObservable<_>) = s.Distinct()  
  [<CustomOperation("exactlyOne")>]  
  member this.ExactlyOne (s:IObservable<_>) = s.Single()  
  [<CustomOperation("exactlyOneOrDefault")>]  
  member this.ExactlyOneOrDefault (s:IObservable<_>) = s.SingleOrDefault()  
  [<CustomOperation("find")>]  
  member this.Find (s:IObservable<_>, [<ProjectionParameter>] predicate : _ -> bool) = s.First(new Func<_,bool>(predicate))  
  [<CustomOperation("head")>]  
  member this.Head (s:IObservable<_>) = s.First()  
  [<CustomOperation("headOrDefault")>]  
  member this.HeadOrDefault (s:IObservable<_>) = s.FirstOrDefault()  
  [<CustomOperation("last")>]  
  member this.Last (s:IObservable<_>) = s.Last()  
  [<CustomOperation("lastOrDefault")>]  
  member this.LastOrDefault (s:IObservable<_>) = s.LastOrDefault()  
  [<CustomOperation("maxBy")>]  
  member this.MaxBy (s:IObservable<'a>,  [<ProjectionParameter>] valueSelector : 'a -> 'b) = s.MaxBy(new Func<'a,'b>(valueSelector))  
  [<CustomOperation("minBy")>]  
  member this.MinBy (s:IObservable<'a>,  [<ProjectionParameter>] valueSelector : 'a -> 'b) = s.MinBy(new Func<'a,'b>(valueSelector))  
  [<CustomOperation("nth")>]  
  member this.Nth (s:IObservable<'a>,  index ) = s.ElementAt(index)  
  [<CustomOperation("sumBy")>]  
  member inline this.SumBy (s:IObservable<_>,[<ProjectionParameter>] valueSelector : _ -> _) = s.Select(valueSelector).Aggregate(Unchecked.defaultof<_>, new Func<_,_,_>( fun a b -> a + b))   
  [<CustomOperation("groupBy", AllowIntoPattern=true)>]  
  member this.GroupBy (s:IObservable<_>,[<ProjectionParameter>] keySelector : _ -> _) = s.GroupBy(new Func<_,_>(keySelector))  
  [<CustomOperation("groupValBy", AllowIntoPattern=true)>]  
  member this.GroupValBy (s:IObservable<_>,[<ProjectionParameter>] resultSelector : _ -> _,[<ProjectionParameter>] keySelector : _ -> _) = s.GroupBy(new Func<_,_>(keySelector),new Func<_,_>(resultSelector))  
  [<CustomOperation("join", IsLikeJoin=true)>]  
  member this.Join (s1:IObservable<_>,s2:IObservable<_>, [<ProjectionParameter>] s1KeySelector : _ -> _,[<ProjectionParameter>] s2KeySelector : _ -> _,[<ProjectionParameter>] resultSelector : _ -> _) = s1.Join(s2,new Func<_,_>(s1KeySelector),new Func<_,_>(s2KeySelector),new Func<_,_,_>(resultSelector))  
  [<CustomOperation("groupJoin", AllowIntoPattern=true)>]  
  member this.GroupJoin (s1:IObservable<_>,s2:IObservable<_>, [<ProjectionParameter>] s1KeySelector : _ -> _,[<ProjectionParameter>] s2KeySelector : _ -> _,[<ProjectionParameter>] resultSelector : _ -> _) = s1.GroupJoin(s2,new Func<_,_>(s1KeySelector),new Func<_,_>(s2KeySelector),new Func<_,_,_>(resultSelector))  
  [<CustomOperation("zip", IsLikeZip=true)>]  
  member this.Zip (s1:IObservable<_>,s2:IObservable<_>,[<ProjectionParameter>] resultSelector : _ -> _) = s1.Zip(s2,new Func<_,_,_>(resultSelector))  
  [<CustomOperation("forkJoin", IsLikeZip=true)>]  
  member this.ForkJoin (s1:IObservable<_>,s2:IObservable<_>,[<ProjectionParameter>] resultSelector : _ -> _) = s1.ForkJoin(s2,new Func<_,_,_>(resultSelector))  
  [<CustomOperation("iter")>]  
  member this.Iter(s:IObservable<_>, [<ProjectionParameter>] selector : _ -> _) = s.Do(selector)  
  
let rxquery = new RxQueryBuiler()
```