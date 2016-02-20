---
layout: post
title: "Set Up qBittorrent Seedbox On Server As Daemon With Web UI"
date:
updated:
---

**Install**

First make sure that your system is up-to-date:

	sudo apt-get update && sudo apt-get upgrade

Add qBittorrent team's PPA to get the latest stable release from:

	sudo add-apt-repository ppa:qbittorrent-team/qbittorrent-stable
	sudo apt-get update

Proceed to install the qBittorrent daemon:

	sudo apt-get install qbittorrent-nox

**User Configuration**

Create the user qBittorrent will run under:

	sudo useradd qbtuser -m

(qBittorrent's configuration files will be placed under this user's directory at `/home/qbtuser/.config/qBittorrent/`.)

**Init script**

Set up an init script `qbittorrent.service` under `/etc/systemd/system/` to maintain the `qbittorrent-nox` daemon:

	sudo nano /etc/systemd/system/qbittorrent.service

And paste the following into the file and save:

	[Unit]
	Description=qBittorrent Daemon Service
	After=network.target

	[Service]
	Type=forking
	User=qbtuser
	ExecStart=/usr/bin/qbittorrent-nox -d

	[Install]
	WantedBy=multi-user.target

**Initialize qBT configuration**

Run qbittorrent as `qbtuser` (i.e. the user qBittorrent will run under) so that it can ask us to accept the disclaimer, and generate the config file to remember this setting:

	sudo su qbtuser
	qbittorrent-nox

Accept the terms (Type `Y`, then hit `ENTER`), and you'll be presented with some useful information:

	******** Information ********
	To control qBittorrent, access the Web UI at http://localhost:8080
	The Web UI administrator user name is: admin
	The Web UI administrator password is still the default one: adminadmin
	This is a security risk, please consider changing your password from program preferences.

Now you may log into qBittorrent's web UI at `http://your-server-ip:8080` with the credentials you received earlier; for e.g. `http://192.168.1.2:8080`.

After the trial run, press `Ctrl + C` to stop the `qbittorrent-nox` instance and type `exit` to return to your normal user account (you are currently logged-in as `qbtuser`, remember?).

**Enable qBittorrent systemd service**

Start the service, verify it is running and enable it to start up on boot:

	sudo systemctl start qbittorrent
	sudo systemctl status qbittorrent
	sudo systemctl enable qbittorrent

After the above, qBittorrent daemon should be running and should now start automatically with reboots.

**TODO:** Enable Logging.

**qBT Configuration**

Now you can access the web UI again and proceed with setting up your preferences. If you need direct access to the config file for advanced configuration, it can be found at: `/home/qbtuser/.config/qBittorrent/qBittorrent.conf`

Also check `qbittorrent-nox -h` for useful command-line options. E.g. use the `--webui-port=<port>` option to set a different port for the web UI (default is 8080). If doing so, don't forget to update the command in the systemd init script.

### Uninstalling qBittorrent:

Stop qBittorrent:

	sudo systemctl stop qbittorrent

Disable qBittorrent:

	sudo systemctl disable qbittorrent

Remove the startup script:

	sudo rm /etc/systemd/qbittorrent.service

Remove the qBittorrent package:

	sudo apt-get remove --purge "qbittorrent*"

Remove log files:

	sudo rm /var/log/qbittorrent.log

Remove qbtuser home folder and config files. If you want to re-install qbittorrent or something you might want to keep this or back it up:

	sudo rm -R /home/qbtuser/.config/qBittorrent

Remove qbtuser:

	sudo userdel qbtuser

### Sources

- qBittorrent Wiki: [Setting up qBittorrent on Ubuntu server as daemon with Web interface (15.04 and newer)](https://github.com/qbittorrent/qBittorrent/wiki/Setting-up-qBittorrent-on-Ubuntu-server-as-daemon-with-Web-interface-(15.04-and-newer))

- qBittorrent Wiki: [Setting up qBittorrent on Ubuntu server as daemon with Web interface (14.04 and older)](https://github.com/qbittorrent/qBittorrent/wiki/Setting-up-qBittorrent-on-Ubuntu-server-as-daemon-with-Web-interface-(14.04-and-older))

- Deluge Wiki: [Systemd User Guide](http://dev.deluge-torrent.org/wiki/UserGuide/Service/systemd#DelugeDaemondelugedService)

- qBittorrent FAQ: [Where does qBittorrent save its settings?](https://github.com/qbittorrent/qBittorrent/wiki/Frequently-Asked-Questions#where-does-qbittorrent-save-its-settings)

- qBittorrent FAQ: [How do I do IP Filtering (eMule and PeerGuardian compatible) in qbittorrent in GNU/Linux?](https://github.com/qbittorrent/qBittorrent/wiki/Frequently-Asked-Questions#how-do-i-do-ip-filtering-emule-and-peerguardian-compatible-in-qbittorrent-in-gnulinux)

To read:

- [qBittorrent WebUI-related FAQ](https://github.com/qbittorrent/qBittorrent/wiki#webui-related)

- [Explanation of Options in qBittorrent](https://github.com/qbittorrent/qBittorrent/wiki/Explanation-of-Options-in-qBittorrent)

<!--

TODO

btpinky - instead of qbtuser
custom port

-->