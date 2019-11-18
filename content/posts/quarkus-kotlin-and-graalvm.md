---
title: "Quarkus, Kotlin and GraalVM"
date: 2019-11-18T22:05:30+01:00
draft: false
---

After a chat with [László](https://github.com/nerg4l), where I've asked some idea to practice Quarkus and GraalVM, he suggested me to code the classic game Snake (AKA Worms by me :P).

### Motivations

Why [Quarkus](https://quarkus.io)? I’ve read an article saying that Quarkus is a great promising solution for Java in the Cloud, as it reduces drastically the application’s footprint and it has an insanely fast startup when running on native code, and here is where GraalVM enters.

[GraalVM](https://www.graalvm.org/) is a general-purpose VM for running applications written in a bunch of different languages, allowing also polyglot code. At the first glance, I felt it was an amazing thing to try on, and so I did, but in the end, it wasn’t fruitful duet to some strange bug on a node app I was coding at that time, so I just gave up for that moment.

[Kotlin](https://kotlinlang.org/) has no reasonable arguments to be here then it is a brand new Java dialect, which I was already learning, but I have never used it on something substantial, so why not?

### Starting the adventure

I’ve started the project just reading the guides on Quarkus.io, which are quite hands-on, but very superficial. Quarkus, as any recent framework for web development, is simple to code, so simple that in five minutes I had a rest service running locally… of course, an HTTP call returning a Hello World string doesn’t deserve to be called a service, does it?

To start, they have implemented a bootstrap command, like the following copied from Quarkus.io.

```
mvn io.quarkus:quarkus-maven-plugin:1.0.0.CR1:create \
    -DprojectGroupId=org.acme \
    -DprojectArtifactId=getting-started \
    -DclassName="org.acme.quickstart.GreetingResource" \
    -Dpath="/hello"
```

A sample on how to add new dependencies:

```
mvn quarkus:add-extension -Dextensions="vertx"
```

Moving on, my first steps were defining the domain, JPA, H2, a couple of CRUD endpoints and some DTOs. After having this squared shape code, I decided to migrate the code to Kotlin, and bugs started popping up as expected.

The first problem which I have to face was JPA. It requires open classes and by default that it’s not true on Kotlin. To fix it, there’s a flag to enable JPA support during compilation. I don’t like this kind of solution, but this is kind of my fault as I wanted to use a mature Java library instead of using something more Kotlin-like.

The second problem was the JSON payload serialization. I had some issues with the default Quarkus dependency, JSON-B, and then I decided to try Jackson, again for the sake of simplicity, as I was already testing tons of new things. It worked nice, the DTOs were cleaner than before. For that, another plugin had to be added to the Kotlin compiler, the no-arg plugin option, and a custom annotation to identify the DTOs.

```
@Dto
data class MatchMapPlayer(
        var playerId: Long,
        var status: MatchPlayerStatus = MatchPlayerStatus.PLAYING,
        var wormLength: Int = 0,
        var direction: Direction = Direction.DOWN,
        var position: List<MapPoint> = arrayListOf()
)
```

The third problem was similar to the JPA issue. All managed beans have to be open, so one more flag to the Kotlin compiler to understand things correctly. Can you guess the solution? Yes, another kotlin compiler plugin.

So, in the end, my Koltin compiler configuration looked like this:

```
                <configuration>
                    <compilerPlugins>
                        <plugin>jpa</plugin>
                        <plugin>no-arg</plugin>
                        <plugin>all-open</plugin>
                    </compilerPlugins>
                    <pluginOptions>
                        <option>all-open:annotation=javax.ws.rs.Path</option>
                        <option>all-open:annotation=javax.enterprise.context.ApplicationScoped</option>
                        <option>all-open:annotation=javax.enterprise.context.RequestScoped</option>
                        <option>all-open:annotation=javax.ws.rs.ext.Provider</option>
                        <option>all-open:annotation=javax.persistence.MappedSuperclass</option>
                        <option>all-open:annotation=javax.persistence.Entity</option>
                        <option>all-open:annotation=javax.persistence.Embeddable</option>
                        <option>no-arg:annotation=com.dev.bruno.worms.dto.Dto</option>
                    </pluginOptions>
                </configuration>
```

I have also created an abstract Repository class to have a common behavior for all entities repos (copying Spring), but here I had a problem with the dependency injection in the superclass. I tried using contructor dependency injection, passing it from the child to the superclass using super, but nothing has worked properly. The work around was injecting the EntityManager directly in the field, like this:

```
    @PersistenceContext
    protected lateinit var em: EntityManager
```

Then, I had to change all @OnToMany from List/Set to MutableList/MutableSet.

And finally, I don’t remember why, but now all fields in the domain and DTO classes are var, instead of val. I have to check why again.

I’ve also implemented some business code, which is not important to discuss here (but if you are interested, go to this [link](https://github.com/brunopacheco1/worms/tree/master/src/main/kotlin/com/dev/bruno/worms/evaluation)).

### And GraalVM enters in the scene...

After trespassing the first barrier, I was ready for the next boss (the final one?). And according to Quarkus.io, it shouldn’t be complicated. They aim to simplify native code compilation using their on maven plugin, and they are not lying, even being simplists in the guides, not mentioning the Reflection limitation, to be explained in the following lines.

To build the app, you have to run a similar mvn command:

```
mvn clean package -Pnative -Dnative-image.docker-build=true
```

By the way, I was building a containerized native code, that’s why there’s this docker-build arg (they build an optimized app for docker). They also provide a dockerfile sample, to follow.

I thought at this stage that would be enough, but no. After running the build and running the app, I faced an error on Jackson, as reflection is not supported by default on native code. It took me a while to understand why, and a bit more to find a way to fix it.

By the way, the build has consumed a lot of memory and CPU, almost killing my old Lenovo. :(

So, GraalVM does allow you to use reflection, but for that, you have to add a config file, located at “META-INF/native-image/reflect-config.json” (it can be in a different folder with a different name, but this is the default path).

First I decided not writing manually this file, and I've found this [blog](https://medium.com/graalvm/introducing-the-tracing-agent-simplifying-graalvm-native-image-configuration-c3b56c486271) presenting an agent to do this job. Its idea is to observe the JVM during runtime and write the config out of the box for you.

To do this, you simply have to run the app with the following command:

```
$JAVA_HOME/bin/java -agentlib:native-image-agent=config-output-dir=META-INF/native-image your.jar
```

So, after running the app with this agent, the generated file was hugely polluted and it didn’t fix the problem anyway, as the agent didn’t add all needed classes. So, I ended up typing the file manually. And surprisingly, I had to add only the DTO classes.

The file has to be something like this:

```
[
  {
    "name": "com.dev.bruno.worms.dto.ExceptionResponse",
    "allDeclaredConstructors": true,
    "allPublicConstructors": true,
    "allDeclaredMethods": true,
    "allPublicMethods": true,
    "allDeclaredClasses": true,
    "allPublicClasses": true,
    "fields": [
      {
        "name": "message",
        "allowWrite": true
      }
    ]
  }
  ...
]
```

### Conclusions

I do recognize the benefits of using Kotlin such as nullability, immutability, less verbose code, tons of new coder-friendly functions, however they are not that important when you have clean code and efficient testing. Also, I don’t like polluted POM files and having to add build plugins and plugin plugins annoyed me. I felt guilty about using JPA and Jackson though, perhaps if I had used something more Kotlin oriented, I wouldn’t have had these issues. Let’s see it again in the next project.

Quarkus has an insanely fast startup indeed, and the memory footprint is great, I have used a micro instance on Google Cloud to test the game I handled properly, without crashing anything. It’s a bit immature yet, as it required some refactoring when I decided to move to newer versions. It is promising anyway, as it has huge support for different technologies and libraries. I’ll use it again.

And I have loved GraalVM at first sight, as the wish of a polyglot code is in my mind for a while and now it’s becoming true. In the end, I didn’t use this feature, however, the native code generation fascinated me as well, and it’s quite nice. I didn’t like to way how it consumes resources though, compiling any other language code is not consuming 10% of what GraalVM has consumed from my machine. I would wait for improvements to benefit from this feature.

To sum up, I have enjoyed playing with all these new technologies at the same time. Due to time and availability constraints, I couldn’t investigate more, perhaps improving and refactoring the code, but the final solution is not bad and it proved to myself that Java definitely can be used in the cloud, it’s just a matter of investment in new solutions, like Quarkus and GraalVM.

Please, refer to the project [repository](https://github.com/brunopacheco1/worms) to have a better picture of what I have coded. And feel free to comment out there.

**See you | Bis geschwënn | Até mais | À bientôt**
