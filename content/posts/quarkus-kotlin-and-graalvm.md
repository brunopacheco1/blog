---
title: "Quarkus, Kotlin and GraalVM"
date: 2019-10-29T22:05:30+01:00
draft: false
---

After a chat with [László](https://github.com/nerg4l), where I've asked some idea to practice Quarkus and GraalVM, he suggested me to code the classic game Snake (AKA Worms by me :P).

### Motivations

Why [Quarkus](https://quarkus.io)? Well, I've read an article saying that Quarkus is a great promising solution for Java in the Cloud, as it reduces drastically the application's footprint and it has an insanely fast startup, when running on native code, and here is where GraalVM enters.

[GraalVM](https://www.graalvm.org/) is a general purpose VM for running applications written in a bunch of different languages, allowing also polyglot code. At the first glance, I felt it was an amazing thing to try on, and so I did, but at the end it wasn't fruitful duet to some strange bug on a node app I was coding in that time, so I just gave up.

[Kotlin](https://kotlinlang.org/) has no reasonable arguments to be here then it is a brand new Java dialect, which I was already learning, but I have never used it on something substantial, so why not?

### Starting the adventure

I've started the project just reading the guides on Quarkus.io, which in the beginning looked quite hands-on, and as any recent framework for web development, it was really simple to code, in five minutes I had a rest service running locally, but of course a http call returning a Hello World string doesn't deserve to be called a service.

First steps, new domain and DTO classes, JPA, H2 and a couple of CRUD endpoints. Now something to start with, isn't it? Yes, and I decided to migrate the code to Kotlin... Bugs were popping up!

First problem, JPA requires open classes and by default it's not true in Kotlin. To fix it, Kotlin compiler has a plugin supporting JPA, just a flag.

Second problem, Quarkus by default uses JSON-B for serialization, but when moving the code to Kotlin, it started failing and some workarounds were required for the DTOs. At the end I decided to try Jackson and it worked nice, not bugs here and the DTOs were cleaner then before. Latter on, I had also to add a no-arg plugin option, a custom annotation in the DTOs in order to remove the extra constructor method, and all properties have to be var.

Third problem, similarly to the JPA entities, all managed beans have to be open, so one more flag to the Kotlin compiler to understand things correctly.

Fourth problem, I created an abstract Repository class, to simulate the spring repositories and have a common behavior for all entities repos, but here again some strange problem with the dependency injection in the superclass. The work around was the following:

```
    @PersistenceContext
    protected lateinit var em: EntityManager
``` 

A bit odd, because I was trying to inject it via constructor, but for some reason it wasn't working, so it is as it is until now.

Fifth problem, for all @OneToMany I had to change from List/Set to MutableList/MutableSet.

Sixth problem, I don't remember why, but now all fields in the domain and DTO classes are var, instead of val. I have to check why again.

### Conclusions

About Kotlin, I do reconize the benefits of using Kotlin such as nullability, immutability, less verbose code, tons of new coder-friendly functions, however they are not that important when you have a clean code and efficient testing.

**See you | Bis dann | Até mais | À bientôt**
