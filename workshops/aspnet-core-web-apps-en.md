---
layout: page
title: Building web applications with ASP.NET Core
---
# Building web applications with ASP.NET Core

Please register [here](https://www.comarch.pl/szkolenia/programowanie/net-c/twarz-aplikacji-internetowych-z-wykorzystaniem-aspnet-core/) or let me know directly
### Opis

The training aims to familiarize participants with the cross-platform ASP.NET Core framework. In addition to theoretical knowledge, participants will acquire practical skills to build a structured, compact, and logical application using the MVC design pattern, which forces the application into three independent layers: the data model, the graphical interface, and the operational logic. The HTTP protocol is not only used for returning web pages, so participants will also acquire skills in designing and building Web APIs using the REST architecture.

Duration: 3 days
### Details

- Introduction to .NET Core
	- What is .NET Standard 2.0 (cross-platform, architecture)
	- Tools (msbuild, cmd, nuget, docker)
	- A brief history of ASP.NET
- Entity Framework Core
	- Data sources used in ASP.NET Core
	- Describing the model using POCO entities
	- CRUD - Creating a relational database from the model, retrieving and modifying data using Entity Framework Code First
- ASP.NET Core - Basics
	- Hosting (kestrel, configuration)
	- Dependency Injection
	- Repository Pattern
	- Middleware
	- Overview of built-in middleware (logging, error handling, CORS, serving static files)
	- Principles of routing mechanisms
	- Areas of application of routing mechanisms (Areas)
	- Processing HTTP requests
	- Building HTTP requests
	- REST architecture
	- Filters (using existing filters, creating custom filters)
- ASP.NET Core - MVC
	- MVC Architecture
	- Basic mechanisms for building controllers in the MVC architecture
	- The ActionResult class and its use in controllers
	- Asynchronous controller operations using Task types
	- Using the ViewData and TempData classes to improve controllers
	- View
		- Methods of defining views
		- Defining page layout
		- Razor syntax
		- Typed and untyped views
		- HTML helper methods
		- Templates
		- Sections
		- Partial views and child action mechanisms
- ASP.NET Core - Advanced
	- Application Testing
		- Designing applications with the testing process in mind
		- The process of integrating unit tests with the application
		- The usefulness of the Dependency Injection and Repository patterns in application testing
	- Security
		- Authentication
		- Authorization
		- Protection against XSS and CSRF
	- Kestrel/IIS hosting
		- Application deployment methods
		- Configuration
		- dotnet publish-iis
	- Caching Mechanisms
	- Client-Side Development, Node Services