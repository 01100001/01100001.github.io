---
layout: post
title: "Set Up Seedbox With rTorrent-PS & PyroCore"
date: 2016-02-17T11:12:01+00:00
updated: 2016-02-20T19:19:37+00:00
order: 5
---

*The [rTorrent seedbox setup guide]({% post_url 2016-02-12-rtorrent-seedbox %}) is also the basis of this guide, so please go and get the gist of it first.*

Let's begin by creating a Docker container on the server in which to run rTorrent-PS (or simply, rTorrent):

	docker run -d -i -t --name="rtorrent" -h="rtorrent" -p 54324:54324 -p 54324:54324/udp -p 54325:54325 -p 54325:54325/udp -v /home/pinky/torrents:/torrents -e "LANG="en_US.UTF-8"" -e "LANGUAGE="en_US:en"" --restart=always ubuntu:trusty /bin/bash

- Where, `54324` is the port through which network traffic to/from rTorrent passes. Port `54325` is to be handy just in case we need another later.

- Configure `ufw` to allow traffic through these ports:

		sudo ufw allow 54324 && sudo ufw allow 54325

Then attach to the running docker container:

	docker attach rtorrent

First things first:

- Generate the default locale(s):

		locale-gen en_US en_US.UTF-8

- Create a new sudo user; then impersonate the user:

		adduser --home /torrents/rtbox dinky
		adduser dinky sudo
		su dinky

	Reason: In Docker containers, the commands are run as root user by default. This could cause permission issues with shared volumes. For example, when I start rTorrent as root user in the container, the files and directories rTorrent creates (e.g. session and downloads), for some weird reason, cannot be modified on the host by the user that runs the Docker container. I faced no issues running rTorrent as a non-root user, hence this is preferred. Besides, you must do this if you going to be installing rTorrent-PS with PyroCore CLI tools. PyroCore was designed to prevent installation as the root user. Also, I use a different username for the container to avoid confusion.

Install the required development tools and just the dependencies required for compiling XML-RPC, libTorrent, rTorrent, rTorrent-PS and PyroScope from source (refer to the [previous rTorrent setup guide]({% post_url 2016-02-12-rtorrent-seedbox %}) for more info):

	sudo apt-get update && sudo apt-get -y install build-essential automake autoconf libtool pkg-config intltool checkinstall curl git subversion vim autotools-dev debhelper libcurl4-openssl-dev quilt debhelper dh-autoreconf libcppunit-dev libcurl4-openssl-dev libsigc++-2.0-dev libssl-dev bc debhelper dh-autoreconf libcppunit-dev libcurl4-openssl-dev libncurses5-dev libncursesw5-dev libsigc++-2.0-dev tmux python-dev python-setuptools python-pkg-resources python-virtualenv python-pip

### Install [rTorrent-PS](https://github.com/pyroscope/rtorrent-ps)

Get the build script and run it to build and install rTorrent and dependencies (libTorrent, XML-RPC, etc.) from source:

	mkdir -p ~/src
	cd ~/src
	git clone https://github.com/pyroscope/rtorrent-ps.git
	cd rtorrent-ps
	./build.sh all

Then, to set up the extended version with some stability fixes and extension patches applied, i.e. install rTorrent-PS on top of rTorrent, run:

	./build.sh extend

> All installations go to `~/lib/rtorrent-<version>`, and you'll find these binaries installed in `~/lib/rtorrent-<version>/bin`:
> 
> - `rtorrent-vanilla`: Unpatched/untouched/original version of rTorrent, created after you run `./build.sh all`.
> 
> - `rtorrent-extended`: rTorrent-PS AKA rTorrent Extended, created after you run `./build.sh extend`.
> 
> - `rtorrent`: First created as a copy of `rtorrent-vanilla` when you run `./build.sh all`. But when you run `./build.sh extend`, the existing `rtorrent` binary is replaced with a copy of the then generated `rtorrent-extended`. How and why?
> 
> When you run `./build.sh extend`, [the build script](https://raw.githubusercontent.com/pyroscope/rtorrent-ps/master/build.sh) used to install rTorrent-PS automatically, symlinks the executable `rtorrent-extended` into your path (i.e. `~/bin`), by doing something like this behind the scenes:
> 
> 	mkdir -p ~/bin
> 	ln -nfs ~/lib/rtorrent-0.9.6/bin/rtorrent-extended ~/bin/rtorrent-0.9.6
> 	ln -s ~/bin/rtorrent-0.9.6 ~/bin/rtorrent
> 
> So when you run `rtorrent` you'd actually be starting rTorrent-PS. If you want to run the unpatched version `rtorrent-vanilla` instead, you can simply switch by changing the symlink in `~/bin`, or by calling it with its full path `~/lib/rtorrent-<version>/bin/rtorrent-vanilla`.

### Install [PyroCore](https://github.com/pyroscope/pyrocore) Utilities

The recommended way to install PyroCore is directly from its GitHub repository which also includes the build script. So:

	mkdir -p ~/bin ~/lib

	git clone "https://github.com/pyroscope/pyrocore.git" ~/lib/pyroscope

	~/lib/pyroscope/update-to-head.sh

	# adds ~/bin to your PATH
	exec $SHELL -l

	# Check success
	pyroadmin --version

### Set Up rTorrent Instance

First, create directories for the instance `rtorrent1`, then get the simple start script needed to start the instance and make it executable (we won't start the instance itself though):

***NOTE:** You might as well go with `rtorrent` instead of `rtorrent1`. The latter has only been used to indicate to you how setting up multiple instances of rTorrent could be done.*

	# Set environment variable `$RT_HOME` pointing to `~/rtorrent1`,
	# the main directory of our rTorrent instance
	export RT_HOME="${RT_HOME:-$HOME/rtorrent1}"

	# Create the necessary directories under `~/rtorrent1`
	mkdir -p $RT_HOME/{.session,work,done,log,watch/start,watch/load}

	# Change directory to `~/rtorrent1`
	cd $RT_HOME

	# Copy the start script provided by PyroCore to our pwd
	cp ~/lib/pyroscope/docs/examples/start.sh ./start

	# Make the start script executable
	chmod a+x ./start

> Instead of naming the rTorrent instances ambiguously such as rtorrent1, rtorrent2, etc., I'd recommend naming them after the category of torrents or the tracker.
> 
> And if you plan to take this route, you could set the sudo user's home directory to `/torrents/rtorrent` as well. That is, instead of this as detailed in one of the earliest steps:
> 
> 	adduser --home /torrents/rtbox dinky
> 
> Do this:
> 
> 	adduser --home /torrents/rtorrent dinky
> 
> Reason being, `/torrents/rtorrent/rtorrent1` doesn't look pretty or organized; `/torrents/rtorrent/movies` does.

### rTorrent Configuration

PyroScope/PyroCore provides an example configuration file, albeit with old rTorrent v0.8.6 syntax (which therefore needs to be converted to the new syntax), that already contains everything needed to use all features of rTorrent and PyroScope. Copy it to the directory of your rTorrent instance (`~/rtorrent1` as per e.g.); **check at least the first section and adapt the values to your environment**. ([My notes on configuring rTorrent]({% post_url 2016-02-12-rtorrent-rc-config %}) might be of some help here.)

	export RT_HOME="${RT_HOME:-$HOME/rtorrent1}"

	## Copies the example rtorrent.rc to your rTorrent instance's home while
	## replacing all instances of`RT_HOME` with the actual path `~/rtorrent1`
	sed -e "s:RT_HOME:$RT_HOME:" <~/lib/pyroscope/docs/examples/rtorrent.rc >$RT_HOME/rtorrent.rc

Immediately, run the migration script on the configuration to convert it to the new syntax:

	export RT_HOME="${RT_HOME:-$HOME/rtorrent1}"

	bash ~/lib/pyroscope/src/scripts/migrate_rtorrent_rc.sh $RT_HOME/rtorrent.rc

### PyroScope Configuration

To create your own configuration, the best way is to start from the default files that are part of your PyroScope installation. Let's create them for our rTorrent instance at the default location (i.e. `~/.pyroscope`):

	pyroadmin --create-config

> If for some reason you want the config. stored at a different location, you may use the `--config-dir` option to do so. And if/when you do that, you also need to tell your rTorrent instance about this. To do so, replace all instances of `~/.pyroscope` in your rtorrent.rc (e.g. `~/rtorrent1/rtorrent.rc`) with `~/some/directory/.pyroscope`.

The following places a minimal configuration in the `config.ini`, so that the defaults are always taken from the installed software. So when you update PyroCore next time, and some defaults have changed in the update, you don't have to do a thing. This makes later updates a lot easier.

	cat >~/.pyroscope/config.ini <<EOF
	# PyroScope configuration file
	#
	# For details, see http://pyrocore.readthedocs.org/en/latest/setup.html
	#

	[GLOBAL]
	# Note that the "config_dir" value is provided by the system!
	config_script = %(config_dir)s/config.py
	rtorrent_rc = ~/rtorrent1/rtorrent.rc

	[ANNOUNCE]
	# Add alias names for announce URLs to this section; those aliases are used
	# at many places, e.g. by the "mktor" tool

	# Public trackers
	# PBT     = http://tracker.publicbt.com:80/announce
	#           udp://tracker.publicbt.com:80/announce
	# OBT     = http://tracker.openbittorrent.com:80/announce
	#           udp://tracker.openbittorrent.com:80/announce
	# Debian  = http://bttracker.debian.org:6969/announce
	EOF

Now that the config. is in place, change the value of `pyro.extended` to `1` in your `rtorrent.rc` so that the additional rTorrent-PS features that are dependent on PyroCore are actually activated.

### Tmux Configuration & Setup

We spruce up `tmux` a bit using a custom configuration, before we start it for the first time. This makes it more homey for long-time `screen` users:

	# Run this with your NORMAL user account!
	cp --no-clobber ~/lib/pyroscope/docs/examples/tmux.conf ~/.tmux.conf

You're now ready to start your first rTorrent instance. So start a `tmux` session to run rTorrent:

	tmux -2u new -s rtorrent -n rtorrent1 "~/rtorrent1/start; exec bash"

The `exec bash` keeps your `tmux` window open if rTorrent exits, which allows you to actually read any error messages in case rTorrent ends unexpectedly.

> So your `tmux` command for running another instance of rTorrent, say `rtorrent2`, would look like:
> 
> 	tmux -2u new -s rtorrent -n rtorrent2 "~/rtorrent2/start; exec bash"
> 
> Then all you have to do is switch windows in the tmux session to access different rTorrent instances.

### Sources

- GitHub Repository: [LibTorrent](https://github.com/rakshasa/libtorrent)

- GitHub Repository: [rTorrent](https://github.com/rakshasa/rtorrent)

	- [rTorrent Wiki](https://github.com/rakshasa/rtorrent/wiki)

	- [Old rTorrent man page](http://web.archive.org/web/20150511123211/http://libtorrent.rakshasa.no/rtorrent/rtorrent.1.html), [2](http://linux.die.net/man/1/rtorrent)

- GitHub Repository: [rTorrent-PS](https://github.com/pyroscope/rtorrent-ps)

- GitHub Repository: [PyroCore](https://github.com/pyroscope/pyrocore)

	- [PyroScope wiki from old Google Code repo](https://github.com/pyroscope/pyroscope/tree/wiki)

	- [PyroCore Docs](https://pyrocore.readthedocs.org/en/latest/)

	- [PyroCore CLI Tools: mktor](https://github.com/pyroscope/pyroscope/blob/wiki/CommandLineTools.md#mktor)

Installation:

- jes.sc: [rTorrent + ruTorrent Configuration Guide](https://jes.sc/kb/rTorrent+ruTorrent-Seedbox-Guide.php)

- Old PyroScope Wiki: [Install from source on Debian](https://github.com/pyroscope/pyroscope/blob/wiki/DebianInstallFromSource.md)

- [PyroCore Docs](https://pyrocore.readthedocs.org/en/latest/)

Configuration:

- [Old rTorrent man page](http://web.archive.org/web/20150511123211/http://libtorrent.rakshasa.no/rtorrent/rtorrent.1.html), [2](http://linux.die.net/man/1/rtorrent)

- [rtorrent.rc template from the Seedbox From Scratch project by Notos](https://github.com/Notos/seedbox-from-scratch/blob/v2.1.9/rtorrent.rc.template)
