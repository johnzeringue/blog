---
title: "GraalVM + Gradle + Docker = Production Ready???"
date: 2018-06-04T20:11:50-04:00
---

[GraalVM][] is a new virtual machine from Oracle that supports several popular
programming languages including JavaScript, Python, Ruby, and Java. One exciting
feature of GraalVM is the ability to create native images from JVM binaries.
These native images promise improved size, speed, startup time, and operational
overhead.

Renato Athaydes first got me interested in GraalVM with
[his blog post on native images](https://sites.google.com/a/athaydes.com/renato-athaydes/posts/a7mbnative-imagejavaappthatrunsin30msandusesonly4mbofram).
I wanted to explore how you'd write a more typical web application using native images,
starting a new codebase from scratch.

In this blog post, I explore how to create a native image for a basic web server
written using [Spark][], a popular Java microframework. At the end, we'll
have a nicely scripted build that you *might* consider for serious projects.

I've chosen Gradle for this project because it's both flexible and my Java build
tool of choice. After installing Gradle on your own, create a new directory,
navigate to it from the command line, and run
`gradle init --type java-application`, which will generate a project skeleton
suitable for this example. Now you should be able to run `./gradlew run` and get
a nice greeting back.

Next we'll flesh out the web application. First, we'll need to add Spark to our
project dependencies. Edit the dependencies block in your `build.gradle` file to look
like this:
    {{< highlight groovy >}}
dependencies {
    compile 'com.sparkjava:spark-core:2.7.2'
    compile 'org.slf4j:slf4j-simple:1.7.13'
}{{< /highlight >}}
Then, update `src/main/java/App.java` to serve a greeting over HTTP instead:
    {{< highlight java >}}
import static spark.Spark.get;

public class App {
    public static void main(String[] args) {
        get("/hello", (req, res) -> "Hello World");
    }
}{{< /highlight >}}

Now comes my only hack in this post. GraalVM won't like that we don't have any
resources later on, so create an empty file called `dummy.txt` in
`src/main/resources`.

If you run `./gradlew run` now, you've got a web server. Awesome! Time to make
a native image out of it.

One cool implementation detail of the GraalVM `native-image` binary is that it
shares a similar interface to the trusty old `java` command. We'll take
advantage of this by using Gradle's `JavaExec` task type to configure tricky
stuff like classpath without too much effort. Add the following to your
`build.gradle`:
    {{< highlight java >}}
task nativeImage(type: JavaExec) {
    classpath = sourceSets.main.runtimeClasspath
    main = project.mainClassName
    executable = 'native-image'
    jvmArgs '-H:+ReportUnsupportedElementsAtRuntime'
}{{< /highlight >}}
This adds a new task that invokes the `native-image` binary using the same
classpath and main class that we've been using with `./gradlew run`.

It's straightforward enough except for the JVM argument at the end. This is
because GraalVM native images don't yet support Java SSL, which Spark includes.
We won't be using Spark's HTTPS functionality, so it's okay to push these
errors to runtime. This flag makes me somewhat uncomfortable about using this
build script in production, since we're keeping GraalVM from catching any *real*
mistakes you might make.

You might be excited to compile your first Java native image now, but we're not
there yet! Unless you've been experimenting with GraalVM already, you probably
don't have it installed on your computer. Also, if you're on a Mac like
me, the only available binary requires an Oracle Developer Network account.
Instead, we'll do everything inside of Docker.

Go ahead and download the GraalVM Linux binary from
https://github.com/oracle/graal/releases/download/vm-1.0.0-rc1/graalvm-ce-1.0.0-rc1-linux-amd64.tar.gz
and put it in the root of your project. Then, copy this Dockerfile:
    {{< highlight docker >}}
FROM ubuntu:18.04
RUN apt-get update && apt-get install -y gcc zlib1g-dev
ADD graalvm-ce-1.0.0-rc1-linux-amd64.tar.gz /
ENV PATH /graalvm-1.0.0-rc1/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
WORKDIR /graalvm-demo
COPY . /graalvm-demo
RUN ./gradlew nativeImage

FROM ubuntu:18.04
WORKDIR /graalvm-demo
COPY --from=0 /graalvm-demo/app .
EXPOSE 4567
CMD ./app{{< /highlight >}}

This `.dockerignore` might also be useful if you plan on experimenting on your
own later:

{{< highlight bash >}}
.gradle
Dockerfile
build{{< /highlight >}}

Again, not too many tricks. The first block in the Dockerfile builds the native
image using the Gradle task we set up earlier, and then the second block copies
it into a clean Docker image without the build tools. It's not especially fast,
so get comfortable and run `docker build -t graalvm-demo .`

Once it finishes, we're all done! On my machine, the native image weighed in at
16 MB and the whole Docker image is 95 MB uncompressed. By comparison, just the
`openjdk` image is currently 625 MB *without any* application code.

Run `docker run -p 4567:4567 graalvm-demo` to start the server, which runs without a
JVM. If you open http://localhost:4567/hello in a browser, you should see
"Hello World".

Conclusions
-----------

I was surprised how easily I was able to integrate GraalVM and Gradle. The
result is something you wouldn't be that crazy to start using for a work project.

The biggest downside to GraalVM native images right now is that
[they don't support reflection well](https://github.com/oracle/graal/blob/master/substratevm/REFLECTION.md).
However, I'm confident that support and tooling will improve, and 
eventually we'll even be able to use reflection-intensive web frameworks like
Jersey and Spring.

Native images could be a big feature for Java, putting it directly in
competition with Golang. Java has been so successful on the server that it's
exciting to imagine the possibilities for CLIs and smaller distributions.

Next Steps
----------

There are a couple of easy improvements to my approach if you really do want
to start using GraalVM native images right now.

First is to package the GraalVM distribution in a base Docker image so that you
don't have the 200 MB tarball in every project you start. You could also try
and get it to work with Alpine Linux instead of Ubuntu to shave another 50 MB
off the final Docker image size.

Next is to package the Gradle code into an easy-to-use plugin. Most of what I
wrote above could be automatically added for the user without any configuration.
`native-image` also has other flags that let you choose the name and location of the
output binary.

[GraalVM]: https://www.graalvm.org/
[Spark]: http://sparkjava.com/
