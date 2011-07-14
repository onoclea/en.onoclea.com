---
layout: bits
title: Why I adore HP ProCurve switches
---
You may say that the subject is over-exaggerated. I do, however, claim that thanks to this fine piece of "automation" I have saved myself loads of time. Instead of logging onto a one server at a time, I could easily create automated tasks without a overpriced and GUI only tools.

{% highlight bash %}
[user@host ~]$ cat download/update
cd os
put H_08_106.swi secondary
[user@host ~]$ eval `ssh-agent`
Agent pid 25066
[user@host ~]$ ssh-add id_dsa
Enter passphrase for id_dsa:
Identity added: id_dsa (id_dsa)
[user@host ~]$ cd download; sftp -b update host
sftp> cd os
sftp> put H_08_106.swi secondary
Uploading H_08_106.swi to /os/secondary
Connection to host closed by remote host.
[user@host download]$
{% endhighlight %}

And that's it - the switch is upgraded!
