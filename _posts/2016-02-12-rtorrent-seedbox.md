---
layout: post
title: "Set Up Seedbox With rTorrent"
date: 2016-02-11T19:42:44+00:00
updated: 2016-03-07T19:19:56+00:00
order: 4
---

*This should be considered a learning excercise for [setting up rTorrent-PS with PyroCore]({% post_url 2016-02-17-rtorrent-ps-seedbox %}), which IMHO is better. Get a gist of this notes to better understand the latter.*

### Prep System For Building Packages

Make sure that your system is up-to-date:

	sudo apt-get update && sudo apt-get upgrade

Install the required development tools:

	sudo apt-get install build-essential automake autoconf libtool pkg-config intltool checkinstall git subversion vim

- `build-essential`, `automake`, etc. are the development tools considered essential for building Debian/Ubuntu packages from source.

- `checkinstall` keeps track of all files created or modified when you run `make install` or equivalent, builds a Slackware (.tgz), RPM (.rpm), or Debian (.deb) package with those files, and installs it in your system giving you the ability to easily uninstall it with your distribution's standard package management utilities.

	Use `checkInstall` instead of just running `make install`, as that will likely put files all over the filesystem, with no easy way of removing them if things go wrong. If in the future you try to install a package that contains the same file as the software you are compiling, you will receive errors and the software you compiled may stop working.

- `git` and `subversion` version control systems to get (pull) source code from remote repositories.

- `vim`, my editor of choice, for editing config. files, etc.

### Install XML-RPC From Source

XML-RPC is required for communication between rTorrent and its plugins (like ruTorrent, PyroScope, etc.).

Before you can compile and install a package from source (`xmlrpc-c` in our case), you need to make sure that all its dependencies are installed. There are at least three straightforward ways to do so:

1. Assuming a maintained or fairly recent version of the package is in Ubuntu's software repositories (in our case it is; source package [`xmlrpc-c`](http://packages.ubuntu.com/source/trusty/xmlrpc-c)), run:

		sudo apt-get build-dep xmlrpc-c

	`build-dep` causes `apt-get` to install the necessary packages in an attempt to satisfy the build dependencies for the `xmlrpc-c` package (which, by the way, is in ), but doesn't install `xmlrpc-c` itself. This is exactly what we need as we want to install `xmlrpc-c` from source.

	Run `apt-get -s build-dep xmlrpc-c` to simulate the entire process and not actually change the system. Useful if you want to know which packages would be installed. For example:

		pinky@rtorrent:~$ apt-get -s build-dep xmlrpc-c

		[...]

		The following NEW packages will be installed:
		  autotools-dev comerr-dev debhelper dh-apparmor diffstat gettext
		  intltool-debian krb5-multidev libcroco3 libcurl4-openssl-dev libgcrypt11-dev
		  libgnutls-dev libgnutlsxx27 libgpg-error-dev libgssrpc4 libidn11-dev
		  libkadm5clnt-mit9 libkadm5srv-mit9 libkdb5-7 libkrb5-dev libldap2-dev
		  libp11-kit-dev librtmp-dev libssl-dev libtasn1-6-dev libunistring0
		  pkg-config po-debconf quilt zlib1g-dev
		0 upgraded, 30 newly installed, 0 to remove and 3 not upgraded.

		[...]

2. `apt-cache showsrc` displays the debian/control file for a given package in the Ubuntu software repository. Amidst the vital information displayed about the package, there's "Build-Depends" which lists the dependencies for the given package.

		pinky@rtorrent:~$ apt-cache showsrc xmlrpc-c

		[...]

		Build-Depends: autotools-dev, debhelper (>= 9), libcurl4-openssl-dev | libcurl3-openssl-dev, quilt

		[...]

	You might as well just run:

		apt-cache showsrc xmlrpc-c | grep ^Build-Depends

	Then manually install the dependencies yourself:

		sudo apt-get install autotools-dev debhelper libcurl4-openssl-dev quilt

	I prefer this method for its simplicity and clarity.

	**NOTE:** Methods (1) and (2) would only be reliable as long as the package in the software repositories is fairly recent and belongs to the same major release (e.g. version 2.x, 3.x, etc.) as the one you are about to install from source.

3. Compiling and installing a package from source, in general, involves running commands such as these:

		cd /tmp/xmlrpc-c
		./configure
		make
		sudo make install

	The `./configure` command checks your system for the required software needed to build the program, and when it notices a dependency missing you'll receive error messages.

	If the error message looks like this:

		configure: error: The intltool scripts were not found. Please install intltool.

	It shouldn't take long for you to understand what you need to do to fix it; install `intltool`:

		sudo apt-get install intltool

	If the error message is about a missing file (e.g. `deluge/event.py`) and isn't specific about which package is missing, you'd run:

		pinky@rtorrent:~$ apt-file search deluge/event.py
		deluge-common: /usr/lib/python2.7/dist-packages/deluge/event.py

	Clearly the missing file is from `deluge-common` package, so install it:

		sudo apt-get install deluge-common

	(That was just an example.)

	This way, fix all the errors that the `./configure` command throws at you, and proceed with compiling and installing the software from source.

Now that we have all the knowledge we need, let's proceed with installing the dependencies for `xmlrpc-c`:

	sudo apt-get install autotools-dev debhelper libcurl4-openssl-dev quilt

Download XML-RPC source from the maintainer's repository into a temporary directory and start the build process:

	svn co https://svn.code.sf.net/p/xmlrpc-c/code/stable ~/tmp/xmlrpc-c
	cd ~/tmp/xmlrpc-c
	./configure
	make
	sudo checkinstall

- For starters, when you run `sudo checkinstall`, you may be presented with some questions. Here's how to deal with them.

	- When asked *Should I create a default set of package docs?  [y]:* type `y` and hit the `ENTER` key.

	- When asked *Please write a description for the package.* simply hit the `ENTER` key (to leave it blank).

	- `dpkg`, which is used by `checkinstall` to install the .deb package that's built during the process, doesn't like it when version number of the package does not start with digit. At some point you'll be greeted with this screen:

			*****************************************
			**** Debian package creation selected ***
			*****************************************

			This package will be built according to these values:

			0 -  Maintainer: [ root@rtorrent ]
			1 -  Summary: [ Package created with checkinstall 1.6.2 ]
			2 -  Name:    [ xmlrpc ]
			3 -  Version: [ c ]
			4 -  Release: [ 1 ]
			5 -  License: [ GPL ]
			6 -  Group:   [ checkinstall ]
			7 -  Architecture: [ amd64 ]
			8 -  Source location: [ xmlrpc-c ]
			9 -  Alternate source location: [  ]
			10 - Requires: [  ]
			11 - Provides: [ xmlrpc ]
			12 - Conflicts: [  ]
			13 - Replaces: [  ]

		So when you're asked *Enter a number to change any of them or press ENTER to continue:* type `3` and hit the `ENTER` key to change the *Version*, and when asked *Enter new version:* type `1.43.01` as we would like to change the value from `c` to `1.43.01` (or whatever the [current stable release](http://xmlrpc-c.sourceforge.net/downloading.php) is when your installing).

- We used `sudo checkinstall` instead of `sudo make install` for reasons mentioned at the beginning of the article.

	You may use `sudo dpkg -r xmlrpc` or `sudo apt-get remove xmlrpc` to uninstall the package but keep the configuration files, and `sudo dpkg -P xmlrpc` or `sudo apt-get purge xmlrpc` to uninstall the package along with the config. files.

- Always read the README (and in our case, also `./doc/INSTALL`) included in the source for any important information that you might otherwise miss.

### Install LibTorrent From Source

LibTorrent is the library on which the rTorrent client runs.

Get Ubuntu's current build dependencies for LibTorrent:

	pinky@rtorrent:~$ apt-cache showsrc libtorrent-dev | grep ^Build-Depends
	Build-Depends: debhelper (>= 9), dh-autoreconf, libcppunit-dev, libcurl4-openssl-dev, libsigc++-2.0-dev, libssl-dev

From that we can deduce which packages need to be installed:

	sudo apt-get install debhelper dh-autoreconf libcppunit-dev libcurl4-openssl-dev libsigc++-2.0-dev libssl-dev

Then get the source code for the latest stable release of LibTorrent from the [official website](http://rakshasa.github.io/rtorrent/), and proceed with the installation:

	cd ~/tmp
	curl http://rtorrent.net/downloads/libtorrent-0.13.6.tar.gz | tar xz
	cd libtorrent-0.13.6
	./autogen.sh
	./configure
	make
	sudo checkinstall

### Install rTorrent

Get Ubuntu's current build dependencies for rTorrent:

	pinky@rtorrent:~$ apt-cache showsrc rtorrent | grep ^Build-Depends
	Build-Depends: bc, debhelper (>= 9), dh-autoreconf, libcppunit-dev, libcurl4-openssl-dev, libncurses5-dev, libncursesw5-dev, libsigc++-2.0-dev, libtorrent-dev (>= 0.13.2~), libxmlrpc-core-c3-dev

From that we can deduce that we need to install these packages:

	bc debhelper dh-autoreconf libcppunit-dev libcurl4-openssl-dev libncurses5-dev libncursesw5-dev libsigc++-2.0-dev libtorrent-dev libxmlrpc-core-c3-dev

Considering that we've already installed XML-RPC and LibTorrent from source and don't want the old versions from Ubuntu's software repos, we won't be installing the `libtorrent-dev` and `libxmlrpc-core-c3-dev` packages, which gives:

	sudo apt-get install bc debhelper dh-autoreconf libcppunit-dev libcurl4-openssl-dev libncurses5-dev libncursesw5-dev libsigc++-2.0-dev

Then get the source code for the latest stable release of rTorrent from the [official website](http://rakshasa.github.io/rtorrent/), and proceed with the installation:

	cd /tmp
	curl http://rtorrent.net/downloads/rtorrent-0.9.6.tar.gz | tar xz
	cd rtorrent-0.9.6
	./autogen.sh
	./configure --with-xmlrpc-c
	make
	sudo checkinstall
	sudo ldconfig

Make the necessary rTorrent directories:

	mkdir -p ~/downloads/{session,watch}

(The `-p` flag makes nested directories as necessary i.e., it will create the directory `downloads` and nest directories `session` and `watch` within it. You may choose any location of your choice.

### Configure rTorrent

Before running rTorrent, find the example configuration file `rtorrent.rc` and copy it to `~/.rtorrent.rc`. The example configuration file is located at `/usr/share/doc/rtorrent/doc/rtorrent.rc` or in `doc/rtorrent.rc` from the source files.

	cp /usr/share/doc/rtorrent/doc/rtorrent.rc ~/.rtorrent.rc

	# Or from the source files #
	cp ~/tmp/rtorrent-0.9.6/doc/rtorrent.rc ~/.rtorrent.rc

Now modify `.rtorrent.rc` to suit your needs. [My notes on configuring rTorrent]({% post_url 2016-02-12-rtorrent-rc-config %}) might be of some help here.

Check if rTorrent starts without issues:

	rtorrent

If you see rTorrent's text UI, you are good to go. Shutdown rTorrent by pressing `Ctrl + q`.

Let's start rTorrent in a screen session:

	screen -S rtorrent -fa -d -m rtorrent

- `-S rtorrent` to set screen session name `rtorrent`.

- `-fa` to set flow-control to automatic switching mode.

- `-d -m` to start screen in detached mode; exit if session terminates.

rTorrent is now running in the background.

**Lastly...**

You might want to delete all the source code you'd downloaded for compiling packages:

	sudo rm -rf ~/tmp
	mkdir ~/tmp

And configure `ufw` to allow traffic through the port(s) defined in our rTorrent configuration (see rtorrent.rc). For example:

	sudo ufw allow 54321

### Sources

- GitHub Repository: [LibTorrent](https://github.com/rakshasa/libtorrent)

- GitHub Repository: [rTorrent](https://github.com/rakshasa/rtorrent)

	- [rTorrent Wiki](https://github.com/rakshasa/rtorrent/wiki)

	- [Old rTorrent man page](http://web.archive.org/web/20150511123211/http://libtorrent.rakshasa.no/rtorrent/rtorrent.1.html), [2](http://linux.die.net/man/1/rtorrent)

- Main guide: Jes.sc's [rTorrent + ruTorrent Configuration Guide](https://jes.sc/kb/rTorrent+ruTorrent-Seedbox-Guide.php)

Compiling and installing package from source:

- Ubuntu Wiki: [Compiling things on Ubuntu the Easy Way](https://help.ubuntu.com/community/CompilingEasyHowTo)

- Debian Wiki: [BuildingTutorial](https://wiki.debian.org/BuildingTutorial)

- Transmission Wiki: [Building Transmission: Install required development tools](https://trac.transmissionbt.com/wiki/Building#Prerequisites)

- Linux Mint Community: [How To Create A .deb Package From Source & Install It Automatically](http://community.linuxmint.com/tutorial/view/162)

- [How To Compile and Install from Source on Ubuntu](http://www.howtogeek.com/105413/how-to-compile-and-install-from-source-on-ubuntu/)

- [How do I find the build dependencies of a package?](http://askubuntu.com/q/21379)

- [How do I find the dependencies when building software from source?](http://askubuntu.com/q/172367)

- Debian Wiki: [CheckInstall](https://wiki.debian.org/CheckInstall)

- Ubuntu Wiki: [CheckInstall](https://help.ubuntu.com/community/CheckInstall)

- [How to use checkinstall instead of make install](http://codeyarns.com/2015/03/11/how-to-use-checkinstall-instead-of-make-install/)

- [Find out if package is installed in Linux](http://www.cyberciti.biz/faq/find-out-if-package-is-installed-in-linux/)

rTorrent configuration & usage:

- ArchWiki: [rTorrent](https://wiki.archlinux.org/index.php/RTorrent)

- Gentoo Wiki: [rTorrent](https://wiki.gentoo.org/wiki/RTorrent)

- FlexGet Wiki: [Complete rTorrent example](http://flexget.com/wiki/Cookbook/rTorrent)

- [Tutorial: Using rtorrent on Linux like a pro](https://harbhag.wordpress.com/2010/06/30/tutorial-using-rtorrent-on-linux-like-a-pro/)

- [Running multiple instances of rtorrent](https://kernelwho.wordpress.com/2011/11/15/running-multiple-instances-of-rtorrent/)

	- [Source another rtorrent.rc file](https://github.com/rakshasa/rtorrent/issues/387#issuecomment-178763051)

- [Reload rtorrent configuration without restart](https://scriptthe.net/2014/05/09/reload-rtorrent-configuration-config-file-rtorrent-rc/)

Misc.

- [Whatbox Wiki](https://whatbox.ca/wiki)

- GitHub Repository: [whatbox/xseed](https://github.com/whatbox/xseed)

- [Feral Hosting FAQ](https://www.feralhosting.com/faq/)

<!--

TODO

rTorrent performance tuning
ruTorrent - Pause Web UI i.e. manual refresh; should be able to handle 1000s of torrents

-->
