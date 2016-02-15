---
layout: post
title:  "Creating Unix Services and RPM/DEB Packages with Spring Boot and Gradle"
date:   2016-02-15 18:47:00
author: Chris Campo
categories: Java Spring Spring-Boot Unix Linux
---

I work with [Spring Boot][boothome] daily, and it's probably one of my favorite frameworks ever. The amount of things
it does that just make every-day JVM-dev life easier is absurd. Spring Boot has probably saved me hundreds of hours
in headaches.

To tack on to the already enormous list of quality-of-life features, Spring Boot version 1.3.0 released the extremely
cool ability to run jar files as `systemd` or `init.d` services (daemons). All you need to do is enable a switch in your
build file (`executable = true`), and then build as normally. You can then run your `jar` file straight from the
command line, like so:

    $ ./myapp.jar

To run as a daemon, you just make a symlink to `/etc/init.d`, e.g.

    $ sudo ln -s /var/myapp/myapp.jar /etc/init.d/myapp
    $ service myapp {start|stop|status|restart...}

This is great. It makes installing new apps literally a few steps: upload the jar, symlink to `init.d`, start. Boom,
done [^1]. At [Aspect][aspect], we link to bundle up our applications as `rpm` packages, as we run most of our apps
on [Centos][centos]. The goal is to have an RPM package that you can just install, and it will do all of the above
steps for you. Ideally, we want a build task for this... something that does a full build and spits out an RPM (or
DEB for Debian based distros) at the end.

Luckily, the guys over at [Netflix Nebula][nebula] have us covered. They produce the amazing [Gradle Linux Packaging
Plugin (ospackage)][ospackage], which fits all the requirements. Throughout the rest of this post, I'll provide
examples on how to create a simple service, package it as an RPM, and finally install and run.

Before we get started, I'd just like to note that this post assumes at least a working knowledge of the following:

* Spring Boot / Spring MVC
* Gradle
* Linux

<sup>(Skip it all and check out the [sample code on GitHub][democode]!)</sup>

So now that that's out of the way,tTo start, I've created a simple Spring Boot web service. Nothing fancy, however it
can literally any Spring Boot 1.3+ application. For simplicity's sake, it's just a single MVC controller that returns
the string `Hello world!`:

{% highlight java %}
package me.ccampo.daemondemo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
public class Application {

    public static void main(final String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @RestController
    public static class HelloController {

        @RequestMapping
        public String helloWorld() {
            return "Hello world!";
        }
    }
}
{% endhighlight %}

Running this and hitting `http://localhost:8080` will just return `Hello world!`. That's it; simple. It is
configured as a standard Spring Boot gradle project, and the full code can be seen on [GitHub][democode]. The cool
part comes when you package this into an RPM and install it as a service.

To use the `ospackage` plugin, first add the necessary dependency to your `build.gradle` file, like so:

{% highlight groovy %}
buildscript {
    ext {
        osPackageVersion = "3.4.0"
    }
    repositories {
        jcenter()
    }
    dependencies {
        classpath("com.netflix.nebula:gradle-ospackage-plugin:${osPackageVersion}")
    }
}

apply plugin: "nebula.ospackage"
{% endhighlight %}

Now we need to configure the `ospackage` plugin to our liking. Here's how I configured the plugin for this sample app:

{% highlight groovy %}
// OS Package plugin configuration
ospackage {
    packageName = "daemon-demo"

    // Uses the main project version
    version = "${project.version}"

    /* Could be anything - in our production builds,
       this is set to the git commit hash */
    release = 1

    os = LINUX
    type = BINARY
    arch = NOARCH

    /* Our install scripts - see the full code for examples.
       They usually do simple prep/cleanup tasks */
    preInstall file("scripts/rpm/preInstall.sh")
    postInstall file("scripts/rpm/postInstall.sh")
    preUninstall file("scripts/rpm/preUninstall.sh")
    postUninstall file("scripts/rpm/postUninstall.sh")

    // Sets our working directory and permissions, basically
    into "/opt/local/daemon-demo"
    user "daemon-demo"
    permissionGroup "daemon-demo"

    // Copy the actual .jar file
    from(jar.outputs.files) {
        // Strip the version from the jar filename
        rename { String fileName ->
            fileName.replace("-${project.version}", "")
        }
        fileMode 0500
        into "bin"
    }

    // Copy the config files
    from("install/unix/conf") {
        fileType CONFIG | NOREPLACE
        fileMode 0754
        into "conf"
    }
}
{% endhighlight %}

One thing to note about above: my project has `scripts` and `install` directories, which contain the necessary
install scripts and configuration files that are copied during install. Please check out the [sample code][democode]
to see how this is all arranged.

Finally, I usually tweak the actual build tasks to do a few more things, and create the symlinks to `/etc/init.d`:

{% highlight groovy %}
// Configure our RPM build task
buildRpm {
    user "daemon-demo"
    permissionGroup "daemon-demo"

    // Creates an empty log directory
    directory("/opt/local/daemon-demo/log", 0755)

    /* Creates a symlink to the jar file as an init.d script
       (this functionality is provided by Spring Boot) */
    link("/etc/init.d/daemon-demo",
         "/opt/local/daemon-demo/bin/daemon-demo.jar")

    /* According to Spring Boot, the conf file needs to sit
       next to the jar, so we just create a symlink */
    link("/opt/local/daemon-demo/bin/daemon-demo.conf",
         "/opt/local/daemon-demo/conf/daemon-demo.conf")
}

// Same as the buildRpm task, but for Debian packages
buildDeb {
    user "daemon-demo"
    permissionGroup "daemon-demo"
    directory("/opt/local/daemon-demo/log", 0755)
    link("/etc/init.d/daemon-demo",
         "/opt/local/daemon-demo/bin/daemon-demo.jar")
    link("/opt/local/daemon-demo/bin/daemon-demo.conf",
         "/opt/local/daemon-demo/conf/daemon-demo.conf")
}
{% endhighlight %}

We're pretty much done at this point. We can now just run the build and it should spit out an RPM, ready to be
installed as a service! To build the RPM:

    $ ./gradlew clean build buildRpm

and to build the DEB:

    $ ./gradlew clean build buildDeb

Gradle will spit out the packages in the `build/distributions` directory. After that, it's just a simple matter of
installing the package! On CentOS, installing the RPM can be done like so:

    $ rpm -Uvh daemon-demo-0.0.1-1.noarch.rpm

In the demo above, I set it to install to `/opt/local/daemon-demo`. In the directory `/opt/local/daemon-demo/conf`,
there are two config files which can be used to alter the runtime behavior of the application. The first,
`daemon-demo.conf` is a standard conf file used to set some variable about the application.
The [Spring Boot docs](confdocs) have more info on this, but the key is that this file needs to be next to the jar,
which is installed to `/opt/local/daemon-demo/bin`, so a symlink is created to the `bin` directory as part of the RPM
build task. The second file is just the standard `application.properties` file, however this can be edited to adjust
runtime properties! This is done by setting the `--spring.config.location` command line switch in the
`daemon-demo.conf` file. Again, see the [source][democode] for more detail. Pretty cool eh?

To run the app, just do `service daemon-demo start` and it should start up on port 8080 by default. You can view the
app's log file in `/opt/local/daemon-demo/log/daemon-demo.log` as well.

To do a full uninstall, just do:

    $ rpm -e daemon-demo-0.0.1-1.noarch

...and the package will be removed from the system, just like any other package.

That's about it! I hope you all find this helpful! Thanks again to the amazing Spring Boot team for making a really
incredible framework for developing applications. I'm sure there are plenty of other cool features like this that will
be coming down the pipe soon!

Once again, the full source of this demo application can be found on [GitHub][democode].

[^1]: If your OS uses `systemd`, it's a little more complicated, but not much; check out [Spring's docs][bootdocs].

[boothome]: http://projects.spring.io/spring-boot/
[bootdocs]: http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#deployment-service
[nebula]: http://nebula-plugins.github.io/
[ospackage]: https://github.com/nebula-plugins/gradle-ospackage-plugin
[democode]: https://github.com/ccampo133/boot-daemon-demo
[confdocs]: http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#deployment-script-customization-conf-file