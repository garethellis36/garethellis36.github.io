---
layout: post
title:  "Handling missing required arguments in PHP"
date:   2015-09-18 16:30:47
comments:   true
categories: php
---
As any PHP developer knows, PHP handles missing arguments in its own idiosyncratic way. That is, if you call a function
or method and don't include all of the required arguments, PHP emits a warning but continues to execute.   
   
Best practice for dealing with this is to ensure that all of your custom functions and methods include guard clauses for
checking for the presence of arguments; you could also go further and check the type of those arguments as well. For 
example:
{% highlight php %}
<?php
use Mynamespace\Exception\MissingArgumentException;

function myFunction($foo, $bar) {
    if (!isset($foo) || !isset($bar)) {
        throw new MissingArgumentException(); 
    }
    //do stuff...
}
{% endhighlight %}

However, sometimes this isn't a viable strategy. In my daily work I am dealing with a large application that has more
than one hundred controller classes and more than a thousand methods - most of which were implemented without guard 
clauses (usually by me before I knew any better!).   
   
Our application uses CakePHP 2, which has automagic routing, e.g. `/posts/view/2` will automatically map to 
`PostsController::view(2)`. Sometimes users type in URLs manually and may miss parameters out by accident. Taking the
above example, this can cause messy error log entries:

{% highlight php %}
<?php
class PostsController extends AppController
{
    public function view($postId)
    {
        $this->set("post", $this->Post->findById($postId));
    }
}
{% endhighlight %}

With this code, in the event of a user visiting `/posts/view/` without passing in the ID, you will get a warning from 
PHP for the missing argument, a notice for using an undefined variable and probably some other errors in your view file 
when you try to use the '$post' variable.   
   
I suppose there is also the potential for destructive behaviour if your method is going on to `INSERT`, `UPDATE` or 
`DELETE` in your database and the code isn't checking for required parameters first...   
   
Rather than go through hundreds of methods to add guard clauses, I came up with a simpler solution. Our application uses
 a custom error handler class. I added the following new method to that class:
 
{% highlight php %}
public static function isMissingParameterWarning($description)
{
    return (preg_match("/expects at least [0-9]* parameters/", $description) || preg_match("/Missing argument [0-9]* for/", $description));
}
{% endhighlight %}

This method is very simple - it checks the error description from PHP to see if it matches the pattern of the warning
message emitted by a call to a function or method with a missing argument, and returns `true` or `false` as appropriate.   
   
The error handler has another method called `handleError` which is called by Cake and which receives a number of parameters,
one of which is `$description`. So, I updated that method to call this new method and throw a custom Exception class:

{% highlight php %}
//in handleError method
if (self::isMissingParameterWarning($description) {
    throw new MissingParameterException($description);
}
{% endhighlight %}

The result of this is that any instances of methods or functions being called with missing parameters show up in the error
log as just one entry, and the script stops executing any further, preventing any destructive behaviour which could occur.   
   
I hope someone else finds this useful!