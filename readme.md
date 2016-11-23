https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/

https://gist.github.com/fnichol/867550#the-manual-way-boring

http://jasonwilder.com/blog/2014/08/19/squashing-docker-images/

http://thephuse.github.io/strange_case/



Ideas
-----
Comment on dependencies in NuGet package

Semantic versioning of nuget packages

A. Docker dotnet core
	api reference
	code generator
	workflow
B. Api performance dotnetcore, node, ruby, owin, webapi etc, local vs docker
	docker compose
C. Async/Await MVC HttpContext lifetimes
	resolve dependencies early
	don't use HttpContext.Current or DependencyResolver.Current
	autofac web module inc. register HttpRequestBase etc.
	strange behaviour of confawait(true/false) in MVC true-deadlocks, false-null httpcontext
    using libraries that do confawait(false) when you need to preserve context
D. Owin Middleware runs twice if request isn't handled
E. To singleton or not to singleton
	state vs stateless
	lifetime conflicts
    use prepop state in constructor as an example
F. Aspects of HttpClient
   //async
   //request
   //reuse
   //ioc request/singleton
   //callcontext
   //consistent retry/attempt/timeout/exception logging
   //strange behaviour of confawait(true/false) true-deadlocks, false-null httpcontext
   //add headers using request context (if not tied to singleton)
   //anti-pattern static HttpContext.Current DependencyResolver.Current
G. Saucy
	dogfooding
H. migrating owin app => dotnet core
I. web api versioning
J. Trimming project.json dependencies
K. What is global.json used for. Appears to be locations of projects when using { target: "project" } instead of nuget version