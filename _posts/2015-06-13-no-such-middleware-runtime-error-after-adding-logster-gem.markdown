---
layout: post
comments: true
title:  "No such middleware RuntimeError after adding logster gem"
date:   2015-06-13 20:25:10
categories: rails
---

[Logster](https://github.com/discourse/logster) is a web log viewer for Rack
applications.
By using it you can view your app logs easily in browser.

Today, I try to set it up for a rails api app so that the mobile app developers
can easily see things like if their request has been successfully received by
the server or not.

The [installation of Logster](https://github.com/discourse/logster#installation) is fairly simple.
And this is not the first time I have done it.
However unexpected things happened.

When I restarted the rails server after installing it, I got error
`actionpack-4.1.10/lib/action_dispatch/middleware/stack.rb:125:in 'assert_index': No such middleware to insert after: ActionDispatch::DebugExceptions (RuntimeError)`.

Then to see what's going wrong with the middlewares, I tried to run `rake
middleware`.
But it failed with the same error.
So I removed Logster and then `rake middleware` showed me the list of all
middlewares the app was using. And `ActionDispatch::DebugExceptions` was in the
list.

My first thought was that Logster tried to insert its middleware after
`ActionDispatch::DebugExceptions`. But for some reason it couldn't find it.

I had no clue what's going wrong. So I created an new rails app and installed
Logster to see if it works or not. Unsurprisingly the rails server started
successfully and it worked well.

Then when I ran `rake middleware` in the new test app, I noticed
that `ActionDispatch::DebugExceptions` wasn't in the list any more.
Instead, `Logster::Middleware::DebugExceptions` appeared at the place where
`ActionDispatch::DebugExceptions` was supposed to be.

!!! The problem wasn't Logster, but a gem after Logster.

So I went back to see the real app's middleware list. The one next to
`ActionDispatch::DebugExceptions` was `BetterErrors::Middleware`.
And in the Gemfile, `gem 'better_errors'` was behind `gem 'logster'`.

Finally, moving `better_errors` ahead of `logster` solved the problem.
