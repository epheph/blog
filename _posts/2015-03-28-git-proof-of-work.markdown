---
layout: post
title:  "Add proof-of-work to your git commits"
date:   2015-03-28 15:25:52
categories: cli
---

Disclaimer: This idea is probably useless, but possibly interesting.

Proof-of-work on the blockchain
===============================

One of the key principles behind the blockchain (i.e. bitcoin) is "proof-of-work". In order to ensure a distribution of authority within the blockchain, one must "prove" that you have worked hard to solve a specific problem, and the chance that you (or your pool) solve this problem consecutively is extremely unlikely. (As a pool approaches 50% of total combined hash power this becomes more likely and creates a problem for overall bitcoin security)

The proof of work that bitcoin uses is hash-based; once a (10-minute) block of transactions is finished, miners race to find a special "nonce", an arbitrary value that, when hashed with other values, including a hash of the transactions that comprise a block, creates a new hash that is less than a specific target when compared numerically. These numbers start with a series of 0's, and look like [00000000000000000528370533e87b66d17c8d1fed6cfc068fbb4b2a2ee3fbcd](http://blockexplorer.com/block/00000000000000001e8d6829a8a21adc5d38d0a473b144b6765798e61f98bd1d)


Proof-of-work in git
====================
There exists significant similarities between git and the blockchain. Most structurally, the main units of data (blockchain's "block" compared to git's "commit") are both identiified by hash, and the parent/child relationship between these is a part of the hashed value (blockhain's "hashPrevBlock" compared to git's "parent"). Through a similar system of nonce & processing power, we can "prove" that we put a lot of computing power into our git commits.

What i've created with a simple bash script is a git plugin that will amend your previous commit's message with a specific value that generates a hash less than a specified target. I'll attach the script in its entireity and show a few sample invocations

{% highlight bash %}

{% endhighlight %}

This script will loop through your current git repository appending a counter to the current commit message and seeing if we hit our target. Once we hit our target, we exit.

Here's an example: 

{% highlight bash %}

$ git clone https://github.com/epheph/epheph.github.io.git
Cloning into 'epheph.github.io'...
remote: Counting objects: 142, done.
remote: Total 142 (delta 0), reused 0 (delta 0), pack-reused 141
Receiving objects: 100% (142/142), 42.77 KiB | 0 bytes/s, done.
Resolving deltas: 100% (58/58), done.
Checking connectivity... done.

$ cd epheph.github.io/

$ git log -1
commit a406d84578ffab5cb647e1a90cf67dba7df15320
Author: Scott Bigelow <scott.bigelow@upsight.com>
Date:   Sun Mar 15 08:29:22 2015 -0700

    Adding disqus comments to all posts. Consider a page variable to selectively enable in the future

$ time git proof
Found 0004e1b3d9440b39e1fcc6a6315d5c69e3b51624 using nonce 4293

real	1m2.509s
user	0m21.370s
sys	0m32.216s

$ git log -1
commit 0004e1b3d9440b39e1fcc6a6315d5c69e3b51624
Author: Scott Bigelow <scott.bigelow@upsight.com>
Date:   Sun Mar 15 08:29:22 2015 -0700

    Adding disqus comments to all posts. Consider a page variable to selectively enable in the future
    4292

{% endhighlight %}

It's worth noting this is extremely hard on your local git clone, as each amended commit is kept around in the reflog 

{% highlight bash %}
$ git reflog | wc -l
    4295

$ git reflog expire --expire-unreachable=now --all

$ git reflog | wc -l
       2

{% endhighlight %}
