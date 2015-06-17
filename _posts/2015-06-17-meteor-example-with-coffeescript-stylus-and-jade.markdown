---
layout: post
title: 'Meteor example with CoffeeScript, Stylus and Jade'
date: '2015-06-17 13:39'
commentIssueId: 3
---

{::options parse_block_html="true" /}
<div class="toc">
Contents

* toc
{:toc}
</div>

## General

I have been checking out [Meteor](https://www.meteor.com) periodically since about version 0.5. It was officially bumped to version 1.0 a while a go and is now at 1.1.0.2, so I decided that now it's a good time to see how mature it is.

While playing around with it in the past I've usually used [CoffeeScript](http://coffeescript.org) which is easily done as Meteor has a good package management system and CoffeeScript was a native package. Same goes with [Stylus](https://learnboost.github.io/stylus/). Changing the HTML templating engine used to be harder, but now packages can also be used for that and I like the [Jade](http://jade-lang.com) syntax, which is very concise, better than SpaceBars syntax that is the default for Meteor.

When creating a new app with Meteor, it generates a few basic files which form an example application that is immediately runnable. The application shows a page with a single button and a counter that is increased when the button is pressed. I converted the application to use CoffeeScript for the events and Jade for the HTML templating and added Stylus support.

## Creating the application

After installing Meteor (instructions can be found [here](https://eee.meteor.com)) creating a new application is done with

{% highlight bash %}
meteor create meteortest
{% endhighlight %}

In that folder are the following files

- meteortest.css
- meteortest.html
- meteortest.js

You can start the example application with running the command `meteor` in that folder. The application can be checked at http://localhost:3000.

## Adding support for CoffeeScript, Jade and Stylus

Using the Meteor package manager is simple and adding the support for the packages is as simple as running

{% highlight bash %}
meteor add coffeescript stylus mquandalle:jade
{% endhighlight %}

in that folder. The packages that are used can be checked with `meteor list`.

## Taking Jade into use

The original HTML in _meteortest.html_ looks like

{% highlight html %}
<head>
  <title>meteortest</title>
</head>

<body>
  <h1>Welcome to Meteor!</h1>
  {% raw %}{{> hello}}{% endraw %}
  </body>

<template name="hello">
  <button>Click Me</button>
  <p>You've pressed the button {{counter}} times.</p>
</template>
{% endhighlight %}

When that is changed to Jade markup the file _meteortest.jade_ becomes

{% highlight jade %}
head
  title meteor-test

body
  h1 Welcome to meteor!
  +hello

template(name=hello)
  button Click me!
  p You've pressed the button #{counter} times.
{% endhighlight %}

### Unwrapped templates as own files

Breaking templates into their own files may make sense and the Jade templating engine used with Meteor supports it. Then the file for template named hello would be _hello.tpl.jade_ and it would contain just

{% highlight jade %}
button Click me!
p You've pressed the button #{counter} times.
{% endhighlight %}

Then the file _meteortest.jade_ would look like

{% highlight jade %}
head
  title meteor-test

body
  h1 Welcome to meteor!
  +hello
{% endhighlight %}

## Taking CoffeeScript into use

The original _meteortest.js_ looks like

{% highlight javascript %}
if (Meteor.isClient) {
  // counter starts at 0
  Session.setDefault('counter', 0);

  Template.hello.helpers({
    counter: function () {
      return Session.get('counter');
    }
  });

  Template.hello.events({
    'click button': function () {
      // increment the counter when button is clicked
      Session.set('counter', Session.get('counter') + 1);
    }
  });
}

if (Meteor.isServer) {
  Meteor.startup(function () {
    // code to run on server at startup
  });
}
{% endhighlight %}

When that it converted to CoffeeScript, the file _meteortest.coffee_ becomes

{% highlight coffeescript %}
if Meteor.isClient
  Session.setDefault 'counter', 0

  Template.hello.helpers
    counter: -> Session.get 'counter'

  Template.hello.events
    'click button': -> Session.set 'counter', Session.get('counter') + 1

if Meteor.isServer
  console.log 'This code runs on server'
{% endhighlight %}

## Taking Stylus into use

Using Stylus is extremely simple. The original _meteortest.css_ file is replaced with _meteortest.styl_ which can for example be

{% highlight css %}
h1
  color: green
{% endhighlight %}

Try changing the color to for example red while you have the application open in a browser and see how quickly the changes are autoloaded to the browser without breaking the session.

## Running the application

After deleting the now redundant _meteortest.css_, _meteortest.js_ and _meteortest.html_ there are four files in the application

- meteortest.jade
- hello.tpl.jade
- meteortest.styl
- meteortest.coffee

The application can be run with just the command `meteor` and by going to the address http://localhost:3000 and it should work exactly as the original example application.

### Cloning from GitHub

The example can also be found from [GitHub](https://github.com/SirIle/meteortest) and can be cloned with

{% highlight bash %}
git clone https://github.com/SirIle/meteortest.git
{% endhighlight %}

After that running it with the command `meteor` in the application folder fetches all the required packages and the application can be tested by going to http://localhost:3000.
