---
layout: post
title: "Using Ghost as a blogging platform"
modified: 2014-06-04 17:03:06 -0400
category: technology
tags: [Blog, technology]
image:
  feature:
  credit:
  creditlink:
comments:
share:
---

A few months ago I started playing with [Ghost](https://ghost.org/) the blogging platform. I ended up not using it (I'm using [Jekyll](http://jekyllrb.com/) now) but I think this notes can be useful for people trying it out.
<br/><br/>
The [documentation](http://docs.ghost.org/) section of the Ghost website is quite complete but I have found the articles at [howtoinstallghost.com](http://www.howtoinstallghost.com/) better at identifying problems and suggesting solutions.
<br/><br/>
So, these are the steps I followed:

1. Create a EC2 instance in AWS -> I followed this [tutorial](http://www.howtoinstallghost.com/how-to-setup-an-amazon-ec2-instance-to-host-ghost-for-free-self-install/) that shows how to install Ghost on an Amazon EC2 instance using the [Free Tier](http://aws.amazon.com/free/). It was really easy to follow but I have a couple of things to note:

    * Everything needs to be done with `sudo`
    * You will need to use absolute paths to call [npm](https://npmjs.org/). For instance: `/usr/local/bin/npm install`
    * There is [another tutorial](http://www.howtoinstallghost.com/how-to-setup-an-amazon-ec2-instance-to-host-ghost-for-free/) that uses a pre-build AMI with Ghost already installed. However that AMI doesn't seem to be available anymore.
<br/><br/>
2. Now you have a Ghost instance running on the cloud but the moment you close the terminal or switch off your laptop the process will stop, so you need to figure out how to make Ghost run _forever_. [This article](http://www.howtoinstallghost.com/how-to-start-ghost-with-forever/) will tell you how.

    When I first tried to run _forever_ the system couldn't find node (even though I installed in the previous step). Thankfully I wasn't the only one facing [this](http://www.howtoinstallghost.com/how-to-start-ghost-with-forever/#comment-216) problem and I solved it by reading [this](http://stackoverflow.com/questions/4976658/on-ec2-sudo-node-command-not-found-but-node-without-sudo-is-ok#answer-5062718) which just tell you how to link the missing files.

3. Finally, make sure you have backup of your content. This [article](https://ghost.org/forum/using-ghost/1067-how-to-backup-ghost-content-data/) will tell you how.
