---
layout: bits
title: csync2 - When all you need is a multi-way synchronization tool
---

Scaling a system horizontally usually requires sharing some data between your worker nodes. If your architecture justifies a database server (be it either SQL or NoSQL) then that's usually a quite trivial task (as long as you assure that your database layer scales, which is another kind of a problem). However, if your requirements are different - like e.g. POSIX compliancy - then you need to look for someting else.

(Un)fortunately, setting up [DRBD](http://www.drbd.org/) and [OCFS2](http://oss.oracle.com/projects/ocfs2/) isn't that easy. It's doable, of course, and works great if done right. Unfortunately, being a complex system makes it susceptible to errors that do occur (by Murphy's law and by statistics) more often than in some less complicated setups.

Luckily, if you need a multi-way synchronization tool and you can deal with some limitations, then you should definitely try [csync2](http://oss.linbit.com/csync2/) - a tool developed by the authors of [DRBD](http://www.drbd.org), that can sometimes turn quite handy.

## What is csync2?

Authors describe [csync2](http://oss.linbit.com/csync2/) as a:

*Csync2 is a cluster synchronization tool. It can be used to keep files on multiple hosts in a cluster in sync. Csync2 can handle complex setups with much more than just 2 hosts, handle file deletions and can detect conflicts.*

So how can it be helpful? I'll describe one of the use cases I came across recently.

## Serving web assets (static files) with multiple http server nodes (nginx)

The system I worked with started as a simple and durable setup. The application layer consisted of two application nodes, working in a hot-standby mode. The initial requirements set to have HA (high availability), not a LB (load balance). So whenever the master node failed (or was set to fail), the slave took over (with some virtual IP magic under the hood). This means, that only one node was active at a time. This was fine until a single machine couldn't handle the load. The system did have a database server, but both application nodes had to have an access to a shared storage - a place where some static files were kept.

## Requirements

First of all I ruled out solutions that required extra nodes (a business requirement). So a pair of new NFS servers was not an option. Moreover, the design had to scale for more than two nodes. This made rsync and similar "one way sync" tools inaplicable as well.

## DRBD - too powerful

After some research two approaches surfaced.

One was to use a full-blown replicated filesystem (OCFS2) with a shared block storage beneath (DRBD). It was a stripped version of the NFS server idea, but turned to be cumbersome to scale and, at least initially (*"The First Rule of Program Optimization: Don't do it. The Second Rule of Program Optimization (for experts only!): Don't do it yet."* - quote by Michael A. Jackson) it was like using cannon to kill a fly. 

## csync2

Then I've tried to setup csync2 - a tool I came across while doing some research about DRBD some time ago. After about 30 minutes the system was up and running. The setup is very simple - all you need is a single configuration file:

{% highlight bash %}
group app {
 host app0.example.com app1.example.com app2.example.com;

 key /etc/csync2.key-app;

 include /vol/0/app/shared;
 include /vol/0/app/spool;

 backup-directory /vol/csync2/backup;
 backup-generations 3;

 auto none;
}
{% endhighlight %}

And that's it - this helps you to have `/vol/0/app/shared` and `/vol/0/app/spool` synchronized between all the nodes. If a files is added, modified or deleted then the change is automatically propagated to other nodes.

## Batch mode of operation

There's one catch, however. Csync2 works in batch mode, so you don't get an on-line synchronization. Running csync2 from cron every minute (or in a loop) doesn't solve that either. Moreover, if you deal with hundreds of thousands of files, then it takes some time to scan all files for potential differences.

However, if you can modify you application to trigger csync2, this can easily be solved by the following wrapper script.

{% highlight bash %}
#!/bin/bash

/usr/sbin/csync2 -v -m $@ 2>&1 | /usr/bin/logger -t csync2
/usr/sbin/csync2 -v -u 2>&1 | /usr/bin/logger -t csync2
{% endhighlight %}

First it sets its argument as "dirty" (so pending synchronization). Then it updates all elements with the "dirty" flag set. It's fast as csync2 doesn't compare all the files and checks only the one that have been pointed out.

The logger part gives you an audit log of all the things that happened - just to check if everything is OK.

## Problems

There was one problem I had to deal with. After generating the certificates the csync2 nodes couldn't connect. It turned to be a known bug:

* [http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=501289](http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=501289)
* [http://archives.free.net.ph/message/20080811.081952.de6b9f3e.ja.html](http://archives.free.net.ph/message/20080811.081952.de6b9f3e.ja.html)

Copying certificate and key files solved the problem.
