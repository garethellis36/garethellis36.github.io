---
layout: post
title:  "Firing off CakePHP shells in tests"
date:   2015-11-17 23:13:10
comments:   true
categories: php cakephp tests phpunit
---
Just a quick post today! This morning I was writing a functional test for my previously discussed [database-backed email
queue system](/php/cakephp/2015/11/06/cakephp-email-queues-the-return.html). As this was a functional test, I wanted to
 do an end-to-end test of the queue system, so I had to do three things:
 
- Create an email message and save it in the database
- Call the email worker shell to retrieve the message and send it
- Check that the email had arrived

The first step was very easy - I just had to create an instance of `CakeEmail` using my `DbQueue` config and call its 
send method. I then tested the success of this by checking the database for a record - as I'm using a test database which
gets flushed out every time tests run, I can simply check for a row count of '1'.

The last part was also simple. In my develop environment I use the wonderful [MailHog](https://github.com/mailhog/MailHog) 
as my local SMTP server. MailHog provides a simple HTTP API and I was able to use this combined with Guzzle to check that
the message sent by the worker in the second step had been sent. To ensure I was checking for the right message, I had
the test generate a UUID for the subject line - then I could just iterate over the Guzzle response to find the matching
subject line.

The second part took me a bit longer to work out. Because the emails are sent by a worker shell, I needed to figure out
how to trigger a CakePHP shell script from within a test. It turns out that this was very simple, and this is all I wanted 
to really share and document in this post. Here's my first version of this section of the test:

{% highlight php %}
<?php
public function testCanSendEmailThroughQueue()
{
    //create email and save to DB
    
    $worker = new DbQueuedEmailWorkerShell();
    $worker->main();
    
    //check MailHog API for the email
}
{% endhighlight %}

This worked perfectly. The only downside was that by creating `DbQueuedEmailWorkerShell` like this, the constructor or
CakePHP's `Shell' class gets called and it uses its `ConsoleOutput` class for stdIn, stdErr and stdOut. As a result, 
when you run your test, you get the output from the shell mixed in with your test results, which isn't so helpful. 
Resolving this was also pretty easy - I just had to provide mocks of ConsoleOutput for stdIn, stdErr and stdOut as follows:

{% highlight php %}
<?php
public function testCanSendEmailThroughQueue()
{
    //create email and save to DB
    
    $mockedConsoleOutput = $this->getMock("ConsoleOutput");
    $worker = new DbQueuedEmailWorkerShell($mockedConsoleOutput, $mockedConsoleOutput, $mockedConsoleOutput);
    $worker->main();
    
    //check MailHog API for the email
}
{% endhighlight %}

This way, the key functionality of the worker shell (i.e. retrieving an email and sending it) is executed but the text
output is not emitted to your console, keeping your test results nice and clean. Cool!

One suggested improvement was to move the code which retrieves and sends the email to another class (rather than have it
as part of the shell class), thus allowing me to test it without having to invoke the shell. However, on reflection I like
this approach as it means I can do a functional test using the actual shell which is called in normal application usage.