---
date: 2020-11-18
title: "Docker, a strong opinion"
tags: ['docker','containers']
draft: true
---

TODO: note that this post is still a draft and is subject to change. It might
even never see the light of day on my actual website.

# Docker, a strong opinion

People that know me in a somewhat IT related capacity know that I have a strong
opinion about using docker in production. And since I have repeated myself often
enough, I'm taking the somebodies advice and created this blogpost about it.
Since the best way to learn how wrong you are, is to say something on the
internet.

Since I hate **TL;DR** at the end, here's this one:

Using barebones docker/docker-compose in production is a terrible idea and
containers should only be used with an orchestrator around them. Simply because
you need something that manages the containers (over time). Containers are easy
to use, difficult to use correctly.

## Disclaimer

While the following text might sound very negative against docker/containers, I
do actually believe there is a time and place to use them. I also realise that a
lot the remarks are probably based on bad experiences with environments that do
not follow best practices in containerisation.

Which is somewhat the point I attempt to make, containers are sometimes seen as
"easy" solutions, which lead to a lot of frustrations with containers. I just
want to urge that when considering containers for services, to do research and
implement best practices.

Anyways, here are the remarks I always make about containers when asked if they
are a good choice for something.

## Remarks

### Containers are not a silver bullet to have "know states"

TODO: this should be rewritten a bit to sound less aggressive and focus more on
that using docker containers does not mean that no config management or system
knowledge is required. I'd argue quite the opposite.

People ofter raise docker as the solution for state on there server. They think
they can put their program in a docker container and run it everywhere, which in
some way, is what docker promises. So people go along, jamming all kinds of
things in containers, thinking that these things will magically work everywhere.

And then it doesn't. Suddenly that site that is completely jammed in a docker
doesn't work anymore and the dev has long left the company. This is a problem
because the runtime settings might have been badly documented, but in my
experience the same issue arises as with Ansible: the Dockerfile becomes the
documentation and issues go from "issue X occured" to "this container doesn't
run".

### Containers their simplicity is in their complexity

TODO: don't think that this post covers the complexity of docker itself enough.
There should be coverage over how volumes, docker networks, file permissions,
... are all manageable by docker, but that it gets very complex if you need to
set all these things

Docker in itself is a complex thing with many parameters to tweak to the users
desires. Not every docker is the same, and not every docker host is the same.
Orchestrators amplify these complexities by an order of magnitude. Look at the
sizes of some Kubernetes manifest, they are incredibly dense with information
and parameters. [1](https://plumbr.io/blog/java/oomkillers-in-docker-are-more-complex-than-you-thought)

Sometimes when the container and host filesystem have a disagreement about the
permissions of volumes. Other times the network of the host doesn't seem to
agree. I have plenty of colleagues that need to run `--network=host` sometimes
to quickly debug things. Some containers require specific runtimes or system
permissions to run. Which brings me to the next point ...

### Containers have security concerns

Docker themselves raise several remakrs about the [docker engine
security](https://docs.docker.com/engine/security/) threat model. Let alone that
most people don't go through the trouble of actually figuring out what
permissions the containers need to function correctly and use the shotgun
approach: run the container as privileged.

Another issue is, how many people validate the images that they pull before
running them? I sometimes wonder how many [malicious
containers](https://www.aquasec.com/research/alert/a-cryptominer-hidden-in-a-docker-hub-image/)
are out there, running on k8s clusters all over the world, silently mining and
allocating resources to botnets.

### Lightweight containers are not always performant

TODO: not sure about the wording here. Maybe this should be focussed more on the
fact that people see docker for seperation more so then performance?

There is somehow the believe that containers are "lightweight". They contain
only the libraries and binaries (or they should) that are needed to run your
service. As such, these containers can be incredibly small in size, sometimes
even single binaries large.

That comes at a cost: debugging additional software needs to be build into the
images. Either by the providers or by making derivations based on those images.
And since these tools need their own libraries to run, these can rapidly start
bulking up container sizes to ridicilous sizes (looking at you NVIDIA).

And then there is the runtime performance. With a solid understanding of the
resource usage and [limits on
these](https://docs.docker.com/config/containers/resource_constraints/), a
docker container is just as dangerous as a rampant piece of software on a normal
host. A single service running in a container can just as easily break the host
when it starts eating memory or cpu.

### A lot of the time development != production

This is a large selling point, you can use it in development and production just
the same. While this can be true if the development environment closely mimics
the production environment. But it doesn't always hold up, after all, what
development environment have load balancer, intricate VPN connection and the odd
duck out that is always lurking behind a corner to trip you up when you deploy?

### [Log output is stdout](https://docs.docker.com/engine/reference/commandline/logs/)

This is a standout thing about Docker and Kubernetes specifically. Logs get
written to stdout and if the program is any solid and writes to a file, you need
to remap that to stdout. Too many times have I needed to fix this or explain to
people without knowledge of this fact, why logs go missing or they aren't able
to see their logs.

## Why do people use containers then?

Since they are usefull when used in a correct context and configured properly.
Microservice architecture components are well suited for containers for example
and containers are *incredbly* powerful in development, abstracting away the
complexities of running DB servers is an absolute blessing at times. But those
same complexities are what sometimes matters in production environments.

In a same way I think that docker offers a good way to "play" with software.
Start out a container, play around with the features to determine it's use and
wheter or not you want to use it in production.

But it's my opinion that using barebones docker/docker-compose in production is
a terrible idea and containers should be only used with an orchestrator around
them. Simply because you need something that manages the containers over long
periods of time.

DO use docker containers:

* when there is a solid in-house understanding of containeristion and it's trade
  offs.
* you have "cattle" services that can be managed by an orchestrator.
* development needs suite the use of containers to quickly set up dependent
  services.
  
DO NOT use containers:

* for your critical services, aka the "kittens".
* to falsely assume a "state" of a service and host.
* there is no in-house knowledge of containerisation.
