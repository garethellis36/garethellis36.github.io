---
layout: post
title:  "Phinx database migrations: be atomic"
date:   2015-12-21 09:57:02
comments:   true
categories: database migrations phinx mysql
---

Managing the version control of databases is not an easy problem to deal with, and until I discovered the wonderful
[Phinx](https://phinx.org/) I was using a variety of sub-optimal home-made solutions using *.sql* files and Git. I was 
never able to get Doctrine Migrations working correctly on my set-up, and Phinx does a similar job very well indeed.

I'd meant to write a short post with a Phinx tip for a while but had forgotten until I was reminded by 
[this blog post on inserting data by Lorna Mitchell](http://www.lornajane.net/posts/2015/insert-data-with-phinx). (Note 
to self: there is now an `insert()` method available for seeding!)

My tip is simple: when writing database migrations in Phinx (this may well apply to similar tools as well), try to be as 
[atomic](https://en.wikipedia.org/wiki/Atomic_commit) as possible. Because of the way Phinx works, if you try to do too
much in one migration class, you can run into problems.

Take this example here from a fictional butchery app:

{% highlight php %}
<?php
use Phinx\Migration\AbstractMigration;

class MeatMigration extends AbstractMigration
{
    public function change()
    {
        //create a table full of cuts of meat, e.g. sirloin, rump, etc
        //associated with animals
        $table = $this->table("cuts");
        $table->addColumn("meat_id", "integer")
            ->addColumn("name", "string")
            ->create();
            
        //try to create a table of animals
        //but oh no, we already did this in a previous migration
        //so this will throw an exception
        $table = $this->table("meats");
        $table->addColumn("name", "string")
            ->create();
    }
}
{% endhighlight %}

In this simple example we are trying to create two tables, one called cuts and one called meats. As you can see from the
comment above the second section, we have forgotten that the `meats` table has already been created - Phinx will therefore
throw an exception and exit.

In this case, the migration will not be marked as complete - however any SQL that Phinx created and executed before the
exception will have 'stuck'. So, when we fix our code by removing the call to create this table and call `phinx migrate` 
again, it will fail because the `cuts` table already exists. Boo. You can't do `phinx rollback` to go back in time to
 the last migration, because as far as Phinx is concerned, this migration has never been executed - a rollback will
 take you further back than you want to go. 
 
With this specific example, one approach would be to surround the calls to create the table with `if ($this->hasTable("tablename")`.
However, my preferred and recommended approach is to make my migration classes `atomic`. That is, don't try to do more
than one thing in a single migration class (hello Single Responsibility Principle). 

Taking our example above, this would mean separating the two calls to create into separate migrations. That way, if one 
of them fails, it doesn't make any unwanted changes to our DB schema that we have to manually undo in order to be able 
to proceed. Similarly, if you were to want to create a table and insert some seed data, being atomic would mean separating
the table creation and the data insertion into two migrations.

It took me a while to adapt to this approach personally, and I still forget to apply it sometimes - mainly out of laziness
(because typing `phinx create` twice is so much effort, obviously!), but I hope this post comes in useful for some other
Phinx users out there! I now await the inevitable responses that I have misunderstood the meaning of atomicity... :-)