---
layout: post
title:  "Configuring Multiple JDBC DataSources with Spring Boot"
date:   2016-02-13 19:54:00
author: Chris Campo
categories: general java spring
---
I spent probably an hour or so yesterday learning how to enable multiple
`DataSource`s in a Spring Boot application, and have them be configurable via
the `application.properties` (or `.yaml`) file. The [Spring Boot docs][bootdocs]
have a simple example and about a paragraph on how to do this, but I found it a
bit lacking in details. After bit of experimentation and exercising my
Google-fu, I was able to piece together what I needed to get this working.

Basically you just need to configure two `DataSource` beans in one of your
`@Configuration` classes, and use the `@ConfigurationProperties` annotation to
specify the property prefix used in the `application.properties` file. Using
that prefix, you can assign any of the Spring supported `DataSource` properties
(the Spring Boot docs have more info on this).

You'll also probably want to annotate one of the beans with `@Primary`, as this
will mark the default `DataSource` that will be autowired if you chose to do so.

Here's an example of the configuration:

{% highlight java %}
import org.springframework.boot.autoconfigure.jdbc.DataSourceBuilder;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import javax.sql.DataSource;

@Configuration
public class ApplicationConfiguration {

    @Bean
    @Primary
    @ConfigurationProperties(prefix = "datasource.primary")
    public DataSource numberMasterDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties(prefix = "datasource.secondary")
    public DataSource provisioningDataSource() {
        return DataSourceBuilder.create().build();
    }
}
{% endhighlight %}

And the `application.properties` file:

{% highlight yaml %}
# Primary DataSource configuration
datasource.primary.url=
datasource.primary.username=
datasource.primary.password=
# Any of the other Spring supported properties below...

# Secondary DataSource configuration
datasource.secondary.url=
datasource.secondary.username=
datasource.secondary.password=
# Any of the other Spring supported properties below...
{% endhighlight %}

Here's a small example demonstrating how autowiring will work. If you want to
autowire the secondary `DataSource`, you'll have qualify it by name.

{% highlight java %}
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.stereotype.Service;

import javax.sql.DataSource;

@Service
public class DataSourceService {

    private final DataSource primaryDataSource;
    private final DataSource secondaryDataSource;

    @Autowired
    public DataSourceService(final DataSource primaryDataSource,
            @Qualifier("secondaryDataSource") final DataSource secondaryDataSource) {
        this.primaryDataSource = primaryDataSource;
        this.secondaryDataSource = secondaryDataSource;
    }
}
{% endhighlight %}

Finally, one last _gotcha_ that nailed me was that I wanted to ensure that
Spring Boot did not enable the default `DataSource` auto-configuration, which
resulted in it running a `schema.sql` file (on your classpath, usually under
the `src/main/resources/` path) to populate the DB schema. Unfortunately, using
this configuration, you can't just set the
`datasource.{primary,secondary}.initialize` property to false. Instead you have
to use `spring.datasource.initialize=false`. This makes sense in the end, but it
tripped me up for a bit.

So just to reiterate, if you want to turn off the
Spring Boot database initialization, set the following in your
`application.properties`:

{% highlight yaml %}
# Disable Spring DataSource auto-initialization
spring.datasource.initialize=false
{% endhighlight %}

Hopefully you all can find this helpful!

[github]: https://github.com/ccampo133/spring-boot-oauth2-demo
[bootdocs]: http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#howto-two-datasources
