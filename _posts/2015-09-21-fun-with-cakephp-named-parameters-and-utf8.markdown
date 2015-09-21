---
layout: post
title:  "Fun with CakePHP named parameters and UTF-8"
date:   2015-09-21 15:55:01
comments:   true
categories: php
---
At work, we have various pages in our main application which list various data in table format. At the top of these pages
are some search filters, allowing the user to tailor the data list according to what they need to find. The filters 
themselves are forms: on submission, the data is taken from CakePHP's `$this->request->data` array and a bookmarkable
URL is built; the user is then redirected silently to the bookmarkable URL.   
   
CakePHP 2.x has a cool feature called named parameters. In short, this allows you to access a URL like 
`/users/index/Name:Joe/City:London` and then access an array called `$this->params['named']` in your controller to do 
what you want with it. The URLs produced by our application's search filters are in this form and make for marginally 
prettier URLs than using query strings. Cake's default paginator component uses named parameters for adding page numbers 
and sort keys to URLs too, so the search filters play nicely with that. However, there are disadvantages to named 
parameters as well, and in fact they have been removed from CakePHP 3 entirely. See 
[this post on Mark Scherer's blog](http://www.dereuromark.de/2013/05/04/passed-named-or-query-string-params/) for more info.   
   
Our search filters convert the named parameters into a conditions array which can be used in a call to `Model::find()`. 
So, for example if you visited `/users/index/Name:Joe/City:London`, you might end up with the following:

{% highlight php %}
//App/Controller/UsersController.php

protected function processFilters()
{
    if ($this->request->is('post')) {
        //build URL of named parameters and redirect
        /...
        return $this->redirect($url);
    }
    
    $conditions = [];
    foreach ($this->params['named'] as $key => $value) {
        if (empty($value)) {
            continue;
        }
        $conditions[strtolower($key) . ' LIKE'] = '%' . $value . '%';
    }
    return $conditions;
}

public function index()
{
    $conditions = $this->processFilters();
    var_dump($conditions);
    //array(
    //  'name LIKE' => '%Joe%',
    //  'city LIKE' => '%London%'
    //)
    
    $users = $this->User->find("all", [
        "conditions" => $conditions
    ]);
}

{% endhighlight %}

Obviously you can get more complex with the way you build the conditions, and in our app we also check the keys against
a white-list to ensure that only filters from an approved list are used. However, we started to experience problems when
users started putting unicode characters into the search boxes. If a user were to visit `/users/index/Name:Bjørn/City:Oslo`,
the filters would return empty result sets. After half a day of head-scratching, I found the problem: browsers
actually encode URLs as ASCII: even if you can see a nice UTF-8-friendly 'ø' in the URL, in the background the request
being sent has been ASCII-encoded. This means that what ends up in Cake's named parameter array is also ASCII-encoded, and 
our code above ends up doing a database search using a 'junk' character that obviously didn't exist in our dataset.   
   
Sidebar - an additional problem when debugging this is that Cake's `debug()` function will display *nothing* if the debug
output contains any character which is in a different encoding type to your application's default. You have to use 
plain-old`var_dump()` instead.   
   
The solution to this issue is very simple - the named parameter values have to be converted into UTF-8 before they can be
included in a model finder call as a condition. We have about a dozen versions of the filter code throughout the app, all 
subtly different (not to mention begging for a re-factor and some unit tests), so rather than update the code in a bunch of 
places, I made a small addition to our `AppController` as follows:
{% highlight php %}
protected function convertNamedParamsToUtf8()
{
    //double use of 'params' here is not a typo
    foreach ($this->params->params[“named”] as &$namedParam) {
        $namedParam = mb_convert_encoding($namedParam, “UTF-8”, “ASCII”);
    }
}

public function beforeFilter()
{
    parent::beforeFilter();
    //other stuff
    
    $this->convertNamedParamsToUtf8();
    
    //more stuff
}
    
{% endhighlight %}

This converts the named parameters available on the controller into UTF-8 and enables direct use of them as part of 
`Model::find()` calls; Bjørn is now findable in our filtered data list!   
   
Join me next time for more fun with UTF-8 and how including UTF-8 characters in file names when running PHP on Windows
can be a right pain in the arse!