---
layout: post
title: "Use Docker Containers For Portability: Basic Setup"
date: 2015-12-09T18:52:10+00:00
updated: 2016-02-20T19:19:04+00:00
order: 2
---

[Once I read this](https://www.reddit.com/r/DataHoarder/comments/3ve1oz/torrent_client_that_can_handle_lots_of_torrents/cxndyzx), I was hooked:

> I seed 85,000 torrents using Docker with 20 Transmission instances. I just moved to a bigger server and Docker is great because I just imported my containers so no torrent rechecking :)

If you've ever moved your torrents between seedboxes or servers (self-hosted), you'd know how painfully long the process can be. So, here's my notes on setting up docker to host my application(s).

**Pre-requisite:** Host system running Ubuntu Server 14.04 LTS 64-bit.

### Install Docker

Make sure that the system is up-to-date:

	sudo apt-get update && sudo apt-get -y upgrade

Make sure that `aufs` storage driver support is available by installing `linux-image-extra` kernel package:

	sudo apt-get install linux-image-extra-$(uname -r)

Reboot:

	sudo reboot

Add docker repository's GPG key to `apt-key` for package verification:

	sudo apt-key adv --keyserver hkp://pgp.mit.edu:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D

Set `apt` sources to use packages from docker's repository:

	echo "deb https://apt.dockerproject.org/repo ubuntu-trusty main" | sudo tee /etc/apt/sources.list.d/docker.list

Update the `apt` package index for the changes to take effect:

	sudo apt-get update

Verify that `apt` is now pulling from the right repository:

	apt-cache policy docker-engine

Install docker:

	sudo apt-get -y install docker-engine

Add the non-root user (with `sudo` priviliges) to the "docker" group so that you don't have to use `sudo` with the `docker` command:

	sudo usermod -aG docker pinky

Either log out/in, or run this command for the new permissions to take effect:

	newgrp docker

### Configure Uncomplicated Firewall (UFW)

By default, UFW drops all forwarding traffic. So, for docker to manage container networking when UFW is enabled, change UFW's forwarding policy in `/etc/default/ufw` file to this:

	DEFAULT_FORWARD_POLICY="ACCEPT"

View current configuration:

	$ sudo ufw status verbose

	Status: active
	Logging: on (low)
	Default: deny (incoming), allow (outgoing), allow (routed)
	New profiles: skip

	To                         Action      From
	--                         ------      ----
	80/tcp                     ALLOW IN    Anywhere
	443                        ALLOW IN    Anywhere
	43713                      ALLOW IN    Anywhere
	80/tcp (v6)                ALLOW IN    Anywhere (v6)
	443 (v6)                   ALLOW IN    Anywhere (v6)
	43713 (v6)                 ALLOW IN    Anywhere (v6)

### Install Base Image

I am going to go with *Ubuntu Server 14.04 LTS (Trusty Tahr)* as the base image for my docker container.

To get the official pre-built image from the docker registry, run any one of the commands:

	docker pull ubuntu:trusty # PREFERRED

	docker pull ubuntu:14.04

	docker pull ubuntu:14.04.3

The [official docker Ubuntu image tags](https://github.com/docker-library/docs/blob/master/ubuntu/tag-details.md) for Ubuntu Server 14.04 LTS (Trusty Tahr), namely `trusty`, `14.04` and `14.04.3`, are aliases of each other. So the three commands above will do the same thing.

### Launch A Container

To launch a container in the base (Ubuntu Server) image we downloaded earlier, issue the command:

	docker run -d -i -t --name="pinkyblog" -h="pinkyblog"  -p 31625:31625 -p 31625:31625/udp -v /home/pinky:/pinky --restart=always ubuntu:trusty /bin/bash

- `docker run` to launch an `ubuntu:trusty` image and run the command `/bin/bash` (launches bash shell) inside a new container

- `-d` to keep the container running in the background (i.e. as a daemon)

- `-i` to start an interactive connection with the new container (and keep `STDIN` open)

- `-t` to assign a pseudo-terminal (tty) inside the new container

- `--name` to assign a name to the container (to use instead of ID)

- `-h` (`--hostname`) to set container hostname

- `-p` to map a container's port, or range of ports, to the host.  You might want to look at the `-P` flag as well

- `-v` to create a bind mount, i.e. mount a directory from host to a location in the container

- [`--restart=always`](http://serverfault.com/a/649835) to auto-start Docker container after system reboot.

(Don't forget to configure your firewall to open the port(s) mentioned in the command. In my case I'd have to run `ufw allow 31625` on the host (NOT within the docker container).)

### Working with Containers

Once you've created a docker container for your application, attach to the running container `pinkyblog` with:

	docker attach pinkyblog

We are now interactively connected to the pseudo-tty inside the container (because we created the container using `-i` and `-t`), and we can start working within the container as we'd on a normal server.

We can detach from the container without stopping it (quiet exit) with `Ctrl-p, Ctrl-q`.

### (Optional) Setting Locale In A Container

When installing some packages in a Docker container, you might notice a warning similar to this:

	perl: warning: Setting locale failed.   
	perl: warning: Please check that your locale settings:   
	        LANGUAGE = "en_US:en",   
	        LC_ALL = (unset),   
	        LC_MESSAGES = "en_US.UTF-8",   
	        LANG = "en_US.UTF-8"   
	    are supported and installed on your system.   
	perl: warning: Falling back to the standard locale ("C").   
	locale: Cannot set LC_CTYPE to default locale: No such file or directory   
	locale: Cannot set LC_MESSAGES to default locale: No such file or directory   
	locale: Cannot set LC_ALL to default locale: No such file or directory

That's because the default locale (`en_US` usually) is missing in the Ubuntu Docker image and needs to be set up for the installation to go smoothly. Just to confirm, run the `locale` and `locale -a` commands; their output would look like this:

	root@pinkyblog:/# locale
	LANG=
	LANGUAGE=
	LC_CTYPE="POSIX"
	LC_NUMERIC="POSIX"
	LC_TIME="POSIX"
	LC_COLLATE="POSIX"
	LC_MONETARY="POSIX"
	LC_MESSAGES="POSIX"
	LC_PAPER="POSIX"
	LC_NAME="POSIX"
	LC_ADDRESS="POSIX"
	LC_TELEPHONE="POSIX"
	LC_MEASUREMENT="POSIX"
	LC_IDENTIFICATION="POSIX"
	LC_ALL=

	root@pinkyblog:/# locale -a
	C
	C.UTF-8
	POSIX

That isn't very good, and on a normal system you'd fix it like this (may also be used to change locale to a different one):

	locale-gen en_US en_US.UTF-8
	dpkg-reconfigure locales

But this doesn't work in a docker container, and [here's why](http://askubuntu.com/a/582197):

> The /etc/default/locale file is loaded by PAM; see /etc/pam.d/login for example. However, PAM is not invoked when running a command in a Docker container. To configure the locale, simply set the relevant environment variable in your Dockerfile. Example:
> 
> 	FROM ubuntu:trusty
> 
> 	# Set the locale
> 	RUN locale-gen en_US en_US.UTF-8
> 	ENV LANG en_US.UTF-8
> 	ENV LANGUAGE en_US:en
> 
> 	CMD ["/bin/bash"]

To put it simply, while you can generate locales using `locale-gen`, it isn't possible to configure the container OS for the new locale to take effect. You'd have to set the environment variables `LANG` and `LANGUAGE` at the time of creation of the container using the `-e` flags in the `docker run` command. Like so:

	docker run -d -i -t --name="pinkyblog" -h="pinkyblog"  -p 31625:31625 -p 31625:31625/udp -v /home/pinky/assets:/assets -e "LANG="en_US.UTF-8"" -e "LANGUAGE="en_US:en"" --restart=always ubuntu:trusty /bin/bash

Then once the container is created, `attach` to it, and in the container, generate the requisite locale(s) like so:

	locale-gen en_US en_US.UTF-8

That's it, but let's confirm just to be sure:

	root@pinkyblog:/# locale
	LANG=en_US.UTF-8
	LANGUAGE=en_US:en
	LC_CTYPE="en_US.UTF-8"
	LC_NUMERIC="en_US.UTF-8"
	LC_TIME="en_US.UTF-8"
	LC_COLLATE="en_US.UTF-8"
	LC_MONETARY="en_US.UTF-8"
	LC_MESSAGES="en_US.UTF-8"
	LC_PAPER="en_US.UTF-8"
	LC_NAME="en_US.UTF-8"
	LC_ADDRESS="en_US.UTF-8"
	LC_TELEPHONE="en_US.UTF-8"
	LC_MEASUREMENT="en_US.UTF-8"
	LC_IDENTIFICATION="en_US.UTF-8"
	LC_ALL=

	root@pinkyblog:/# locale -a
	C
	C.UTF-8
	en_US
	en_US.iso88591
	en_US.utf8
	POSIX

**NOTE:** It'd be a good idea to run the `locale` and `locale -a` commands on the host to see how it's configured as you'd probably want the same in your container(s).

### (Optional) Sudo User Instead Of Root User

Another thing that I do as soon as I set up my Docker container is create a new user with `sudo` privileges and use that user instead of the root user. As in...

	adduser dinky
	adduser dinky sudo
	su dinky

This seems especially important if you are mounting data volume(s) in your container. You may face permission issues on the host while modifying files and directories created by those applications being run by the root user in the container. Run those applications as a normal user with sudo privileges and you'll never have to turn back again.

**TIP!** For easy access to the mounted data volume via command-line within the container, and better sharing of data between the host and the container, I normally set the home directory of the sudo user (in the container) to a directory inside the mounted volume, like so:
	
	# /home/pinky/assets (Host) --> /assets (Container)
	adduser --home /assets/dinky dinky

This way I'll have full access to any of the user's files and applications from the host, and no permission issues whatsoever.

### Sources

- Docker Docs: [Installation on Ubuntu](https://docs.docker.com/engine/installation/ubuntulinux/)

- SO: [Some Docker image tags are aliases of each other...](http://stackoverflow.com/a/30383971)

- Discourse: [How Do I Install Discourse?](https://github.com/discourse/discourse/blob/master/docs/INSTALL.md)

	- [Beginner Docker install guide](https://github.com/discourse/discourse/blob/master/docs/INSTALL-cloud.md)

- Linode: [Docker Quick Reference](https://www.linode.com/docs/applications/containers/docker-quick-reference-cheat-sheet)

- [Docker Command Cheat Sheet](https://developer.basespace.illumina.com/docs/content/documentation/native-apps/manage-docker-image#DockerCommandCheatSheet)

- Linode: [How to Configure a Firewall with UFW](https://www.linode.com/docs/security/firewalls/configure-firewall-with-ufw)

- DO: [How To Set Up a Firewall with UFW on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-14-04)

- [How do I fix my locale issue?](http://askubuntu.com/q/162391)

- [How to configure locales to Unicode in a Docker Ubuntu 14.04 container?](http://askubuntu.com/q/581458)

- [Docker and Locales](http://jaredmarkell.com/docker-and-locales/)

<!--

### Temp Notes

Don't ever `apt-get upgrade` your containers!
http://crosbymichael.com/dockerfile-best-practices-take-2.html
Instead simply run `docker pull ubuntu:trusty` to update the base image.
Then restart all relevant containers for the updates to take effect.

docker exec -t -i 589f2ad30138 /bin/bash
docker exec -it ubuntu_bash bash
docker stats e64a279663aa # container stats

Copying files in and out of the container with `docker cp`
Saving a container's filesystem to a tarball with `docker export`
Saving an image to a tarball with `docker save`
Loading an image from a tarball with `docker import`

Stop a container

	docker stop <container_id>

Stop all containers

	docker stop `docker ps -a -q`

Start a container

	docker start <container_id>

Delete all containers:

	docker rm `docker ps -a -q`

Force delete all containers (without having to stop 'em first):

	docker rm -f `docker ps -a -q`

Delete all images:

	docker rmi `docker images -a -q`

Docker command help:

	docker --help
	docker run --help
	docker attach --help

See all running containers:

	docker ps

See all containers:

	docker ps -a

Exposing a Port on a Live Docker Container:
http://stackoverflow.com/a/21374974

	docker ps
	docker stop <container_id>
	docker commit <container_id> foo/ubuntu:trusty
	docker rm <container_id>
	docker run -i -p 22 -p 8000:80 -m /data:/data -t foo/ubuntu:trusty /bin/bash

-->