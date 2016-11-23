---
layout: post
title: "Implementing continuous delivery of NuGet packages"
date: 2016-10-12
categories: dotnet cd continuous delivery nuget appveyor
featured_image: /images/cover.jpg
---
In my [last post]({% post_url 2016-10-11-building-multi-platform-nuget-packages %}) we created a solution to build multi-platform **NuGet** packages. Along with the packaged code, it consisted of a test project and some build scripts; enough to form the basis of a continuous delivery pipeline.  I'm a big fan of [AppVeyor](https://www.appveyor.com/){:target="_blank"}, which apart from it being configurable by code, is free for Open Source Software, so we will be using it to build, test, package and publish our multi-platform **NuGet** package.

### Prerequisites
Further to completing the [last post]({% post_url 2016-10-11-building-multi-platform-nuget-packages %}){:target="_blank"} you will need a [GitHub](https://github.com/){:target="_blank"} or [Bitbucket](https://bitbucket.org/product){:target="_blank"} account to host the repository (repo) containing your solution. **AppVeyor** will allow you to authenticate with either of these so there's no need to register with them directly. I use **GitHub** for public repos and **Bitbucket** for private repos. Beware though, at the time of writing, **AppVeyor** will charge you for building private repos.

Finally, you will need access to [NuGet](https://www.nuget.org/){:target="_blank"} in order to be able to publish your package. Unfortunately **NuGet** can't authenticate with either **GitHub** or **Bitbucket**, but you can use your **Microsoft** account instead. Log in to **NuGet** and take a copy of your API Key for later use.

### Create a repository
Create a [new repository](https://github.com/new){:target="_blank"} in **GitHub**. You can chose a default `.gitignore` file at this point if you wish, but it doesn't require too many entries, so manually add the following to your working folder. [gitignore.io](https://www.gitignore.io/) is a nifty tool for more complicated use-cases.

<div class="figcaption">.gitignore</div>
{% highlight text %}
.vs/
obj/
bin/
artifacts/
*.user
project.lock.json
TestResult.xml
{% endhighlight %}

In a command window in your working folder, run the following commands to create and push the code to your new **GitHub** repo, replacing `{username}` with your **GitHub** username and `{reponame}` with the name of your new repo.

{% highlight batch %}
git init
git remote add origin https://github.com/{username}/{reponame}.git
git add .
git commit -m "Initial commit"
git push -u origin master
{% endhighlight %}

### Configure AppVeyor

Log in to **AppVeyor**, select **New Project**, identify your repo from **GitHub** and click **Add**. You can click **New Build** to confirm **AppVeyor** can pull your repo at this point, but it won't build successfully just yet because we haven't configured it to.

As mentioned above, **AppVeyor** can be configured by code, meaning if your repo has an `appveyor.yml` file in its root it will override almost all the settings in the **AppVeyor** site. You will need to encrypt your **NuGet** API Key using **AppVeyor's** [Encrypt data](https://ci.appveyor.com/tools/encrypt) and replace the `{encrypted-api-key}` with the encrypted value.

<div class="figcaption">appveyor.yml</div>
{% highlight yml %}
branches:
  only:
    - master
version: 'build{build}'
skip_tags: true
clone_depth: 1
cache: C:\Users\appveyor\.nuget\packages
configuration: Release
before_build:
  - ps: Import-Module .\build\psake.psm1
build_script:
  - ps: |
      Invoke-Psake .\build\build.ps1 -properties @{"Configuration"=$env:CONFIGURATION}
      if ($psake.build_success -eq $false) { exit 1 }
test: off
artifacts:
  - path: 'artifacts\*.nupkg'
deploy:
  provider: NuGet
  api_key:
    secure: {encrypted-api-key}
  artifact: /.*\.nupkg/
{% endhighlight %}

Before we push `appveyor.yml` to the repo, you will need to update your package's `project.json` file or the publish stage will fail due to the existence of another package with the same name. The `name` and `title` tags are the important changes, but you should change the `authors` and `url` tags too. [This post](http://dotnetliberty.com/index.php/2016/01/26/where-does-dotnet-get-nuget-package-metadata/){:target="_blank"} from Armen Shimoon lists some of the other options.

<div class="figcaption">src\MyNuGetPackage\project.json</div>
{% highlight json %}
{
   "version": "1.0.0-*",
   "name": "Example.MultiPlatform",
   "title": "Example.MultiPlatform",
   "description": "Multi-platform library",
   "authors": [ "{yourname}" ],
   "packOptions": {
      "tags": [ "multi-platform", "example" ],
      "repository": {
         "type": "git",
         "url": "git://github.com/{username}/{reponame}"
      }
   },
   "frameworks": {
      "netstandard1.6": {
         "dependencies": {
            "Microsoft.AspNetCore.Http.Abstractions": "1.0.0"
         }
      },
      "net451": {
         "dependencies": {
            "Microsoft.Owin": "3.0.1"
         }
      }
   }
}
{% endhighlight %}

In the same command window as above, run the following commands to push the changes. **AppVeyor** will automatically start what will hopefully be a successful build in response to this push.

{% highlight batch %}
git add appveyor.yml
git add src/MyNuGetPackage/project.json
git commit -m "Add AppVeyor configuration"
git push
{% endhighlight %}

Once the **AppVeyor** build has completed successfully (green), the package will be listed on [NuGet](https://www.nuget.org/){:target="_blank"}. It may take a few moments to be publically available, but it should be visible on [Manage My Packages](https://www.nuget.org/account/Packages){:target="_blank"} immediately.

### Summary
We have created a simple continuous deployment pipeline that can build, test, package and publish changes made in a **GitHub** repo to **NuGet** for near-instant consumption.

However, this approach is not ideal for two reasons; firstly, each push to the `master` branch of the repo will trigger a new package to be published (usually you will want to isolate consumers from your packages until they are ready), and secondly, you need to change the `version` tag each push, otherwise the publish stage will be skipped. In the next post in the series, we will look at one possible workflow that addresses these issues.
