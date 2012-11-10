---
layout: bits
title: How to make the HP MicroServer Remote Access Card to work via IPMI (ipmitool) on ArchLinux (or any other Linux distro in general)?
---
Couple of days back I wanted to access the HP MicroServer Remote Access Card via the IPMI (ipmitool) interface. The reason was quite simple - I locked myself out from the web interface by setting a password that included "special" (in my case the "&") characters. UI allowed me to change the password, but then refused to let me in... Getting back to the point.

After probing the respective modules (`ipmi_devintf` & `ipmi_si`) nothing worked.

{% highlight bash %}
[root@host]# ipmitool chassis status
Could not open device at /dev/ipmi0 or /dev/ipmi/0 or /dev/ipmidev/0: No such file or directory
Error sending Chassis Status command
{% endhighlight %}

It turned out to be a problem with the default internall communication port, that was not picked up correctly. The manual (see the link below) states it clearly - quote from the docs (page no. 5):

> The default system base address for the I/O mapped KCS Interface is 0xCA2 (...)

So, to make things right, one must pass the correct `ports` parameter to the `ipmi_si` module while probing:

{% highlight bash %}
[root@host]# modprobe ipmi_devinf
[root@host]# modprobe ipmi_si type=kcs ports=0xca2
{% endhighlight %}

Now the kernel recognizes the card without any problems:

{% highlight bash %}
[root@host]# dmesg
[32559.752311] IPMI System Interface driver.
[32559.752321] ipmi_si: probing via hardcoded address
[32559.752326] ipmi_si: Adding hardcoded-specified kcs state machine
[32559.752336] ipmi_si: Trying hardcoded-specified kcs state machine at i/o address 0xca2, slave address 0x0, irq 0
[32560.211697] ipmi_si ipmi_si.0: Found new BMC (man_id: 0x000001, prod_id: 0x3431, dev_id: 0x20)
[32560.211973] ipmi_si ipmi_si.0: IPMI kcs interface initialized
{% endhighlight %}

More importantly, `ipmitool` just works:

{% highlight bash %}
[root@host]# ipmitool chassis status
System Power         : on
Power Overload       : false
Power Interlock      : inactive
Main Power Fault     : false
Power Control Fault  : false
Power Restore Policy : always-off
Last Power Event     : command
Chassis Intrusion    : inactive
Front-Panel Lockout  : inactive
Drive Fault          : false
Cooling/Fan Fault    : false
{% endhighlight %}

## ArchLinux specific settings

In ArchLinux, in order to set those options permanently, you need to create two files, in order to...

### ...load the necessary modules at boot

Create a file in `/etc/modules-load.d`, e.g. `ipmi.conf`:

{% highlight bash %}
# cat /etc/modules-load.d/ipmi.conf 
ipmi_devintf
ipmi_si
{% endhighlight %}

Now, on boot, ArchLinux will load those two modules automatically.

### ...set the respective options while loading `ipmi_si` module

Create a file named `ipmi_si` in `/etc/modprobe.d`:

{% highlight bash %}
# cat /etc/modprobe.d/ipmi_si.conf 
options ipmi_si type=kcs ports=0xca2
{% endhighlight %}

This will make sure that the `ipmi_si` module receives the correct options upon loading.

## Useful links

While applying the RTFM method, the *M* part is sometimes quite handy:

* [http://h10032.www1.hp.com/ctg/Manual/c02948881.pdf](http://h10030.www1.hp.com/ctg/Manual/c02948881.pdf "HP MicroServer Remote Access Card manual")
