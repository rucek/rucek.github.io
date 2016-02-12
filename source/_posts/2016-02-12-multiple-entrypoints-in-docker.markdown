---
layout: post
title: "Multiple entrypoints in Docker"
date: 2016-02-12 21:35:00 +0100
comments: true
categories: docker
---

## Background
When using [Docker](https://www.docker.com/) containers for a number of building blocks of your application, the recommended approach is to run a separate container for every building block, so that the components are well separated. And the [Docker best practices](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#run-only-one-process-per-container) suggest running a single process per container.

However, you may imagine a scenario in which it's reasonable to run multiple services in a single container (an example will follow). Then the question arises how to run those services using a single `ENTRYPOINT` command in the `Dockerfile`.

## Example scenario
Let's say your application communicates with an external service through an SSH tunnel (or a VPN, or anything more than a direct connection - you name it). Then, there is a number of ways to set up the tunnel along with the application itself:

  1. Setup the tunnel on the host, run the container with the `host` network mode (i.e. with the `--net=host` switch) - which means that the container has no separate network stack but uses the host's one, so it can just access the tunnel.

  2. Setup the tunnel on the host, run the container in the default network mode (i.e. `bridge`) and somehow access the tunnel on the host from the container. Somehow here means e.g. using the `--add-host` switch (with the host's IP) when running the container, which adds an `/etc/hosts` entry in the container (see [the docs](https://docs.docker.com/engine/reference/run/#managing-etc-hosts) for details). Or you can try [some other hacks](http://stackoverflow.com/questions/22944631/how-to-get-the-ip-address-of-the-docker-host-from-inside-a-docker-container) as well.

  3. Run the tunnel in the container.

Now, the problem with options 1 and 2 is that you lose the _run anywhere_ part of the Docker philosophy, since _anywhere_ becomes limited to _anywhere-with-an-ssh-tunnel_. Plus, your infrastructure is no more immutable, since by setting up the tunnel you just made a change to the host. A change about which you need to remember every time you run the container somewhere else.

Therefore, option 3 seems to be the way to go. But since Docker allows only a single `ENTRYPOINT` (to be precise, only the last `ENTRYPOINT` in the `Dockerfile` has an effect), you need to find a way to run multiple processes (the tunnel and the application) with a single command. Let's see how you can do it.

## Solution

The simplest idea is to create a shell script to run the required processes and use that script as the `ENTRYPOINT`. But I wouldn't write a blogpost about writing a shell script, would I? Instead, let's dig into the recommended technique which uses [_supervisord_](http://supervisord.org/) - a process control system.

In the big picture, _supervisord_ is a tool which lets you run multiple programs at once from a single place. The benefits over a plain old shell script are the numerous configuration and monitoring options. Here I'm going to cover only the basic usage, which is just enough for our scenario - feel free to explore the more advanced stuff yourself.

To use _supervisord_, you first need to install it, preferably using a package manager. In an Ubuntu/Debian-based container you need to add the following to the `Dockerfile`:

```
RUN apt-get update && apt-get install -y supervisor
```

Since in this example you're also going to need an SSH tunnel, let's install the SSH client too:

```
RUN apt-get update && apt-get install -y supervisor openssh-client
```

**Note:** always remember to combine `apt-get update` and `apt-get install` into a single command in order to get the latest package versions - see [the docs](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#apt-get) for details.

Now that _supervisord_ and the SSH client are installed, it's time for some configuration. Let's assume that your application's entrypoint is `/opt/myapp/bin/myapp`. The _supervisord_ configuration will then be something like:

```ini
[supervisord]
nodaemon=true
logfile=/var/log/supervisord/supervisord.log
childlogdir=/var/log/myapp

[program:ssh]
command=ssh -N -L8080:localhost:8080 user@example.com

[program:myapp]
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
command=/opt/myapp/bin/myapp --some-configuration
```

**Note:** If you put the congfiguration file in the defult location inside the container, i.e. `/etc/supervisor/conf.d/supervisord.conf`, it will automatically be picked up by _supervisord_. However, for the sake of this example, let's assume you chose a custom location inside the container, e.g. `/etc/myapp/supervisord.conf`.

The `[supervisord]` section configures the main _supervisord_ process. The `nodaemon=true` indicates that the process should stay in the foreground (as the container would be terminated otherwise). Additionally, you can specify a log file for _supervisord_ logs (with the `logfile` parameter) and a directory for the messages captured from the `stdout` and `stderr` of the child processes (here: `ssh` and `myapp`) - with the `childlogdir` parameter.

Next come the configurations of your processes that will run in the container.

For `ssh` you define your arbitrary port forwarding with the `-L` switch. Plus you can use the `-N` flag, which means that no remote command would be executed - which is _useful for just forwarding ports_, according to the SSH man page.

In `myapp` configuration the `stdout_logfile` parameter indicates where the `stdout` of the process should go - in this case it goes to the container's `stdout`. A log rotation policy can be configured  with `stdout_logfile_maxbytes`, where a value of zero means no rotation. The `command` parameter is self-explanatory - this is the full command to run your application.

Having configured _supervisord_, the last step is to actually run it when the container starts - with the `ENTRYPOINT` command:

```
ENTRYPOINT ["/usr/bin/supervisord", "-c", "/etc/myapp/supervisord.conf"]
```

Here you can see that we use the `-c` switch to provide the path to a custom configuration file. This wouldn't be necessary if the configuration was in `/etc/supervisor/conf.d/supervisord.conf`.

After running the container you should see a few logs from _supervisord_, indicating that both the daemon and your processes have been started:

```
2016-02-09 16:47:13,949 CRIT Supervisor running as root (no user in config file)
2016-02-09 16:47:13,951 INFO supervisord started with pid 1
2016-02-09 16:47:14,953 INFO spawned: 'ssh' with pid 8
2016-02-09 16:47:14,954 INFO spawned: 'myapp' with pid 9
2016-02-09 16:47:16,754 INFO success: ssh entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
2016-02-09 16:47:16,755 INFO success: myapp entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
```

## Summary
You have just learned the recommended way to run multiple processes in a Docker container - if you know what you're doing, i.e. you really need more than one process in your container. Please refer to [_supervisord_ docs](http://supervisord.org/) if your scenario is anything more than this basic one.
