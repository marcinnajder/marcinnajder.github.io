---
layout: post
title: .NET project overview [F#]
date: 2023-08-16
tags: 
series: immutable-list-csharp1
---
Let's say we have a quite big .NET project we have never seen before. A microservice sample [eShop](https://github.com/dotnet/eShop/tree/main) project would be a good example. If we open up the solution in Visual Studio, we will see more than 20 different C# projects.

The first questions that come to my mind are:
- Where should I start scanning the project structure?
- Which projects are the most important?
- What are the dependencies between projects and external NuGet packages?

I wrote a small F# script to help me answer those questions. Each C# project in .NET has its project `*.csproj` XML file, and all projects are bound together using a solution `*.sln` text file. My script takes a solution source path, parses all necessary files, and prints an overview to the console. 

I will say a few words about the F# language in this article. I love the simplicity and clarity of the code written in F#. Some say functional code is read "from the top to bottom, from the left to right." I try to explain what they mean by that.

The script is a single F# `PrintSolutionSummary.fsx` file. Let's assume we have installed .NET and downloaded a .NET project, which we will analyze, as well as the script file. We run the script by executing the following command: `dotnet fsi PrintSolutionSummary.fsx "/Volumes/data/github/dotnet9eshop/eShop/eShop.sln"` from the terminal. Of course, your eShop project can look different when you execute the script; my output looks like this:

```txt
Solution file: /Volumes/data/github/dotnet9eshop/eShop/eShop.sln

-- Projects
eShop.ServiceDefaults - 
EventBus - 
Ordering.Domain - 
WebAppComponents - 
ClientApp - 
EventBusRabbitMQ - EventBus
Identity.API - eShop.ServiceDefaults
IntegrationEventLogEF - EventBus
Mobile.Bff.Shopping - eShop.ServiceDefaults
WebhookClient - eShop.ServiceDefaults
HybridApp - WebAppComponents
ClientApp.UnitTests - ClientApp
Basket.API - eShop.ServiceDefaults, EventBusRabbitMQ
Catalog.API - EventBusRabbitMQ, IntegrationEventLogEF, eShop.ServiceDefaults
OrderProcessor - eShop.ServiceDefaults, EventBusRabbitMQ
Ordering.Infrastructure - IntegrationEventLogEF, Ordering.Domain
PaymentProcessor - eShop.ServiceDefaults, EventBusRabbitMQ
WebApp - eShop.ServiceDefaults, EventBusRabbitMQ, WebAppComponents
Webhooks.API - EventBusRabbitMQ, IntegrationEventLogEF, eShop.ServiceDefaults
Ordering.API - EventBusRabbitMQ, IntegrationEventLogEF, eShop.ServiceDefaults, Ordering.Domain, Ordering.Infrastructure
Basket.UnitTests - Basket.API
Catalog.FunctionalTests - Catalog.API
eShop.AppHost - Mobile.Bff.Shopping, Basket.API, Catalog.API, Identity.API, Ordering.API, OrderProcessor, PaymentProcessor, Webhooks.API, WebApp, WebhookClient
Ordering.FunctionalTests - Ordering.API, Identity.API, Ordering.Domain, Ordering.Infrastructure
Ordering.UnitTests - Ordering.API, Ordering.Domain, Ordering.Infrastructure

-- Packages
- eShop.ServiceDefaults
 - Asp.Versioning.Mvc.ApiExplorer
 - Microsoft.AspNetCore.OpenApi
 - Scalar.AspNetCore
 - Microsoft.AspNetCore.Authentication.JwtBearer
 - Microsoft.Extensions.Http.Resilience
 - Microsoft.Extensions.ServiceDiscovery
 - OpenTelemetry.Exporter.OpenTelemetryProtocol
 - OpenTelemetry.Extensions.Hosting
 - OpenTelemetry.Instrumentation.AspNetCore
 - OpenTelemetry.Instrumentation.GrpcNetClient
 - OpenTelemetry.Instrumentation.Http
 - OpenTelemetry.Instrumentation.Runtime

- EventBus
 - Microsoft.Extensions.DependencyInjection.Abstractions
 - Microsoft.Extensions.Options

- Ordering.Domain
 - MediatR
 - System.Reflection.TypeExtensions

- WebAppComponents
 - Microsoft.AspNetCore.Components.Web

- ClientApp
 - Google.Protobuf@3.29.3
 - Grpc.Net.Client@2.67.0
 - Grpc.Tools@2.69.0
 - IdentityModel.OidcClient@6.0.0
 - Microsoft.Maui.Controls@9.0.30
 - Microsoft.Maui.Controls.Compatibility@9.0.30
 - Microsoft.Maui.Controls.Maps@9.0.30
 - Microsoft.Extensions.Logging.Debug@9.0.0
 - CommunityToolkit.Maui@9.1.1
 - IdentityModel@7.0.0
 - CommunityToolkit.Mvvm@8.3.2

- EventBusRabbitMQ
 - Aspire.RabbitMQ.Client
 - Microsoft.Extensions.Options.ConfigurationExtensions
 - Polly.Core

- Identity.API
 - Duende.IdentityServer.AspNetIdentity
 - Duende.IdentityServer.EntityFramework
 - Duende.IdentityServer.Storage
 - Duende.IdentityServer
 - Microsoft.AspNetCore.Identity.EntityFrameworkCore
 - Microsoft.AspNetCore.Identity.UI
 - Microsoft.EntityFrameworkCore.Tools
 - Aspire.Npgsql.EntityFrameworkCore.PostgreSQL
 - Microsoft.Web.LibraryManager.Build

- IntegrationEventLogEF
 - Npgsql.EntityFrameworkCore.PostgreSQL

- Mobile.Bff.Shopping
 - Yarp.ReverseProxy
 - Microsoft.Extensions.ServiceDiscovery.Yarp
 - AspNetCore.HealthChecks.Uris

- WebhookClient
 - Asp.Versioning.Http.Client
 - Microsoft.AspNetCore.Authentication.OpenIdConnect
 - Microsoft.AspNetCore.Components.QuickGrid

- HybridApp
 - Microsoft.AspNetCore.Components.WebView.Maui@9.0.30
 - Microsoft.Maui.Controls@9.0.30
 - Microsoft.Maui.Controls.Compatibility@9.0.30
 - Microsoft.Extensions.Http@9.0.0
 - Microsoft.Extensions.Logging.Debug@9.0.0

- ClientApp.UnitTests
 - Microsoft.Maui.Controls@9.0.40
 - Microsoft.Maui.Controls.Compatibility@9.0.40
 - Microsoft.Maui.Controls.Maps@9.0.40
 - Microsoft.NET.Test.Sdk@17.13.0
 - coverlet.collector@6.0.4
 - MSTest.TestAdapter@3.8.2
 - MSTest.TestFramework@3.8.2

- Basket.API
 - Aspire.StackExchange.Redis
 - Grpc.AspNetCore

- Catalog.API
 - Asp.Versioning.Http
 - Aspire.Npgsql.EntityFrameworkCore.PostgreSQL
 - CommunityToolkit.Aspire.OllamaSharp
 - Microsoft.EntityFrameworkCore.Tools
 - Microsoft.Extensions.ApiDescription.Server
 - Aspire.Azure.AI.OpenAI
 - Microsoft.Extensions.AI
 - Microsoft.Extensions.AI.OpenAI
 - Pgvector
 - Pgvector.EntityFrameworkCore

- OrderProcessor
 - Aspire.Npgsql

- Ordering.Infrastructure
 - Npgsql.EntityFrameworkCore.PostgreSQL

- PaymentProcessor


- WebApp
 - Asp.Versioning.Http.Client
 - Aspire.Azure.AI.OpenAI
 - CommunityToolkit.Aspire.OllamaSharp
 - Microsoft.Extensions.ServiceDiscovery.Yarp
 - Microsoft.Extensions.AI
 - Microsoft.Extensions.AI.OpenAI
 - Microsoft.AspNetCore.Authentication.OpenIdConnect
 - Google.Protobuf
 - Grpc.Net.ClientFactory
 - Grpc.Tools

- Webhooks.API
 - Asp.Versioning.Http
 - Aspire.Npgsql.EntityFrameworkCore.PostgreSQL
 - Microsoft.EntityFrameworkCore.Tools

- Ordering.API
 - Asp.Versioning.Http
 - Aspire.Npgsql.EntityFrameworkCore.PostgreSQL
 - FluentValidation.AspNetCore
 - Microsoft.EntityFrameworkCore.Tools

- Basket.UnitTests
 - Microsoft.AspNetCore.Mvc.Testing
 - Microsoft.Extensions.Identity.Stores
 - Microsoft.NET.Test.Sdk
 - MSTest.TestAdapter
 - MSTest.TestFramework
 - NSubstitute
 - NSubstitute.Analyzers.CSharp

- Catalog.FunctionalTests
 - Asp.Versioning.Http.Client
 - Aspire.Hosting.AppHost
 - Aspire.Hosting.PostgreSQL
 - Microsoft.AspNetCore.Mvc.Testing
 - Microsoft.AspNetCore.TestHost
 - Microsoft.NET.Test.Sdk
 - xunit.runner.visualstudio
 - xunit

- eShop.AppHost
 - Aspire.Hosting.AppHost
 - Aspire.Hosting.RabbitMQ
 - Aspire.Hosting.Redis
 - Aspire.Hosting.PostgreSQL
 - Aspire.Hosting.Azure.CognitiveServices
 - CommunityToolkit.Aspire.Hosting.Ollama

- Ordering.FunctionalTests
 - Asp.Versioning.Http.Client
 - Aspire.Hosting.AppHost
 - Aspire.Hosting.PostgreSQL
 - Microsoft.AspNetCore.Mvc.Testing
 - Microsoft.NET.Test.Sdk
 - Microsoft.AspNetCore.TestHost
 - xunit.runner.visualstudio
 - xunit

- Ordering.UnitTests
 - Microsoft.NET.Test.Sdk
 - MSTest.TestAdapter
 - MSTest.TestFramework
 - NSubstitute
 - NSubstitute.Analyzers.CSharp
```

There are two main sections: `-- Projects` and `-- Packages`. The first one displays information only about projects with project dependencies that are part of the solution, while the second one displays the dependent NuGet packages.

We cannot define circular dependencies between projects in .NET, so the project structure can be presented as a tree, not a graph. Let's take a look at the projects once again.

```text
-- Projects
eShop.ServiceDefaults - 
EventBus - 
Ordering.Domain - 
WebAppComponents - 
ClientApp - 
EventBusRabbitMQ - EventBus
Identity.API - eShop.ServiceDefaults
IntegrationEventLogEF - EventBus
Mobile.Bff.Shopping - eShop.ServiceDefaults
WebhookClient - eShop.ServiceDefaults
HybridApp - WebAppComponents
ClientApp.UnitTests - ClientApp
Basket.API - eShop.ServiceDefaults, EventBusRabbitMQ
Catalog.API - EventBusRabbitMQ, IntegrationEventLogEF, eShop.ServiceDefaults
OrderProcessor - eShop.ServiceDefaults, EventBusRabbitMQ
Ordering.Infrastructure - IntegrationEventLogEF, Ordering.Domain
PaymentProcessor - eShop.ServiceDefaults, EventBusRabbitMQ
WebApp - eShop.ServiceDefaults, EventBusRabbitMQ, WebAppComponents
Webhooks.API - EventBusRabbitMQ, IntegrationEventLogEF, eShop.ServiceDefaults
Ordering.API - EventBusRabbitMQ, IntegrationEventLogEF, eShop.ServiceDefaults, Ordering.Domain, Ordering.Infrastructure
Basket.UnitTests - Basket.API
Catalog.FunctionalTests - Catalog.API
eShop.AppHost - Mobile.Bff.Shopping, Basket.API, Catalog.API, Identity.API, Ordering.API, OrderProcessor, PaymentProcessor, Webhooks.API, WebApp, WebhookClient
Ordering.FunctionalTests - Ordering.API, Identity.API, Ordering.Domain, Ordering.Infrastructure
Ordering.UnitTests - Ordering.API, Ordering.Domain, Ordering.Infrastructure
```

Projects are sorted in a particular order. At the top are projects without references, like leaves in a tree project structure—for example, the `Ordering.Domain` project represents the domain using the DDD approach, and the `eShop.ServiceDefaults`, which is part of the .NET Aspire infrastructure. The projects below are referencing the projects above, like `Basket.API - eShop.ServiceDefaults, EventBusRabbitMQ`. One project has a reference to two other projects.

I don't want to go into too much detail about the eShop example specifically, but by looking at project names and dependencies, we can guess that projects named with `API` at the end represent REST or gRPC endpoints. Some functionality is split into many projects, such as `Ordering`, which is implemented in `Ordering.API` pointing to `Ordering.Domain` and `Ordering.Infrastructure`. This solution also includes unit and functional tests.

In the second part, Packages, we can quickly scan what external packages are used and by which projects. The list of packages is sorted by name; projects from the solution are not included. We immediately can see that databases like PostgreSQL, Redis, and RabbitMQ are used, popular libraries like FluentValidation, MediatR, Polly, and Yarp are used, and for unit tests, NSubstitute, xunit, and coverlet packages.

As a new developer on the team, I already know a lot about the projects just by looking at the information above. Furthermore, we can decide in which order we want to analyze the code, from the top to bottom, or the opposite. 

Now we will discuss the script source code from the beginning to the end.

```fsharp
open System
open System.IO
open System.Text.RegularExpressions
open System.Xml.Linq

let args = fsi.CommandLineArgs
let slnFilePath = if args.Length > 1 then args.[1] else "/Volumes/data/github/dotnet9eshop/eShop/eShop.sln"

type Project = { Path: string; ProjectRefs: string []; PackageRefs: string [] }

// string -> string
let adjustPath (path: string) =
    let segments = path.Split([| '/'; '\\' |], StringSplitOptions.RemoveEmptyEntries)
    Path.Combine segments

// string -> string -> string
let joinPath folderPath filePath = Path.GetFullPath(Path.Combine(folderPath, adjustPath (filePath)))

// string -> option<string>
let parseSolutionLine line =
    Regex.Matches(line, "\"(.+?)\"")
    |> Seq.cast<Match>
    |> Seq.map (fun m -> m.Value.Trim("\""[0])) // fix markdown :) was '"'
    |> Seq.tryPick (fun s -> if s.EndsWith(".fsproj") || s.EndsWith(".csproj") then Some s else None)

// string -> []<string>
let parseSolutionFile (solutionFilePath: string) =
    let solutionFolderPath = Path.GetDirectoryName(solutionFilePath)
    File.ReadAllLines(solutionFilePath)
    |> Seq.choose parseSolutionLine
    |> Seq.map (joinPath solutionFolderPath)
    |> Seq.toArray

// string -> Project
let parsePojectFile (projectFilePath: string) =
    let projectFolderPath = Path.GetDirectoryName(projectFilePath)
    let xmlRoot = XElement.Load(projectFilePath)
    let projectRefs =
        xmlRoot.Descendants("ProjectReference")
        |> Seq.map (fun xel -> xel.Attribute("Include").Value)
        |> Seq.map (joinPath projectFolderPath)
        |> Seq.toArray
    let packageRefs =
        xmlRoot.Descendants("PackageReference")
        |> Seq.map (fun xel ->
            String.Join(
                "@",
                [ "Include"; "Version" ]
                |> Seq.choose (fun a -> xel.Attribute(a) |> Option.ofObj |> Option.map (fun aa -> aa.Value))
            ))
        |> Seq.toArray
    { Path = projectFilePath; ProjectRefs = projectRefs; PackageRefs = packageRefs }
```

The first part of the script is responsible for parsing project and solution files. The main function is `parsePojectFile`. It takes the path to the project XML file and returns a record type `Project`, which stores information about the whole source file path and dependent projects and packages. The NET solution file `.sln` is not formatted as XML (the new `slnx` format is), so it's read and parsed line by line using regex expressions. 

F# is a statically typed language with advanced type inference capabilities. For instance, argument and result types can be inferred from the function's body. I left some extra comments on the function signature for readability. It's not entirely correct, but the code `string -> string -> string` can be read as "takes two strings and returns a string". Variables and functions are defined the same way using the `let` keyword. There are no curly brackets designating blocks of code; similarly to Python, indentations matter. The last F# seen everywhere is pipe operator `|>`, instead of calling nested functions like `func2(func1(val))`, we can write `val |> func1 |> func2`. `Seq` is a module (like a static class in C#) that contains helper functions working with the `seq` type (alias to `IEnumerable<T>` type), so it's very similar to the `Enumerable` static class providing LINQ operators.  

What does reading functional code "from the top to bottom, from the left to right" mean? F# is immutable by default; variables (values), objects, or collections can not be changed after creation. Once we read and understand a particular line of code defining a variable using `let` keyword, we don't have to search for other places changing them. It is not possible. There are no loops mutating the program's state; we use higher-order functions like `map, filter, reduce` to calculate new data. The order of files and even members inside the files matters. We can use variables or functions defined only above in the same file. It is easy to find them. There is a lot of jumping between many lines of code in typical imperative programming with variables, if-then-else, and loops. In each moment, anything can be changed. Such a code is much harder to understand and maintain. 

The following two functions are the more interesting ones.

```fsharp
// list<Project> -> list<Project>
let orderProjectsByRefs projects =
    let rec loop projs paths =
        match projs with
        | [] -> []
        | _ ->
            let referenced, referencing =
                List.partition (fun p -> Seq.forall (fun n -> Set.contains n paths) p.ProjectRefs) projs
            referenced @ loop referencing (referenced |> Seq.fold (fun names' p -> Set.add p.Path names') paths)
    loop projects Set.empty

// string -> seq<string> -> string
let joinStrings (sep: string) (items: seq<_>) = String.Join(sep, items)

// list<string> -> string
let findPrefix (paths: string list) =
    match paths with
    | []
    | _ :: [] -> ""
    | firstPath :: restPaths ->
        let segments = firstPath.Split('.')
        { segments.Length - 1 .. -1 .. 1 }
        |> Seq.map (fun n -> segments |> Seq.take n |> joinStrings ".")
        |> Seq.tryFind (fun path -> restPaths |> Seq.forall (fun p -> p.Length <> path.Length && p.StartsWith(path)))
        |> Option.defaultValue ""

```


The `orderProjectsByRefs` function takes an immutable list of projects and returns a new list of projects. It sorts the projects by the level in the dependency tree from the bottom to the top. The first on the list are projects without dependencies, followed by projects pointing to them, and so on. Defining the internal recursive `loop` function representing a loop is a standard pattern in functional code. The `paths` argument represents the state "change inside the loop", a set of all already checked project paths. `List.partition` is a built-in collection operator that splits a list of items into two lists according to a passed predicate function. The `referenced` variable represents projects pointing to already processed projects stored in `paths` set. They are appended to the returned result using `@` operator. The `referencing` stores the remaining projects, which are passed to the next iteration of the loop.

.NET projects often follow a naming convention where all projects start with the same prefix, for example, `MyCompany.MyProject`. The function `findPrefix` takes a list of project names, like  `MyCompany.MyProject.Domain`, `MyCompany.MyProject.Intrastructure`, `MyCompany.MyProject.Shared`, and finds the common prefix `MyCompany.MyProject`. This function is used in the last part of the script where the final result is formatted and printed to the console.

```fsharp
let projects = parseSolutionFile slnFilePath |> Seq.map parsePojectFile |> Seq.toList |> orderProjectsByRefs

let getProjectFullName (path: string) = Path.GetFileNameWithoutExtension(path)

let prefix = projects |> List.map (fun p -> getProjectFullName p.Path) |> findPrefix

let getProjectShortName (path: string) =
    if prefix = "" then getProjectFullName path else (getProjectFullName path).Substring(prefix.Length + 1)

let projectsContent =
    projects
    |> Seq.map (fun p ->
        $"""{getProjectFullName p.Path} - {p.ProjectRefs |> Seq.map getProjectShortName |> joinStrings ", "}""")
    |> joinStrings Environment.NewLine

let formatPackagesContent p =
    $"""- {getProjectFullName p.Path}
{p.PackageRefs |> Seq.map (fun p -> " - " + p) |> joinStrings Environment.NewLine}"""

let packagesContent =
    projects |> Seq.map formatPackagesContent |> joinStrings (Environment.NewLine + Environment.NewLine)

let content =
    $"""
Solution file: {slnFilePath}

-- Projects
{projectsContent}

-- Packages
{packagesContent}
"""

printfn "%s" content
```

In the last part, a few variables were calculated using previously defined functions, the final string content was built with simple string concatenation, and it was printed to the console.

Complete script is available [here](https://github.com/marcinnajder/misc/blob/master/2023_08_16_print_solution_summary/PrintSolutionSummary/PrintSolutionSummary.fsx "PrintSolutionSummary.fsx") 