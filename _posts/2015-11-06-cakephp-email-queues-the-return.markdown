---
layout: post
title:  "CakePHP email queues: the return"
date:   2015-11-06 17:00:00
comments:   true
categories: php cakephp
---
Yesterday I wrote about [how I implemented a simple message queue-like approach for handling emails in CakePHP 2](/php/cakephp/2015/11/05/implementing-a-simple-email-queue-system-in-cakephp2.html). Today I 
want to share some lessons I learned post-implementation and some changes I had to make to my classes.

The first issue occurred when a user tried to send an email with a couple of large attachments (several MB). By default, 
MySQL's `max_packet_size` is set to 1MB, and when CakeEmail calls send(), it reads the data from the attachments and 
stores it in the `_message` property. Thus, when you serialize it and try to persist it in the database, MySQL can have issues with the size of the packet you 
are sending it. 

My initial solution to this was to simply increase `max_packet_size`. Our SMTP server has a hard limit of 20MB on emails
and our application is set to automatically reject any email greater than that size, so I decided that setting `max_packet_size = 30M`
would be fine - it would cover the largest possible email, plus some head-room. But then I started thinking about what
could happen if two users sent 17MB emails at once and then a queue worker tried to retrieve 30+ MB of data in one query. 
Would it fail then?

I decided that it was best not to find out and decided to tackle the problem another way. As mentioned above, the reason the
stringified object is so large is that CakeEmail's send method "stringifies" the attachments. I thought it would be optimal
to do this only when actually trying to send the email, and that it wasn't necessary to do this when saving in the database.

Our app was already using an extended version of CakeEmail called `CustomEmail`, so I had to override the `send()` method as follows:
  
{% highlight php %}
<?php
App::uses("CakeEmail", "Network/Email");

class CustomEmail extends CakeEmail
{
    protected $_content;
    
    public function send($content = null)
    {
        $transportClass = $this->transportClass();

        //if sending via DbQueueTransport, just call the transportClass send method immediately
        //results in smaller serialized object as attachments aren't in-lined
        if ($transportClass instanceof DbQueueTransport) {
            $this->_content = $content;
            return $transportClass->send($this);
        }

        return parent::send($content);
    }
    
    public function getContent()
    {
        return $this->_content;
    }
}
{% endhighlight %}

This is a very simple piece of code. All it does is determine what transport class our email object is using, and if it
is using `DbQueueTransport` rather than `SmtpTransport` (or anything else for that matter), it skips the rest of Cake's
usual send() routine and goes straight to the send method of our `DbQueueTransport` class (see previous post for this). 
That's it! I found when testing this on my development environment that sending an email with a 4MB attachment and a long 
 message body resulted in a stored string of more than 6MB; using this method the stored value was far smaller - ~200KB.
 Most importantly, when the worker retriever the email later, it sent it with the attachments successfully.
 
You may notice that I've also added a protected property `$_content` to this custom class (along with a getter method) and set it before calling send 
on the transport class. This is to address the second issue I encountered. You may recall that in my original post I had the following code
 in the worker class:

{% highlight php %}
<?php
class DbQueuedEmailWorkerShell extends AppShell
{
    public function main()
    {
        //stuff, loop over queued emails
        
        $emailContent = $email->message();
        $email->send($emailContent);
        
        //more stuff
    }
}
{% endhighlight %}

The reason for these two lines of code was simple: you can call `CakeEmail::send()` with or without a parameter `$content`.
Most of the time you will want to build a pretty HTML email and you would use the `tempate()` method for that, calling `send()` without a parameter. But for some
very short notifications it can be useful to just do something like `CakeEmail::send("Your message here")`. If you were to call
`$email->send()` in the worker without passing in the content parameter, you would lose "Your message here" completely.

However, I discovered that `CakeEmail::message()` returns more than simply what you pass in as the parameter to `send()`; 
if you are *also* attaching files, `message()` returns the "content" of those files and you end up with an email full of 
garbled junk data that is meaningless to the sender or recipient.

To solve this, I added the `$_content` property to `CustomEmail` and then set it using the `$content` value passed in to 
`CustomEmail::send()`, and then added a getter method to retrieve the value. The updated worker class looks like this:

{% highlight php %}
<?php
class DbQueuedEmailWorkerShell extends AppShell
{
    public function main()
    {
        //stuff, loop over queued emails
        
        $emailContent = $email->getContent();
        $email->send($emailContent);
        
        //more stuff
    }
}
{% endhighlight %}