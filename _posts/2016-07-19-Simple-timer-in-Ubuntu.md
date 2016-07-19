---
layout: post
title: "Simple timer in Ubuntu Unity"
category: posts
---

From time to time I like to make a tea during my play with notebook but I don't want to constantly check clock whether time is up to pick out tea bag from the cup. I found out that there is no simple timer app in Ubuntu Unity.
Thanks to Linux we have `at` command which can schedule any action on system and `notify-send` command which can display notifications on desktop. By combination of both we have very elegant and easy to use timer without necessity to install anything.


## How to set timer to 10 minutes ahead? 

Run terminal (Ctrl-Alt-T) and type 

    at now + 10 minutes

you will get into `at` prompt and you can schedule notification in next 10 minutes

    notify-send "Pick out tea bag :)"

And now, just wait 10 minutes and you once time is up, you will see notification on your desktop.

