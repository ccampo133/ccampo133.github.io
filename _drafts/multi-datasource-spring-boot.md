---
layout: post
title:  "Configuring Multiple JDBC DataSources with Spring Boot"
date:   2016-02-12 18:20:00
author: Chris Campo
categories: General Java Spring
---

Java:

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


[github]: https://github.com/ccampo133/spring-boot-oauth2-demo
