---
title: "Java Web Server in a 20 MB Docker Image"
date: 2018-06-04T20:11:50-04:00
---

[GraalVM][] is a new virtual machine from Oracle with a lot of cool features. One
of the most exciting is the ability to create native Java binaries.
These native binaries promise improved speed, size, startup time, and operational
overhead compared to a typical JVM runtime.

In this blog post, I show how to create a tiny Docker image with a simple web server
by using native Java binaries. At the end, we'll have a nicely scripted build that
you *might* consider using for serious projects.

After installing Gradle on your own, create a new directory, navigate to it from
the command line, and run `gradle init --type java-application`, which will
generate a project skeleton suitable for this example. Now you can
run the application with `./gradlew run` and get a nice greeting back.

Next, we'll flesh out the web server. I chose [Spark][], so we need to add it to our
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

Now comes a quick hack. GraalVM won't like that we don't have any
resources later on
([something I'd like to change](https://github.com/oracle/graal/issues/456)),
so create an empty file called `dummy.txt` in `src/main/resources`. In a more
substantial project, you'd likely have some resources anyway and wouldn't need
this file.

If you run `./gradlew run` now, you've got a web server. Awesome! Time to make
it native.

One cool implementation detail of the GraalVM `native-image` binary is that it
has a similar interface to the trusty old `java` command. We'll take
advantage of this by using Gradle's `JavaExec` task type to configure tricky
stuff like the classpath without too much effort. Add the following to your
`build.gradle`:
    {{< highlight java >}}
task nativeImage(type: JavaExec) {
    classpath = sourceSets.main.runtimeClasspath
    main = project.mainClassName
    executable = 'native-image'
    jvmArgs '--static', '-H:+ReportUnsupportedElementsAtRuntime'
}{{< /highlight >}}
This adds a new task that invokes the `native-image` binary using the same
classpath and main class that we've been using with `./gradlew run`.

It's straightforward enough except for the JVM arguments at the end. `--static`
tells GraalVM to make a static binary including `glibc` and `zlib`.
`-H:+ReportUnsupportedElementsAtRuntime` is a workaround for [issue #390][].
Unfortunately, this flag shifts a whole class of build errors to runtime,
but it's unavoidable for now.

You might be excited to compile your first native Java binary, but we're not
there yet! Unless you've been experimenting with GraalVM already, you probably
don't have it installed on your computer. Also, if you're on a Mac like
me, the only available binary requires an Oracle Developer Network account.
Instead, we'll do everything inside of Docker.

Go ahead and download the GraalVM Linux binary from
https://github.com/oracle/graal/releases/download/vm-1.0.0-rc2/graalvm-ce-1.0.0-rc2-linux-amd64.tar.gz
and put it in the root of your project. Then, copy this Dockerfile:
    {{< highlight docker >}}
FROM ubuntu:18.04
RUN apt-get update && apt-get install -y gcc zlib1g-dev
ADD graalvm-ce-1.0.0-rc2-linux-amd64.tar.gz /
ENV PATH /graalvm-ce-1.0.0-rc2/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
WORKDIR /graalvm-demo
COPY . /graalvm-demo
RUN ./gradlew nativeImage

FROM alpine
WORKDIR /graalvm-demo
COPY --from=0 /graalvm-demo/app .
EXPOSE 4567
CMD ./app{{< /highlight >}}

Again, not too many tricks. The first block in the Dockerfile builds a native
binary using the Gradle task we set up earlier, and the second block copies
it into a clean Docker image without the build tools. It's not especially fast,
so get comfortable and run `docker build -t graalvm-demo .`

Once it finishes, we're all done! On my machine, the native binary weighed in at
14 MB, and the whole Docker image is just 19 MB uncompressed. By comparison, the
`openjdk` image is 450 MB *without any* application code.

Run `docker run -p 4567:4567 graalvm-demo` to start the server. If you open
http://localhost:4567/hello in a browser, you should see "Hello World".

Takeaways
---------

I'm surprised how easy it is to integrate GraalVM and Gradle. As long as you're
only targeting Docker/Linux, this is something Java developers should already
consider using for projects.

The biggest downside to the native binaries right now is that
[they don't support reflection well](https://github.com/oracle/graal/blob/master/substratevm/REFLECTION.md).
However, I'm confident that support and tooling will improve, and that
eventually we'll even be able to use reflection-intensive web frameworks like
Jersey and Spring.

Native binaries could be a huge feature for Java, putting it in direct
competition with Golang. Java has been so successful on the server that it's
exciting to imagine the possibilities for CLIs and smaller distributions.

[GraalVM]: https://www.graalvm.org/
[Spark]: http://sparkjava.com/
[issue #390]: https://github.com/oracle/graal/issues/390
