---
layout: post
title:  "Using namespaced classes in CakePHP2"
date:   2015-10-01 21:12:00
comments:   true
categories: php
---
Because it supports PHP 5.2, CakePHP version 2 does not use any namespaced classes in its core, and as a result the convention
for classes used in Cake 2 apps is to not use namespaces. Note, I am not talking about packages loaded through composer - 
you can use namespaced classes through composer to your heart's content.
   
So until recently, when I wanted to add custom classes which I wasn't loading through composer, I placed them 
(without namespaces) in the `Lib` folder and then used `App::uses()` to lazy-load in the rest of the app.   
    
But there is a way to namespace these custom classes, autoload them with composer and include them in your app in the 
"proper and modern" PHP style. This method assumes that you are 
[already installing CakePHP through composer](http://mark-story.com/posts/view/installing-cakephp-with-composer) and
using composer's autoloader.   
   
First, decide where you want to save your namespaced classes. I put mine in `app/Lib` along with my other non-namespaced
classes. Open your `composer.json` and insert the following:
 
{% highlight json %}
"autoload": {
  "psr-4": {
    "App\\": "app/Lib/"
  }
}
{% endhighlight %}

Next, run `composer dump-autoload` to re-generate your autoload.php. Composer will now look in your `app/Lib` folder and
autoload any classes in that folder under the `App` namespace.   
   
For example, if you were writing a custom error handler and saving it in `app/Lib/Error/CustomErrorHandler.php` 
it would look like this:

{% highlight php %}
<?php
namespace App\Error;

class CustomErrorHandler
{
}
{% endhighlight %}

You can then use it in your app by putting `use App\Error\CustomErrorHandler` at the top of any PHP files where you wish 
to make use of the class.   
   
There's nothing particularly wrong with using `App::uses();` but if you're like me and like to make use of as many PHP.new
shiny features as possible, then hopefully you'll find this to be a useful tip. Make sure you don't namespace any classes
in your app which are automagically invoked by the Cake core, for example your controllers, models, views, tests, etc - 
adding namespaces to these classes will break your app. Enjoy!
