+++
title = "Network Wide Ad Blocking With Pi-hole"
tags = ["ads", "tracking", "privacy", "raspberry-pi", "open-source"]
date = 2024-03-02
+++

Today, many of us rely on browser extensions to block annoying ads and tracking while browsing on our computers. They work well for desktop browsers, but what about our mobiles, tablets, smart TVs, and game consoles? They're left vulnerable to ads because these extensions don't work on them. We've come to accept ads as a necessary evil, but it doesn't have to be that way.

With a simple setup using a **Raspberry Pi** and a tool called **Pi-hole**, you can block ads and tracking across all devices on your home network. It's not limited to a Raspberry Pi; you can run it on any machine using Docker, but the Pi is preferred for its low power consumption.

My frustration with ads reached its peak while watching a Dota 2 tournament on Twitch. Every time I switched channels, the same unskippable ad played, and I'd had enough. I remembered hearing about Pi-hole and decided to give it a try with my idle **Raspberry Pi 400**.

Installing Pi-hole was straightforward using their one-step automated install guide [here](https://docs.pi-hole.net/main/basic-install/).

Pi-hole's capabilities can be extended by adding more domain blocklists from **Firebog**, a site that provides lists for blocking ads, tracking, crypto mining, and more [here](https://firebog.net/), which I did.

After installation, I configured Pi-hole as the DNS server for all devices on my home network through my router's admin page. I also reserved a static IP address for the Raspberry Pi to avoid network interruptions.

The result? No more ads on Twitch, and when I switched from mobile data to WiFi, Pi-hole blocked ads on websites like [CNN](https://cnn.com).

Pi-hole works by checking each DNS query against its blocklist, allowing or blocking queries accordingly. It's effective because many sites use separate domains for ads and tracking. However, ads served from the same domains as regular content, like on YouTube, can't be blocked by Pi-hole.

![{Pi-hole Dashboard}](https://github.com/roopeshvs/roopeshvs.github.io/blob/main/static/images/pi-hole-dashboard.png?raw=true)

Notice that over one out of ten DNS queries are getting blocked by Pi-hole.

Now, what about blocking ads and trackers when you're away from home? You can set up a VPN server on the same Raspberry Pi to enjoy ad-free browsing anywhere. I tried setting up **OpenVPN**, but my ISP's [Carrier-Grade NAT (CGNAT)](https://en.wikipedia.org/wiki/Carrier-grade_NAT) blocked port forwarding. After some research, I found **Tailscale**, a beautiful and easy solution for connecting to Pi-hole remotely, even behind different NATs. You can read in detail about their NAT Traversal solution [here](https://tailscale.com/blog/how-nat-traversal-works).

**Tailscale** allows devices to connect to Pi-hole from anywhere, and setting it up took just a few minutes with their [guide](https://tailscale.com/kb/1114/pi-hole). With my mobile device connected to Tailscale, I was now able to view [CNN's site](https://cnn.com) without ads even when I was using my mobile data.

No more ads or tracking, anywhere! 🎉
