---
title: "Creating A CI/CD Workflow With Gitlab and Portainer"
date: 2022-04-18T09:29:19-04:00
draft: true
tags: ['gitlab', 'portainer', 'docker', 'cicd']
---
![portainer](/img/portainer.png)

---

{{< toc >}}

--- 

## To-Do for this writeup
- Documentation for Gitlab calling webhooks
- Adding stacks to portainer from git repo
- Notes about migrating current docker-compose over to portainer

## Backstory

I've been trying to look for good ways to automate the deployment / updates of docker containers, while keeping the simplicity of writing docker-compose files, for quite awhile. I started off trying to implement this via Ansible and [AWX](https://github.com/ansible/awx), as thats the automation platform I use for a lot of my other homelab configuration, but it does not really seem designed to do that sort of thing. The docker module in Ansible allows for the deployment of containers, but will only accept its own type of `yml` config. I already had a bunch of docker-compose files, so that was a no go. 

Then I had the idea to go full bore and try and migrate everything over to Kubernetes using Rancher. While this isnt a bad idea, and definitely something I want to shoot for in the future, it was _way_ more complicated than what I was looking for at the time. I was looking for something simple, and able to use what configuration I already had to manage it. I was getting to the point where I thought I would have to build an application to deploy docker-compose files over the Docker API using the TCP socket.

Eventually, while I was talking with one of my co-workers about some other container related stuff he was working on, I remembered Portainer! I had used it when I first started learning about containers, and used it as an easy way to get docker containers up and running. But when I had learned more about how docker works, and especially once I learned about docker-compose, I kind of left it in the dust.

I went ahead and had a look at what they had to offer in terms of deployment and CI/CD in general, and it was _exactly_ what I was looking for.

---
## Overview

![portainer-ci](/img/portainer-ci.png)

Essentially, I can setup a git repo (in this case I'm using Gitlab) that holds the docker-compose file and any supporting config files to send a webhook call to Portainer whenever a push is made to the master branch. The webhook will trigger Portainer to pull the updated config files and re-deploy the stack. 

This workflow allows for plenty of CI testing inside of Gitlab using its runners to test images and config. For the time being I wont be setting up that side, but will be sure to link to an update once I get this setup.

---

## Setting Everything Up

### Install Portainer

First things first is to get Portainer installed and setup. Most of what I'm about to say can be found in the [Portainer Docs](https://docs.portainer.io), and I would encourage you to go there to get the most up to date info on how to properly deploy it.

With that, there is actually already an important distinction that you should be aware of. Portainer offers two different versions of their product: Portainer Community Edition and Business Edition. For most people, Community Edition will likely be just fine, but some of the more advanced features (Such as forcing a redeployment when Portainer pulls the new git config, which I need for this setup) are locked behind Business Edition. As of writing, Portainer offers [5 Free Nodes](https://www.portainer.io/pricing/take5?hsLang=en) of Business Edition when you sign-up with a business email. So if you have a business you are using this for, sign-up and get the extra features! 

I deployed Portainer on my docker host using the following command:

```
docker run -d -p 8000:8000 -p 9000:9000 -p 9443:9443 --name portainer \
--restart=always \
-v /var/run/docker.sock:/var/run/docker.sock \
-v ~/portainer/portainer-config:/data \
portainer/portainer-ce
```

Its very important that you give Portainer access to `/var/run/docker.sock`, this is how Portainer will communicate with the Docker API, without it you will be unable to manage containers properly. _Note that in doing so you are giving Portainer administrative access to all of your other containers, but most importantly can give access to the host system as the docker daemon runs as root. While Portainer is likely trustworthy, use your best judgement on what you give access to use this socket. See more info [here](https://raesene.github.io/blog/2016/03/06/The-Dangers-Of-Docker.sock/)._

Once you ran the docker command, Portainer should be reachable on port `9000` or `9443` if you want to use HTTP or HTTPS respectively. From here you should be prompted to setup your admin credentials (If you are using Business Edition, you will be prompted for your license key first). 

Next, you will need to add your first environment. You can go ahead and select Docker and connect to the local Docker socket. If it worked correctly you should see a Docker Environment labelled _local_ on the Home tab:

![portainer-local-env](/img/portainer-local-env.png)

If you already have other containers / docker-compose stacks on this instance, they should show up already (Though they wont be able to be controlled by Portainer, since docker-compose is the owner of those stacks). If everything shows up, then it should be installed and good to go!

### Attaching Additional Docker Hosts

If you want to be able to deploy to and monitor additional Docker instances, then you will need to connect Portainer to that machines Docker socket as well. There are a few ways to go about it, mainly exposing that socket to the network. _I am taking the lest secure way in this write-up, as I wanted to get it running first and foremost, however I do **NOT** recommend doing it this way, as it will openly expose your Docker socket to your network unauthenticated. I recommend reading up on the [Docker Wiki](https://docs.docker.com/engine/security/protect-access/) for best practices._

We will be editing the Docker daemon to add a flag to expose the socket over TCP.

1. Log into your machine
2. Assuming you are running docker via systemd, edit `/etc/systemd/system/multi-user.target.wants/docker.service`
3. Add `-H tcp://0.0.0.0:2375` to the command in the `ExecStart` parameter under `[Service]` and save the file
4. Reload the systemd daemon: `sudo systemctl daemon-reload`
5. Restart the docker service: `sudo systemctl restart docker.service`

Here is what the Service section for my docker service looks like for reference:

```
[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2375 --containerd=/run/containerd/containerd.sock
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
```

You can verify that docker came up properly via `systemctl status docker.service`, if you see `-H tcp://0.0.0.0:2375` in the `usr/bin/dockerd` command, then it should be working. 

From there, we can add the environment to Portainer: 
1. Select _Environments_ on the side tab and _Add Environment_.  
2. Select _Docker_ for the Environment Type, and give it an appropriate name. 
3. Put the IP address of the docker host in _Environment URL_ along with port `2375`: `10.0.5.253:2375` for example
4. For _Public IP_, just put in the IP address of the host
5. Click 'Add Environment'

![portainer-add-env](/img/portainer-add-env.png)

Repeat all these steps for any other host that you want to add, and they should appear in your Home tab just the same as your local environment:


![all-envs](/img/all-envs.png)

And thats it for now on the Portainer side