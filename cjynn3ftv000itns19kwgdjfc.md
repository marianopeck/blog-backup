## Docker Swarm cloud on a ARM64 DIY SBC cluster running a Smalltalk webapp

In a [previous post](https://martinezpeck.hashnode.dev/step-2-single-node-docker-swarm-and-smalltalk-cjy9ct3y3000u9ss1mprgnq9l) we saw how to build a single-node [Docker Swarm](https://docs.docker.com/engine/swarm/) running [VASmalltalk](https://twitter.com/instantiations). That stack involved one web server ([Traefik](https://traefik.io/)) load balancing across 10 VASmalltalk images running a [Seaside](http://seaside.st/) web application. All containers in same node.

### Preparing the physical cluster

Before getting started, know that the code has been [published on Github in the same place as the previous posts](https://github.com/vasmalltalk/docker-examples).

The single-node of the previous post was a Raspberry Pi 3B+ running Raspbian 32 bits. But VASmalltalk can also runs 64 bits (x86, ARM, etc). In part, thanks to libffi and LLVM portable and power capabilities. So… I prepared the `ubuntu` node of our cluster with a Ubuntu Server 18.04 ARM 64 bits running in the Raspberry Pi 3B+

%[https://twitter.com/martinezpeck/status/1121137764949536768]

But to be able to call it “a cluster” I need at least one more node, right? Getting another Raspberry Pi 3 with Raspbian was boring. So [Instantiations](https://www.instantiations.com/) purchased me a different device, also being able to run 64 bits: the Rock 64. This is an incredibly nice machine with much more power than the Pi3 in some regards (4GB of RAM, eMMC, etc). I had Raspbian and Ubuntu Server already so for this `rock` node I installed Armbian Debian Stretch (BTW, runs very very nice and stable!)

%[https://twitter.com/martinezpeck/status/1120690282427748353]

**Cluster conclusion:** `ubuntu` node with Ubuntu Server 18.04 on Raspberry Pi3 and `rock64` node with Armbian on a Rock64.  All 64 bits.

### Preparing the Docker Swarm

In the previous post, before starting the swarm we executed `docker swarm init`. That made that single node be the “manager” of the cloud. But on a real cloud, you have managers and workers nodes. Usually there is 1 manager node and N worker nodes. However, as far as I understand, there are cases where there could be more than 1 manager. But for the moment, let’s just take 1: the `rock64`​.

The first step is to go to the manager (`rock64`) and execute `docker swarm init` if you haven’t done it before.  Then, run `docker swarm join-token worker`. That will print something like below:

```
To add a worker to this swarm, run the following command:
 
    docker swarm join --token SWMTKN-1-32xpsbsdu9zxzf3cejp6b459ubcgyoyxvvifuv75kf17ld2kd9-7ow4qvkpf30wn7aad0a3vvlui 192.168.7.108:2377
```

Then, go to each of the worker nodes and execute that docker command in the previous step. If that worker was already on a swarm, you would need to first do `docker swarm leave`. In this example, I did it for `ubuntu` node.

And that’s all! The Docker Swarm should now have 2 nodes. You can use `docker node ls` to confirm the results:

![Screen Shot 2019-05-14 at 4.21.40 PM](https://marianopeck.files.wordpress.com/2019/05/screen-shot-2019-05-14-at-4.21.40-pm.png?w=748)

You can notice how `rock64` “Manager Status” is “Leader” while `ubuntu` is not.

### Swarm specific settings in docker-compose.yml

The `docker-compose.yml` of this example is the following:

```yaml
version: '3'
services:
  seaside:
    image: seaside-debian-slim
    environment:
      TZ: America/New_York
      SEASIDE_PORT: 7777
    deploy:
      replicas: 10
      labels:
        traefik.port: 7777
        traefik.frontend.rule: "HostRegexp:{catchall:.*}"
        traefik.backend.loadbalancer.stickiness: "true"
    volumes:
      - /var/log/docker-logs:/opt/log
  balancer:
    image: traefik:v1.7
    # The webserver should only load on manager nodes
    deploy:
      placement:
        constraints: [node.role == manager]
    # To see all available  command line options: docker run --rm traefik:v1.7 --help | less
    command: --docker --docker.swarmmode --retry --loglevel=WARN
    ports:
      - "80:80"
    volumes:
      # Required because of docker backend, so traefik can get docker data.
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

The only particularities for running inside a Swarm are:

- The balancer `constraints: [node.role == manager]` because the Traefik web server must run in the manager (cannot accidentally be started in a worker node)
- The argument `--docker.swarmmode` to Traefik command.
- `/var/log/docker-logs` must exist on each node. There are better ways to solve this but exceeds the scope of this post.

### Running the Swarm!

The way to start the swarm is exactly the same as the previous post, as it doesn’t matter (it’s transparent) how many physical nodes are inside that cloud.  Same goes for all `service` commands we saw before.

The only important part here is that the `seaside-debian-slim` should have been already built in each node or available at some registry. See previous post for more details.

Below you can see the full sequence:

![Screen Shot 2019-05-14 at 4.27.27 PM.png](https://marianopeck.files.wordpress.com/2019/05/screen-shot-2019-05-14-at-4.27.27-pm.png?w=748)

Note that Traefik is running on the manager and that the 10 worker containers were split in half between the manager and the one worker.

### Interesting points

You can see how easy it is to scale to multiple nodes. In this example I just used 2, but normally you could have many more nodes. It’s also nice to see how it helps for availability. See what happens when I disconnect (power-off) of the workers (`ubuntu` in this case):

![Screen Shot 2019-05-14 at 4.32.44 PM.png](https://marianopeck.files.wordpress.com/2019/05/screen-shot-2019-05-14-at-4.32.44-pm.png?w=748)

See? `ubuntu` appears as "down" and the containers that were running on it are automatically re-started on different nodes (here the only possibility was the manager) to satisfy the specified number of replicas (10). Sure, Seaside users would loose their session, but that’s still awesome.

%[https://twitter.com/martinezpeck/status/1126143503262728197]

Hope you enjoyed,