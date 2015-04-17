---
layout: post
title:  "How to buid an OAuth2 protected REST API, using Spring Boot, Spring Security, and Spring Security OAuth2"
date:   2015-04-16 23:46:00
categories: General
---

My goal with this post is to provide a quick and informative tutorial on how to generate a fully secured OAuth 2.0 RESTful API using Spring Boot, Spring Security, and Spring Security OAuth2. I'll also point out all of the "gotchas!" that got me the first time around doing this, and clarify some details that may not be so apparent in the code itself. While I love [Spring](http://spring.io), there is no doubt that their documentation can be opague, or even lacking at times, like in the case of the [Spring Security OAuth2](http://projects.spring.io/spring-security-oauth/docs/Home.html) project.

If you aren't familiar with the [OAuth 2.0 specification](http://oauth.net/2/), I suggest you familarize yourself at some point. It doesn't have to be now, but it may help you understand some concepts at a higher level. Also, general knowledge of Java and Spring is assumed. For this tutorial, I'll be using [Gradle](http://gradle.org), although feel free to use Maven or another build tool if you so desire. Also, all of the code is available on [GitHub](github) for you to pull down and use yourself :)

...so without further ado, lets get started!

In our project root directory, we need a `build.gradle` file to handle all of our dependencies. We're going to need to rely on Spring Boot, Spring Security, and Spring Security OAuth2. To make things quick, just use the following code for your `build.gradle` contents:

{% highlight java %}
buildscript {
    ext {
        springBootVersion = "1.2.3.RELEASE"
        springSecurityVersion = "4.0.0.RELEASE"
        springSecurityOAuth2Version = "2.0.7.RELEASE"
    }
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
    }
}

apply plugin: "java"
apply plugin: "spring-boot"

jar {
    baseName = "spring-boot-oauth2-demo"
    version = "0.0.1-SNAPSHOT"
}

sourceCompatibility = 1.7
targetCompatibility = 1.7

repositories {
    mavenCentral()
}

dependencies {
    compile("org.springframework.boot:spring-boot-starter-web")
    compile("org.springframework.security:spring-security-web:${springSecurityVersion}")
    compile("org.springframework.security.oauth:spring-security-oauth2:${springSecurityOAuth2Version}")
    testCompile("org.springframework.boot:spring-boot-starter-test")
}
{% endhighlight %}


If you aren't familiar with Gradle, without going in to too much detail, this pulls down all the necessary dependencies to get up and running quickly. We'll structure our project like any other standard Java project, and we'll use the base package `oauth2demo`. Take a look at the code on [GitHub](github) for the full project layout.

Next thing is to create our application entry point in the base `oauth2demo` package. This is a class with a `main` method, just like a standard Java command line application... except it's Spring Bootified! Call it `Application.java`:

{% highlight java %}
package oauth2demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
{% endhighlight %}

That's it. At this point, you can literally compile run the application and you'll have a full Spring MVC web app hosted at `http://localhost:8080`. Of course, it won't do much of anything.... yet :)


[github]: https://github.com/ccampo133/spring-boot-oauth2-demo