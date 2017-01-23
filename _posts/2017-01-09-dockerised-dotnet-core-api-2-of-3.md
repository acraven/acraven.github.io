---
layout: post
title: "A Dockerised .NET Core API (part 2 of 3)"
date: 2017-01-09
categories: dotnet core api docker
featured_image: /images/cover.jpg
tags:
- draft
---
In my [last post]({% post_url 2017-01-09-dockerised-dotnet-core-api-1-of-3 %}) we created a web API using **.NET Core**. The aim of this particular post is demonstrate how we can create a **Docker** image of our web API in a robust, reliable and repeatable CI (Continuous Integration) process.

### Getting it into Docker
I have come across several solutions for getting the published binaries into **Docker**, and none have left me with the satisfying glow of job well done, not even my chosen solution. I want to build in **Docker**, but I don't want to link a volume, besides which linked volumes need extra configuration with **boot2docker** (so I'm ruling those out). I want a minimal image size, which means I need to build in a different container to which is deployed. 

I've settled on a two step process, the first of which performs the build and runs any unit tests in a build image and the second extracts the artifacts and creates a deploy image. The really cool thing here is that the base images are completely different, the base build image can contain all the tools required for CI, whereas the base deploy image is focused on being as lightweight as possible.

### Gotchas ###
When creating the `.sh` files in **Windows**, you need to ensure the line endings are set to `LF` as opposed to `CRLF`.  The resulting not found errors have cost me much time and frustration on more than one occasion.

{% highlight shell %}
/bin/sh: 1: ./dotnet-build-and-test.sh: not found
The command '/bin/sh -c ./dotnet-build-and-test.sh' returned a non-zero code: 127
{% endhighlight %}

If you are using **Linux** or plan too, you will need to set the executable attribute of the `.sh` files. You can change them all in **Linux** using `find . -name '*.sh' | xargs chmod +x`. To change them in **Windows**, you will need to commit the files to a **Git** repository and use `find . -name '*.sh' | xargs git update-index --chmod=+x`. If you are just using **boot2docker** in **Windows**, you don't need to worry about this, all files will be set to be executable as they are copied to the container. Scary.

{% highlight shell %}
SECURITY WARNING: You are building a Docker image from Windows against a non-Windows
Docker host. All files and directories added to build context will have '-rwxr-xr-x'
permissions. It is recommended to double check and reset permissions for sensitive
files and directories.
{% endhighlight %}

### The build
The purpose of the build step is to build and test our web API in a known and repeatable environment. When **Docker** has done its bit, we will have an image containing our binaries.

Before we get to **Docker** though, we need to create a script so it knows how to build our web API and run the inside-out (unit) tests. If you wish, you can run this script locally in order to build and run the tests.

<div class="figcaption">app\dotnet-build-and-test.sh</div>
{% highlight shell %}
#!/bin/bash
set -e

dotnet restore
dotnet publish DotnetApiReference --configuration Release --output binaries
dotnet test DotnetApiReference.Tests --configuration Release
{% endhighlight %}

A `dockerfile` describes how our **Docker** image is assembled, and usually follows the pattern: specify base image, set working folder, copy some files and run some commands.

The base image `acraven/dotnet:build-api-1.0.1` was created in order to improve build performance. It simply contains a cache of most of the **NuGet** packages used by the web API; any that aren't cached will be downloaded for each build. As the web API moves towards production, we would create a customised base image containing any additional tools used in the build. 

The `dotnet-build-and-test.sh` script is run inside the container and the binaries are exposed in the `build/binaries` folder. If any of the tests failed the image will not be created and the build will fail.

<div class="figcaption">app\Dockerfile.build</div>
{% highlight text %}
FROM acraven/dotnet:build-api-1.0.1
WORKDIR /build
COPY . /build
RUN ./dotnet-build-and-test.sh
VOLUME /build/binaries
{% endhighlight %}

A `.dockerignore` file identifies which local files shouldn't be available when building the container. We definitely don't want any local build artifacts finding their way into containers, so we need to exclude `bin` and `obj` folders and any `binaries` created by local builds.

<div class="figcaption">app\.dockerignore</div>
{% highlight text %}
**/bin
**/obj
binaries/
{% endhighlight %}

You can now test the build step by running `docker build --file Dockerfile.build .` from the `app` folder. We will extend this into a script in the next step.

### The package
The purpose of the package step is to run the container that performed the build, extract the binaries and copy them into a new deploy container.

Like the base image used in the build step, it is expected that it will be customised to satisfy future requirements. For now though, we are just using `microsoft/dotnet:1.0.1-core` which contains the **.NET Core** runtime.

Our `dockerfile` is expecting a single file `binaries.tar`; we will see how this is created in a moment. The `ADD` instruction understands `.tar` files and simply extracts the contents into the `app` folder [in this case].

If you recall, our web API listened on port 9000, so we need to instruct **Docker** to forward requests to this port.

There are a number of different methods for running a process in a container, we will be using what is known as the exec form; this allows an INT signal (SIGINT) to be passed from container to web API enabling it to terminate gracefully should the container be stopped.

<div class="figcaption">package\Dockerfile.package</div>
{% highlight text %}
FROM microsoft/dotnet:1.0.1-core
WORKDIR /app
ADD binaries.tar /app/
EXPOSE 9000
CMD ["dotnet", "DotnetApiReference.dll"]
{% endhighlight %}

We have to tag our image during the build so we are able to interact with it. Once built, we can spin the image up as a container and copy the binaries. **Docker** has a pretty cool `cp` command that allows us to extract the files into a `.tar` file. This file containing our binaries is then packaged into our deployable container.

<div class="figcaption">docker-build-and-package.sh</div>
{% highlight shell %}
#!/bin/bash
set -e

location="`dirname \"$0\"`"

rm -f $location/package/binaries.tar

echo "Building application binaries"
docker build --tag=dockerised-dotnet-core-api/build:latest --file $location/app/Dockerfile.build --no-cache=true $location/app

echo "Extracting binaries"
id=$(docker create dockerised-dotnet-core-api/build:latest)
docker cp $id:/build/binaries/. - > $location/package/binaries.tar
docker rm -v $id

echo "Packaging binaries"
docker build --tag=dockerised-dotnet-core-api/api:latest --file $location/package/Dockerfile.package --no-cache=true $location/package
{% endhighlight %}

### The run

We can run our container using the tag given to it in the package step. We must first stop and remove the container beforehand if it is already running.

<div class="figcaption">docker-run.sh</div>
{% highlight shell %}
#!/bin/bash
set -e

id=$(docker ps -aq --filter name=dockerised-dotnet-core-api)

if [ -n "$id" ]; then
  isrunning=$(docker inspect --format="{% raw %}{{ .State.Running }}{% endraw %}" $id)

  if [ "$isrunning" == "true" ]; then
    echo "Stopping container (sending SIGINT)"
    docker kill --signal=INT $id
  fi

  echo "Removing container"
  docker rm $id
fi

echo "Starting new container"
docker run \
  --name dockerised-dotnet-core-api \
  -d \
  -p 9000:9000 \
  dockerised-dotnet-core-api/api:latest
{% endhighlight %}

You can confirm the API is running by navigating to [192.168.99.100:9000/ping](http://192.168.99.100:9000/ping) or [localhost:9000/ping](http://localhost:9000/ping) depending on your **Docker** installation.

### Summary
We have dockerised our web API and we can build and run our web API with just a couple of commands. This is great from a development workflow point of view; for those that just want to consume this service all we need on our development machines is a **Git** client and a **Docker** client. As our web API evolves we could use **Docker Compose** to run any downstream messaging and database services.

In the next post we will look into how we might productionise the web API. We need to secure our web API using the OWASP guidelines, we need a Continuous Integration pipeline and we need a mechanism of deploying our container from this pipeline into the wild.
