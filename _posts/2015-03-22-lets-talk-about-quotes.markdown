---
layout: post
title:  "Let's talk about [bash] quotes part 1"
date:   2015-03-22 15:25:52
categories: cli
---

{% highlight bash %}
{% endhighlight %}

For nearly 20 years, I have used Linux, and bash, on a daily basis. However, until very recently, I didn't fully understanding bash quoting. I understood how to make bash invoke programs with the arguments I wanted, mostly, but it was more of a rain dance than it was an application of something well-understood. And this lack of a complete understanding left me both sub-optimal on the command line and sometimes wondering exactly WHY an invocation needed to be called in a specific way. Take, for instance, the following example:

We want to awk. But we want to use a variable from the calling script. But there are spaces within the awk we want to run.

1.) You can't understand what you can't test
--------------------------------------------

First off, let's create a quick script to play around with. We can write this in both bash and python, take your pick; they should act identical (leave comment if you can find a case in which they don't!).

Let's name this `argp` in your $PATH

Python
{% highlight python %}
#!/usr/bin/env python
from sys import argv

print( "\n".join( [ "%s: ^%s$" % (num, argv[num]) for num in range(1,len(argv) ) ] ) )
{% endhighlight %}

bash
{% highlight bash %}
#!/bin/bash
i=0
for var in "$@"
do
    echo "$((i++)):^$var$"
done
{% endhighlight %}

A quick test run should give:

{% highlight bash %}
$ argp a bb ccc
1: ^a$
2: ^bb$
3: ^ccc$
{% endhighlight %}



---------------------------------------------------------------------------------------------------------------

This one is a little vague, and i would work with the candidate to make sure they understood what is being asked, but anyone interviewing at mid-level or greater position should have run into what seems like a low memory issue in on a Linux box, but isn't. The answer I'm looking for here is "cached" memory. Let's look at a standard `free` output:

{% highlight bash %}
$ free -m | head -2
             total       used       free     shared    buffers     cached
Mem:         96695      94316       2379          0          0      71689
{% endhighlight %}

At first glance, one might be extremely concerned; the box has 96GB of memory, of which 94GB are in use! It would seem we have just 2GB left! However, the "cached" column at the far right is extremely important. Let's look at the very next line:

{% highlight bash %}
$ free -m | head -3
             total       used       free     shared    buffers     cached
Mem:         96695      94316       2379          0          0      71689
-/+ buffers/cache:      22612      74083
{% endhighlight %}

`free` is nice enough to calculate the ACTUAL used and free in the "-/+ buffers/cache" line of the output, which is the most important line in `free`'s output.  If you'd like to play around with this cache, the kernel provides a method to request a cache drop:

{% highlight bash %}
$  echo 3 | sudo tee /proc/sys/vm/drop_caches
3

$ free -m
             total       used       free     shared    buffers     cached
Mem:         96695      20462      76233          0          0         78
-/+ buffers/cache:      20384      76311
{% endhighlight %}

The "cached" column represents memory that, although currently being used as a cache of files on disk, the kernel would quickly give it up if it had a reason to, such as an application allocating memory. This is an extremely important concept to understand if you are monitoring your system's memory usage, troubleshooting performance issues, or even A/B testing code for deployment.
