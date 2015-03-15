---
layout: post
title:  "Tricky DevOps/Systems/Site Reliability Engineer Interview Questions"
date:   2015-03-15 15:25:52
categories: cli
---
There are a plethora of really basic (Linux) systems-oriented questions floating out around there, but here are a few of my favorite, insidious questions that I have found can start a great conversation and highlight the level of a candidate's understanding.

1.) Subnet Mask: 255.255.255.1
------------------------------
Under the assumtion it is valid (*technically*, [it is](https://tools.ietf.org/html/rfc1219)), how many hosts would be in a subnet with this network mask? What would the network number and broadcast be? What would be unusual about the host numbering? How would you represent it with CIDR?

255.255.255.1 is quite an odd subnet mask, with a bitmask of:

11111111 11111111 11111111 00000001

Remembering that the network is the part that matches all the '1's (and in this case, the 1's are non-contiguous), let's try a few out. Below are a few samples starting from 8.8.8.0, bolding the host portions of the netmask applied each IP:

8.8.8.0 - 00001000 00001000 00001000 **0000000**0<br />
8.8.8.1 - 00001000 00001000 00001000 **0000000**1<br />
8.8.8.2 - 00001000 00001000 00001000 **0000001**0<br />
8.8.8.3 - 00001000 00001000 00001000 **0000001**1<br />
8.8.8.4 - 00001000 00001000 00001000 **0000010**0<br />
8.8.8.5 - 00001000 00001000 00001000 **0000010**1

Notice that .0, .2, and .4 all have the same network bits matching, while .1, .3, and .5 have their own set of network bits matching. The 255.255.255.1 subnet mask would split a /24 into two networks; one of evens and one of odds. So let's go back and answer our original questions:

### How many hosts would be in this subnet?
126 (the same as a /25 (there are 25 bits, after all), 2^7 minus 2 for network & broadcast)

### What would the network number and broadcast be?
You can't answer this until you have the final octet of an address you are applying the netmask to, as it is part of the network, too. 
If the last octet is even: X.X.X.0/X.X.X.254
If the last octet is odd: X.X.X.1/X.X.X.255

### What would be unusual about the host numbering?
Odds and evens would make up two different networks in the last octet

### How would you represent it with CIDR notation?
You can't. CIDR allows you to quickly specify a subnet mask by counting the 1 bit's in a subnet mask before the 0's start. Because the 1's are not contiguous, there is no way to represent it using CIDR

This question can help determine if a candidate has an understanding of both network subnets as well as bit-wise operations.

2.) Write the alphabet from A to Z vertically, write an executable starting with every letter
---------------------------------------------------------------------------------------------
Have the candidate write the most basic executable they can come up with that starts with each letter of the alphabet, with a focus on commands that are most likely to be included with a base distribution. NO shell builtin's (but shells themselves are OK).

A few letters can be pretty hard to come up with on the spot. Notably difficult letters:

- E (so many of the ones you'd tend to are builtin's: exit, exec, etc. Good answers are `ex`, `expr`)
- H (`host` and `hostname` are usually forehead-slappers once you get them)
- J (`java` isn't that basic and `join` isn't commonly used)
- K (`kill` is NOT a builtin, so it flies!)
- O (`od`, `openssl`)
- Q (`quotaadd`)

You can use `type -1` to verify the answers if you are not familiar with any of them (and watch out for commands that can be both builtin's and executuables):

{% highlight bash %}
$ type -a cd
cd is a shell builtin

$ type -a ex
ex is /bin/ex
ex is /usr/bin/ex

$ type -a time
time is a shell keyword
time is /usr/bin/time
{% endhighlight %}

This question not only shows a good working set and exposure to basic unix tools (and each answer could be an interesting discussion), but also which ones are core to Linux in addition to the difference between builtins and executables.

3.) How do you find file creation time in Linux?
------------------------------------------------

Basically, you don't. The common answer here is `stat`, but that's not correct. stat reports these times:

 - mtime - file modification time
 - atime - file access time
 - ctime - file **change** time (the greater of the last time the file was modified OR the last time the file had permissions/owner/metadata changed)

ctime is **not** create time, as one might expect (and as would be infinitely more useful). There are [debugfs methods in ext4](http://tecadmin.net/file-creation-time-linux/) and other less common file systems include this feature, but stat is not the answer in common Linux file systems.

A great answer would be something along the lines of: "if you knew it was the last file (or directory) created or deleted in the directory that contains it, the mtime of the directory would be the creation time of the file"

4.) Your Linux system is not experiencing a performance issue, but `free` reports very little memory free. Why?
---------------------------------------------------------------------------------------------------------------

This one is a little vague, and i would work with the candidate to make sure they understood what is being asked, but anyone interviewing at mid-level or greater position should have run into what seems like a low memory issue in on a Linux box, but isn't. The answer i'm looking for here is "cached" memory. Let's look at a standard `free` output:

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

`free` is nice enough to calculate the ACTUAL used and free in the "-/+ buffers/cache" line of the output, which is the most important line in `free`'s output.  If you'd like to play around with thi cache, the kernel provides a method to request a cache drop:

{% highlight bash %}
$  echo 3 | sudo tee /proc/sys/vm/drop_caches
3

$ free -m
             total       used       free     shared    buffers     cached
Mem:         96695      20462      76233          0          0         78
-/+ buffers/cache:      20384      76311
{% endhighlight %}

The "cached" column represents memory that, although currently being used as a cache of files on disk, the kernel would quickly give it up if it had a reason to, such as an application allocating memory. This is an extremely important concept to understand if you are monitoring your system's memory usage, troubleshooting performance issues, or even A/B testing code for deployment.
