---
title: Intro to vagrant
date: 2021-03-03
tags: [ vagrant, vm, linux ]
draft: true
---

# Intro to Vagrant

So I need to do a lot of system testing and recently I've fallen short of a
virtualized compute to do testing through VMs. So,
[Vagrant](https://www.vagrantup.com/) to the resecue for some small scale local
demos.

Vagrant's punchline is "Development environemnts made easy". It has to goal to
have easily reproducible environments that are as close to the production
environments that is possible. Which is exactly what I need.

## Why not containers?

Containers are all fine and dandy when you need to test applications. But for
some "real production" env tests that I want to do, they are sometimes lacking.
VMs work for these since each instance stand completely on it's own as they would
in the production environment.

Containers are fine when I need a quick DB or service for development, not when I
want to test some behavior that goes through LB, proxies, application servers,
... , in that case I need a bit more.

## Vagrant basics

Lets first take a look at the basics of Vagrant before diving into the deep end.
Vagrant works with a `Vagrantfile` while like many of Hashicorp's products uses
`HCL`. It has several "blocks" that define different steps of what we want.

```
Vagrant.configure("2") do |config|
    config.vm.define "example" do |foo|
        foo.vm.box = "ubuntu/focal64"
        foo.vm.network "private_network", ip: "192.168.50.10"
        foo.vm.hostname = "example-01"
        foo.vm.provision "ansible" do |ansible|
            ansible.playbook = "setup/playbook.yml"
            ansible.extra_vars = {
                node_ip: "192.168.50.10",
            }
        end
    end
end
```

That's a lot of information there, lets break things down a bit. First of all, we
have the `Vagrant.configure("2") do |config|` line. This declares that we'll be
using version `2` of Vagrant and declare it a `config` variable. This means
within this block we can use `config` to refer to the Vagrant config.

`config.vm.define "example" do |foo|` defines a VM of which we can access the
config through the `foo` keyword. We theb define the `box` we want to use for the
VM with `foo.vm.box = "ubuntu/focal64"`. A `box` is the base upon which the VM
is build. It's a package closely resembling an OS image. In this example Ubuntu
20.04 (codename focal) is being used. These boxes vary a bit depending on which
"provider" you are using. Providers are what you are using to run the
virtualization (libvirt, virtualbox, hyperv, ...) and not all providers support
all boxes. A box can be downloaded with `vagrant box add <user>/<name>`. With
`vagrant box list` you can see which boxes are available locally and which
providers they support.

`foo.vm.network "private_network", ip: "192.168.50.10"` assigns a private network
to the VM with the IP set to `192.168.50.10`. What exactly this private network
is, again depends on the provider. More info on these can be found
[here](https://www.vagrantup.com/docs/networking/private_network). Aside from
private networks, there are some other options available for the
[networking](https://www.vagrantup.com/docs/networking). The `foo.vm.hostname`
speaks for itself, it sets the hostname.

`foo.vm.provision "ansible" do |ansible` allows us to provision the VM after boot
with a tooling of [choice](https://www.vagrantup.com/docs/provisioning). Here we
use `ansible` with a playbook to accomplish that goal.

In this example, we'll be using [Virtualbox]() as the provider. After installing
Virtualbox, Vagrant and Ansible, we can drop the config into a `Vagrantfile` and
type `vagrant up` in the directory. Then we lean back and watch the magix happen:
first the VM gets created and afterwards you can see the ansible output of the VM
being provisioned.


## First attempt: Kubernetes

So, in the previous section we discovered Vagrant can be used to create VMs and
provision them through Ansible. People that know me, know my dislike for
Kubernetes (but that's for another blog). yet I encounter it in many jobs, so
lets take a look at what is needed to have a small Kubernetes cluster setup with
Vagrant. For this testing cluster, I'd like:

* 1 control node
* 2 worker nodes

Baby steps first, later on we can start adding LB in front of this cluster and
expand on the control nodes to have it HA for example. But for the small scale
testing, this should suffice. Luckily, I already have an ansible laying around to
configure a Kubernetes cluster, so that's one part already done.

```
IMAGE_NAME = "ubuntu/focal64"
N = 2

Vagrant.configure("2") do |config|
    config.ssh.insert_key = false

    config.vm.provider "virtualbox" do |v|
        v.memory = 2048
        v.cpus = 2
    end
      
    config.vm.define "k8s-leader" do |leader|
        leader.vm.box = IMAGE_NAME
        leader.vm.network "private_network", ip: "192.168.50.10"
        leader.vm.hostname = "k8s-leader"
        leader.vm.provision "ansible" do |ansible|
            ansible.playbook = "kubernetes-setup/leader-playbook.yml"
            ansible.extra_vars = {
                node_ip: "192.168.50.10",
            }
        end
    end

    (1..N).each do |i|
        config.vm.define "node-#{i}" do |node|
            node.vm.box = IMAGE_NAME
            node.vm.network "private_network", ip: "192.168.50.#{i + 10}"
            node.vm.hostname = "node-#{i}"
            node.vm.provision "ansible" do |ansible|
                ansible.playbook = "kubernetes-setup/node-playbook.yml"
                ansible.extra_vars = {
                    node_ip: "192.168.50.#{i + 10}",
                }
            end
        end
    end
end
```

# Sources
