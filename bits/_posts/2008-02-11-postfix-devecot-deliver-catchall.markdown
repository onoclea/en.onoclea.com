---
layout: bits
title: Postifx + Dovecot Deliver + Catchall Addresses - Impossible is nothing!
---
For over a year now I've been struggling with something I thought would never get solved. Though the setup I have (postfix + dovecot) isn't really complicated, it does have some nifty features. Among many of them is the fact that I use catchall addresses in most of my domains. On the top of that I wanted to use Sieve to filter some of my mail on the server side (especially those "you always read them" daily log reports).

But there was a problem. I've even made a post on the Dovecot mailinglist:

[http://www.dovecot.org/list/dovecot/2006-March/012064.html](http://www.dovecot.org/list/dovecot/2006-March/012064.html)

In general - dovecot cannot mimic postfix' way of handling catch-all addresses - there was nothing even similar to the "table search order" found in my favorite MTA. So I made a rather ugly patch... and it worked. But it was such a dirty hack that merging it with the dovecot's trunk wasn't really an option. So whenever a new dovecot release was published I had to manually prepare my "own" version. So I ended up having two separate setups depending on the client's demands - either a vanilla dovecot without my patch (that was updated on a regular basis) or a hacked version that... well... should be updated :)

Couple of days ago I was contacted by Maciej Paczesny asking if there was any progress with the problem I reported back in 2006.

After exchanging some emails and thoughts I think I've finally found a neat way of getting "things done right".

[http://www.postfix.org/canonical.5.html](http://www.postfix.org/canonical.5.html)

All you have to do in fact is to rewrite the recipient address using postfix' recipient_canonical_maps table. That's all. Thanks to this little trick, dovecot-deliver receives an address that is final and unique so it doesn't have any troubles locating a proper message store directory.

Technically speaking, you must define recipient_canonical_maps in such a way that it would return final and unique address for any of your users (here we can benefit from postfix' table search order, so catch-all addresses do work!). Then postfix will rewrite the envelope "To:" header, pass it to dovecot and voila - problem solved.

Thanks Maciej! :)
