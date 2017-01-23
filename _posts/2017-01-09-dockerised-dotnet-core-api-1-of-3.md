---
layout: post
title: "A Dockerised .NET Core API (part 1 of 3)"
date: 2017-01-09
categories: dotnet core api docker
featured_image: /images/cover.jpg
tags:
- draft
---
I love the concept of **Docker** and ever since **.NET Core** was released I have been eager to combine the two, but couldn't find a good resource for doing so. My ultimate goal of this series of posts therfore, is to bring them together to create a base from which enterprise micro-services can easily be spawned. The aim of this particular post is the creation of a web API using **.NET Core**. Later posts will focus on dockerising and productionising the API.

### Requirements
Before anything else, I scribled down some requirements that were important (in no particular order).

- Be built and tested on a build/CI agent using **Docker**.
- Allow non-committed changes to be built and tested using **Docker**.
- Build in a clean repeatable environment.
- Have minimal final image size.
- Support inside-out testing before the image is built.
- Support outside-in testing after the image is built.
- Have minimal time to failure.
- Link image version to a commit.
- Support **Windows boot2docker** and **Linux**.
- Have minimal impact to dev workflow.

I will review them at the end to see how I got on.

### Prerequisites
For **Windows**, you will need **Visual Studio 2015 Update 3** and **.NET Core tools for Visual Studio** as described on the [.NET Core](https://www.microsoft.com/net/core#windows){:target="_blank"} site. You will also need [Docker for Windows](https://www.docker.com/products/docker#/windows) or [Docker Toolbox](https://www.docker.com/products/docker-toolbox). I use **Docker Toolbox** since **Docker for Windows** requires **Hyper V** which is incompatible with **VMware Workstation**.

For **Linux**, you will need [.NET Core](https://www.microsoft.com/net/core#linuxubuntu){:target="_blank"} and an editor of your choice, I use [Visual Studio Code](https://code.visualstudio.com/docs/setup/linux).

### Cut to the chase
If you would rather just dive in, fork the **GitHub** repository [dockerised-dotnet-core-api](https://github.com/acraven/dockerised-dotnet-core-api){:target="_blank"}, start **Docker** and run `./docker-build-and-package.sh` followed by `./docker-run.sh`.

You can confirm the API is running by navigating to [192.168.99.100:9000/ping](http://192.168.99.100:9000/ping) or [localhost:9000/ping](http://localhost:9000/ping) depending on your **Docker** installation. 

### Creating the solution
My solution consists of two projects that reside in the `app` folder of the repo. This will contain the application itself and any inside-out tests that will be run before we bother creating the final image.

<div class="figcaption">c:\blog\dockerised-dotnet-core-api\</div>
{% highlight tree %}
 └─app
    ├─DotnetApiReference
    │  ├─Program.cs
    │  ├─project.json
    │  └─Startup.cs
    └─DotnetApiReference.Tests
       ├─StatusScenarios
       │  └─ping.cs
       └─project.json
{% endhighlight %}

I have included the **NuGet** packages `Bivouac` and `Burble`; the former contains middleware for server logging and status endpoints, the latter adds HTTP client logging, retrying and throttling. I will delve into these packages in another post.

<div class="figcaption">app\DotnetApiReference\project.json</div>
{% highlight json %}
{
   "version": "1.0.0-*",
   "buildOptions": {
      "emitEntryPoint": true
   },
   "dependencies": {
      "Microsoft.NETCore.App": {
         "type": "platform",
         "version": "1.0.1"
      },
      "Microsoft.AspNetCore.Mvc": "1.0.1",
      "Microsoft.AspNetCore.Server.Kestrel": "1.0.1",
      "Burble": "0.0.4-preview0023",
      "Bivouac": "0.0.2-preview0016"
   },
   "frameworks": {
      "netcoreapp1.0": {}
   }
}
{% endhighlight %}

The web API uses the **Kestrel** web server which in this example listens to port 9000.

<div class="figcaption">app\DotnetApiReference\Program.cs</div>
{% highlight csharp %}
namespace DotnetApiReference
{
   using Microsoft.AspNetCore.Hosting;

   public class Program
   {
      public static void Main(string[] args)
      {
         var host = new WebHostBuilder()
             .UseKestrel()
             .UseUrls("http://*:9000/")
             .ConfigureServices(Startup.ConfigureServices)
             .Configure(Startup.Configure)
             .Build();

         host.Run();
      }
   }
}
{% endhighlight %}

We need to configure our application using the usual `ConfigureServices` and `Configure` methods. This is where we add the server logging and status endpoints to the request pipeline.

<div class="figcaption">app\DotnetApiReference\Startup.cs</div>
{% highlight csharp %}
namespace DotnetApiReference
{
   using System;
   using Bivouac.Abstractions;
   using Bivouac.Events;
   using Bivouac.Middleware;
   using Burble.Abstractions;
   using Microsoft.AspNetCore.Builder;
   using Microsoft.Extensions.DependencyInjection;

   public static class Startup
   {
      public static void ConfigureServices(IServiceCollection services)
      {
         if (services == null) throw new ArgumentNullException(nameof(services));

         services.AddStatusEndpointServices("dockerised-dotnet-core-api");
         services.AddServerLoggingServices();

         services.AddTransient<IHttpServerEventCallback>(CreateHttpServerEventCallback);
         services.AddTransient<IHttpClientEventCallback>(CreateHttpClientEventCallback);

         services.AddMvc();
      }

      public static void Configure(IApplicationBuilder app)
      {
         if (app == null) throw new ArgumentNullException(nameof(app));

         app.UseStatusEndpointMiddleware();
         app.UseServerLoggingMiddleware();

         app.UseMvc();
      }

      private static IHttpServerEventCallback CreateHttpServerEventCallback(IServiceProvider serviceProvider)
      {
         var requestIdGetter = serviceProvider.GetService<IGetRequestId>();
         var correlationIdGetter = serviceProvider.GetService<IGetCorrelationId>();

         return new IdentifyingHttpServerEventCallback(
            requestIdGetter,
            correlationIdGetter,
            new JsonHttpServerEventCallback(Console.WriteLine));
      }

      private static IHttpClientEventCallback CreateHttpClientEventCallback(IServiceProvider serviceProvider)
      {
         return new JsonHttpClientEventCallback(Console.WriteLine);
      }
   }
}
{% endhighlight %}

We are using XUnit to run the inside-out (unit) tests, and in addition to `Bivouac` we will be using `Banshee` which creates a lightweight API pipeline suitable for unit tests.

<div class="figcaption">app\DotnetApiReference.Tests\project.json</div>
{% highlight json %}
{
   "version": "1.0.0-*",
   "testRunner": "xunit",
   "dependencies": {
      "Microsoft.NETCore.App": {
         "type": "platform",
         "version": "1.0.1"
      },
      "xunit": "2.2.0-beta2-build3300",
      "dotnet-test-xunit": "2.2.0-preview2-build1029",
      "Microsoft.AspNetCore.TestHost": "1.0.0",
      "DotnetApiReference": { "target": "project" },
      "Bivouac": "0.0.2-preview0016",
      "Banshee": "0.0.2"
   },
   "frameworks": {
      "netcoreapp1.0": {}
   }
}
{% endhighlight %}

We need a couple of tests to confirm the status endpoints middleware has been added to the request pipeline.

<div class="figcaption">app\DotnetApiReference.Tests\StatusScenarios\ping.cs</div>
{% highlight csharp %}
namespace DotnetApiReference.Tests.StatusScenarios
{
   using System;
   using System.Net;
   using System.Net.Http;
   using Banshee;
   using Bivouac.Abstractions;
   using Bivouac.Events;
   using Microsoft.Extensions.DependencyInjection;
   using Xunit;

   public class ping
   {
      private readonly HttpResponseMessage _response;

      public ping()
      {
         Action<IServiceCollection> configureServices = services =>
         {
            Startup.ConfigureServices(services);
            services.AddTransient<IHttpServerEventCallback, NoOpHttpServerEventCallback>();
         };
         var testHost = new LightweightWebApiHost(configureServices, Startup.Configure);

         _response = testHost.Get("/ping");
      }

      [Fact]
      public void should_return_status_code_200()
      {
         Assert.Equal(_response.StatusCode, HttpStatusCode.OK);
      }

      [Fact]
      public void should_return_content_pong()
      {
         var content = _response.Content.ReadAsStringAsync().Result;

         Assert.Equal(content, "Pong!");
      }
   }
}
{% endhighlight %}

The tests can be run locally using `dotnet test`.  

{% highlight batch %}
dotnet restore
dotnet test app\DotnetApiReference.Tests\project.json
{% endhighlight %}

The API can be run locally using `dotnet run`, remembering to use the `-p` argument.

{% highlight batch %}
dotnet restore
dotnet run -p app\DotnetApiReference\project.json
{% endhighlight %}

The status endpoints middleware will have exposed [localhost:9000/ping](http://localhost:9000/ping) and [localhost:9000/status](http://localhost:9000/status).

### Summary
We have created a very simple web API in .NET Core that exposes two diagnostics endpoints. In the [next post]({% post_url 2017-01-09-dockerised-dotnet-core-api-2-of-3 %}) of the series, we will use **Docker** to build and run the web API.
