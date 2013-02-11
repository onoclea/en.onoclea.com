---
layout: bits
title: Forwarding (routing) DLNA/UPnP broadcasts on Linux (virtual setup using KVM).
---
One of the recent updates in my home infrastructure was to convert from VMware to Linux KVM in terms of the virtualisation platform. Everything went smoothly, virtual machines have been successfuly converted. There was one tiny problem however - my UPnP media server, running PS3 Media Server, stopped to be visible from both the Playstation3 and the AVR (H/K AVR-270). It took me a day of troubleshooting but finally everything is working once again.

## Symptoms

From the "business" side effects were quite obvious. Neither the PS3 nor the H/K AVR were able to see the media server. Technically, after a bit of digging, it turned out that the multicast discovery sent from the virtual machine hasn't been forwarded by the virtual hypervisor. The VM host saw the multicast communication sent to 239.255.255.250, port 1900 but it didn't put it through.

## Fix

The fix turned out to be quite simple yet it took a lot of time to figure out - [https://github.com/troglobit/smcroute](smcroute) or "simple multicast router" came to the rescue.

### VM hypervisor

The VM hypervisor configuration (br0 is the bridge that all virtual machines are connected to):

{% highlight bash %}
[root@blackbox ~]# smcroute -j br0 239.255.255.250
[root@blackbox ~]# smcroute -a br0 0.0.0.0 239.255.255.250 br0
{% endhighlight %}

### media server

The media server configuration:

{% highlight bash %}
[root@media ~]# smcroute -j eth0 239.255.255.250
[root@media ~]# smcroute -a eth0 0.0.0.0 239.255.255.250 eth0
{% endhighlight %}

And that's it - now both the game console and the AVR could see the UPnP server.
