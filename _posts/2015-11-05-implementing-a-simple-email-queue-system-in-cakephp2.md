---
layout: post
title:  "Implementing a simple email queue system in CakePHP 2"
date:   2015-11-05 15:17:00
comments:   true
categories: php
---
The principle application I manage and develop for on a day-to-day basis sends many hundreds of emails every day. Until recently, all of these emails were sent as part of the HTTP lifecycle, that is, a user would perform an action in their browser, the application would do something server-side (e.g. update database) and then send an email notification, then the user would see an updated page in their browser.

Handling emails like this has many downsides, the main ones being:
- The user suffers from any latency issues between the web server and the email server
- The application can become more brittle if there are problems with the email server (e.g. downtime)

Just yesterday we suffered some SMTP server issues which resulted in something in the region of 150 emails failing to send because Cake's connection to the SMTP server timed out. When this happens, CakeEmail throws `SocketException` - our application catches these and handles the problems gracefully, but it is still a pain for the user because they will often have to repeat their action if the action is something that depends on an email being sent (e.g. using the application to send a document to a customer, rather than simply performing an action and then receiving an email notification).

One approach to this problem is to use a `message queue`. Message queues are exactly what they sound like: they take a "message" from one system, and hold it in a queue; another system can then retrieve that message at a later time and process it as required. Sidebar: in the context of "message queue", "message" can mean anything - it does not specifically refer to emails. This is a distinction that I did not quite understand at first.

There are several dedicated applications out there for handling message queues, including [RabbitMQ](http://www.rabbitmq.com/) and [Beanstalkd](http://kr.github.io/beanstalkd/). Beanstalkd is Linux-only and we are Windows-hosted, so that was off the table immediately. I had a look at RabbitMQ and was impressed by how simple it was to get it to interact with PHP using [videlalvaro/rabbitmq](https://github.com/videlalvaro/php-amqplib), but ultimately I decided that I didn't want to introduce another element into our single-server "architecture". 

Instead, I ended up implementing a database-backed queue system. The application already used MySQL as its datastore, and at the scale we are dealing with, using MySQL to queue emails didn't seem like a big issue. So, here are the details of that implementation. Please note that this is quite a specific implementation in that it only handles emails - it is not a fully-fledged message queue at all. There are solutions for CakePHP out there already if you want a database-backed message queue that can handle different types of tasks, for example Mark Scherer's [Queue plugin](https://github.com/dereuromark/cakephp-queue). You might also implement this in a completely different way if you are starting an application from scratch, but for our application and its existing codebase this solution works beautifully.

First of all, I created a database table for storing the emails. In order to play nicely with CakePHP's model conventions, I named it `email_queues`. This doesn't make much sense out of context (because it is not multiple queues but one queue of multiple emails), but it does give me a model in Cake called EmailQueue, which perfectly describes its function to me. Here's the schema I went with:

{% highlight mysql %}
+------------+------------+------+-----+---------+----------------+
| Field      | Type       | Null | Key | Default | Extra          |
+------------+------------+------+-----+---------+----------------+
| id         | int(11)    | NO   | PRI | NULL    | auto_increment |
| email      | longtext   | NO   |     | NULL    |                |
| locked     | tinyint(4) | NO   |     | 0       |                |
| attempts   | int(11)    | NO   |     | 0       |                |
| last_retry | datetime   | YES  |     | NULL    |                |
| created    | datetime   | YES  |     | NULL    |                |
| failed     | tinyint(4) | NO   |     | 0       |                |
+------------+------------+------+-----+---------+----------------+
{% endhighlight %}

These should hopefully be self-explanatory, but here's what each field is for: `email` stores the serialized `CakeEmail` object - this is effectively the email itself that we will send later; `locked` is a boolean switch to indicate to a queue worker that this email is being processed already; `attempts` is the number of times a worker has tried to send the email previously; `last_retry` is a datetime stamp of the last occasion a worker tried to send the email; `created` is the time that the email was put into the queue (useful for debugging); finally, `failed` is a boolean switch to indicate that the email has failed to send.

In pure database terms, the flow for queuing emails is simple: whenever the application calls `CakeEmail::send()` to send an email, instead of sending it via SMTP it serializes the email object and saves it in the database. A worker can later retrieve the email from the database, mark it as locked (so another worker doesn't attempt to send it) and try to send it. If it sends successfully, the worker deletes the email from the database. If it fails to send, the worker unlocks the email, increments the number of attempts and stamps the last_retry field with the current date & time. Then, another worker can attempt to send the email again at a future time. Once a prescribed number of attempts has been exceeded for the email, the worker will mark the email as failed and no workers will attempt to send it again.

This means that in our code we only need two components: a way of adding the email to the queue, and a way of retrieving and processing the email. As our application already had hundreds of calls to `CakeEmail::send()`, the easiest way for me to achieve the first component was to create a new transport class. `CakeEmail::send()` in turn calls the send method on the transport class, so in this case it would just save the email to the database.

First, I added an email configuration which would use the new transport class. As standard, you would add this to the `EmailConfig` class in `app/Config/email.php`:

{% highlight php %}
    public $dbQueue = [
        "transport" => "DbQueue",
    ];
{% endhighlight %}

Then, whenever your app calls `new CakeEmail("default")` I updated it to call `new CakeEmail("dbQueue")`. In our case, we were instantiating CakeEmail via a static factory method already, so we just needed to update that:

{% highlight php %}
<?php
App::uses("CakeEmail", "Lib");

/*
	Factory class for creating new Email object
 */

class EmailFactory
{
	/*
		Create Email object
		@param 		$cfg 	Email config to use
		@return 	Instance of CakeEmail (or whatever you choose to return)
	 */
	public static function create($cfg = "dbQueue") {
		return new CakeEmail($cfg);
	}
}
{% endhighlight %}

Whenever our app needs an instance of CakeEmail, it just calls `EmailFactory::create()`. Because we were already doing it this way, updating the app to use the new dbQueue config was a doddle. Next, I needed to create the transport class itself. As mentioned above, this only needs to take the CakeEmail object, serialize it and save it in the database.

{% highlight php %}
<?php
App::uses('AbstractTransport', 'Network/Email');

/*
 * Transport class for sending emails via queue from database
 * This class provides a send() method which serializes the CakeEmail object and saves it in database for a worker to pick up at a later time
 */
class DbQueueTransport extends AbstractTransport
{
    /*
     * "Send" an email by serializing CakeEmail object and saving to database for retrieval by a worker
     */
    public function send(CakeEmail $email)
    {
        $data = [
            $this->model->alias => [
                "email" => base64_encode(serialize($email))
            ]
        ];

        $model = ClassRegistry::init("EmailQueue");
        $model->create();
        if ($model->save($data)) {
            return true;
        }
        throw new Exception("Failed to save email to database");
    }
}
{% endhighlight %}

Very simple - now when you call `CakeEmail::send()` while using the `dbQueue` config, the email object is serialized and saved to the database, i.e. added to the queue.

A few notes here:
- You have to `base64_encode()` the serialized object, or you may have problems retrieving it later. [See this article by David Walsh for more](https://davidwalsh.name/php-serialize-unserialize-issues)
- If the save fails, the code throws a vanilla `Exception` so that I didn't have to update our existing try/catch blocks. Throwing vanilla `Exception` is not best practice!
- The above differs slightly from our actual implementation - instead of instantiating the model class in the method, I used dependency injection in the constructor method. `CakeEmail` instantiates the transport class without any constructor parameters, so you also have to extend `CakeEmail` to be able to do this. I elected not to go into that level of detail in this blog post.

Next, I needed a worker to retrieve emails from the queue and process them. The easiest way to achieve this was to use CakePHP's built-in shell. I created a shell class called `DbQueuedEmailWorkerShell` as follows:

{% highlight php %}
<?php
App::uses("CakeEmail", "Network/Email");

class DbQueuedEmailWorkerShell extends AppShell
{
    public $uses = [
        "EmailQueue"
    ];

    const MAX_ATTEMPTS = 50;

    const RETRY_DELAY_SECONDS = 15;
{% endhighlight %}

Here we set-up the class. `$uses` gives the shell class access to the `EmailQueue` model via `$this->EmailQueue` and the two constants are settings that will be used later on - they are hopefully self-explanatory.

Next, I needed a `main()` method that would do the heavy lifting. I have tidied the code up a bit here and obfuscated a few bits and pieces for brevitys sake.

{% highlight php %}
    public function main()
    {

        //get queued emails that haven't been attempted yet or which haven't been attempted for at least RETRY_DELAY_SECONDS
        $data = $this->EmailQueue->find("all", [
            "conditions" => [
                "locked" => 0,
                "failed" => 0,
                "OR" => [
                    "last_retry <=" => $date("Y-m-d H:i:s", strtotime("-" . self::RETRY_DELAY_SECONDS . "seconds")),
                    "last_retry IS NULL"
                ]
            ]
        ]);

        if (empty($data)) {
            return $this->out("No queued emails found");
        }

        //mark locked
        $data = array_map(function ($value) {
            $value["EmailQueue"]["locked"] = 1;
            return $value;
        }, $data);
        $this->EmailQueue->saveMany($data);

        //process the emails
        $requeue = [];
        foreach ($data as $queuedEmail) {

        	//unserialize the stored object - occasionally things can go wrong when unserializing, so I included a check that the resulting variable is in fact a CakeEmail instance
            try {
                $email = unserialize(base64_decode($queuedEmail["EmailQueue"]["email"]));
                if (!$email instanceof CakeEmail) {
                    throw new Exception("Failed to unserialize email");
                }
            } catch (Exception $e) {
                MyErrorHandler::handleException($e, false);
                continue;
            }

            try {
                //revert email to use the default email config, e.g. SMTP, otherwise it'll just put it back in the database!
                $email->config("default");

                //if original CakeEmail::send() included a $content parameter, it needs to be pulled out and inserted in the new call to send()
                //otherwise send() just sends a blank message body
                $emailContent = $email->message();
                $email->send($emailContent);

                //sent successfully, delete from queue
                $this->EmailQueue->delete($queuedEmail["EmailQueue"]["id"]);

                $this->out("[x] SENT: " . $email->subject(), 2);

                //do something to log the email success here

            } catch (Exception $e) {

                $nextAttempt = $queuedEmail["EmailQueue"]["attempts"] + 1;

                $this->out("[ ] FAIL: " .$email->subject(), 2);
                //do something to log the email failure here

                //if within max # attempts, unlock, increment attempts and save timestamp of this attempt
                if ($nextAttempt < self::MAX_ATTEMPTS) {
                    $requeue[] = [
                        "id" => $queuedEmail["EmailQueue"]["id"],
                        "attempts" => $nextAttempt,
                        "last_retry" => date("Y-m-d H:i:s"),
                        "locked" => 0
                    ];
                    continue;
                }

                //max # attempts have been exceeded
                $requeue[] = [
                    "id" => $queuedEmail["EmailQueue"]["id"],
                    "attempts" => $nextAttempt,
                    "last_retry" => date("Y-m-d H:i:s"),
                    "locked" => 0,
                    "failed" => 1
                ];

                $this->out("[ ] FAIL: exhausted maximum number of attempts: " . $email->subject(), 2);

                //do something to log the fact that max attempts were exhausted
            }
        }

        //update database with emails that need to be re-queued
        if (!empty($requeue)) {
            $this->EmailQueue->saveMany($requeue);
            return $this->out("Saved " . count($requeue) . " emails for re-queuing");
        }

        return $this->out("No emails re-queued");
    }
{% endhighlight %}

Finally, I decided that I wanted a method to attempt to process the failed emails. This means that if an email fails for some reason that is not to do with a temporary outage and will *never* send correctly, I have the opportunity to address the fault and then try to re-send at a later time.

{% highlight php %}
    public function processFailed()
    {
        //get failed emails
        $data = $this->EmailQueue->find("all", [
            "conditions" => [
                "locked" => 0,
                "failed" => 1,
            ]
        ]);

        if (empty($data)) {
            return $this->out("No failed emails found");
        }

        //mark locked
        $data = array_map(function ($value) {
            $value["EmailQueue"]["locked"] = 1;
            return $value;
        }, $data);
        $this->EmailQueue->saveMany($data);

        $unlock = [];
        foreach ($data as $k => $queuedEmail) {

            try {
                $email = unserialize(base64_decode($queuedEmail["EmailQueue"]["email"]));
                if (!$email instanceof CustomEmail) {
                    throw new Exception("Failed to unserialize email");
                }
            } catch (Exception $e) {
                MyErrorHandler::handleException($e, false);
                continue;
            }

            try {
                $email->config("default");
                $emailContent = $email->message();
                $email->send($emailContent);

                //sent successfully, delete from queue
                $this->EmailQueue->delete($queuedEmail["EmailQueue"]["id"]);

                $this->out("[x] SENT: " . $email->subject(), 2);

            } catch (Exception $e) {
                $this->out("[ ] FAIL: " . $e->getMessage() . " - " . $email->subject(), 2);
                $unlock[] = [
                    "id" => $queuedEmail["EmailQueue"]["id"],
                    "locked" => 0
                ];
                continue;
            }

        }

        if (!empty($unlock)) {
            $this->EmailQueue->saveMany($unlock);
        }
    }
{% endhighlight %}

This works very similarly to the `main()` method - it retrieves and locks the failed messages from the database, and attempts to send them. If successful, the message is deleted; if not, it is unlocked again.

Once this was implemented, all I had to do was set-up a scheduled task on the production server to run the worker at set intervals (I elected for every 20 seconds), et voila - we had an email queue system in plcae and ready to handle our application's email requirements.