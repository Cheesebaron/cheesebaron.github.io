---
layout: post
title: Matching UIKeyboard speed and curve to your animation
date: '2014-01-21T13:55:00.000+01:00'
author: Tomasz Cielecki
tags:
- Xamarin.iOS
modified_time: '2014-01-22T12:50:06.573+01:00'
blogger_id: tag:blogger.com,1999:blog-3433282516380174051.post-5382291753974613584
blogger_orig_url: http://blog.ostebaronen.dk/2014/01/matching-uikeyboard-speed-and-curve-to.html
---

I have a ViewController in my Xamarin.iOS application, where i scale the View Frame when the keyboard shows.
However, I had some trouble with matching the speed and curve of the animation to the keyboards animation.
Though I found a pretty good solution, which takes the values from the keyboard itself, through the *UIKeyboardWillShowNotification.*

{% gist 8539458 %}
