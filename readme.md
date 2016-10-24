https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/

https://gist.github.com/fnichol/867550#the-manual-way-boring
   set SSL_CERT_FILE=C:\Work\cacert.pem

bundle exec jekyll serve

http://thephuse.github.io/strange_case/



Ideas
-----
A. Docker dotnet core
	api reference
	code generator
	workflow
B. Api performance dotnetcore, node, ruby, owin, webapi etc, local vs docker
	docker compose
C. Async/Await MVC HttpContext lifetimes
	resolve early
	no use of HttpContext.Current or DependencyResolver.Current
	autofac web module inc. register HttpRequestBase etc.
D. Owin Middleware runs twice if request isn't handled
E. To singleton or not to singleton
	state vs stateless
	lifetime conflicts
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
