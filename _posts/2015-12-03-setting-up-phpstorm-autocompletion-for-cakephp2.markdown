---
layout: post
title:  "Setting up PHPStorm autocompletion for CakePHP 2"
date:   2015-12-03 17:58:01
comments:   true
categories: php cakephp phpstorm
---
Back in August I started working in [PHPStorm](https://www.jetbrains.com/phpstorm), and as I've delved into the more
advanced features recently, I've begun to wonder how I ever managed without it. The feature-set of this application is
 just mind-blowing and I've been able to automate many tasks which might otherwise have involved a lot of tedious 
 repetition.
 
One thing that always bugged me though was that auto-completion didn't work for my CakePHP 2 projects. That is, I wanted
to be able to type `$this->` in a controller class and a the list of properties/methods including components and models.
 Further, I wanted to be able to hit ctrl-b and jump to method declarations in model classes. Likewise, I wanted auto-complete
 and 'jump' support in view files.
 
Setting up PHPStorm for auto-completion with CakePHP 2 is pretty straightforward. Google is full of decent tutorials - 
[here's the one I used](http://blog.hwarf.com/2011/08/configure-phpstorm-to-auto-complete.html). In most cases it's a case
of adding some doc block properties to the appropriate classes. The app that I work on most has more than 150 model classes,
a similar number of controllers and many hundreds of view files (*.ctp). The prospect of having to manualy add in those
doc blocks was quite daunting, and also seemed like a waste of time. 

So, I found a way to automate it! 

# Controllers #

I wanted to be able to auto-complete components and models from within controllers. Components were pretty easy - I just 
added the following to my AppController doc block:

{% highlight php %}
<?php
/**
 * @property AuthComponent $Auth
 * ...
 * @property FooComponent $Foo
 */
{% endhighlight %}

As I only use a handful of components in my app, this was easy to do manually. I didn't want to go down this route with
models and add those to the AppController because I wanted my auto-completion to be a bit smarter and more specific to each
controller class. 

So, I wrote a CakePHP shell script wich did the following:
- Scanned the controller folder
- For each found controller file (matching *FooController.php*), assume we need a matching *Foo* model property
- Scan the contents of the controller for any calls to `$this->loadModel('Bar')` (in this case we would have a property called `$Bar`
- Generate the doc block contents and automatically insert it into the controller class, being careful not to overwrite any existing doc block

If a controller had a dependency on a model which didn't have its own class file, I had the doc block use `AppModel` as 
the class for auto-completion. I'd end up with something like this in a controller doc block:

{% highlight php %}
<?php
/**
 * @property Foo $Foo Foo model
 * @property AppModel $Bar Bar model
 * /
 class FooController extends AppController {
 //..
{% endhighlight %}

Easy peasy!

# Models #

Model classes need similar doc blocks for auto-completing associated models. For example, if model `Bar` has a `belongsTo`
relationship with model `Foo`, I want to be able to have auto-completion for doing `$this->Foo` within the `Bar` class.

With 150+ model classes and even more model associations to manage, again I wanted to automate this. I wrote another shell
script which did the following:
- Scanned the model folder
- For each found model file (matching *Foo.php*), used reflection to determine which models it had associations with
- Examined the association definitions to see if the `className` property was set, in which case the doc block had to be 
tailored accordingly
- Checked to see if the class exists, and if not use `AppModel` for auto-completion
- Generate and insert the doc block into the model class

This resulted in something like:

{% highlight php %}
<?php
/**
 * @property Bar $Bar
 * @property Baz $Bat
 * @property AppModel $Garr
 * /
 class Foo extends AppModel
 {
    public $belongsTo = [
        "Bar",
        "Bat" => [
            "className" => "Baz"    //in this case "Bat" is just an alias for the "Baz" class
        ],
        "Garr"  //the class "Garr" doesn't exist, even though there is a table called "garrs" in the DB, so auto-completion has to use AppModel
    ];
 }
 
{% endhighlight %}
 
# Views #
 
To get auto-completion in view files (.ctp), you need to add a docblock which defines the meaning of `$this` within the file.
Most tutorials advise telling you to define it as referring to the standard Cake `View` class. In Cake 2 at least,
the standard `View` class contains doc blocks defining all of the core helpers as properties. However, it obviously
doesn't offer anything for your own custom helpers. 

To get around this, I created a class called `AppView` which extends the core View class. This class isn't touched in any by the actual application - it's 
only there for auto-completion, so it feels a bit hacky but personally I think this is an acceptable solution ;-) Here's
what the class contains:

{% highlight php %}
<?php
/**
 * This class does nothing. It is not touched in any way by the application. It is simply here to define the helpers used
 * in views, allowing auto-completion in PHPStorm custom helpers in view files (*.ctp)
 *
 * @property FooHelper $Foo
 * @property ExtendedFormHelper $Form
 */
 class AppView extends View {}
{% endhighlight %} 

Just by creating this class and including it in the project, PHPStorm can use it for auto-completion in view files. You
didn't even have to include this file in version control if you don't want to. Again, I generated the list of helpers
to include in the doc block using a PHP script.

Then, all you have to do is add this to all of your application's view (.ctp) files:
{% highlight php %}
<?php
/**
 * @var $this AppView
 */
?>
{% endhighlight %}

Once again, I achieved this using an automated shell script.

# The scripts #

[The scripts I used can be found here](https://gist.github.com/garethellis36/da9a32e437f157ee1aca). I would suggest using
this as a base for creating your own shell class. You could then invoke these from the CLI like so:

`$ Console/cake tools modelDocBlocks`

As they are, these scripts will change the content of your class files so use with caution!

# QA # 

After running all of these scripts, I did a few QA steps to make sure I hadn't broken my app:

- I ran my test suite (obvs)
- I did a few manual inspections of the affected files (I used Git to double-check that only the intended files had been touched)
- I did a few Git diffs to double-check that only the doc blocks had been touched
- I then used PHP's built-in linter to double-check that I hadn't introduced any parse errors:

`find app/Model \( -name "*.php" \) -print0 | xargs -0 -n 1 php -lf`

Thanks to [twisty](https://github.com/twisty) for [this gist](https://gist.github.com/twisty/747668) which shows how to 
lint multiple fies at once.