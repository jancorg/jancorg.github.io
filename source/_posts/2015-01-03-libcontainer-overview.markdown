---
layout: post
title: "LibContainer overview"
date: 2015-01-03 18:14:14 +0100
comments: true
categories: [ devops, libcontainer, docker ] 
---



Libcontainer is now the default docker execution environment. It is driver (named native) and a library. 

Or in other words, it is a replacement (since version 0.9) for formerly LXC execution environment (that can be easily brought back using `-e` switch). 

This library is developed by docker.io, written in go and C/C++, in order to support a wider range of isolation technologies. It also can be used in python through python bindings.

![docker execdriver](http://blog.docker.com/wp-content/uploads/2014/03/docker-execdriver-diagram.png)

It is meant to be a cross-system abstraction layer being an attempt to standarize the way apps are packed up, delivered, and run in isolation.

Libcontainer as a stand alone project, makes possible to other game players to adopting it. Google, parallels (openvz), redhat, ubuntu (lxc). are also contributing to this project.

![libcontainer_place](http://1.bp.blogspot.com/-sxNaakTEBlg/U6Bbtglx00I/AAAAAAAAAcw/xgreTVZP4F4/s1600/libcontainer+intro.png)

This way, container features available in linux kernel API are provided as a unique library in a consistent way. LibContainer addresses the problem of having an unique kernel API and several implementations. (call them, LXC, libvirt, lmctfy, ...)

> Libcontainer enables containers to work with Linux namespaces, control groups, capabilities, AppArmor, security profiles, network interfaces and firewalling rules in a consistent and predictable way.

Currently, docker can support these kernel features out-of-the-box, since it no longer depends on LXC.

It introduces a new container specification as we can see [here](https://github.com/docker/libcontainer/blob/4940cee052ece5a8b2ea477699e7bb232de1e1f8/SPEC.md)

>  It includes namespaces, standard filesystem setup, a default Linux capability set, and information about resource reservations. It also has information about any populated environment settings for the processes running inside a container.


Let`s see a little example of how it was replaced on docker code.

`nsinit` and `nsenter` are the user-land tools that now are needed in order to replace `lxc-*` user-land tools. 
`nsinit` is part of libcontainer package and it is meant to load a [config json file](https://github.com/docker/libcontainer/blob/master/sample_configs/minimal.json). It replaces `lxc-start`.

`nsenter` replaces `lxc-attach` and it is not part of libcontainer package, but it is part of `util-linux` [package](https://www.kernel.org/pub/linux/utils/util-linux/).


When formerly we ran `docker run ...`, (by default) it called `lxc-start` to run the container. 

``` go lxc-start https://github.com/docker/docker/blob/17cacf3326edde6d177e12132f74fc0174bda1d2/daemon/execdriver/lxc/driver.go
	params := []string{
		"lxc-start",
		"-n", c.ID,
		"-f", configPath,
	}
```

Currently, native execdriver is called. 

``` go native_exec_driver https://github.com/docker/docker/blob/17cacf3326edde6d177e12132f74fc0174bda1d2/daemon/execdriver/native/driver.go
func (d *driver) Run(c *execdriver.Command, pipes *execdriver.Pipes, startCallback execdriver.StartCallback) (execdriver.ExitStatus, error) {
	// take the Command and populate the libcontainer.Config from it
	container, err := d.createContainer(c)
```

``` go native_syscalls https://github.com/docker/libcontainer/blob/1597c68f7b941fd97881155d7f077852e2914e7b/namespaces/utils.go
var namespaceInfo = map[libcontainer.NamespaceType]int{
	libcontainer.NEWNET:  syscall.CLONE_NEWNET,
	libcontainer.NEWNS:   syscall.CLONE_NEWNS,
	libcontainer.NEWUSER: syscall.CLONE_NEWUSER,
	libcontainer.NEWIPC:  syscall.CLONE_NEWIPC,
	libcontainer.NEWUTS:  syscall.CLONE_NEWUTS,
	libcontainer.NEWPID:  syscall.CLONE_NEWPID,
}
```


General go syscalls can be found [here.](http://golang.org/pkg/syscall/). These ones used above are Linux especific.


Master Libcontainer branch supports right now [lxc and native](https://github.com/docker/docker/tree/master/daemon/execdriver) execdrivers. Hence it means that it can only used in linux for now. 

``` go supported_exec_drivers https://github.com/docker/docker/blob/master/daemon/execdriver/execdrivers/execdrivers.go
package execdrivers

import (
...
	"github.com/docker/docker/daemon/execdriver/lxc"
	"github.com/docker/docker/daemon/execdriver/native"
...
)
``` 

Other systems, like FreeBSD, implements 'containers' using Jails. It is an older concept, and some people note that it could not be as sofisticated as linux one is. 

[Here](https://github.com/kzys/docker/commit/f8c4d49fda9eb7e35c88532c174fa8dca9d831ba) is as an example of an execution driver in development for FreeBSD Jails. Well, it is an execution driver for FreeBSD, therefore not using libcontainer.

Unfortunately, FreeBSD (and others flavours) mention on docker & libcontainer are only for go language.


A full list of OS level virtualization implementations can be found  on [wikipedia](http://en.wikipedia.org/wiki/Operating_system%E2%80%93level_virtualization#Implementations)
