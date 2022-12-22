# Docker limit resource utilization using cgroup-parent


<img class="cp t u fz ak" src="https://miro.medium.com/max/7842/1*hjJCRo-Rk3E_WufIFE2duQ.jpeg" role="presentation"/>

One of the things you have to keep in mind in the cloud journey is limiting the resources (CPU and memory) for all running containers. Restrict CPU and Memory utilization across all containers is very important especially if you are running multiple containers. Docker presents an option called cgroup-parent for this purpose. You can check the official docker documentation on cgroup-parent [here](https://docs.docker.com/engine/reference/commandline/dockerd/#miscellaneous-options). Unfortunately, docker documentation doesn’t provide enough details for you start using use cgroup-parent. You can continue to read this post if you are still looking a way to apply resource limitation to your containers.

First, What is cgroups (control groups)?
========================================

cgroups is a feature provided by Linux kernel which allows a user to set limits on resource usage — _cpu, memory, network, disk i/o_. In another way, this allows specifying process level resource limitations in Linux. Read more about cgroups in [redhat docs](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/ch01).

Can I use docker --memory, --cpus options? Yes, only if you have one container per host!
========================================================================================

Yes, you can use the memory and cpu limit options mention mentioned on [docker documentation](https://docs.docker.com/config/containers/resource_constraints/#limit-a-containers-access-to-memory). One of the things you need to keep in mind _these options are per containers_. For example, if you have two core cpu and you start two containers with option `--cpus="1"`, your containers are allowed to consume 100% of cpu (2 cores). You can run two containers with one cpu each using below commands.

```
⇒ docker run -it --rm --cpus="1" busybox
⇒ docker run -it --rm --cpus="1" busybox
```

Once these containers are up, check the cpu allocated in cgroups. You will see that docker set the cpu limit you mentioned at the container start but per container.

```
/ # cat /sys/fs/cgroup/cpu/cpu.cfs\_quota\_us
```

😕So, how do we apply the restrictions consistently?

Here comes the savior cgroup-parent
===================================

For you to use the cgroup-parent option, first you have to define a cgroup on the host with desired limits. Different host os provides packages for creating cgroups, in this article I am going to explain how to use that using centos. I am going to do this in three steps _(1) create cgroup on host for limiting cpu and memory (2) adding the cgroup-parent to the containers (3) stress test the containers._

To make this easier, I have created a vagrant box with **centos7** as the host and use an **alpine** image with [stress](https://people.seas.harvard.edu/~apw/stress/) for the containers. I also added `htop` on the centos to check the resource usage. You can clone [repo](https://github.com/asishrs/docker-cgroup-parent.git) and use `vagrant up --provision` to provision the box.

```
⇒ git clone [https://github.com/asishrs/docker-cgroup-parent.git](https://github.com/asishrs/docker-cgroup-parent.git) && docker-cgroup-parent
⇒ vagrant up --provision
```

Create cgroup on host for limiting cpu and memory
-------------------------------------------------

I am using `cgcreate` command for creating a new cgroup with name c_group-limit._

```
/ # cgcreate -g cpu:cgroup-limit
Next, I am going to set cpu usage by 50%
/ # echo 100000 > /sys/fs/cgroup/cpu/cgroup-limit/cpu.cfs\_quota\_us
/ # echo 100000 > /sys/fs/cgroup/cpu/cgroup-limit/cpu.cfs\_period\_us
```

You need to create one cgroup per resource type, in this case, I am trying to limit the cpu hence `cpu:` in my command. You can check more about _cpu.cfs\_quota\_us_ and _cpu.cfs\_period\_us_ [here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/sec-cpu).

Next, I am going to create a cgroup to limit memory to _100mb (_104857600 _bytes)._

```
/ # cgcreate -g memory:cgroup-limit
/ # echo 104857600 > /sys/fs/cgroup/memory/cgroup-limit/memory.limit\_in\_bytes
```

👁️Check `memory:` in the command and the name _cgroup-limit_. Since I want to use a single cgroup to control the cpu and memory, I am using the same name for both cgroups.

You can check more about _memory limits_ [here](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/sec-memory).

Adding the cgroup-parent to the containers
------------------------------------------

You can add cgroup-parent to docker in a few different ways. The goal here is to tell all containers running on the daemon to use the same resource limits.
**1\. Use** `**--cgroup-parent**` **option**
You can pass option `--cgroup-parent` during the container creation.

```
/ # docker run -it --rm --cgroup-parent=/climit-cgroup/ <<image-name>>
```

**2\. Use** `**cgroup_parent**` **option in compose file**
You can specify cgroup as an optional parameter for the container in the docker compose.
⚠️According to [docker compose docs](https://docs.docker.com/compose/compose-file/#cgroup_parent), this option is _ignored when deploying a stack in swarm mode with a (version 3) Compose file_.
**3\. Use** `**--cgroup-parent**` **option on dockerd**
You can use the `--cgroup-parent` option on dockerd command or specify this in the config.json

I am using option #1`--cgroup-parent` at container creation for testing this.

Stress test the containers
--------------------------

To verify the settings, I am going to use [stress](https://people.seas.harvard.edu/~apw/stress/) which allows to reserve cpu and memory so that I can see how the resource limits are working at peak.

Here is the `htop` output before starting any containers. CPU is 0% and memory is at 115mb, we will check this once we start the containers with and without cgroups.

<img class="cp t u fz ak" src="https://miro.medium.com/max/1278/1*jOxpz6Os5AEFWBoVWSs8wA.png" width="639" height="87" role="presentation"/>

**Test the containers without cgroup limit.**

I am going to run this from the vagrant host as I have a specific image (asishrs/alpine-stress) I can use for stress testing.

```
⇒ vagrant up --provision // (Waiting...) This might take a while.
⇒ vagrant ssh
// Now we are inside the vagrant box.
// Here I am creating two conatiners without cgroup parent.
\[vagrant@cgrouphost ~\]$ sudo docker run -it --rm -d asishrs/alpine-stress stress -c 2 -i 1 -m 1 --vm-bytes 300M -t 60s
\[vagrant@cgrouphost ~\]$ sudo docker run -it --rm -d asishrs/alpine-stress stress -c 2 -i 1 -m 1 --vm-bytes 300M -t 60s
```

As you notice, I have started stress as part of the container. We are asking stress to use 2 cpus `-c2`, one IO `-i 1`, one worker `-m 1`, allocate 300M memory`--vm-bytes 300M`and run for 60 seconds `-t 60s`. You can learn all options by adding `--verbose` at the end of stress command.

Here is the screenshot from `htop`. You can see that the cpu utilization is nearly 100% and the memory is at 607mb.

<img class="cp t u fz ak" src="https://miro.medium.com/max/1250/1*Z7rvCk-5QmVJbXbVjNwz1w.png" width="625" height="94" role="presentation"/>

**Test the containers with per container cgroup limit.**

```
// Here I am creating two conatiners with per container limits
\[vagrant@cgrouphost ~\]$ sudo docker run -it --rm --cpus="1" --memory="300m" -d asishrs/alpine-stress stress -c 2 -i 1 -m 1 --vm-bytes 300M -t 60s
\[vagrant@cgrouphost ~\]$ sudo docker run -it --rm -cpus="1" --memory="300m" -d asishrs/alpine-stress stress -c 2 -i 1 -m 1 --vm-bytes 300M -t 60s
```

I have used options `--cpus="1"`and `--memory="300m"` during the container creation but this is applying the limit per container, that means both of the containers collectively going to consume 100% of cpu and 600mb memory.

Here is the output of `htop`. You can see that the cpu utilization is nearly 100% and the memory is at 721mb.

<img class="cp t u fz ak" src="https://miro.medium.com/max/1238/1*K6CT8lkHBMU6eZz6ehkI4g.png" width="619" height="88" role="presentation"/>

**Now run the container with** `**--cgroup-parent**`**.**

```
// Here I am creating two conatiners with cgroup-parent.
\[vagrant@cgrouphost ~\]$ sudo docker run -it --rm -d --cgroup-parent=/cgroup-limit/ asishrs/alpine-stress stress -c 2 -i 1 -m 1 --vm-bytes 300M -t 60s
\[vagrant@cgrouphost ~\]$ sudo docker run -it --rm -d --cgroup-parent=/cgroup-limit/ asishrs/alpine-stress stress -c 2 -i 1 -m 1 --vm-bytes 300M -t 60s
```

In the above command, I am passing the option`—-cgroup-parent=/cgroup-limit/`and that will restrict the overall all CPU utilization to 1 cpu and memory to 100mb.

Here is the screenshot from `htop`.

<img class="cp t u fz ak" src="https://miro.medium.com/max/1246/1*4aVlP0rRxtP_tV-UgBFMGA.png" width="623" height="87" role="presentation"/>

😲 You can see that the cpu utilization is nearly 50% and the memory is at 223mb (this includes the memory for other processes on the host).

**Next Steps:** Chekc how can you persist the cgroups so that can create atomic hosts.

* * *

📗Reference:
------------

[https://docs.docker.com/engine/reference/commandline/dockerd/#miscellaneous-options](https://docs.docker.com/engine/reference/commandline/dockerd/#miscellaneous-options)
[https://access.redhat.com/documentation/en-us/red\_hat\_enterprise\_linux/6/html/resource\_management\_guide/ch01](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/ch01)


