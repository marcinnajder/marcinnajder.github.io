---
layout: post
title: Simplifying XNA development using Async and F# asynchronous workflow [C#, F#]
date: 2011-05-05
tags:
  - csharp
series: asyncctp
---
[Last time](http://mnajder.blogspot.com/2011/01/async-ctp-on-wp7.html) I have been writing about launching the Async CTP on WP7 platform. The [Async CTP Refresh](http://msdn.microsoft.com/en-us/vstudio/async/) has been released during MIX 2011 and finally WP7 platform is supported, so my last post is quite obsolete right now. But I hope that, after reading it, you better understand how Async CTP works underneath.

This time we will go one step forward. Once we have Tasks on WP7 and two new keywords await and async in C# we can use all that to change the way we write games in XNA.

If you know something about TPL and XNA framework it may seem a bit strange how I am going to mix them both since at first look TPL is all about building multithreaded applications and XNA application uses one single thread in most cases.

I will not not write too much about XNA framework itself  (especially because I am a beginner in game development :) ) but if you try writing even a very simple 2D game you will find out that code for XNA is really hard to understand and maintain. Probably you will have many fields storing state of the game items, many boolean flags and a lot of if-then-else statements changing that state. You can say that the game is actually one big state machine. That’s what I’ve found out after writing my fist game in XNA.

Async CTP can change the structure of the code dramatically so we can easily discover the flow of the game logic and what is most interesting everything is running on single thread.

In this post, I will show you three implementations of a very simple 2D game called Smiley. All of those implementations use the XNA framework, but the first one is a typical XNA approach, and the last two are totally different. They use Async CTP and the F# asynchronous workflow.

![xnagame](/assets/images/xnagame.png)
I took the idea for the game from the [Tomas Petricek’s master thesis](http://tomasp.net/academic/reactive-thesis/thesis.pdf) and it can be presented on the three screens:

![game1](/assets/images/game1.png)

![game2](/assets/images/game2.png)

![game3](/assets/images/game3.png)

We tap the screen to start the game, then we have 20 seconds to hit moving Smiley image as many times as we can and at the end the number of hits is displayed for 4 seconds. The first screen appears again. The Smiley position is changing in 2 seconds periods or directly after tapping on it.

Originally, that game was implemented in F# using [Joinads](http://tomasp.net/blog/fsharp-variations-joinads.aspx). A few months ago I have implemented that game also in F# using Reactive Framework and treating the `IObservable<T>` type as the [Monad](http://en.wikipedia.org/wiki/Monad_\(functional_programming\)) type. With that I built my own workflow builder. I will try to describe that approach in the next blog post but now let’s come back to the subject of today’s post.

As you can see in the class diagram above, all three implementations have a common base class called SmileyGameBase. That class inherits from Game class which is the main XNA component representing the whole game. And all we need to do when writing an XNA game is to override two methods: Update and Draw. Update method is responsible for recalculating the game state and the Draw method draws all game items like texts, textures and so on. Both methods are called by XNA environment (Update before Draw) many times during each second when the game is running (about 30 times per second on WP7). In our game the Draw method looks the same in all three cases but the Update is specific for each of them.

```csharp
public abstract class SmileyGameBase : Game
{
    // ... 

    protected override void Draw(GameTime gameTime)
    {
        GraphicsDevice.Clear(Color.Black);

        _spriteBatch.Begin();

        if (_isRunning)
        {
            float fontScale = 2;
            _spriteBatch.DrawString(_spriteFont, "time : " + _time, new Vector2(10, 5), 
                Color.White, 0, Vector2.Zero, fontScale, SpriteEffects.None, 0);
            _spriteBatch.DrawString(_spriteFont, "score : " + _score, new Vector2(500, 5), 
                Color.White, 0, Vector2.Zero, fontScale, SpriteEffects.None, 0);

            _spriteBatch.Draw(_smileyTexture, _smileyPosition, Color.White);
        }
        else
        {
            float fontScale = 3;
            _spriteBatch.DrawString(_spriteFont, _message, new Vector2(10, 100), 
                Color.Red, 0, Vector2.Zero, fontScale, SpriteEffects.None, 0);
        }

        _spriteBatch.End();

        base.Draw(gameTime);
    }
}
```

The code is pretty simple, we have few fields representing the game state and the Draw method draws Smiley image and prints some text messages. Now let’s look at the XNA and Async CTP implementations to compare them together.

This is the Async CTP implementation.

```csharp
using System.Threading.Tasks;
using Microsoft.Xna.Framework.Input.Touch;
    
namespace SmileyGame.Common
{
    using AsyncXnaIntegration;
   
    public abstract class AsyncCtpGame : SmileyGameBase
    {
        private GameAwaiter _gameAwaiter;

        protected AsyncCtpGame()
        {
            _gameAwaiter = new GameAwaiter(this);
            Components.Add(_gameAwaiter);
        }

        protected override void LoadContent()
        {
            base.LoadContent();
            StartMenu();
        }

        async private void StartMenu()
        {
            while (true)
            {
                _message = "tap to start the game ... ";
                await _gameAwaiter.Gesture(GestureType.Tap);
                _isRunning = true;
                await StartGame();
                _isRunning = false;
                _message = "your score : " + _score;
                await _gameAwaiter.Delay(4000);
            }
        }

        async private Task StartGame()
        {
            _score = 0;
            var timer = StartTimer(GameDuration);

            while (true)
            {
                var match = await _gameAwaiter.WhenAny(timer,
                    _gameAwaiter.Delay(ChangePositionPeriod), _gameAwaiter.Gesture(IsSmileyClicked));
                switch (match.Index)
                {
                    case 0:
                        return;

                    case 1:
                        _smileyPosition = GetRandomPostion();
                        break;

                    case 2:
                        _smileyPosition = GetRandomPostion();
                        _score++;
                        break;

                    default: break;
                }

            }
        }

        async private Task StartTimer(int n)
        {
            while (n >= 0)
            {
                _time = n;
                await _gameAwaiter.Delay(TimerPeriod);
                n--;
            }
        }
    }
}
```

And here is the pure XNA implementation.

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Input.Touch;

namespace SmileyGame.Common
{
    public abstract class XnaGame : SmileyGameBase
    {
        protected XnaGame()
        {
            _message = "tap to start the game ... ";
        }

        private TimeSpan _nextTimerTime;
        private TimeSpan _nextChangePostionTime;
        private TimeSpan? _nextDisplayFinalScoreTime;
        
        protected override void Update(GameTime gameTime)
        {
            base.Update(gameTime);

            var gestures = GetGestures();

            if (!_isRunning)
            {
                if (_nextDisplayFinalScoreTime.HasValue)
                {
                    if (gameTime.TotalGameTime > _nextDisplayFinalScoreTime.Value)
                    {
                        _nextDisplayFinalScoreTime = null;
                        _message = "tap to start the game ... ";
                    }   
                }
                else if (gestures.Any(g => IsGestureType(g, GestureType.Tap)))
                {
                    _isRunning = true;
                    _score = 0;
                    _time = GameDuration;
                    _nextTimerTime = gameTime.TotalGameTime + TimeSpan.FromMilliseconds(TimerPeriod);
                    _nextChangePostionTime = gameTime.TotalGameTime + TimeSpan.FromMilliseconds(ChangePositionPeriod);
                }
            }
            else
            {                
                if (gameTime.TotalGameTime > _nextTimerTime )
                {
                    _time--;
                    _nextTimerTime = gameTime.TotalGameTime + TimeSpan.FromMilliseconds(TimerPeriod);
                    if (_time < 0)
                    {
                        _isRunning = false;
                        _nextDisplayFinalScoreTime = gameTime.TotalGameTime + TimeSpan.FromMilliseconds(4000);
                        _message = "your score : " + _score;
                    }                    
                }
                else if (gameTime.TotalGameTime > _nextChangePostionTime )
                {
                    _smileyPosition = GetRandomPostion();
                    _nextChangePostionTime = gameTime.TotalGameTime + TimeSpan.FromMilliseconds(ChangePositionPeriod);
             
                }
                else if (gestures.Any(IsSmileyClicked))
                {
                    _smileyPosition = GetRandomPostion();
                    _nextChangePostionTime = gameTime.TotalGameTime + TimeSpan.FromMilliseconds(ChangePositionPeriod);
                    _score++;
                }
            }
        }

        private static bool IsGestureType(GestureSample gestureSample, GestureType gestureType)
        {
            return (gestureSample.GestureType & gestureType) == gestureType;
        }

        private static List<GestureSample> GetGestures()
        {
            var gestures = new List<GestureSample>();
            while (TouchPanel.IsGestureAvailable)
                gestures.Add(TouchPanel.ReadGesture());
            return gestures;
        }
    }
}
```

When we read first implementation we can easily discover the flow of the program and the order of particular pieces of code look natural. In the pure XNA approach it is really hard to understand how the game works internally and how to extend them. Let’s image that we would like to display sequentially “3” “2” “1” “start” just after first tap in 1 second periods. Think how to do it in each of these two approaches ?

So now lets go to F# implementation. F# has something called asynchronous workflow which was available for F# developers long before Async CTP (and in fact it was the inspiration for Async CTP authors) the F# implementation is very similar to Async CTP one.

```fsharp
namespace SmileyGame.FSharp

open SmileyGame.Common
open AsyncXnaIntegration
open SmileyGame.FSharp.Extensions
open Microsoft.Xna.Framework.Input.Touch

type FSharpGame() as this =
  inherit SmileyGameBase()
  let _gameAwaiter = new GameAwaiter(this)
  do this.Components.Add(_gameAwaiter)

  // access to protected members
  member private this.set_smileyPosition p = this._smileyPosition <- p
  member private this.set_time t = this._time <- t 
  member private this.set_score s = this._score <- s
  member private this.set_message m = this._message <- m
  member private this.set_isRunning r = this._isRunning  <- r
  member private this.inc_score() = this._score <- this._score + 1  
  member private this.get_score = this._score
  member private this.get_TimerPeriod = SmileyGameBase.TimerPeriod
  member private this.get_GameDuration = SmileyGameBase.GameDuration  
  member private this.get_ChangePositionPeriod = SmileyGameBase.ChangePositionPeriod  
  member private this.IsSmileyClicked g = base.IsSmileyClicked g
  member private this.GetRandomPostion() = base.GetRandomPostion()
  
  member private this.StartTimer n =
    let n = ref n
    async {      
      while !n >= 0 do        
        this.set_time !n
        do! _gameAwaiter.Delay(this.get_TimerPeriod) |> Async.AwaitTask
        n := !n - 1
    }

  member private this.StartGame() =
    async {
    do this.set_score 0
    let timer = this.StartTimer(this.get_GameDuration) |> Async.ToTask
    let c = ref true
    while !c do
      let! m = _gameAwaiter.WhenAny(timer,_gameAwaiter.Delay(this.get_ChangePositionPeriod), _gameAwaiter.Gesture(this.IsSmileyClicked)) |> Async.AwaitTask
      match m.Index with
      | 0 -> c := false
      | 1 -> this.set_smileyPosition(this.GetRandomPostion())
      | 2 -> this.set_smileyPosition(this.GetRandomPostion()) ; this.inc_score()
      | _ -> ()
    }

  member private this.StartMenu() =
    async {
      while true do
        this.set_message "tap to start the game ... "
        let! _ = _gameAwaiter.Gesture(GestureType.Tap) |> Async.AwaitTask
        this.set_isRunning true
        do! this.StartGame()
        this.set_isRunning false
        this.set_message ("your score : " + (this.get_score).ToString() )
        do! _gameAwaiter.Delay(float 4000) |> Async.AwaitTask
    }

  override this.LoadContent() =
    base.LoadContent()
    this.StartMenu() |> Async.StartImmediate
```

Fine, but how does it work? All I had to do was to implement class GameAwaiter deriving from GameComponent which means that we can add it to the components stored inside Game class and its Update method will be called automatically when Game’s Update method is being called. GameAwaiter gives us basically a few public methods such as Gesture, Delay and WhenAny. All of them return Task object which means that at some point in the future the result will come. Let’s see the details:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Input.Touch;

namespace AsyncXnaIntegration
{
    using AsyncXnaIntegration;

    public class GameAwaiter : GameComponent
    {
        private readonly List<Job> _jobs = new List<Job>();

        public GameAwaiter(Game game)
            : base(game)
        { }

        public Task Delay(TimeSpan interval)
        {
            return RegisterJob(new TimerJob { Interval = interval });
        }
        
        public Task Delay()
        {
            return RegisterJob(new TimerJob());
        }
     
        public Task While(Func<bool> condition)
        {
            return RegisterJob(new WhileJob { Condition = condition });
        }
  
        public Task<GestureSample[]> Gesture(Func<GestureSample, bool> predicate)
        {
            return RegisterJob(new GestureJob { GestureCondition = predicate });
        }

        public void TryDisposeTask(Task tt)
        {
            var job = _jobs.FirstOrDefault(j => j.TaskO == tt);
            if (job != null)
                job.IsDisposed = true;
        }

        public override void Update(GameTime gametime)
        {
            var state = new State { GameTime = gametime, TouchState = TouchPanel.GetState(), };
            while (TouchPanel.IsGestureAvailable)
                state.Gestures.Add(TouchPanel.ReadGesture());

            foreach (var job in _jobs.ToArray())
            {      
                if (job.IsDisposed || job.Update(state))                
                    _jobs.Remove(job);
            }
        }


        private Task<T> RegisterJob<T>(Job<T> job)
        {
            _jobs.Add(job);
            return job.Task;
        }

        #region inner types

        private class State
        {
            public GameTime GameTime;
            public TouchCollection TouchState;
            public readonly List<GestureSample> Gestures = new List<GestureSample>();
        }

        private abstract class Job
        {
            public bool IsDisposed { get; set; }
            public abstract Task TaskO { get; }

            public abstract bool Update(State state);
        }

        private abstract class Job<T> : Job
        {
            protected TaskCompletionSource<T> _source = new TaskCompletionSource<T>();
            public Task<T> Task { get { return _source.Task; } }
            public override Task TaskO { get { return Task; } }
        }

        private class TimerJob : Job<object>
        {
            private TimeSpan? StartTime;
            public TimeSpan? Interval;

            public override bool Update(State state)
            {
                if (!Interval.HasValue)
                {
                    _source.TrySetResult(null);
                    return true;
                }

                if (!StartTime.HasValue)
                    StartTime = state.GameTime.TotalGameTime;

                if (state.GameTime.TotalGameTime - StartTime >= Interval)
                {
                    _source.TrySetResult(null);
                    return true;
                }

                return false;
            }
        }

        private class WhileJob : Job<object>
        {
            public Func<bool> Condition;

            public override bool Update(State state)
            {
                if (Condition())
                {
                    _source.TrySetResult(null);
                    return true;
                }
                return false;
            }
        }

        private class GestureJob : Job<GestureSample[]>
        {
            public Func<GestureSample, bool> GestureCondition;

            public override bool Update(State state)
            {
                if (state.Gestures.Count > 0)
                {
                    if (GestureCondition == null)
                    {
                        _source.TrySetResult(state.Gestures.ToArray());
                        return true;
                    }

                    var gestures = state.Gestures.Where(GestureCondition).ToArray();
                    if (gestures.Length > 0)
                    {
                        _source.TrySetResult(gestures);
                        return true;
                    }
                }

                return false;
            }
        }

        #endregion
    
    }

public static class GameAwaiterExtensions
    {
        public static Task<Branch> WhenAny(this GameAwaiter gameAwaiter, params Task[] tasks)
        {
            return GameAwaiterExtensions.WhenAny(tasks)
                .ContinueWith(t =>
                {
                    try
                    {
                        foreach (var tt in tasks)
                            gameAwaiter.TryDisposeTask(tt);
                        return t.Result;
                    }
                    catch (Exception exception)
                    {
                        Debugger.Break();
                        return t.Result;
                    }
                }, TaskContinuationOptions.ExecuteSynchronously);
        }

        public static Task<Branch> WhenAny(params Task[] tasks)
        {
            return TaskEx
                .WhenAny(tasks)
                .ContinueWith(t =>
                {
                    var task = t.Result;
                    var taskType = task.GetType();
                    object value = null;

                    // we cannot read nonpublic types via reflection in silverlight 
                    if (taskType.IsGenericType && taskType.GetGenericArguments()[0].IsPublic)
                        value = task.GetType().GetProperty("Result").GetValue(task, null);

                    return new Branch { Index = Array.IndexOf(tasks, task), Value = value };                    
                }, TaskContinuationOptions.ExecuteSynchronously);
        }

        public static Task Delay(this GameAwaiter gameAwaiter, double milliseconds)
        {
            return gameAwaiter.Delay(TimeSpan.FromMilliseconds(milliseconds));
        }

        public static Task<GestureSample[]> Gesture(this GameAwaiter gameAwaiter, GestureType gestureType)
        {
            return gameAwaiter.Gesture(g => (g.GestureType & gestureType) == gestureType);
        }

        public static  Task<GestureSample[]> Gesture(this GameAwaiter gameAwaiter)
        {
            return gameAwaiter.Gesture(g => true);
        }
    }


    public class Branch
    {
        public int Index;
        public object Value;
    }
}
```

The last thing I would like to mention is a really tricky stuff :) What was really important for me when writing GameAwaiter, was the linear execution of the code. I didn’t want to introduce any new threads or use the SynchronizationContext object underneath. XNA framework calls Update and Draw methods one by one in the single thread many times per second and next iteration can start after the previous one has finished, so it’s really easy to debug such a single-threaded application. The problem is that, by default, Async CTP uses SynchronizationContext‘s Post method when the continuation delegate passed to TaskAwaiter object is called. Of course that happens if any context exists and in XNA case the “SynchronizationContext.Current” property returns the instance of the context. The latest version of Async CTP gives us the ability to configure that behavior but we would need to call “task.ConfigureAwait(false)” in every place where we use await keyword. That would be pretty inconvenient. So I have implemented my own extension method for Task object, that creates appropriately configured awaiter object.

```csharp
namespace AsyncXnaIntegration
{    
    public static class Extensions
    {
        public static ConfiguredTaskAwaitable.ConfiguredTaskAwaiter GetAwaiter(this Task task)
        {
            return task.ConfigureAwait(false).GetAwaiter();
        }

        public static ConfiguredTaskAwaitable<T>.ConfiguredTaskAwaiter GetAwaiter<T>(this Task<T> task)
        {
            return task.ConfigureAwait(false).GetAwaiter();
        }
    }
}
```

Now we need to find a way to force C# compiler to use our extension method instead of that provided in AsyncCtpLibrary_Phone.dll library. We can do it for example by placing “using AsyncXnaIntegration;” statement inside the namespace declaration in all files where we are using Async CTP. Thanks to that little trick our method will be “closer” than all others defined outside the namespace declaration.

Of course GameAwaiter class can be extended (and it most likely will be) by many other useful methods simplifying use of phone’s Keyboard, Geo-Position or Accelerometer APIs, but this is just a proof of concept. As always I encourage you to download the [source code](http://archive.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=mnajder&DownloadId=15375) and definitely play the Smiley game :) Thanks for reading this post and I hope it will open your eyes for many new very interesting scenarios where the Async CTP can change XNA game development.

[downlaod](http://archive.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=mnajder&DownloadId=15375)