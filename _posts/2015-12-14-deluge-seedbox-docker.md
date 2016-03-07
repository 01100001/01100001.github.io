---
layout: post
title: "Set Up Seedbox With Deluge: Server & Thin Client Config."
date: 2015-12-14T17:21:27+00:00
updated: 2016-02-20T19:19:34+00:00
order: 3
---

From the [official Deluge wiki](http://dev.deluge-torrent.org/wiki/UserGuide/ThinClient), about the server/thin-client setup:

> Deluge can be setup in such a way that a Deluge daemon, *deluged*, can be running on a central computer, *server*, which can then be accessed and controlled by other computers, *clients*, using one of Deluge's UIs.

To put it simply, we'll be running the Deluge daemon `deluged` (headless, i.e. no GUI) on a server and connect to it remotely via Deluge (with GUI) installed on our computer. The latter acts as an interface to the daemon running on the server.

We'll be running the Deluge daemon (`deluged`) within a Docker container on the server. Here's the command I use to create the docker container for this purpose:

	docker run -d -i -t --name="deluge" -h="deluge" -p 54321:54321 -p 54321:54321/udp -p 54322:54322 -p 54322:54322/udp -v /home/pinky/torrents:/torrents -e "LANG="en_US.UTF-8"" -e "LANGUAGE="en_US:en"" --restart=always ubuntu:trusty /bin/bash

> Make sure your firewall is configured to allow traffic through the port(s) mentioned in the command. Going by the example above, I'd just have to run this command on the host beforehand:
> 
> 	sudo ufw allow 54321 && sudo ufw allow 54322
> 
> As for why these ports are necessary, you'll learn as the tutorial progresses.

Then attach to the running docker container:

	docker attach deluge1

First things first:

- Generate the default locale(s):

		locale-gen en_US en_US.UTF-8

- Create a new sudo user; then impersonate the user:

		adduser pinky
		adduser pinky sudo
		su pinky

	Reason: In Docker containers, the commands are run as root user by default. This could cause permission issues with shared volumes. For example, when I start Deluge as root user in the container, the files and directories Deluge creates (e.g. the config. directory), for some weird reason, cannot be modified on the host by the user that runs the Docker container. No issues with running Deluge as a non-root user, hence preferred. (I even use the same username as the host user running Docker; with a simpler password of course.)

### Install Deluge (Server Setup)

To install the latest Deluge packages, add the official PPA to the sources:

	$ sudo add-apt-repository ppa:deluge-team/ppa
	bash: add-apt-repository: command not found

`apt-add-repository` is just not in the base Ubuntu image. You'll first have to install it.

By the way, you don't need to use sudo in the Docker container because the commands are run as root by default. ([Source](http://stackoverflow.com/a/32487026); [also see](http://askubuntu.com/a/55484).) Install `vim` as we might need it later, even if not in this tutorial.

	sudo apt-get update && sudo apt-get -y install software-properties-common vim

Now add the official PPA and install the Deluge daemon and console packages:

	sudo add-apt-repository ppa:deluge-team/ppa
	sudo apt-get update && sudo apt-get -y install deluged deluge-console

Always combine `apt-get update` with `apt-get install` in the same command as shown above. Using `apt-get update` alone in a container causes caching issues and subsequent `apt-get install` instructions may fail. [More info here](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#apt-get).

Start the Deluge daemon; this creates the config directory and populates with the default files:

	deluged -p 54321 -c /torrents/deluge1/.config

- The default Deluge daemon port is `58846`. It's through this port that the Deluge daemon on the server and the thin-client on computer connect and communicate with each other. But we'll use a different port (between [49152-65535 is recommended](http://dev.deluge-torrent.org/wiki/Faq#WhichportsshouldIuse)) to keep any easy attacks at bay. The `-p` flag in the command above allows us to define the port the daemon will listen on.

- We also used the `-c` flag to set a different config. location for our Deluge instance (default is `~/.config/deluge`). As you can see, I've asked for the config. to be stored in a directory within the shared volume (accessible on both the host and the container). All our torrents and downloads will also be stored under `/torrents/deluge1` directory as we'll configure in the thin-client later.

- Similarly, you can run multiple instances of Deluge simultaneously, if you need to, by passing different ports and config. locations while starting the daemon each time.

### Basic Deluge Daemon Config.

[Add new user](http://dev.deluge-torrent.org/wiki/UserGuide/Authentication) to the authentication file located in the config. directory:

	echo "pinky:PaSsWoRD123:10" >> /torrents/deluge1/.config/auth

Open `deluge-console` and [connect with the daemon](http://dev.deluge-torrent.org/wiki/UserGuide/ThinClient#Console):

	deluge-console -c /torrents/deluge1/.config
	connect 127.0.0.1:54321

Normally you could just run `deluge-console` and you'd be automatically connected to the daemon. But as we are using different port and config. location, we need to pass them along in two commands.

In the same console screen, configure the daemon to allow remote connections, verify the same, and then exit:

	config -s allow_remote True
	config allow_remote # TO VERIFY
	exit

Now would be a good time to restart (stop, then start) the daemon for the config. changes to take effect:

	killall deluged
	deluged -p 54321 -c /torrents/deluge1/.config

If you get *"command not found"* error on running `killall`, you'll first have to install the `psmisc` package. To do so, run `apt-get update && apt-get -y install psmisc`.

### Client Setup

**Pre-requisite:** Deluge GUI installed on your PC. [Download it here](http://dev.deluge-torrent.org/wiki/Download).

Configure the client to connect to the Deluge daemon on the server:

- Go to *Preferences > Interface* and disable *Classic Mode*.

- Restart Deluge and you should see the *Connection Manager*. You can also access it from *Edit > Connection Manager*.

- Create a new entry with the *Add* button:

	- Hostname is your server's IP address.

	- Port should be the one the daemon will be listening on (and as per our example earlier, it's 54321).

	- Username and Password are those we added to the `deluged` config auth file (*pinky* and *PaSsWoRD123* in our case).

	If this was successful a green indicator should now appear as the status for the daemon you just added. Click on *Connect* and the *Connection Manager* should disappear.

- Expand *Options* and modify the additional settings to suit your specific needs.

You are now connected to the remote daemon on the server from Deluge (GUI) client on your PC.

**Disconnect Properly:**

If you don't disconnect from the remote daemon on the server properly, you could end up stopping the daemon each time you close the Deluge client on your PC. This is how it's done:

1. Go to *Edit > Connection Manager* (or `Ctrl+M`).

2. Select the remote host that you are currently connected to.

3. Click *Disconnect* (or simply hit the `Enter` key).

4. Proceed with closing Deluge client on your PC; the remote deluge daemon will continue to run on the server.

### Deluge Configuration (Using Thin-Client)

With the basic stuff out of the way, we are ready to deal with the actual configuration.

(May come at a later time. Meanwhile, take a look at [my notes on configuring rTorrent]({% post_url 2016-02-12-rtorrent-rc-config %}). It should be of some help with configuring Deluge as well.)

### Sources & Further Reading

- What.cd: [Ubuntu Server 12.04 LTS / Latest Deluge (ThinClient & Web-UI) / autodl-irssi](https://what.cd/forums.php?action=viewthread&threadid=176875)

- Deluge Wiki: [Index](http://dev.deluge-torrent.org/wiki/TitleIndex), esp.

	- [Ubuntu Install Guide](http://dev.deluge-torrent.org/wiki/Installing/Linux/Ubuntu)
	- [Thin Client Setup](http://dev.deluge-torrent.org/wiki/UserGuide/ThinClient)
	- [FAQ](http://dev.deluge-torrent.org/wiki/Faq)
	- [User Guide](http://dev.deluge-torrent.org/wiki/UserGuide)

- ArchWiki: [Deluge](https://wiki.archlinux.org/index.php/Deluge)

- Worth checking: Deluge [ltconfig plugin](https://github.com/ratanakvlun/deluge-ltconfig) lets users modify libtorrent settings that are otherwise not exposed through Deluge. E.g. you can allow multiple connections per IP, then you'll be able to connect to all the seed boxes on same IP.

- [Feral Hosting - FAQ](https://www.feralhosting.com/faq/)

- [Whatbox.ca - Wiki](https://whatbox.ca/wiki)

	- [Deluge](https://whatbox.ca/wiki/deluge)
	- [Deluge Console Documentation](https://whatbox.ca/wiki/Deluge_Console_Documentation)

- [ChmuraNet Forum](http://www.chmuranet.com/forum/index.php) & [FAQ](http://www.chmuranet.com/forum/index.php?p=/faq/faq)

- Jesse Schoff's [rTorrent + ruTorrent Configuration Guide](https://jes.sc/kb/rTorrent+ruTorrent-Seedbox-Guide.php)

- GitHub Repository: [Deluge ltConfig Plugin](https://github.com/ratanakvlun/deluge-ltconfig)


<!--

TO DOs

- How to do this with Deluge: Initial seeding AKA Super seeding

- How to backup and transfer Deluge seedbox from one server to another without going through all the rechecking process? It'd probably involve copying over the config directory (~/.config/deluge), esp. the `state` directory in it (~/.config/deluge/state).

- deluge speed optimization & fine tuning

-->