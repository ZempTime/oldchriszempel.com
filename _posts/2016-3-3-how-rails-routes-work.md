---
layout: post
title:  "How Rails Routes Work, Simplified"
date:   2016-3-3 12:22:25
---

One of the most confusing things for people new to Rails are routes. What makes them so confusing?

1. Understanding a route means you must also understand a variety of different parts of Rails all at the same time.
2. It's not clear what a route produces.
3. Differentiating between what's Ruby and convention (Rails-introduced behavior)

When you're new to Rails, routes will likely feel pretty confusing at the start, really hard to *get*, to *understand*. Well, its programming. It **is** hard. But you can do it. There's no magic involved.

## Parts of a Route

We've got an <a href="https://github.com/ZempTime/lemons/blob/master/config/routes.rb" target="_blank">app all about lemons</a>, where in our routes file we simply have `resources :lemons`, and you can see it in action <a href="http://lemonsnotavailable.herokuapp.com/" target="_blank">here</a>, unless you get to the end of this terrible run-on sentence first.

When you run `rake routes`, you'll get something like this:

<code>
Prefix Verb&nbsp;&nbsp;&nbsp;URI Pattern&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Controller#Action
lemons&nbsp;GET&nbsp;&nbsp;&nbsp;&nbsp;/lemons(.:format)&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;lemons#index
</code>

This information is super useful when you've wrapped your head around routes, which is why its displayed. But if you haven't, where do you start? How should you think about it?

The best way to understand something is to grapple with the problem it was introduced to solve. The problem routes were introduced to solve is this: how do you get from one part of your app to another?

![Routes 1](/assets/routes-1.png)

You can think of a route like a strange device that has three keyholes in it. For some reason, in my head, I always imagine it as a potato. Anyway, when you push the right key into one of the holes, keys emerge from the other two key holes that unlock the other areas you're going to want to go.

Honestly, there are only like 3 paths a route really helps with that you gotta wrap your head around.

## 1. When a Request Comes In

So when someone types in `localhost:3000/lemons/1/`, we need to get from this url to the right place in the app to render the page that person wanted to go to. And that's exactly what the route helps with.

![Routes 2](/assets/routes-2.png)

So when `localhost:3000/lemons/1/` goes in from the browser, `lemons#show` comes out for the app. Now here's where there are some Rails conventions kick in.

We want to render the `show` page for that particular lemon, and let the user see the information they requested. What happens next is Rails takes the `controller#action` part of that route, and does this with it:

Rails takes `lemons`, capitalizes it, and adds `Controller` to the end. Sound familiar? Now we have `LemonsController`!

But we still want to render back an html page, and we're not there yet. So Rails does `LemonsController.new`, then calls the `show` action on it. So like this:

`@controller = LemonsController.new`

`@controller.show`

Then your "show action" does a couple things. It runs whatever logic you put in there, like:

`@lemon = Lemon.find(params[:id])`

Then it calls `render show.html` implicitly...except, which show? `lemons/show.html` of course!

So it will go grab `lemons/show.html.erb` from your `/views` directory. But wait - `.erb`? The `.erb` stands for "embedded ruby," and so any instance variables that you created in your controller action - anything with an `@` in front of it, in this case, `@lemon` - your view can dip its hand into the controller and grab those out.

So `<%= @lemon.name %>` will simply put whatever results out of that into the response you're sending back, exactly where you put it in your `show.html.erb` file.

What you're left with is a bunch of html, which is perfect! That's what gets sent back to the user's browser. There's a little bit of Ruby and a lot of Rails conventions happening here, but this is how a route helps you get from a browser url to the page you want to go.

But what happens if we want to let a user click on a link to add a new lemon? (damn lemons)

## 2. Creating a Link

In this case, we're going to be starting in Ruby, and we're gonna want the ultimate result: a url our user can click on.

![Routes 3](/assets/routes-3.png)

In our view, we'll have: `<%= link_to "New Lemon", new_lemon_path %>`

The key thing here is `_path`. When Rails sees this, it will automatically know that you're talking about a route, and that you want the url aspect of it meant for the browser. And since no two routes can have the same path and verb, all its gotta do is find the match.

Our Ruby up there will evaluate to:

`<a href="/lemons/new">New Lemon</a>`

In fact, Rails conventions are so strong, that if you wanted to link to a specific lemon, you could:

`<%= link_to @lemon.name, lemon_path(@lemon) %>`

Rails will see the route is `lemons/:id`, note the `:id` part in the route, then go ahead and call `@lemon.id` for you and put it in there.

In fact, you could do this:

`<%= link_to @lemon.name, @lemon %>`

And Rails would see that you're talking about a singular lemon, and just go ahead and assume you wanted to show it, and put the `lemon_path` url `/lemons/2` in there for you.

## 3. The Verb Part, aka Creating a Lemon

Aka giving some lemons back to life, instead of the other way around.

There are only a couple places you can store information in every request:

1. The url
2. The HTTP request body
3. In a cookie (aka on the user's machine)

Thing is, you can only store 2,083 characters in a url. And unless you're Amazon, no way would you want your urls to be anywhere near that long. Amazon's are really long, ugly, and not immediately understandable. You want yours to be immediately understandable.

The reason there's a `verb` part of the route is to help the app put the incoming request in the right place.

`new_lemon_path` -> `lemons#new` page.

But when we go ahead and [fill out the form](http://lemonsnotavailable.herokuapp.com/lemons/new) and hit submit, the description of our lemon could far exceed 2,083 characters, especially if we're talking about something more substantial and with many more fields than a simple lemon. So we've got to send that data over somewhere else, like the body of our request. We have to POST it.

![Routes 4](/assets/routes-4.png)

You'll see, even though the url portion is the same, the verb is different. A POST request means there's info in the body (the information about our new lemon), so we need to send it to a different place, meaning `lemons#create`.

Except I'm not going to go into how the params work. Not when there's a much better explanation readily available.

## Best Next Resources

The next best things to watch are this:

<a href="https://gorails.com/episodes/the-params-hash" target="_blank">The Params Hash</a>, which goes in depth on how information gets from the http request to your controller.

Now, I've talked a lot about Rails conventions here assuming you know what they are. What I didn't do was talk about the problem they were introduced to solve. For more information on that, <a href="https://player.vimeo.com/video/15772341" target="_blank">Tour of the Development Process at 37Signals</a> is a video from the creators of Rails themselves explaining it.

This will be a somewhat permanent WIP because I'm pretty sure things can always be explained better. So let me know what parts really worked, or which ones weren't clear at [@ZempTime](https://twitter.com/zemptime), especially if there are any good resources to add to the end!
