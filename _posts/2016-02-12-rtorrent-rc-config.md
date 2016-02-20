---
layout: post
title: "Basic rTorrent Configuration Guide"
date: 2016-02-11T19:45:39+00:00
updated: 2016-02-11T19:45:39+00:00
---

**NOTE:** The following settings are configured for a server with: 8 GB RAM, 3.40 GHz dual-core processor (4 threads), 2 TB storage and 200 Mbps - 800 Mbps bandwidth (let's just say ~ 300 Mbps), and considering that there are mostly good peers (with excellent internet speeds) on private trackers.

### min_peers, max_peers, min_peers_seed & max_peers_seed

*The `min_peers` & `max_peers` configuration settings apply to downloads (leeching) only. OTOH, the `min_peers_seed` & `max_peers_seed` settings apply to uploads (seeding) only.*

The `min_peers` setting makes rTorrent use more aggressive settings until it finds that number of peers, and it won't stop hammering the tracker until it does. It's all about acquiring that specific number of peers.

The `max_peers` setting tells rTorrent that it has sufficient number of peers and it needn't look for more.

In the range between `min_peers` and `max_peers`, rTorrent would be on the look out for peers, but not as aggressively as it would be until it reaches the `min_peers` mark. During this time I suspect some algorithms come into play wherein, for example, banned, [snubbed](https://en.wikipedia.org/wiki/Glossary_of_BitTorrent_terms#Snubbing), and/or badly performing (slow speeds) peers could be dropped and new ones added, probably depending upon your upload/download speeds, etc. etc.

Here's an (untested) idea:

- **If you just uploaded a torrent to a tracker**, the initial rush of downloads is from the auto-downloading seedboxes and lasts for a very short period of time. For this reason, we need rTorrent to be all active and agressive about finding peers to upload to during this time. Once it passes, everything goes back to normal; there's only a download every now and then, if there is. So, **the rest of the time**, where there will be a downloader only occasionally, we only need rTorrent to look for peers normally. rTorrent will do that anyway, no matter what your config. is, i.e. even if you set `min_peers` to `40` rTorrent can't behave aggressively when the tracker is reporting `0` peers (downloaders).

- **If you are downloading a torrent that's just been added**, you are competing with other downloaders to complete your download asap and start seeding. For this reason, we need rTorrent to be all active and agressive about finding peers to quickly download from during this time. **Once you've downloaded the torrent**, rTorrent will start seeding, so config. doesn't matter.

What I am essentially saying is that your rTorrent configuration should be about using your internet speeds to their full potential *during the initial rush*. That's the formula.

Considering:

- This is only about private trackers where there are mostly good peers with excellent internet speeds.

- s1 = My server's internet speed = ~ 300 Mbps (Remember! Yours will be different.)

- s2 = The lowest upload speed that I can expect the good peers to have on an avg. = ~ 8 Mbps (1 MB/s)

	Lower this to 4 Mbps (500 KB/s) if you are paranoid.

rTorrent config.:

- My formula for `min_peers` = s1 / s2 = 300 / 8 = ~ 40
- My formula for `max_peers` = min_peers x 2 = 80

Therefore:

	min_peers = 40
	max_peers = 80

The default `min_peers_seed` and `max_peers_seed` values are not as aggressive. Set them to `-1` to use the same settings as `min_peers` and `max_peers` respectively.

	min_peers_seed = -1
	max_peers_seed = -1

*Start with these settings, see how rTorrent performs (if it's performing to its potential), and then increase or decrease as you see fit. (For e.g. if rTorrent seems to be performing to its full potential, your internet speeds are probably faster, so try increasing the values further.) The same applies to other config. settings in `.rtorrent.rc`.*

### max_uploads

Let's look at the `min/max_peers*` setting the other way. While `min/max_peers*` controls the aggressiveness of scraping to find peers, `max_uploads` controls how many peer connections are actually allowed to be made for uploading to or downloading from.

This setting should be set to max out your internet connection. And as you've set `min_peers` to an optimistic value sufficient to max out your connection, I feel that `max_uploads = min_peers` is a safe setting to start with, so:

	max_uploads = 40

### download_rate & upload_rate

My internet speed isn't guaranteed to stick at a set rate. From what I've been told it'd linger between 200 Mbps to 800 Mbps, and I have no doubt that it could go lower than 200 Mbps during bad times. So, I'll leave the defaults in place (i.e. `0` for unlimited).

	download_rate = 0
	upload_rate = 0

### Misc.

The rest of the configuration is pretty straighforward, so I won't go into detailing each other config. setting. Instead here's what my `~/.rtorrent.rc` looks like (and it's based on [this](https://github.com/pyroscope/pyrocore/blob/master/docs/examples/rtorrent.rc)):

	# general
	upload_rate = 0
	download_rate = 0
	max_uploads = 40
	tracker_numwant = 80
	max_open_http = 99
	max_open_files = 600
	max_open_sockets = 300
	encryption = allow_incoming,try_outgoing,enable_retry
	umask = 0027
	key_layout = qwerty
	check_hash = no

	# range for listening port
	port_range = 49164-49164
	port_random = no

	# tracker-less torrent support
	dht = disable
	peer_exchange = no
	use_udp_trackers = no

	# peer settings
	min_peers = 40
	max_peers = 80
	min_peers_seed = -1
	max_peers_seed = -1

	# resource usage
	max_memory_usage = 8192M
	xmlrpc_size_limit = 4M

Notes:

- Leave the scheduled job named `untied_directory` commented out. It's unnecessary and can be confusing.

		# Stop torrents which have been deleted from the watch directory(-ies)
		# schedule = untied_directory,5,5,stop_untied=

- The `port_range` option sets which port(s) to use for listening. It is recommended to use a port between 49152-65535. Although, rTorrent allows a range of ports, a single port is recommended. Don't forget to make sure port forwarding is enabled for the proper port(s).

- Disable DHT and peer exchange, which is a requirement of most private trackers.

- Read the comments in your default rtorrent.rc file. Most of the settings aren't that difficult to understand.
