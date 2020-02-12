+++ 
draft = false
date = 2020-02-12T20:37:41+01:00
title = "How I failed using Google Cloud Functions"
slug = "" 
tags = []
categories = []
thumbnail = "images/tn.png"
description = ""
+++

This post follows up the exercise described on my previous post about [Quarkus, Kotlin and GraalVM](/posts/quarkus-kotlin-and-graalvm).

I was proud for finishing the Snake game, it was working on Multiplayer, but there were two points I thought it would worth spending more efford, in order to make the gaming experience better: Interface and Scalability. So, I have chosen the Scalability to talk in more details in this post.

#Motivation

I was using VertX for in-memory PubSub. It is a powerful and versatile lib and it has simplified a lot my job, mainly because it provided me an uncomplicated way to use EventSource and Server Sent Events, so the interface was reactive. But this was not a reliable solution if I wanted to scale the game horizontaly (having multiple servers running the same app, load-balacing the requests among them).

