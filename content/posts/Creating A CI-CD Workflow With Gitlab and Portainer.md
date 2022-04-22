---
title: "Creating A CI/CD Workflow With Gitlab and Portainer"
date: 2022-04-18T09:29:19-04:00
draft: false
tags: ['gitlab', 'portainer', 'docker', 'cicd', 'linux']
---

![portainer](/img/portainer.png)

---

{{< toc >}}


---
## Backstory

Docker is awesome. It allows for easy deployment of apps that I or others have made. Docker Compose makes the deployment of container stacks even easier by allowing you to control the containers via a YAML file. This is all fantastic, but the thing that has always driven me mad is finding a way to deploy them via automation.

For quite awhile, I've been trying to look for good ways to automate the deployment and updates of Docker containers, while keeping the simplicity of writing docker-compose files. I started off trying to implement this via Ansible and [AWX](https://github.com/ansible/awx), as that is the automation platform I use for a lot of my other homelab configurations, but it does not really seem designed to do that sort of thing. The Docker module in Ansible allows for the deployment of containers, but will only accept its own type of `yml` config. I already had a bunch of docker-compose files, so that was a no go. 

Then I had the idea to go full bore and try and migrate everything over to Kubernetes using Rancher. While this isn't a bad idea, and definitely something I want to shoot for in the future, it was _way_ more complicated than what I was looking for at the time. I was looking for something simple and was able to use what configuration I already had to manage it. I was getting to the point where I thought I would have to build an application to deploy docker-compose files over the Docker API using the TCP socket.

Eventually, while I was talking with one of my co-workers about some other container related stuff he was working on, I remembered Portainer! I had used it when I first started learning about containers, and used it as an easy way to get docker containers up and running. But when I learned more about how Docker works, and especially once I learned about docker-compose, I kind of left it in the dust.

I went ahead and had a look at what they had to offer in terms of deployment and CI/CD in general, and it was _exactly_ what I was looking for.

---

## Overview

![portainer-ci](/img/portainer-ci.png)

Essentially, I can setup a git repo (in this case, I'm using Gitlab) that holds the docker-compose file and any supporting config files to send a webhook call to Portainer whenever a push is made to the master branch. The webhook will trigger Portainer to pull the updated config files and re-deploy the stack. 

This workflow allows for plenty of CI testing inside of Gitlab using its runners to test images and config. For the time being, I won't be setting up that side, but will be sure to link to an update once I get this setup.

Notes:
- I am assuming you have already written your docker-compose files and stored them in git.
- The same thing should work on Github, or any git server that supports webhooks, but in this case I am using Gitlab.
- Because I am using the SaaS version of Gitlab, I have to expose the Portainer service externally. I will be using Cloudflare for setup.

---

## Setting Everything Up

### Install Portainer

First things first is to get Portainer installed and setup. Most of what I'm about to say can be found in the [Portainer Docs](https://docs.portainer.io), and I would encourage you to go there to get the most up to date info on how to properly deploy it.

With that, there is actually an important distinction that you should be aware of. Portainer offers two different versions of their product: Portainer Community Edition and Business Edition. For most people, Community Edition will likely be just fine, but some of the more advanced features (such as forcing a redeployment when Portainer pulls the new git config, which I need for this setup) are locked behind Business Edition. As of writing, Portainer offers [5 Free Nodes](https://www.portainer.io/pricing/take5?hsLang=en) of Business Edition when you sign-up with a business email. So if you have a business you are using this for, sign-up and get the extra features! 

I deployed Portainer on my Docker host using the following command:

```

docker run -d -p 8000:8000 -p 9000:9000 -p 9443:9443 --name portainer \

--restart=always \

-v /var/run/docker.sock:/var/run/docker.sock \

-v ~/portainer/portainer-config:/data \

portainer/portainer-ce

```

It is very important that you give Portainer access to `/var/run/docker.sock`, this is how Portainer will communicate with the Docker API. Without it, you will be unable to manage containers properly. _Note that in doing so, you are giving Portainer administrative access to all of your other containers, but most importantly, can give access to the host system as the Docker daemon runs as root. While Portainer is probably trustworthy, use your best judgment when granting access to this socket. More information can be found [here] (https://raesene.github.io/blog/2016/03/06/The-Dangers-Of-Docker.sock/).

Once you run the docker command, Portainer should be reachable on port `9000` or `9443` if you want to use HTTP or HTTPS respectively. From here, you should be prompted to setup your admin credentials (if you are using Business Edition, you will be prompted for your license key first). 

Next, you will need to add your first environment. You can go ahead and select Docker and connect to the local Docker socket. If it worked correctly, you should see a Docker Environment labelled _local_ on the Home tab:

![portainer-local-env](/img/portainer-local-env.png)

If you already have other containers or docker-compose stacks in this instance, they should show up already (though they wont be able to be controlled by Portainer since docker-compose is the owner of those stacks). If everything shows up, then it should be installed and good to go!

### Connecting Additional Docker Hosts

If you want to be able to deploy to and monitor additional Docker instances, then you will need to connect Portainer to that machine's Docker socket as well. There are a few ways to go about it, mainly exposing that socket to the network. _In this write-up, I took the least secure route because I wanted to get it running as soon as possible; however, I do **NOT** recommend doing it this way because it will openly expose your Docker socket to your network unauthenticated.I recommend reading up on the [Docker Wiki](https://docs.docker.com/engine/security/protect-access/) for best practices. _

We will be editing the Docker daemon to add a flag to expose the socket over TCP.

1. Log into your machine
2. Assuming you're using systemd to run Docker, modify `/etc/systemd/system/multi-user.target.wants/docker.service`.
3. Add `-H tcp://0.0.0.0:2375` to the command in the `ExecStart` parameter under `[Service]` and save the file
4. Reload the systemd daemon: `sudo systemctl daemon-reload`
5. Restart the Docker service: `sudo systemctl restart docker.service`

Here is what the service section for my Docker service looks like for reference:

```

[Service]
Type=notify
# The default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2375 --containerd=/run/containerd/containerd.sock
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

```

You can verify that Docker came up properly via `systemctl status docker.service`, if you see `-H tcp://0.0.0.0:2375` in the `usr/bin/dockerd` command, then it should be working. 

From there, we can add the environment to Portainer: 

1. Select _Environments_ on the side tab and _Add Environment_.  
2. Select _Docker_ for the Environment Type, and give it an appropriate name. 
3. Put the IP address of the docker host in _Environment URL_ along with port `2375`: `10.0.5.253:2375` for example
4. For _Public IP_, just put in the IP address of the host
5. Click 'Add Environment'

![portainer-add-env](/img/portainer-add-env.png)

Repeat all these steps for any other host that you want to add, and they should appear in your Home tab just the same as your local environment:

![all-envs](/img/all-envs.png)

### Setting up a Portainer Stack

Now, we can setup a docker stack using Portainer. 

- To get started, go into the node you want to deploy to and head to Stacks and select `Add Stack`.
- For the Build Method, select `Repository`.
- Add the GitLab repo link under in `Repository URL`
- For the repository reference, add the branch you would like to pull from. In my case, I want to pull from prod, so I would use: `refs/heads/prod`
- If you need to authenticate to the git repo, enable Authentication and add your username and a Personal Access Token.
- If your Docker stack requires additional environment variables that can't go in your compose stack, you can either import them via a .env file or add them one by one under Environment Variables.

![add-stack.png](/img/add-stack.png)

Once done, you can test a deployment and make sure that the stack downloads and deploys properly. Next, it is on to creating the webhooks.

### Setup GitLab Webhooks

Now that Portainer is setup locally, we can setup our repo's inside of GitLab to call Portainer WebHook's when we push to a branch. Note that you will need to expose the Portainer webhook url to the internet, or at least GitHub, to allow connections in. Once you have your git repo created with your docker compose file and any supporting config, you can get started by creating the webhook on Portainer. [Here](https://gitlab.com/dmarshall-homelab/docker/shlink) is a basic example of one of my repo's for a link shortener I use. 

#### Creating the Webhook URL

- Go into your stack config and edit the stack you deployed earlier.
- Check `Automatic Updates`
- Set `Mechanism` to Webhook
- Enable `Force Redeployment` (This is the feature that was Business Edition only)
- Copy the webhook URL (Note that it uses the internal hostname of Portainer, so change the beginning URL to whatever domain you are using externally).
- Save Settings

![auto-updates.png](/img/auto-updates.png)

#### Setup Webhook Events in Gitlab

- Go to Settings -> Webhooks inside the repo
- For the URL - Put the publicly accessible webhook URL here
- Under Triggers: Make sure `push events` is selected and you put in the branch you want to cause triggers to the webhook, in my case I am using `prod`
- Save Your settings

![webhook.png](/img/webhook.png)

From here, you can use the 'Test' button next to your webhook to verify that it works. You should see the portainer restart the container. At this point, whenever you push to the branch you specified, it should deploy your new configuration. 

---

## Wrapping Up

After that, you are pretty much good to go! Changes to your compose stacks will be automatically deployed on pushes to the repo. You will still need to add stacks manually initially, but afterwards it is completely automated. Next, I will be working on interacting with the Portainer API to deploy these stacks automatically, so when I create a github repo, I can simply trigger some form of automation to create the stack on the Portainer itself. But that will be a future project. For now, I am happy with where I am at. I will surely make tweaks and improvements as I use it more.

If anything didn't make sense, or if you have any questions, feel free to reach out to me and I would be happy to lend a hand!
