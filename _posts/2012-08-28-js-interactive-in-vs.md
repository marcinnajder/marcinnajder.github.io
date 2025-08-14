---
layout: post
title: JavaScript Interactive in Visual Studio [JS]
date: 2012-08-28
tags:
  - csharp
series: asyncctp
---
The time has come. The time to learn JavaScript :) As usually when I‘m learning some new technology, I spent most of my time inside IDE like Visual Studio.  I’ve opened some good resources on the tab (this time it’s a book: [JavaScript: The Definitive Guide](http://shop.oreilly.com/product/9780596101992.do)), a raw text file with my notes on the other . You may ask: why inside IDE ? Because I always try to use new technologies I’m reading about in practice. The problem with JavaScript is that in most cases it runs inside web browser. But I don’t want to make any web pages to test some feature of JavaScript language. I just want to write a few lines of JavaScript code and execute them. So I need something called [REPL](http://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop) for JavaScript. REPL works very similar to Windows command line but it usually provides one great feature, the session state. In case of JavaScript that means that every time we execute some code we have access to all state - like variables, functions, etc. defined by previous executions. Each modern web browser has a tool called console where we can execute any piece of JavaScript code and its result is retuned immediately. Some tools like [Firebug](https://getfirebug.com/) go even further providing a nice code editor with syntax highlighting and intellisense support where we can select part of script and “send it to REPL”. Image below shows how we use such a tool.jsp

![jsrepl](/assets/images/jsrepl.png)

If we work with F# we can do the same thing directly from Visual Studio via F# Interactive windows. If we work with C# we have to install [Roslyn](http://msdn.microsoft.com/en-us/vstudio/roslyn.aspx) and we will get C# Interactive window. Unfortunately we don’t have anything similar for JavaScript (at least so far ;) ).

In this post I will show you how we can accomplish very similar behavior to that presented above with JavaScript language. Here is the full list of steps that need to be done:

1. Install [nodejs](http://nodejs.org/) (don’t be afraid, It’s free and very light - just a “single” exe file)
2. Create a new project in Visual Studio, for example “ASP.NET Empty Web Application” (my solution doesn’t work with Web Site projects)
3. Add a new template file called JsRepl.tt and copy its content from code below (this template doesn’t generate any output text so we optionally clear its default Custom Tool property using property grid)
4. Add a new JS file next to JsRepl.tt and set its Custom Tool property to TextTemplatingFileGenerator. A new file will be automatically added to the project under JS file. Open those two files one above the other, JS file is our input windows and generated file is our output window. Each time we save JS file, its content will be executed immediately.
5. Now we can write any JavaScript code in JS file except the first and the last line. The first line should look like this **//<#@ include file="JsRepl.tt"#>** . It’s a T4 directive that includes (pastes) template file we just added to the project. The last line specifies how we want to execute JS code:

- **//<# Generate();#>** – executes the whole JS file
- **//<# Generate(useClipboard:true);#>** – executes code from the clipboard so selecting the code and pressing Ctrl+C, Ctrl+S simulates sending code to REPL :)
- **//<# Generate(useClipboard:true, historyFile:"history.js");#>** – works the same as code above but additionally stores all previous successfully executed code inside specified helper JS file (here history.js). Every time we try to execute code from the clipboard, the one big file is created underneath and executed. This file combines the content of history file with the code from the clipboard. This is how we can easily simulate REPL session state mechanism :)

Let’s look at a sample usage:repl

![jsrepl2](/assets/images/jsrepl2.png)

The output window (ReplSample.txt file) displays the result of calling selected text from input windows (ReplSample.js file). Before that I did two additional executions, first one with text declaring _count_ variable, the second one with definition of the function _printNumber_. The history file looks like this:

![jsrepl3](/assets/images/jsrepl3.png)

Now it’s time to reveal the secret how the JS REPL in Visual Studio was implemented. JsRepl.tt file is the whole implementation:

```csharp
<#@ template hostspecific="true"#>  
<#@ output extension=".txt"#>  
<#@ assembly name="System.Windows.Forms" #>  
<#@ assembly name="System.Core" #>  
<#@ import namespace="System" #>  
<#@ import namespace="System.Diagnostics" #>  
<#@ import namespace="System.Text" #>  
<#@ import namespace="System.Windows.Forms" #>  
<#@ import namespace="System.IO" #>  
<#@ import namespace="System.Linq" #>  
<#@ import namespace="System.Threading" #>  
  
<#+  
    public void Generate(bool useClipboard = false, string historyFile = null)      
    {  
        GenerationEnvironment.Clear();  
          
        if(!useClipboard)  
        {              
            RunNode(Host.TemplateFile, WriteLine, WriteLine);  
        }  
        else  
        {  
            var text = GetClipboardText();  
            var tempFile = Path.GetTempFileName();  
  
            if(string.IsNullOrEmpty(historyFile))  
            {                                          
                File.WriteAllText(tempFile,text);  
                RunNode(tempFile, WriteLine, WriteLine);  
            }  
            else  
            {  
                var historyFilePath = Host.ResolvePath(historyFile);  
                if(!File.Exists(historyFilePath))  
                {  
                    WriteLine("Specified history file path '{0}' does not exist.", historyFilePath);  
                }  
                else  
                {                                          
                    var historyLines = File.ReadAllLines(historyFilePath);  
                    int? printedOutputLinesCount = ExtractOutputLinesCount(historyLines);  
                    long i = 0, max = printedOutputLinesCount ?? 0;  
                    var tempFileLines =                                                       
                        (printedOutputLinesCount == null ? new [] {"//0"} : new string[0])  
                        .Concat(historyLines)                              
                        .Concat(text.Split(new string[] {Environment.NewLine},StringSplitOptions.None))  
                        .Concat(new [] {new string('/',100)})                              
                        .ToArray();  
  
                    File.WriteAllLines(tempFile, tempFileLines);  
  
                    var result = RunNode(tempFile, s => { if(++i >= max) WriteLine(s); }, WriteLine );  
                  
                    if(result == 0) // ok  
                    {  
                        tempFileLines[0] = @"//" + i;  
                        File.WriteAllLines(historyFilePath, tempFileLines);  
                    }  
                }  
            }  
  
            if(File.Exists(tempFile))  
                File.Delete(tempFile);  
        }  
    }  
  
    private static int? ExtractOutputLinesCount(string[] allLines)  
    {  
        int count;  
        string firstLine = null;  
  
        return (allLines != null) && ((firstLine = allLines.FirstOrDefault()) != null) && int.TryParse(firstLine.Replace(@"//",""), out count) ?  
            (int?)count : null;  
    }  
  
    private static int RunNode(string jsFile, Action<string> onOutputDataReceived, Action<string> onErrorDataReceived)  
    {  
        var processStartInfo = new ProcessStartInfo(@"node", jsFile);  
  
        processStartInfo.RedirectStandardInput = true;  
        processStartInfo.RedirectStandardOutput = true;  
        processStartInfo.RedirectStandardError = true;  
        processStartInfo.CreateNoWindow = true;  
        processStartInfo.UseShellExecute = false;  
  
        var process = Process.Start(processStartInfo);  
  
        process.OutputDataReceived += (sender, args) => onOutputDataReceived(args.Data);  
        process.ErrorDataReceived += (sender, args) =>  {  if(args.Data!=null) onErrorDataReceived(args.Data); };  
        process.BeginOutputReadLine();  
        process.BeginErrorReadLine();  
        process.WaitForExit();  
        return process.ExitCode;  
    }  
      
    private static string GetClipboardText()  
    {  
        try  
        {  
            return Clipboard.GetDataObject().GetData(DataFormats.Text) as string;          
        }  
        catch  
        {  
            return "";  
        }  
    }  
#>
```

There are a couple of things worth explanation here. You may be wondering how I actually execute JavaScript code outside of the browser ? I use node.exe console application passing JavaScript file path as an argument and I listen what result is printed. T4 template mechanism use _WriteLine_ method to write something to the input file. This method appends passed string argument to the internal StringBuilder object stored in property called _GenerationEnvironment_. By default each line of the template file (JS file in this case) which is not a directive or control flow statement (code inside regions <#, <#+ or <#= ) is passed to the _WriteLine_ method. That’s why the main method called _Generate_ is placed at the end of template file and clears everything that was written before its execution by calling StringBuilder _Clear_ method. Because the whole history file is executed from the beginning to the end but we want to see results only of the latest part (that from the clipboard) I use special counter stored in the first line in history file. This counter represents a number of lines printed during history file execution and it’s used to suspend printing process till the appropriate moment when the code from the clipboard is executed.

And that’s all. Simple as it is I hope you find it useful like I did :)



