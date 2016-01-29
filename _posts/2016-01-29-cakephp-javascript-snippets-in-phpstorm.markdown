---
layout: post
title:  "JavaScript snippets & CakePHP's HtmlHelper in PHPStorm"
date:   2016-01-29 16:58:01
comments:   true
categories: cakephp phpstorm
---
Have you ever used CakePHP's `HtmlHelper::scriptBlock()` method to insert JavaScript into a view? This method takes a string
of JavaScript code as the first argument and outputs a set of `<script>` tags in your view HTML. The problem is that trying
to write even trivial JavaScript code inside PHP code can be messy and hard to stay on top of. You don't get any of 
PHPStorm's usual awesome JavaScript support such as auto-completion or syntax highlighting.

![default behaviour](/assets/scriptBlock1.png)

There is a way to get PHPStorm to play ball with this and recognize whatever you enter as the first argument as JavaScript.
Firstly I need to thank by buddy [Nils](https://twitter.com/nilsluxton) for doing the hard work here and figuring out how 
to do this. I cannot take any credit for this awesomeness, so thank you Nils!

The way to address this issue is via Language Injections. Go to this settings menu to configure a new injection: `File | Settings | 
Editor | Language injections`. Next, click the green `+` symbol at the top right of the settings window and choose 
"Generic PHP". Give the injection a name (I called it `CakePHP HtmlHelper::scriptBlock`), choose JavaScript from the `ID`
 dropdown and then enter the following code in the `Places Patterns` field:
  
{% highlight php %}
+ phpLiteralExpression().inside(phpFunctionReference().withText(string().startsWith("$this->Html->scriptBlock"))).andNot(phpLiteralExpression().inside(phpElement().withText(string().startsWith("["))))
{% endhighlight %}

Save the injection and you should then be able to take advantage of syntax highlighting when using `HtmlHelper::scriptBlock()`.
 Note, this code says to treat any code that inside a `scriptBlock()` call that doesn't start with a '[' as JavaScript. 
 This is so that PHPStorm doesn't treat the second argument to this method - an array of options - as JavaScript.
 This does mean that if you use traditional array syntax (`array()` vs `[]`), or if you have defined your options as an 
  array variable elsewhere, then the code above would not work without some modification.

![fixed behaviour](/assets/scriptBlock2.png)

Once you've entered any JavaScript code at all (e.g. just the keyword `var`), you can then use a separate editor pane
to write the JavaScript and not have to worry about escaping quotes and the like. With the cursor inside the JavaScript
string, press ALT+ENTER to bring up the context menu and select `Edit JavaScript fragment`.
 
![bringing up the editor](/assets/scriptBlockEdit.png)

This will bring up a separate pane below the editor where you can edit your JavaScript and make full use of PHPStorm's
JavaScript features, such as auto-completion and more.

I hope that this is useful to someone else.

![editor pane](/assets/scriptBlockFragmentEditor.png)