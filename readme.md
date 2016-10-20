https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/

https://gist.github.com/fnichol/867550#the-manual-way-boring
   set SSL_CERT_FILE=C:\Work\cacert.pem

bundle exec jekyll serve

http://thephuse.github.io/strange_case/



Ideas
-----
E. Multi-platform NuGet packages
	appveyor
	dotnet core
	project.json references dependencies
	workflow master/stable semver
A. Owin Middleware runs twice if request isn't handled
B. Saucy
	dogfooding
C. Docker dotnet core
	api reference
	code generator
	workflow
D. Api performance dotnetcore, node, ruby, owin, webapi etc, local vs docker
	docker compose
F. Async/Await MVC HttpContext lifetimes
	resolve early
	no use of HttpContext.Current or DependencyResolver.Current
	autofac web module inc. register HttpRequestBase etc.
G. To singleton or not to singleton
	state vs stateless
	lifetime conflicts
H. Aspects of HttpClient
   //async
   //request
   //reuse
   //ioc request/singleton
   //callcontext
   //consistent retry/attempt/timeout/exception logging
   //strange behaviour of confawait(true/false) true-deadlocks, false-null httpcontext
   //add headers using request context (if not tied to singleton)
   //anti-pattern static HttpContext.Current DependencyResolver.Current
I. owin app => dotnet core
J. web api versioning
