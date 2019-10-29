---
title: "Quarkus, Kotlin and GraalVM"
date: 2019-10-29T22:05:30+01:00
draft: false
---

After a chat with [László](https://github.com/nerg4l), where I was asking some ideas to practice Quarkus and GraalVM, he suggested me to code the classic game Snake (AKA Worms by me :P).

First, why [Quarkus](https://quarkus.io)? I've read an article somewhere and it was said that Quarkus is a great promising solution for the Java in the Cloud, as it reduces drastically the application's footprint and it has an insanelly fast startup when running on native code, and here is where GraalVM enters.

[GraalVM](https://www.graalvm.org/) is a general porpouse VM for running applications written in a bunch of different languages, allowing polyglot code. The first time I've read about it was a year ago(?) and I felt that was a amazing thing to try on but still immature. I did some test back then anyway, but not that fruitfull on a node app I was coding, so I just gave up.

[Kotlin](https://kotlinlang.org/) has no reasonable arguments to be here then that it is a brand new Java dialect, which I was already learning, but I have never used it on something substantial as this game was planed to be. I do reconize the benefits of using Kotlin such as nullability, imutability, less verbose code, tons of new coder-friendly functions, however they are not that important when you have a clean code and efficient testing.


