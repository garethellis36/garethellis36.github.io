---
layout: post
title:  "Setting up Let's Encrypt on Ubuntu 14.04 & Apache"
date:   2015-12-14 18:00:01
comments:   true
categories: ssl https letsencrypt ubuntu apache
---

Let's Encrypt is a fantastic new open-source tool for the creation and installation of *free* SSL certificates. I'd write
a bit on what these are and why they are a Good Thing, but instead I'll provide a link to this lovely [DreamHost blog 
article](https://www.dreamhost.com/blog/2015/12/03/lets-encrypt-and-dreamhost/) which explains that far better than I can.

Setting up Let's Encrypt is really easy. I managed to get it up and running on my 
[social cricket club's website](https://ivcc.co.uk) today, and the site is now served over HTTPS - hooray. I encountered
a couple of teething problems so I just wanted to write a quick post to go over the installation process.

These instructions are only relevant if you are installing Let's Encrypt on Ubuntu 14.04.

The first thing you need to do is clone the Let's Encrypt git repository on the server where you are installing it:

{% highlight bash %}
user@webserver:~$ git clone https://github.com/letsencrypt/letsencrypt
user@webserver:~$ cd letsencrypt
{% endhighlight %}

From here it's usually as simple as using the provided `letsencrypt-auto` script to set things up. However, in Ubuntu 14.04
there is a minor issue with Python. Let's Encrypt has a dependency on version 2.7.9 or higher, and by default you can't
install higher than version 2.7.6 through *apt*. When trying to install using version 2.7.6, you'll get an `InsecurePlatformWarning`
and installation won't complete properly.

So, next you need to add a new *PPA* to apt to allow you to install Python 2.7.9 (or higher):

{% highlight bash %}
user@webserver:~$ sudo add-apt-repository ppa:fkrull/deadsnakes-python2.7
user@webserver:~$ sudo apt-get update
user@webserver:~$ sudo apt-get upgrade
{% endhighlight %}

This should install Python 2.7.10 (or whatever is current when you run it). You can check by running `python --version` 
and checking the output.

Now you can set-up Let's Encrypt using a single command. Here's what I used (our site runs on Apache):

{% highlight bash %}
user@webserver:~$ sudo ./letsencrypt-auto --apache -d ivcc.co.uk
{% endhighlight %}

In my case, I had tried to run this before upgrading Python, and when I ran this again *after* upgrading Python it wouldn't
work because of the virtual environment installed by the first attempt (it failed with the error 
 `ImportError: cannot import name _compare_digest`). I had to delete the virtual environment by deleting the folder at 
`~/.local/share/letsencrypt`. After this, the certification creation and installation worked perfectly. The installation
also configured Apache to automatically redirect HTTP requests to HTTPS - sweet! 

I hope that someone else finds this useful!