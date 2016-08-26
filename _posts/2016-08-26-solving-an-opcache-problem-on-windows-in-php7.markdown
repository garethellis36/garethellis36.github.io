---
layout: post
title:  "Solving an OpCache problem on Windows in PHP7"
date:   2016-08-26 15:01:00
comments:   true
categories: php php7 windows iis opcache
---
Howdy readers; happy Friday! It's been a while since I've blogged and I
was trying to think about something to write about, so here's a quick one
to share something I learned during the PHP7 migration process.

I installed PHP7 on my local machine (Windows/IIS) and set about making
sure our application code worked fine. After a few tweaks, all our tests
passed and everything seemed rosey.

However, over the course of the next few days I noticed that IIS was
intermittently responding with error 500s on requests for static assets
(i.e. CSS/JS and occasinoally images). The problem seemed to disappear
when I switched IIS back to using PHP 5.6, even though PHP wasn't actually
involved in the requests for CSS/JS.

I was usually able to make the problem go away by clearing my app cache,
restarting IIS and recycling my application pools. However, a couple of days
ago I started getting the 500s from PHP requests as well, which gave me
a lovely IIS error screen including a FastCGI error code (the helpfully
named `0xfffffffe`).

The Google rabbit hole eventually lead me to suspecting that OpCache
was somehow involved. I enabled the PHP.ini directive `opcache.error_log`,
and sure enough, there was an entry in this for each of the 500s that IIS
 was dishing out:
```log
Base address marks unusable memory region. Please setup opcache.file_cache and opcache.file_cache_callback directives for more convenient Opcache usage.
Attempt to access invalid address.
```

It seems like OpCache was having some memory issues, and by default it has no
fallback-options for writing to file cache. By setting the following INI 
directives, it gives OpCache some fallback options and the 500-coded
responses disappeared - hooray!

```ini
opcache.file_cache="C:\Windows\temp\php_opcache"
opcache.file_cache_fallback=1_
```