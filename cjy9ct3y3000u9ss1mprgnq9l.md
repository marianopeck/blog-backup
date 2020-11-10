## Step 2: Single-node Docker Swarm and Smalltalk

## Previously…

In a [previous post](https://martinezpeck.hashnode.dev/getting-started-with-docker-and-smalltalk-cjy5vhta8002x1ms1c9lth917) we saw an introduction to Docker and Linux containers and one possible usage for Smalltalk. We saw how to create the Dockerfile for a Seaside web application developed and running within [VASmalltalk](https://www.instantiations.com/products/vasmalltalk/index.html), build a Docker image, and finally run a container with it.

However, one of the conclusions from that post is that with plain `docker` command we can only run one container at a time and only in that node docker is running. So, in this post I would like to give you an introduction to Docker Swarm, which would allow us to do that. But, to make it easy to understand, in this post we will see a single-node example (but multiple containers).

BTW, thanks a lot to Julian Maestri and Norbert Schlemmer for your help!

## Docker Swarm and docker-compose

Docker can run in “swarm mode”, which basically allows you to create a cluster of docker engines. That way, you can run X number of containers within Y number of nodes. Of course, [Docker Swarm](https://docs.docker.com/engine/swarm/) comes with a lot of tools to ease the management of the cluster.

docker-compose is another tool which is kind of in the middle between plain docker (only one container running) and Swarm. It allows you to run multiple containers but only within that node, as far as I understand. Because of this, I don’t think is worth spending too much time explaining/learning about that and just move to Swarm.

## Let’s start using Docker Swarm for our Smalltalk application

We will continue with the very [same example](https://github.com/vasmalltalk/docker-examples/tree/master/source/SeasideTrafficLights/Raspberry) in the previous post, which is this Traffic Light Seaside demo running on a Raspberry Pi 3.

%[https://twitter.com/martinezpeck/status/1124321499043848192]

It’s important to note that for running Swarm you don’t need any extra packages or anything, it just comes with the normal Docker installation.

In the same way the Dockerfile was the most important file for defining a Docker image, the `docker-compose.yml` is the one for a Swarm. That’s where you define which services you would like in your cluster, how many “instances” of them, and all possible configuration. Below is our example:

```yaml
# To start it up.
# ./startSwarm.sh
 
# To stop it
# ./stopSwarm.sh
 
version: '3'
services:
  seaside:
    image: seaside-debian-slim
    deploy:
      replicas: 10
      labels:
        traefik.port: 7777
        traefik.frontend.rule: "HostRegexp:{catchall:.*}"
        traefik.backend.loadbalancer.stickiness: "true"
  balancer:
    image: traefik:v1.7
    deploy:
      placement:
        # The webserver should only load on manager nodes
        constraints: [node.role == manager]
    # To see all available  command line options: docker run --rm traefik:v1.7 --help | less
    command: --docker --docker.swarmmode --retry --loglevel=WARN
    ports:
      - "80:80"
    volumes:
      # Required because of docker backend, so traefik can get docker data.
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

Important things to note:

- I am defining 2 services `seaside` and `balancer`
- For the service `balancer` there is nothing extra to do because `traefik:v1.7` should be found and downloaded from known online repositories (like dockerhub)
- Instead, for the service `seaside` I am using an image called `seaside-debian-slim` which is **not**  on a public repository. So you have 3 alternatives:
  1. Build the image in each node: `docker build -f ./debian_slim_Dockerfile -t seaside-debian-slim .` (we will use this one for this example)
  2. Build the image just once, export it to a .tar and import .tar in the rest of the nodes (you can use `docker export` and `docker import` commands for this)
  3. Use an internal registry/repository. This obviously sounds like the best alternative if you have many nodes.
- Note that I am specifying 10 replicas for the `seaside` service. That means I will have 10 containers running the `seaside-debian-slim`  image.
- All Seaside containers will be sharing the same port 7777. How is that possible? Why don’t you get a `Error: Address already in use`? Docker virtualization magic. This port is not exposed and I don’t think you can do it even if you have replicas.
- `HostRegexp:{catchall:.\*}` is just a rule in Traefik web server to redirect everything to the underlying service (Seaside in our case)
- `traefik.backend.loadbalancer.stickiness: “true”` is necessary because Seaside framework is stateful and we need session affinity.
- We serve to the outside world over the port 80.

## How to run the swarm now?

First let’s get the example and build the image:

```bash
cd $HOME
git clone https://github.com/vasmalltalk/docker-examples.git
cd docker-examples/source/SeasideTrafficLights/Raspberry/
docker build -f ./debian_slim_Dockerfile -t seaside-debian-slim .
```

Then you need to initialize the Swarm. As this will be a single-node cluster, the only thing you need to execute is:

```bash
docker swarm init
```

Finally, all you have to do is to start Swarm using my provided `startSwarm.sh`:

```bash
#!/bin/bash docker stack deploy --compose-file docker-compose.yml seaside-debian-slim
```

You can see there the docker command to deploy a stack of services using the `docker-compose.yml` and naming it `seaside-debian-slim`.

To verify everything is working, you should take a web browser and enter to the host machine on port 80. Example [http://marianopi3.local/trafficlight](http://marianopi3.local/trafficlight) or [http://192.168.7.91/trafficlight](http://192.168.7.91/trafficlight). That will show the Seaside Traffic Light example we have been seeing so far in the examples.

You can then stop the swarm with `stopSwarm.sh` which has some magic so that to wait for the network to shutdown correctly… otherwise it will fail once you try to start again:

```bash
#!/bin/bash
 
readonly STACK_NAME=seaside-debian-slim
 
docker stack rm $STACK_NAME
declare TIMEOUT=0
readonly WAIT_INTERVAL=1
readonly LIMIT=120
 
echo -n 'waiting for network to shutdown'
while [ "$(docker network ls | grep --count "$STACK_NAME")" -gt 0 ] && [ $TIMEOUT -lt $LIMIT ]; do
    echo -n .
    sleep $WAIT_INTERVAL
    (( TIMEOUT+=WAIT_INTERVAL ))
done
if [ $TIMEOUT -ge $LIMIT ]; then
    echo ' failed' 1>&2
    exit 1
else
    echo ' ok'
fi
```

In the previous post we used the commands `container` and `image` to see our results. However, with Swarm you have to use the `service` commands. Some examples below:

 

![Screen Shot 2019-05-08 at 4.36.29 PM](https://marianopeck.files.wordpress.com/2019/05/screen-shot-2019-05-08-at-4.36.29-pm.png?w=748)

So…to conclude, we have 1 container running Traefik web server load balancing across 10 containers running a Docker image that has VASmalltalk with Seaside inside a single node (Raspberry Pi 3+ in this case).

## Interesting tips

Running `docker service logs seaside-debian-slim_seaside` is very handy to watch for logs of services.

Another magic thing is the default policy of auto-restart of the containers. Kill one of the containers… by kill -9 or whatever…and see how magically Docker starts a new container to supply the one that went down and satisfy the required replicas (10 in this example).

![Screen Shot 2019-05-08 at 4.44.10 PM.png](https://marianopeck.files.wordpress.com/2019/05/screen-shot-2019-05-08-at-4.44.10-pm.png?w=748)

See seaside containers 2 and 7? I killed them and Docker automatically started them over for me :)

## What’s next?

Obviously, the next step is to make the Swarm to work with more than one node :) So here it is a teaser of the next post:

%[https://twitter.com/martinezpeck/status/1126143505095700480]

## [UPDATE: See next post about this topic!](https://martinezpeck.hashnode.dev/docker-swarm-cloud-on-a-arm64-diy-sbc-cluster-running-a-smalltalk-webapp-cjynn3ftv000itns19kwgdjfc)

