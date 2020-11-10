## Getting started with Docker and Smalltalk!

## What is Docker and Linux Containers?

[Linux containers](https://linuxcontainers.org/) are a way of virtualization that has been around since quite some time already. Compared to the classical “Virtual Machine” like VMWare, VirtualBox etc that are bulky and heavy to run, containers are lightweight, much smaller and they share the host’s kernel. That is, X number of small isolated Linux systems (containers) can be run and controlled within host’s Linux kernel. Those containers are able to run “applications” (web server, DB, domain apps, mail server, monitor tooling, whatever). This yields a much better optimization of hardware resources, control, etc.

[Docker](https://www.docker.com/) went a bit further and give us a tool for managing applications that run on top of containers. It eases the tasks for creating, deploying and maintaining containers. What is funny for us (Smalltalkers) is that Docker introduced again the concept of “image” (Hey, Docker don’t steal from us, that word is ours!!!!). And guess what? It’s the same idea. It’s basically an executable result of running some instructions to build and run an application. Then, when Docker runs that image, it “becomes” one (or more) containers.

[Here](https://www.infoworld.com/article/3204171/what-is-docker-docker-containers-explained.html) is a nice summary of what I just tried to explain above.

%[https://twitter.com/martinezpeck/status/1115731825140350978]

## Why is Docker relevant for Smalltalk?

Some of the Docker features don’t sound so surprising to us.  However, there still are quite a few:

* Ease deployment.
* Being able to replicate the same “infrastructure” to multiple places (testing, production, etc) regardless of the running host.
* Better optimization of hardware resources.
* Ease horizontal scaling. Each physical node can run N containers, and you can have M nodes in a “cloud” (see more below).
* Allows the trendy “Infrastructure as code” (although I prefer to call it “Infrastructure as config files”).
* Your deployment / build is reproducible, code versioned, etc.
* There are plenty of Docker definitions out there for existing softwares (web servers, databases, tools, etc) so rather than having to install all that you simply start from an existing image.
* Big players behind it ([RedHat OpenShift](https://www.openshift.com/), [Google Kubernetes](https://cloud.google.com/kubernetes/), etc)
* Many others….

Obviously, I think that the answer (“why it’s relevant?”) would depend a lot to whom you ask: a developer, a sysadmin or a DevOps.


## How to Dockerize a Smalltalk application?

The first thing you must to is to create a Dockerfile for your application what would specify all what your Docker image/container will need. Normally, you don’t start from scratch but from existing images. [Dockerhub](https://hub.docker.com/)  is the largest library of existing images. Of course, there are others too.

Some weeks ago, some customers from [Instantiations](https://twitter.com/instantiations) where inquiring about using Docker with [VASmalltalk](https://www.instantiations.com/products/vasmalltalk/index.html). And so, with the help of a few folks (thanks Julian Maestri and Norbert Schlemmer!)  from the community we got some [ready-to-go-examples on github](https://github.com/vasmalltalk/docker-examples).

%[https://twitter.com/martinezpeck/status/1102629181551230977]

The example on github is a [Seaside](https://github.com/SeasideSt/Seaside) web application that simulates a Traffic Light. It’s a very simple example but enough to show several features. [Note that the `.icx` image size is only 3.5MB](https://github.com/vasmalltalk/docker-examples/blob/master/source/SeasideTrafficLights/Raspberry/app/seasideTrafficLight-unix.icx) and it allows stack dumping, remote debugging, etc etc (topics for future posts).

The web application is running under port `7777` and so you get to it from a web browser in a similar way to this: [http://yourhostname.local:7777/trafficlight](http://yourpihostname.local:7777/trafficlight)

And to make it even more funny, our host (where Docker was running) was not a regular PC with Linux but a Raspberry Pi 3B+ running Raspbian Linux :) However, the same example should work for any regular Linux and x86 too.  Anyway, the entry point for all the details is [here](https://github.com/vasmalltalk/docker-examples/tree/master/source/SeasideTrafficLights/Raspberry).

As you can see there are multiple `docker\*` files. This is because we experimented with different OS and flavors. The one we are currently using is [debian_slim_Dockerfile](https://github.com/vasmalltalk/docker-examples/blob/master/source/SeasideTrafficLights/Raspberry/debian_slim_Dockerfile "debian_slim_Dockerfile") which looks like this:

```dockerfile
FROM debian:9-slim
 
# Comment build and run commands
# docker build -f ./debian_slim_Dockerfile -t seaside-debian-slim .
# docker run -e TZ=America/New_York --mount type=bind,source="$(pwd)",target=/opt/log -p 7777:7777 seaside-debian-slim
 
LABEL version="0.2.0"
LABEL maintainer="mpeck@instantiations.com"
LABEL description="VAST SeasideTrafficLight example"
 
# Install VASmalltalk Dependencies
RUN apt-get update \
&& apt-get install --assume-yes --no-install-recommends libc6 libssl-dev locales \
&& apt-get clean \
&& rm --recursive --force /var/lib/apt/lists/* /tmp/* /var/tmp/* \
&& echo en_US.ISO-8859-1 ISO-8859-1 >> /etc/locale.gen \
&& locale-gen
 
# set working directory
WORKDIR /opt/app
 
ADD ./vast92 /opt/vast92
ADD ./app /opt/app
 
RUN mkdir /opt/log
 
EXPOSE 7777
 
# Start application 
 
CMD ["./seasideTrafficLight.sh"]
```

Important points of this file:

1. `FROM debian:9-slim` defines from which image we start off. In this case debian:9-slim which is found automatically on Dockerhub.
2. We install VASmalltalk dependencies and generate necessary `$LOCALE`.
3. We copy the directories `vast92` (VASmalltalk Virtual Machine) and `app` from the host into the container
4. We just start the bash script `./seasideTrafficLight.sh`.

The bash script is as simple as invoking the Smalltalk VM and passing the image and ini files are arguments:

```bash
#!/bin/sh
 
cd /opt/app
 
export LANG=en_US.iso88591
export VAST_ROOT="/opt/vast92"
export LD_LIBRARY_PATH="${VAST_ROOT}/bin:$LD_LIBRARY_PATH"
$VAST_ROOT/bin/esnx -no_break -msd -mcd -i./seasideTrafficLight-unix.icx -ini:./seasideTrafficLight-unix.ini
 
cp *.sdf /opt/log
```

## How to run with Docker now?

The obvious first step is to have Docker installed. Google is your friend.

OK, we have the Dockerfile and Docker working. What do we do now? The first step is to “build” an image out from that Dockerfile and second, to run it (which will mean a container running). This is how you could do it:

```bash
cd $HOME
git clone https://github.com/vasmalltalk/docker-examples.git
cd docker-examples/source/SeasideTrafficLights/Raspberry/
docker build -f ./debian_slim_Dockerfile -t seaside-debian-slim .
docker run -e TZ=America/New_York --mount type=bind,source="$(pwd)",target=/opt/log -p 7777:7777 seaside-debian-slim
```

And that’s all. You should have created an image and started a container running it.

**Something very important:** Docker containers are NOT persisted. What does it mean? That whatever you write into the container, will be erased/lost as soon as the container is shutdown. There are many ways to add “persistency” but it depends a lot on what you need.

Useful commands to confirm our steps are `docker images` and `docker containers ps`. See image below:

![Screen Shot 2019-05-06 at 5.28.30 PM](https://marianopeck.files.wordpress.com/2019/05/screen-shot-2019-05-06-at-5.28.30-pm.png?w=748)

**Important:** Notice that the size of the container is just 71MB. I think it’s quite good for a starting point.

And obviously you should see the Traffic Light up an running:

![Screen Shot 2019-05-06 at 5.29.43 PM](https://marianopeck.files.wordpress.com/2019/05/screen-shot-2019-05-06-at-5.29.43-pm.png?w=748)

## Troubleshooting and Docker tips

Something very useful is to read from stdout or some other possible log that the app running in the container may have written. For that you can do `docker logs -f 6f7bc37b14e6` (where `6f7bc37b14e6` is the container ID you want to check). Example:

![Screen Shot 2019-05-06 at 5.33.06 PM](https://marianopeck.files.wordpress.com/2019/05/screen-shot-2019-05-06-at-5.33.06-pm.png?w=748)

The other very useful thing is to run a interactive container for a given image. For example:

![Screen Shot 2019-05-06 at 5.35.01 PM.png](https://marianopeck.files.wordpress.com/2019/05/screen-shot-2019-05-06-at-5.35.01-pm.png?w=748)So…basically you can open an interactive bash inside your container!!!! This is VERY useful.

Here are a [list of the most useful Docker commands](https://medium.com/the-code-review/top-10-docker-commands-you-cant-live-without-54fb6377f481) in case you wanna explore further (like how to stopping and removing containes, etc etc).

## Next steps?

With the regular `docker` command you can actually run **one** container at a time and in a **single** node. But, the joy of this is to have X containers split across Y nodes, right? So… below is a teaser of my next post on this subject:

%[https://twitter.com/martinezpeck/status/1124321499043848192]

## [UPDATE: See next post about this topic!](https://martinezpeck.hashnode.dev/step-2-single-node-docker-swarm-and-smalltalk-cjy9ct3y3000u9ss1mprgnq9l)
