---
layout: post
title:  "Using Spring with AWS Lambda"
date:   2016-11-27 15:00:00
author: Chris Campo
categories: java spring aws
---

I love [AWS Lambda][lambda]. Removing the concept of a "server" from your application is huge. I won't go into all 
the benefits of Lambda, but I can say from first hand experience, it not only eliminates countless hours of worrying 
about servers and infrastructure, but also makes backend application development a breeze. Oh, and for most small 
stuff, [it's pretty much free][pricing].

Lambda currently supports three languages for application development - Javascript (via NodeJS), Python, and Java. My
preferred language for Lambda is Java [^1] for its static typing, maturity, and open source community... plus it's 
what I write mostly in my day job. The Lambda API for Java is simple - you write a class that implements one of their
interfaces (`RequestHandler`) or you write a "handler" method with a specific signature and tell Lambda to call it. You 
can [read the docs][lambda-java] for the specifics, but suffice to say that it's simple.
 
While the Java API is nice, it's pretty bare-bones. Also, since AWS Lambda requires your `RequestHandler` 
implementation to have a default, no-arg constructor, writing `RequestHandler`s with dependencies becomes tricky from
a testability standpoint. For example, imagine our `RequestHandler` requires a data-access-object (DAO) class. If we
want to write some unit tests for this `RequestHandler`, we have no way of stubbing this DAO since it has to be 
instantiated either as a field or in the no-arg constructor. Typically the way to solve this problem is using 
Dependency Injection/Inversion of control.

My library of choice for dependency injection (among with a bunch of other cool features) is [Spring][spring]. There is 
a bit of boilerplate with integrating Spring into AWS Lambda, so I wrote a small library for eliminating some of that, 
as well as providing a pattern for writing testable, Spring-enabled lambda functions.

Enter [Spring AWS Lambda][spring-aws-lambda] (clever title, I know!). The general gist here is that this library 
provides a few abstract classes which build on top of the AWS Lambda Java API's `RequestHandler` interfaces. Instead 
of implementing AWS's interfaces directly, you extend one of these new abstract classes and voila!, you can now use 
Spring's dependency injection with your Lambda functions.

The meat of the [Spring AWS Lambda][spring-aws-lambda] library is the `SpringRequestHandler` class. Basically, 
instead of implementing `RequestHandler` directly, you extend this class and provide it with a 
Spring `ApplicationContext`. Then you can implement as many `RequestHandler`s as you want and provide injectable 
dependencies to them. See the library source code for more details, but examples are worth more than words, so here's a 
quick one (NOTE: the rest of this post assumes a working knowledge of how Spring dependency injection works):

{% highlight java %}
public class MainHandler extends SpringRequestHandler<Request, Response> {

    /**
     * Here we create the Spring ApplicationContext that will
     * be used throughout our application.
     */
    private static final ApplicationContext context =
            new AnnotationConfigApplicationContext(ApplicationConfiguration.class);

    @Override
    public ApplicationContext getApplicationContext() {
        return context;
    }
}
{% endhighlight %}

`MainHandler` is the entry point to your application. On the AWS Lambda configuration console, you actually set this 
as your "Handler" (see screenshot below). Note that `MainHandler` will rarely have any logic itself. Instead, you 
simply tell it the request and response types via the generic type parameters (these can be anything you want), and 
implement the `getApplicationContext` abstract method, which should provide a reference to your application's main 
`ApplicationContext`.

<img src="{{ site.baseurl }}/assets/images/lambda-conf.png" title="AWS Lambda console">

If you look at the guts of [`SpringRequestHandler`][spring-request-handler-source], you'll notice that it actually 
implements the `RequestHandler` class itself and provides a default `handleRequest` method implementation, that 
actually retrieves a Spring bean of type `RequestHandler` and calls that bean's `handleRequest` method. You are free 
to override this method yourself if you want to add any additional logic, but that's the general gist of it.

More on that - since `SpringRequestHandler` gets a bean of type `RequestHandler` and calls that bean's `handleRequest` 
method, that means that all of your main application logic should go in your own implementation of `RequestHandler`, 
and this class should be declared a bean.

This might seem not much different than the default AWS programming model. Well you're right - it's not. In fact, 
it's almost exactly the same. The key difference here though is that now any `RequestHandler` you implement yourself 
can take advantage of using Spring's annotations and dependency injection. All you have to do is register it as a bean.

Here's an example of your own Spring-enabled request handler:

{% highlight java %}
@Component
public class ExampleRequestHandler implements RequestHandler<Request, Response> {

    /*
     * These can be anything you want; if they're registered as 
     * Spring beans, they'll be injected below via autowiring.
     */
    private final ExampleServiceA exampleServiceA;
    private final ExampleServiceB exampleServiceB;

    /**
     * Dependency injection is handled via autowiring!
     */
    @Autowired
    public ExampleRequestHandler(final ExampleServiceA exampleServiceA, final ExampleServiceB exampleServiceB) {
        this.exampleServiceA = Objects.requireNonNull(exampleServiceA, "exampleServiceA");
        this.exampleServiceB = Objects.requireNonNull(exampleServiceB, "exampleServiceB");
    }

    @Override
    public Response handleRequest(final Request input, final Context context) {
        final String responseMessage = "Request message: " + input.getMessage()
                + ", Service A message: " + exampleServiceA.getMessage()
                + ", Service B message: " + exampleServiceB.getMessage();
        final Response response = new Response();
        response.setMessage(responseMessage);
        response.setStatus(Response.Status.OK);
        return response;
    }
}
{% endhighlight %}

Note that we declare this `RequestHandler` as a bean by use of the `@Component` annotation. When your AWS Lambda 
function is run, it will actually invoke the `handleRequest` method of `ExampleRequestHandler` (via the `MainHandler`
proxy).

The key to take away here is that in our `ExampleRequestHandler` class, we are now able to take advantage of using 
Spring for dependency injection. This means we can inject dependencies, like services, DAOs, etc., via the 
constructor. This is huge for testability! Since AWS Lambda requires that your handler class have a default, no-arg 
constructor, dependencies would be typically hardcoded and tough to mock or stub in a testing environment. Now, we 
have a logic-less `MainHandler` class which we pass to AWS Lambda's console, and a fully 
testable/mockable `RequestHandler` implementation!

For more information, and a more complete example, please see the docs and `example` module on [Spring AWS Lambda's 
Github page][spring-aws-lambda]. I hope this helps you enjoy using AWS Lambda even more!

[^1]: I really prefer Kotlin, since the Lambda runtime is really just the JVM and Kotlin works great with Lambda!

[lambda]: https://aws.amazon.com/lambda/
[pricing]: https://aws.amazon.com/lambda/pricing/
[lambda-java]: http://docs.aws.amazon.com/lambda/latest/dg/java-programming-model.html
[spring]: http://spring.io
[spring-aws-lambda]: https://github.com/ccampo133/spring-aws-lambda
[spring-request-handler-source]: https://github.com/ccampo133/spring-aws-lambda/blob/master/spring-aws-lambda/src/main/java/me/ccampo/spring/aws/lambda/SpringRequestHandler.java
