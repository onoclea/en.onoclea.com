---
layout: bits
title: How to host your site on Dreamhost (or any shared hosting site) with an external DNS server
---

Dreamhost is one of the best shared-hosting sites I've came across. It has its ups and downs, but all in all it's very useful in many situations. In my particular case I use it to host some of my family's sites and files. I could do it on my own, but I prefer to pay $10 a month and sleep safe and sound :)

## The problem

There's one problem, however. I don't use [Dreamhost](http://www.dreamhost.com/r.cgi?215063) to host my domains. So the question is - how do you reliably setup a DNS record (e.g. go.pjs.name used in the example below) to point it to a Dreamhosts' IP address, which can change? The solution is quite straightforward and easy to implement. What you need to to is to point your subdomain's (e.g. go.pjs.name) domain name servers (it's legit and nothing out of ordinary) to the [Dreamhost](http://www.dreamhost.com/r.cgi?215063) defined ones (`ns1.dreamhost.com`, `ns2.dreamhost.com`, `ns3.dreamhost.com`). This way you:

* retain full control over your domain
* delegate authority for the subdomain (the one hosted on Dreamhost) so whenever somebody at Dreamhost decides to change the IP address of the server that keeps your data, you are completely safe and don't have to change anything

BTW - this technique is universal and can be applied to any shared hosting site, or any similar situation.

## Example

See the following example as it may shed some light on the solution, since the particular implementation depends on where you keep your domain.

### My domain - `pjs.name`

I own a `pjs.name` domain and host its DNS servers externally:

{% highlight bash %}
$ dig -t ns pjs.name

;; QUESTION SECTION:
;pjs.name.                      IN      NS

;; ANSWER SECTION:
pjs.name.               86400   IN      NS      ns2.onoclea.net.
pjs.name.               86400   IN      NS      ns1.onoclea.net.
pjs.name.               86400   IN      NS      ns3.onoclea.net.
pjs.name.               86400   IN      NS      ns5.onoclea.net.
pjs.name.               86400   IN      NS      ns6.onoclea.net.
pjs.name.               86400   IN      NS      ns4.onoclea.net.
{% endhighlight %}

OK, so what about the site I wish to keep at my shared hosting site?

### My site at Dreamhost - `go.pjs.name`

I also have a site at Dreamhost - `go.pjs.name`. It serves me as a place where I can share data with my friends. I just upload some data, create a unique link  (e.g. http://go.pjs.name/d/9f166080-2fe1-4600-92bb-d42dc19c453b/) and voila. I could, of course, use a specialized service, e.g. [Dropbox](http://db.tt/2450WkK) or do some other nifty things like setting up a username and password or maybe even SSL. But still - there are situations that this level of security is all I need.

There is a problem with how you set up the DNS records. The straightforward approach would be to get the current host at Dreamhost that keeps your data, get its IP address and create an `A` records pointing to that IP address. This is not a bulletproof solution, unfortunately. If the IP addresses at Dreamhost changes, you end up with an unreachable site.

### Solution

The solution is very simple. All you need to do is to create `NS` records for the site/domain you wish to host at Dreamhost and point it to the Dreamhost's DNS servers. So, in our example, this would look like the following:

{% highlight bash %}
$ dig -t ns go.pjs.name @ns1.onoclea.net

;; QUESTION SECTION:
;go.pjs.name.                   IN      NS

;; AUTHORITY SECTION:
go.pjs.name.            86400   IN      NS      ns2.dreamhost.com.
go.pjs.name.            86400   IN      NS      ns3.dreamhost.com.
go.pjs.name.            86400   IN      NS      ns1.dreamhost.com.

{% endhighlight %}

And that's it.  Now you don't have to worry that your site becomes unreachable because somebody decides to change an IP address somewhere at Dreamhost. Notifying all the customers that could be potentially affected is almost impossible, so don't expect it. What you can do instead is to rely on the fact that the DNS data must be kept valid. And this is one way of doing that :)

## Conclusions

DNS is not limited just to `A`, `MX` and `CNAME` records... and this one tiny feature of it just made my life a bit easier :)
