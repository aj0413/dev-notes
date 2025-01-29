# REST API Service Guidelines

An opinionated guidline of basic coding tools and practices to use when creating a REST API service.

Quicklinks:
- [DOs and DO NOTs](#dos-and-do-nots)
- [Tools and Libraries](#tools-and-libraries)

## DOs and DO NOTs

### DO NOT use controllers

Controllers are historically part of the MVC framework/pattern of doing things in dotnet, but create performance and feature overhead that isn't needed in pure RESTful services the majority of the time.

Additionally, devs should start thinking in terms of **[REPR Design Pattern (Request-Endpoint-Response)](https://deviq.com/design-patterns/repr-design-pattern)**.

MSFT intorduced **[Minimal APIs](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis/overview?view=aspnetcore-9.0)** to address the above, but I found many legacy .Net devs are uncomfortable with the more functional programming style that introduces.

[Here's a side-by-side from MSFT.](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/apis?view=aspnetcore-9.0)

### DO NOT use Swashbuckle for Swagger

[Swashbuckle.AspNetCore is being removed in .NET 9](https://github.com/dotnet/aspnetcore/issues/54599) and devs need to be prepared for switching to a different library for this due to the project being effectively dead; [NSwag](https://github.com/RicoSuter/NSwag) is a popular alternative that also allows for client side generation of code from Swagger docs.

MSFT will be extending it's `Microsoft.AspNetCore.OpenApi` library to provide OpenAPI document generation, but a Swagger UI tool will still be needed. 

Devs should read [MSFT docs on Swagger vs OpenAPI](https://learn.microsoft.com/en-us/aspnet/core/tutorials/web-api-help-pages-using-swagger?view=aspnetcore-8.0#openapi-vs-swagger) to understand how this workds under the hood.

### DO NOT use NewtonSoft.Json

MSFT has it's own [System.Text.Json (STJ)](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/overview) library for handeling JSON serialization and deserialization.

There's no reason to bloat a project's dependencies and introduce vectors for security vulnerabilities and bugs by using a third party for this anymore.

Further, STJ is more performant than Newtonsoft.Json and is the default mechanism the vast majority of the .Net ecosystem now uses for things.

### DO use Command Query Responsibility Segregation (CQRS)

[Heres an article explaining CQRS](https://www.milanjovanovic.tech/blog/cqrs-pattern-with-mediatr#what-exactly-is-cqrs), but, essentially, it boils down to thinking of operations in terms of commands (changing system state and maybe returns data) and queries (doesn't change system, but always returns data), which should each be handled differently.

A RESTful API service is already broken down into commands (POST, DELETE, PUT) and queries (GET), so using CQRS should be natural.

[MediatR](https://github.com/jbogard/MediatR) is one of the most widly used tools to implement this, but as long as you're breaking down classes into individual commands and queries to be handled by associated handler it's not needed.

NOTE: Think of this as making individual objects to represent the methods on a traditional service/business class.

### DO use Vertical Slice Architechure (VSA)

[As described in this article](https://www.milanjovanovic.tech/blog/vertical-slice-architecture), the project structure is about thinking in terms of features instead of layers.

The goal is to organize code not by what it is, but by what *feature* is pertains to. Shared items, Middleware, Database stuff, etc... can still go into its own spaces, but combining REPR with CQRS and you start seeing how each endpoint is it's own vertical slice of components/logic for grouping.

This simplifies PRs (only need to look at files containing code for a specific feature), maintaince and debugging, and adding/moving/removing features from a codebase. It also greatly simplifies code files by focusing them; the cognitive complexity and overhead of 1k+ files is too much for most people to parse and fully keep in mind.

NOTE: A repository would just go in the feature folder, if you decided to make one. Again, the goal here is to avoid large generic classes and organize code into what belongs to together.

![example-image](vsa-example.png)

## Tools and Libraries

All of the following barring FastEndpoints (it's still too new) are generally considered best in class for their use cases.

### [FastEndpoints](https://github.com/FastEndpoints/FastEndpoints)

This is a FOSS library built from the ground up for RESTful API services and helping devs 'fall into the pit of success'.

It does many things. Including, but not limited too:
- Making testing of endpoints easier using tools built off [XUnit](https://github.com/xunit/xunit)
- Allowing for use of Attributes to register classes for Dependency Injection
- Provides traditional classes for Minimal APIs
- Fascilitates the REPR and VSA patterns 
- Uses [FluentValidation](https://docs.fluentvalidation.net) for validating requests out-of-the box
- Uses [NSwag](https://github.com/RicoSuter/NSwag) for Swagger UI generation
- Uses [System.Text.Json (STJ)](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/overview) for JSON serialization and deserialization
- Provides an ICommand and ICommandHandler for implementing CQRS
- API Versioning
- Rate Limiting
- Response Caching
- Global Exception Handler
- Pre / Post Processors
- etc....

There's a bunch of [documentation](https://fast-endpoints.com/docs/get-started) covering the above and more.

The main point of this project is that it basically gives you everything you need out-of-the-box to create a good API.

### [AspNetCore.Diagnostics.HealthChecks](https://github.com/Xabaril/AspNetCore.Diagnostics.HealthChecks)

This is a FOSS library for building in healthchecks for an application quickly and easily. The healthcheck implementations generate their own logs, provide a means of exposing a reporting function, and support a wide variety of platforms.

I've seen too many devs try and handroll how healthchecks should work and it's created an inconsistency across teams and projects. Additionally, the vast majority of those implementations have some issue or other. Why re-invent the wheel?

### [Duende.AccessTokenManagement](https://github.com/DuendeSoftware/foss/tree/main/access-token-management)

This is a FOSS library for automatic handeling of access tokens. Especially convenvient when you need to attach these tokens to requests to other backend services.

> This directory contains a set of .NET libraries that manage OAuth and OpenId Connect access tokens. These tools automatically acquire new tokens when old tokens are about to expire, provide conveniences for using the current token with HTTP clients, and can revoke tokens that are no longer needed.

### [Refit](https://github.com/reactiveui/refit)

This is a FOSS library for generating type-safe and performant REST clients (using HttpClient internally) to call a REST API.

Too many devs break best practices when using HttpClient directly:
- Directly instatiating one instead of letting the runtime dictate lifetime
- Binding requests to `object` instead of an actual type
- Messing with the encoding directly and confusing recipients of requests
- And more....

This library was chosen both because it's convenient and easy to use, but also because it removes the burden on devs on thinking above stuff related to how HttpClient works and how they're suppose to use it.

### [Serilog](https://github.com/serilog/serilog)

This is a FOSS library for handeling logging, in all its various aspects, but it can also be used to trigger external systems (ex. PagerDuty) as well.

> Serilog is a diagnostic logging library for .NET applications. It is easy to set up, has a clean API, and runs on all recent .NET platforms. While it's useful even in the simplest applications, Serilog's support for structured logging shines when instrumenting complex, distributed, and asynchronous applications and systems.

### [FluentValidation](https://docs.fluentvalidation.net/)

This is a FOSS library for providing an easier and more declaritive way for applying validation rules to an object.

> FluentValidation is a .NET library for building strongly-typed validation rules.

### [NSwag](https://github.com/RicoSuter/NSwag)

This is a FOSS *toolchain* more than it is a library. 

While it's most basic use can be for Swagger UI generation similar to Swashbuckle, it also can handle client code generation for consuming your REST API endpoints based on the OpenAPI spec it creates.

### [XUnit](https://github.com/xunit/xunit)

This is a FOSS library for writing tests in .Net.

### [FluentAssertions](https://github.com/fluentassertions/fluentassertions)

This is a FOSS library for asserting outcomes of tests.

### [Moq](https://github.com/devlooped/moq)

This is a FOSS library for mocking/stubbing objects in .Net.

### [Coverlet](https://github.com/coverlet-coverage/coverlet)

This is a FOSS library for generating code coverage reports based on tests.

> Coverlet is a cross platform code coverage framework for .NET, with support for line, branch and method coverage. It works with .NET Framework on Windows and .NET Core on all supported platforms.