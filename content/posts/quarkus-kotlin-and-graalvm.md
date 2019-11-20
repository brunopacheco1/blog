---
title: "Quarkus, Kotlin and GraalVM"
date: 2019-11-18T22:05:30+01:00
draft: false
---

During a chat with [László](https://github.com/nerg4l), I've asked him for some ideas to practice Quarkus and GraalVM, he suggested to code the classic game Snake (AKA Worms by me :P).

### Motivations

Why [Quarkus](https://quarkus.io)? I've read an article saying that Quarkus is a great and promising solution for Java in the Cloud, as it drastically reduces the application's footprint and has an insanely fast startup when running on native code.

[GraalVM](https://www.graalvm.org/) is a general-purpose VM for running applications written in a bunch of different languages, allowing a polyglot code. At first glance, I felt it was an amazing thing to try, and so I did. In the end, this first trial wasn't fruitful, due to some strange bug on a node app I was coding at that time, so I just gave up for that moment. 

[Kotlin](https://kotlinlang.org/) has no reasonable arguments to be here except it is a brand-new Java dialect which I was already learning, but I have never used it on something substantial, so why not?

### Starting the adventure

I've started the project just reading the guides on Quarkus.io, which are quite hands-on, but superficial. Quarkus is simple to use, as any recent framework for web development. So simple that in five minutes I had a rest service running locally... of course, an HTTP call returning a "Hello World" string doesn't deserve to be called a service, does it?

So, to start your project from scratch, they have implemented a bootstrap command. Please refer to the followin command:

```
mvn io.quarkus:quarkus-maven-plugin:1.0.0.CR1:create \
    -DprojectGroupId=org.acme \
    -DprojectArtifactId=getting-started \
    -DclassName="org.acme.quickstart.GreetingResource" \
    -Dpath="/hello"
```

And an easy way to add new dependencies:

```
mvn quarkus:add-extension -Dextensions="vertx"
```

Moving on, my first steps were: defining the domain, JPA, H2, a couple of CRUD endpoints and some DTOs. After having this square-shaped code, I decided to migrate it to Kotlin, and bugs started popping up as expected.

The first problem which I had to face was JPA. It requires open classes and by default it isn't true for Kotlin. To fix this, there's a flag to enable JPA support during compilation. I don't like this kind of solution, but this is somewhat my fault as I wanted to use a mature Java library instead of using something more Kotlin-like.

The second problem was the JSON payload serialization. I had some issues with JSON-B (the default option), and then I decided to try Jackson. Again, just for the sake of simplicity, as I was already testing tons of new things.

It worked nicely and the DTOs were cleaner than before. However, another plugin had to be added to the Kotlin compiler (the no-arg plugin option) and a custom annotation to help the compiler identifying the DTOs.

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

The third problem was similar to the JPA issue. All managed beans have to be open. Can you guess the solution? Yes another Kotlin compiler flag.

So, in the end, my Koltin compiler configuration looked like the following:

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

I've also created an abstract Repository class to have a common behavior for all entity repos (copying Spring), but I had a problem with the dependency injection in the superclass.

I tried using constructor dependency injection, passing it from the child to the superclass using super, but it hasn't worked properly. The work around was injecting the `EntityManager` directly in the field, like this:

```
    @PersistenceContext
    protected lateinit var em: EntityManager
```

Then, I had to change all `@OnToMany` from `List`/`Set` to `MutableList`/`MutableSet`.

And finally, I don't remember why, but now all fields in the domain and DTO classes were `var`, instead of `val`. I have to check why again.

I've also implemented some business code, which is not important to discuss here (but if you are interested, go to
[this link](https://github.com/brunopacheco1/worms/tree/master/src/main/kotlin/com/dev/bruno/worms/evaluation)).

### And GraalVM enters the scene...

After trespassing the first barrier, I was ready for the next boss (the final one?): compiling the project to native code.

According to Quarkus.io, this shouldn't be complicated as they aim to simplify native code compilation using their on maven plugin. Even their guides being simplistic, they are not lying. But they could have mentioned the Reflection limitation in GraalVM, which will be explained in the following lines.

So, to build the app, you have to run a similar mvn command:

```
mvn clean package -Pnative -Dnative-image.docker-build=true
```

By the way, I was building a containerized native code, that's why there's this docker-build arg. They also provide a dockerfile sample.

I thought at this stage that would be enough, but no. After running the build and running the app on Docker, I faced exceptions on Jackson. So, reflection is not supported by default on native code and it took me a while to understand why, and a bit more to find a way to fix it.

On a side note, the build consumes a lot of memory and CPU, almost killing my old Lenovo. :(

So, GraalVM does allow you to use reflection, but for that, you need to add a config file, located at `META-INF/native-image/reflect-config.json` (it can be in a different folder with a different name, but this is the default path).

First I decided not to manually write this file, and after googling I've found this [blog](https://link.medium.com/n2us7AWxF1) presenting an agent to do this job. The idea is to observe the JVM during runtime and generate the config.

To do this, you simply have to run the app with the following command:

```
$JAVA_HOME/bin/java -agentlib:native-image-agent=config-output-dir=META-INF/native-image your.jar
```

After running the app with this agent, the generated file was hugely polluted and it didn't fix the problem anyway, as the agent didn't add all needed classes. So, I ended up typing the file manually. And surprisingly, I only had to add the DTO classes.

The file should look something like this:

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

I do recognize the benefits of using Kotlin such as nullability, immutability, less verbose code, tons of new coder-friendly functions, however they are not that important when you have clean code and efficient testing.

Also, I don't like polluted POM files, and it annoyed me having to add build plugins, compiler flags and configurations. I felt guilty about using JPA and Jackson though. Perhaps, if I had used something more Kotlin oriented, I wouldn't have had these issues. Let's see what happens in the next project.

Quarkus has an insanely fast startup indeed, and the memory footprint is great. I have used a micro instance on Google Cloud to test the game, and it handled properly the load, without crashing anything. Quarkus is a bit immature yet, as it required some refactoring when I decided to move to newer versions. Nonetheless, it is promising, as it has huge support for different technologies and libraries. I'll use it again.

I have loved GraalVM at first sight, as the wish of a polyglot code is in my mind for a while, and now it's becoming reality. In the end, I didn't use this feature, however, the native code generation fascinated me as well. I didn't like the way it consumes resources though. Compiling any other language code to native is not consuming 10% of what GraalVM has consumed from my machine (not judging...). I would wait for improvements on this area before using it on commercial solutions.

To sum up: I have enjoyed playing with all these new technologies all together. Due to time and availability constraints, I couldn't investigate more. Perhaps improving and refactoring the code. The final solution is not bad and it has proven to me that Java definitely can be used in the cloud, it's just a matter of investment in new solutions, like Quarkus and GraalVM.

Please, refer to the project [repository](https://github.com/brunopacheco1/worms) to have a better picture of what I have coded. And feel free to comment out there.

**See you | Bis geschwënn | Até mais | À bientôt**
