---
title:  "Some tips when migrating your project to .NET Core"
date: 2020-08-25 10:54
categories: dotnet
redirect_from:
  - /2020/08/25/some-tips-when.html
---

Sometimes, moving your project from .NET Framework to .NET Core can be super easy, and sometimes it can be a daunting task. There's several factors that can complicate matters:

* Having .NET Framework only-components like Windows Communication Foundation (WCF) or Workflow Foundation in your project.
* The project is heavily based on ASP.NET with a lot of HTTP modules and other ASP.NET (Framework version)-only constructs.
* The project uses ASP.NET WebForms, which may never be supported.
* The project is simply HUGE and complicated in itself.
* Using third-party components that are not compatible with .NET Core.

We're going through the motions of porting our main software product to .NET Core in my current organization at the moment and figured I'll share a few tips based on our learnings so far. 

To set the scene, we develop a .NET based case and document management solution that is used by local, regional and national governments and organizations in the nordics. It's developed on .NET Framework, has 20 years of legacy, and ticks all the boxes above. We're porting it to .NET Core for cross-platform hosting, future proofness and performance. One of the goasl is to run our software on Linux containers on Kubernetes, instead of having to rely on Windows-based containers, which has its issues.

Now... let's take this one point at a time. Note that you won't see a package version on the package references. Feel free to add that as you see fit. We use centralized package versions: [https://github.com/microsoft/MSBuildSdks/tree/master/src/CentralPackageVersions](https://github.com/microsoft/MSBuildSdks/tree/master/src/CentralPackageVersions)

## Porting huge solutions
You may have a monolithic application that is hard to port all at once. You may need to maintain it and release new functionality while you are doing the work on porting it, and porting it may be complex and take a long time itself. 

What we've done here is:
* Convert all project files (.csproj files) to the new project format introduced in Visual Studio 2017. Here's a quick guide on how to do this: [https://natemcmaster.com/blog/2017/03/09/vs2015-to-vs2017-upgrade/](https://natemcmaster.com/blog/2017/03/09/vs2015-to-vs2017-upgrade/). Apart from ASP.NET web projects, this can be done without changing the target framework.
* For one project at a time, you can add both .NET Framework AND .NET Core (or .NET Standard) as build targets, and then just go through each of the build errors as needed. Note that if you previously used a "TargetFramework" element, you need to change this to "TargetFrameworks" (plural): 
  ```xml
  <TargetFrameworks>net48;netcoreapp3.1</TargetFrameworks>
  ```
* Sometimes you may need to have different code for .NET Core and .NET Framework. For instance, there may be libraries that should only be used for .NET Core, while you use framework assemblies in .NET Framework. In those cases, you can use `#if` directives to tell the compiler to compile that code only for specific frameworks. Example: 
```csharp
#if NETFRAMEWORK
//.NET Framework-specific code here
#else
//.NET Core-specific code here
#endif
```
* Related to the previous point - sometimes, there are NuGet packages, or assembly references, that only make sense for .NET Core, and some that only make sense to .NET Framework. In your project file, these can be conditionally included like this:
  ```csharp
  <ItemGroup Condition="!$(TargetFramework.StartsWith('net4'))">
    <PackageReference Include="System.Configuration.ConfigurationManager" />
    <PackageReference Include="SoapCore" />
  </ItemGroup>
  ```
  Here we include the SoapCore and System.Configuration.ConfigurationManager packages only if **not** compiling for .NET Framework 4+.

## Porting ASP.NET code to ASP.NET Core while staying compatible with both .NET Framework and Core
ASP.NET and ASP.NET Core is quite different. It's a different way of initializing the application, different classes that area used for controllers, no more HTTP modules, and the list goes on. Therefore, when porting ASP.NET code, what we decided was to split our ASP.NET project into three projects:

* One ASP.NET specific project, which contains the plumbing, lifecycle hooks for ASP.NET, HTTP modules and other .NET Framework specific constructs. This project targets only .NET Framework.
* One ASP.NET Core specific project, which contains the plumbing for ASP.NET Core (startup/configuration code, etc.), middleware, and other ASP.NET Core-specific code. This project only targets .NET Core.
* One project that contains all the application logic. This includes API controllers and all the logic that is defining your actual application. Add this project as a reference to the other two projects, and make sure this common project targets both .NET Core and .NET Framework.

The first two projects is fairly straight forward. However, the common project needs some thought, because ASP.NET Core controllers are a completely different type from the ASP.NET controllers. Thankfully, there's a neat package: Microsoft.AspNetCore.Mvc.WebApiCompatShim. This allows you to re-use the typenames for ASP.NET controllers, and act as an adapter between ASP.NET Core & ASP.NET and your controller definitions. So here's some steps you can take for this project:

* Include the ASP.NET and ASP.NET Core packages conditionally, including the WebApiCompatShim package (which has the required ASP.NET Core packages as dependencies):
  ```xml
  <ItemGroup Condition="$(TargetFramework.StartsWith('net4'))">
    <PackageReference Include="Microsoft.AspNet.WebApi.WebHost" />
  </ItemGroup>
  <ItemGroup Condition="!$(TargetFramework.StartsWith('net4'))">
    <PackageReference Include="Microsoft.AspNetCore.Mvc.WebApiCompatShim" />
    <PackageReference Include="Microsoft.AspNetCore.Routing" />
  </ItemGroup>
  ```
* In the controller code, you need to have some conditional `#if` directives. First off, *RoutePrefix* is no longer a thing, but *Route* serves the same purpose when put on the controller level. You also need some conditional *using* statements:
```csharp
#if !NETFRAMEWORK
using Microsoft.AspNetCore.Mvc;
using RoutePrefixAttribute = Microsoft.AspNetCore.Mvc.RouteAttribute;
#endif
```
This serves the purpose of: 1. Creating an alias for RoutePrefix with ASP.NET Core, and 2. Include the ASP.NET Core namespace.
* HttpContext.Current is no longer a thing as well - so you will have to make an adjustment for this. You can use dependency injection with ASP.NET Core, and a `#if` directive to instead use HttpContext.Current in .NET Framework.
* Keep the controller base class as before (ApiController). The WebApiCompatShim package defines ApiController for .NET Core.

As for HTTP modules, which is .NET Framework specific, these should be implemented as ASP.NET Core middleware. This is described elsewhere: [https://docs.microsoft.com/en-us/aspnet/core/migration/http-modules?view=aspnetcore-3.1](https://docs.microsoft.com/en-us/aspnet/core/migration/http-modules?view=aspnetcore-3.1)

Other than this, there should not be a lot of changes needed, apart from general porting of code.

## Porting Workflow Foundation and Windows Communication Foundation code
I won't write a lot here because I'm not an authority when it comes to WF and WCF. However, I want to highlight two Nuget packages that can be used to ease the transition to .NET Core:

* [CoreWf](https://github.com/UiPath-Open/corewf) - for Workflow Foundation
* [SoapCore](https://github.com/DigDes/SoapCore) - For WCF

Both of these are available in the public nuget feed. Kudos to the authors of these packages for significantly easing the transition of legacy WF and WCF code.

That's it for now! You can reach me on [Twitter](https://twitter.com/hallgeirl) if you have any comments or feedback.