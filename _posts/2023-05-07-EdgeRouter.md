---
layout: post
title: Sleeping schedule for EdgeOS
author: Călin
keywords:
    - EdgeRouter
    - EdgeOS
    - firewall rules
    - block internet for children
    - OpenWRT
---

I spent today figuring out how to setup a schedule to block internet access at night. Victor started going to sleep later and later, to the point where it is no longer acceptable to leave his devices free to access internet at night.

So why a blog about this? Because I spent an inordinate amount of time hardware resetting the router to get it working and I hope that someone else who may want to do the same thing avoids the pain. And because I used [OpenWRT](https://openwrt.org/docs/guide-user/firewall/firewall_configuration) in the past and it was this easy:

1. add a rule in `/etc/config/firewall` that looked something like this:
```
config rule
        option enabled '1'
        option src '*'
        option dest 'wan'
        option proto '0'
        option target 'REJECT'
        option name 'device_name'
        option src_mac 'xx:xx:xx:xx:xx:xx'
        option start_time '22:00:00'
        option stop_time '07:30:00'
```

2. `service firewall restart`

I loved the flexibility of the OpenWRT system, but the hardware I ran it on, an TPLink Archer A7, was cheap, and gave up pretty fast.

After some research, I upgraded the home networking to a more professional setup:

- [Ubiquity U6 Long-Range](https://store.ui.com/products/u6-lr-us) access point.
- [Ubiquity EdgeRouter X-SFP](https://store.ui.com/collections/operator-edgemax-routers/products/edgerouter-x-sfp)

And with this, we're talking endless options for configuration. Ubiquity has a cloud console and network management software, but interestingly enough, their network management and WiFi management are different package. Beside the fact that for the cloud console they charge monthly subscriptions, it makes absolutely no sense to me why would you break the security of your network by having the management in the cloud! Especially when the devices themselves sport a command line interface. The router runs EdgeOS, a Debian 9 clone. And the access point is based on OpenWRT.

I did not play much with the access point, beside setting up a non-default root account and the WiFi network(s). All the configuration is on the router, which runs the DHCP server, DNS, and firewall (among other things).

A bit of digging through the web documentation got me to
[EdgeRouter - How to Create a Guest\LAN Firewall Rule](https://help.ui.com/hc/en-us/articles/218889067), which covers pretty much what I wanted.

First, I setup the DHCP server to statically map the addresses, and then an firewall address group for all the devices of interest:

```
set firewall group address-group nighttime description "devices"
set firewall group address-group address xxx.xxx.xxx.xxx description "dev1"
set firewall group address-group address xxx.xxx.xxx.xxx description "dev2"
```

Then added the rules, first to block the traffic at night, and second to allow the devices to get to DHCP and DNS so that they do have a connection (that was what someone pointed out as nice to have).

```
set firewall name nighttime_in rule 1 action drop
set firewall name nighttime_in rule 1 description "pause at night time"
set firewall name nighttime_in rule 1 protocol all
set firewall name nighttime_in rule 1 source group address-group nighttime
set firewall name nighttime_in rule 1 time starttime 22:00:00
set firewall name nighttime_in rule 1 time stoptime 07:30:00
set firewall name nighttime_in rule 1 log disable

set firewall name nighttime_local rule 1 action accept
set firewall name nighttime_local rule 1 description "allow dhcp"
set firewall name nighttime_local rule 1 protocol tcp_udp
set firewall name nighttime_local rule 1 destination port 67-68
set firewall name nighttime_local rule 1 source group address-group nighttime
set firewall name nighttime_local rule 1 log disable

set firewall name nighttime_local rule 2 action accept
set firewall name nighttime_local rule 2 description "allow dns"
set firewall name nighttime_local rule 2 protocol tcp_udp
set firewall name nighttime_local rule 2 destination port 53
set firewall name nighttime_local rule 2 source group address-group nighttime
set firewall name nighttime_local rule 2 log disable
```

And finally, activate the rules on the network interface where the access point is connected (it so happens that it was the same port as in the documentation example!)
```
set interfaces ethernet eth2 firewall in name nighttime_in
set interfaces ethernet eth2 firewall local name nighttime_local
```

And then ... nothing! Traffic continued to flow unrestricted! Ok, let's reboot the router so that the rules take effect. Unfortunately, once rebooted, I couldn't connect to the router on anything else: wired, wireless, different ports (including `eth0`), everything was reporting: `the host is not available`.
Switching to debug mode, I reset the router to its factory configuration (hardware reset, since I could not get to it otherwise) and started incrementally:

- first, disable IPv6 -- kind of useless to block IPv4 traffic if you allow IPv6 traffic!
- then start modifying the rules, changing the rules that they were all accepting traffic;
- then one rule at time, `nighttime_in` and then `nighttime_local`.

Nothing worked, every time I rebooted the router it became a brick. Fortunately, you can backup and restore the configuration on the router, so de-bricking _only_ involved pressing the reset button, switching a couple of cables, loading the backup configuration, and rebooting again.

Three hours later, I was almost ready to give up and move to look at how to block the traffic on the access point, when I had the idea of using the switch for the interface. So I changed the last two lines to:

<div class="language-plaintext highlighter-rouge"><pre class="highlight"><code>set interfaces ethernet <span class="red">switch0</span> firewall in name nighttime_in
set interfaces ethernet <span class="red">switch0</span> firewall local name nighttime_local
</code></pre></div>

and that was it!

Anca says it was worth it. I tend to disagree. I'd like to know why using `eth2` in a firewall rule, even when the port is not connected disables all access to the device, but I already spent enough time, and in fact using `switch0` makes more sense to me, so I will leave it here. Except for writing this quick blog so that others won't have to spend the hours.





