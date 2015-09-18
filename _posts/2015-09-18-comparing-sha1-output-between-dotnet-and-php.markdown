---
layout: post
title:  "Comparing SHA1 output between .NET and PHP"
date:   2015-09-18 17:30:47
categories: php
---
I have recently been working on a PHP library to integrate a PHP application with a variety of services via a SOAP API. 
One of the first things I had to do was get my PHP library to authenticate with the target service via a username and 
password. The API documentation indicated that the password had to be sent as a hash, encoded using SHA1 and salted. The
salt was within the documentation and apparently had to be placed after the password, so I wrote the following method to 
generate the hash:
{% highlight php %}
protected function hashPassword($password, $salt)
{
    return sha1($password . $salt);
}
{% endhighlight %}

The target service was returning a failure on the login attempt using this, so after much head-scratching and checking
of the basic things (i.e. I had the password correct), I went back to the documentation. Helpfully, the author(s) had provided
an example hash method in .NET C#:
{% highlight c# %}
public static string HashPassword(string password)
{
    return System.Web.Security.FormsAuthentication.HashPasswordForStoringInConfigFile(password + salt, "sha1");
}
{% endhighlight %}

Working on the assumption that the authors would not have provided an example which didn't work, I decided I should 
compare the output of the above rather verbose method in .NET C# with that of PHP. With a cheeky `var_dump()` I 
ended up with a hash of something like `a1b2c3d4`. A quick Google search found various .NET C# "fiddlers" that I could play
with in order to get the output of the above method. I quickly discovered that this method is deprecated in the latest 
version of .NET (4.5) so I had to find another fiddler which would play ball with .NET 4.0. 
[RexTester](http://rextester.com/runcode) did exactly this and allowed me to quickly work out what came out of this method:
the same input password and salt produced a hash like `A1B2C3D4` - the only difference was that the letters were in 
upper case. Thus, a quick update to my PHP resulted in a successful login to the target service:

{% highlight php %}
protected function hashPassword($password, $salt)
{
    return strtoupper(sha1($password . $salt));
}
{% endhighlight %}

Hopefully this post will help someone else who is trying to integrate a PHP app with a service which was clearly rooted
in the .NET C# world.   
   
Final note - [this StackOverflow post](http://stackoverflow.com/questions/13527277/drop-in-replacement-for-formsauthentication-hashpasswordforstoringinconfigfile)
provides an approach for hashing in SHA1 in .NET 4.5.