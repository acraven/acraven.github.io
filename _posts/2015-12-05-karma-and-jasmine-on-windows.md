---
layout: post
title: "Installing and configuring Karma and Jasmine on Windows"
date: 2015-12-05
categories: karma jasmine windows
featured_image: /images/cover.jpg
---

I have wanted to use Karma to run Jasmine tests ever since I worked at a company that screen-scraped the Jasmine test runner results from a browser using Selenium. This must be reasonably common, as lo and behold, the next company I worked for were doing exactly the same thing.

Being a massive fan of NCrunch which makes my C# life so much better, I wanted my Javascript life to benefit from this instant feedback test running goodness. Having previously tried, and failed, to get Karma to run my Jasmine tests I was again motivated after witnessing a colleague share my frustrations yesterday.

My failure was possibly due to not starting with a blank canvas; I have had Jasmine tests already running in a Visual Studio web app to which I tried unsuccessfully to apply Karma to. Recently I created a VM which has Chocolately, Chrome, Firefox, Visual Studio 2015 Express installed and is void of pretty much any other stuff, so this seems like a good place to start.

npm will be our friend, but we will also need Python to make sure our new friend plays nicely witfh PhantomJS.

{% highlight batch %}
choco install nodejs
choco install python2
npm config set python c:\tools\python2\python
{% endhighlight %}

You will need to ensure that Visual C++ is installed. The default installation of Visual Studio 2015 doesn't do this, so unless you customised the install you should navigate to **File | New | Project | C++** in Visual Studio and follow the prompt.

From my experience using NuGet, it felt to me like I needed a **package.json** file. This allows us to restore our dependencies without storing them in a Source Code Control System.

{% highlight batch %}
mkdir karma-jasmine-example
cd karma-jasmine-example
npm init --yes
{% endhighlight %}

We need to install the necessary packages initially, but once the **package.json** file contains these dependencies, a simple **npm install** is good enough to restore them.

{% highlight batch %}
npm install jasmine --save-dev
npm install karma --save-dev
npm install phantomjs --save-dev
npm install karma-phantomjs-launcher --save-dev
npm install karma-jasmine --save-dev
{% endhighlight %}

An finally "Install this module if you wanna be able to use karma in your command line"; for me, that's the whole point.

{% highlight batch %}
npm install karma-cli -g
{% endhighlight %}

Karma is now installed, but it needs a configuration file to make sense of anything. Fortunately, it contains a wizard for doing so.

{% highlight batch %}
karma init karma.config.js
{% endhighlight %}

Having run the wizard, my generated **karma.config.js** looks like this (with comments removed).

{% highlight javascript %}
module.exports = function(config) {
  config.set({
    basePath: '',
    frameworks: ['jasmine'],
    files: ['js/*.js', 'specs/**/*.js'],
    exclude: [],
    preprocessors: {},
    reporters: ['progress'],
    port: 9876,
    colors: true,
    logLevel: config.LOG_INFO,
    autoWatch: true,
    browsers: ['Chrome', 'Firefox', 'PhantomJS'],
    singleRun: false,
    concurrency: Infinity
  })
}
{% endhighlight %}

You can run Karma at this point, but it won't find any tests unless you're already ahead of me. Unless you specify the **--single-run** argument, Karma will run indefinitely, which is what you want for development. A single run will come in handy for Continuous Integration at a later point.

{% highlight batch %}
karma start karma.config.js
{% endhighlight %}

Add the following Jasmine spec to a file **karma-jasmine-example\specs\Calculator.js**

{% highlight javascript %}
describe('Calculator', function() {
  it('should add two numbers correctly', function() {
    var calculator = new Calculator();
	
    expect(calculator.add(1, 2)).toEqual(3);
  });
});
{% endhighlight %}

Karma will discover this new file and will attempt to run it. It will fail obviously since we haven't defined **Calculator**. Adding the file **karma-jasmine-example\js\Calculator.js** with the following will make the test pass.

{% highlight javascript %}
var Calculator = function() {};

Calculator.prototype.add = function(a, b) {
   return a + b;
};
{% endhighlight %}

Enjoy your instant feedback test running goodness.