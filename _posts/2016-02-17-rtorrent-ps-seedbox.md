---
layout: post
title: "Set Up Seedbox With rTorrent-PS & PyroCore"
date: 2016-02-17T11:12:01+00:00
updated: 2016-02-20T19:19:37+00:00
---

> **NOTE:** This tutorial assumes that you logged with a normal user account with sudo privileges, i.e. not as root user. This is necessary if you going to be installing rTorrent-PS with PyroCore CLI tools. PyroCore was designed to prevent installation as the root user.
> 
> The [rTorrent seedbox setup guide]({% post_url 2016-02-12-rtorrent-seedbox %}) is also the basis of this guide, so please go and get the gist of it first.

Let's begin with updating the system; then install the required development tools and only the dependencies required for compiling XML-RPC, libTorrent, rTorrent, rTorrent-PS and PyroScope from source. Or precisely:

	sudo apt-get update && sudo apt-get upgrade

	sudo apt-get install build-essential automake autoconf libtool pkg-config intltool checkinstall curl git subversion vim autotools-dev debhelper libcurl4-openssl-dev quilt debhelper dh-autoreconf libcppunit-dev libcurl4-openssl-dev libsigc++-2.0-dev libssl-dev bc debhelper dh-autoreconf libcppunit-dev libcurl4-openssl-dev libncurses5-dev libncursesw5-dev libsigc++-2.0-dev tmux python-dev python-setuptools python-pkg-resources python-virtualenv python-pip

### Install [rTorrent-PS](https://github.com/pyroscope/rtorrent-ps)

Get the build script and run it to build and install rTorrent and dependencies (libTorrent, XML-RPC, etc.) from source:

	mkdir -p ~/tmp
	cd ~/tmp
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

	cd ~/tmp
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
	# the home directory of our rTorrent instance
	export RT_HOME="${RT_HOME:-$HOME/rtorrent1}"

	# Create the necessary directories under `~/rtorrent1`
	mkdir -p $RT_HOME/{.session,work,done,log,watch/start,watch/load}

	# Change directory to `~/rtorrent1`
	cd $RT_HOME

	# Copy the start script provided by PyroCore to our pwd
	cp ~/lib/pyroscope/docs/examples/start.sh ./start

	# Make the start script executable
	chmod a+x ./start

### rTorrent Configuration

PyroScope/PyroCore provides an example configuration file, albeit with old rTorrent v0.8.6 syntax (which therefore needs to be converted to the new syntax), that already contains everything needed to use all features of rTorrent and PyroScope. Copy it to the home directory of your rTorrent instance rTorrent1, i.e. `~/rtorrent1`; **check at least the first section and adapt the values to your environment**. ([My notes on configuring rTorrent]({% post_url 2016-02-12-rtorrent-rc-config %}) might be of some help here.)

	export RT_HOME="${RT_HOME:-$HOME/rtorrent1}"

	## Copies the example rtorrent.rc to your rTorrent instance's home while
	## replacing all instances of`RT_HOME` with the actual path `~/rtorrent1`
	sed -e "s:RT_HOME:$RT_HOME:" <~/lib/pyroscope/docs/examples/rtorrent.rc >$RT_HOME/rtorrent.rc

Immediately, run the migration script on the configuration to convert it to the new syntax:

	export RT_HOME="${RT_HOME:-$HOME/rtorrent1}"
	bash ~/lib/pyroscope/src/scripts/migrate_rtorrent_rc.sh $RT_HOME/rtorrent.rc

### PyroScope Configuration

To create your own configuration, the best way is to start from the default files that are part of your PyroScope installation. Let's create them for our rTorrent instance:

	export RT_HOME="${RT_HOME:-$HOME/rtorrent1}"
	mkdir -p $RT_HOME/special/.pyroscope
	pyroadmin --create-config --config-dir $RT_HOME/special/.pyroscope

The following places a minimal configuration in the `config.ini`, so that the defaults are taken from the installed software (i.e. from the `rtorrent.rc` that we created earlier), which makes later updates a lot easier.

	cat >~/rtorrent1/special/.pyroscope/config.ini <<EOF
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

As you can see, we are storing our PyroCore configuration within our rTorrent instance's home directory, i.e. at `~/rtorrent1/special` so as to keep it separate from any other rTorrent instance that we may spin up later on. We need to tell our rTorrent instance about this. To do so, replace all instances of `~/.pyroscope` in rtorrent.rc (now located at `~/rtorrent1`) with `~/rtorrent1/special/.pyroscope`. You could use `vim` for this.

Also change the value of `pyro.extended` to `1` so that the additional rTorrent-PS features that are dependent on PyroCore are actually activated.

### Tmux Configuration & Setup

We spruce up `tmux` a bit using a custom configuration, before we start it for the first time. This makes it more homey for long-time `screen` users:

	# Run this with your NORMAL user account!
	cp --no-clobber ~/lib/pyroscope/docs/examples/tmux.conf ~/.tmux.conf

You're now ready to start your first rTorrent instance. So start a `tmux` session to run rTorrent:

	tmux -2u new -s rtorrent -n rtorrent1 "~/rtorrent1/start; exec bash"

The `exec bash` keeps your `tmux` window open if rTorrent exits, which allows you to actually read any error messages in case rTorrent ends unexpectedly.

### Configure `ufw`

Tell `ufw` to allow traffic through the port(s) defined in our rTorrent configuration (see rtorrent.rc):

	ufw allow 49164

### Docker

Just a note to self. A Docker container that runs rTorrent-PS would be created like so:

	docker run -d -i -t --name="rtorrent1" -h="rtorrent1" -p 49164:49164 -p 49164:49164/udp -p 49165:49165 -p 49165:49165/udp -v /home/pinky/torrents:/torrents -e "LANG="en_US.UTF-8"" -e "LANGUAGE="en_US:en"" --restart=always ubuntu:trusty /bin/bash

In which case, you must remember to replace all instances of `~/rtorrent1` (`$RT_HOME`) in the tutorial with `/torrents/rtorrent1`, `/torrents` being our mounted data volume in the container.

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
