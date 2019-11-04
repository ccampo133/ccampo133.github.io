---
layout: post
title:  "Java in a Container: Resource Allocation Guidelines"
date:   2019-10-31 17:09:00
author: Chris Campo
categories: java docker containers kubernetes
---

Containers have changed the face of the software industry in just a few short years. Perhaps you've gotten to the point
where you're now running Java in a container. That's great! Unfortunately, there are still some things to be aware of
when it comes to CPU and memory utilization for containerized Java applications, which I'll outline below.

This article assumes some familiarity with Java and containers in general. If you need more background, check out some 
(or all) of the [references](#references).

## Heap Space

If there's one key statement regarding running Java in a container, it's the following:

**DO NOT SET THE JVM HEAP SPACE MANUALLY FOR ANY JAVA PROCESS RUNNING IN A CONTAINER. INSTEAD, SET THE CONTAINER LIMITS.**

Why?

* First and foremost, setting container limits accomplishes the very basic objective of containers/cgroups to begin 
with... that is, to isolate resource usage of a collection of processes. When you allocate the heap space manually via 
JVM arguments for example, you disregard that entirely.

* It allows easy tuning of container resource allocation. Need more memory? Bump the container limit. Need less? 
Downsize it. This opens the door for automated processes to scale containers accordingly (such as the [k8s vertical pod 
autoscaler][vpa]) without having to manually tweak JVM parameters.
    
* If running in a container orchestration environment (such as Kubernetes), the container limits become extremely 
important for both node health and scheduling. The scheduler will use these limits to find an appropriate node to run a 
container, and ensure that equal load is spread across nodes. If you are setting memory usage via JVM arguments, this 
information is not available to the scheduler, and therefore the scheduler has no idea how to effectively spread load 
for your containers.

* If you do not set container limits, and Java is running in a container without any JVM memory flags explicitly set, 
the JVM will automatically set the max heap to 25% of the RAM on the **node** it is running on. For example, if your 
container is running on a 64 GB node, your JVM process heap space can max out at 16 GB. If you're running 10 containers 
on a node (a common occurrence due to auto-scaling), then all of the sudden you're on the hook for 160 GB of RAM. It's a 
disaster waiting to happen.[^1]

### What can you do about it?

Set the container memory (and CPU) limits.[^2] It is not sufficient to rely on resource requests (soft limits) only. 
Requests are great for helping out the scheduler, but setting hard limits allows Docker (or whatever container runtime 
you're using) to allocate the specified resources to the container itself, and no more. This will also allow Java 
(which is "container aware" by default as of Java 8u191) to properly allocate the memory based on the resource 
limits placed on the container itself, instead of the node it is running on.

### Regarding the `[Min|Max|Initial]RAMPercentage` Parameters

In a [fairly recent version of Java][jvmrelnotes], the following JVM parameters were introduced (and back-ported to Java
8u191).

 * `-XX:MinRAMPercentage`
 * `-XX:MaxRAMPercentage`
 * `-XX:InitialRAMPercentage`
 
I won't go into detail about how these exactly work,[^3] but the key takeaway is that they can be used to fine tune the 
JVM heap size without setting the heap size directly. That is, the container can still rely on the limits imposed on it.

So what are the correct values to use? The answer is - "it depends"... particularly on the limits imposed on the 
container.

By default, the JVM heap gets 25% of the container's memory. You can tune the initial/min/max heap parameters to change 
this... e.g. setting `-XX:MaxRAMPercentage=50` will allow the JVM to consume 50% of the container memory for the heap, 
instead of the default 25%. When this is safe depends largely on how much memory the container has to work with, and 
what processes are running in the container. 

For example, if your container is running a single Java process, has 4 GB of RAM allocated to it, and you set 
`-XX:MaxRAMPercentage=50`, the JVM heap will get 2 GB. This is opposed to the 1 GB default it would normally get. In 
this case, 50% is almost certainly perfectly safe, and perhaps optimal, since much of the available RAM was probably 
under-utilized. However, imagine the same container has only 512 MB of RAM allocated to it. Now setting 
`-XX:MaxRAMPercentage=50` gives the heap 256 MB of RAM and only leaves the other 256 MB to rest of the entire container. 
This memory would need to be shared by all other processes running in the container, plus the JVM Metaspace/PermGen, 
etc. allocations. Perhaps 50% is not so safe in this case.

Therefore, I can offer the following recommendations:

* Don't touch these parameters unless you want to squeeze some extra memory headroom for your Java process. In most 
cases, the 25% default is a safe approach to managing memory. It might not be the most memory efficient, but RAM is 
cheap and it's always better to err on the side of caution, versus having your JVM process OOM-killed for some unknown 
reason.

* If you do want to tweak these, be conservative. 50% is most likely a safe value where you can expect to avoid issues 
(in most cases), however this is still is largely dependent on how big (memory-wise) your container is. I don't 
recommend going to 75% unless your container has at least 512 MB of RAM (preferably 1 GB), and you have a good 
understanding of the memory usage of the application in question.

* If your container is running multiple processes in addition to Java, be extra cautious when tweaking these values.
The container memory is shared across all of these processes, and understanding total container memory usage in these 
cases is more complicated.

* Anything over 90% is probably asking for trouble.

## What about Metaspace/PermGen/etc?

That's beyond the scope of this article but rest assured that this can also be tweaked, but likely shouldn't. The 
default JVM behavior be fine for majority of use cases. If you find yourself trying to tackle an obscure memory issue, 
then it might be time to look into fiddling with this somewhat esoteric area of JVM memory, but otherwise I would avoid 
doing anything directly. 

## And CPU?

There's not much to do here. As of Java 8u191, the JVM is pretty "container aware" by default and interprets CPU share 
allocation correctly. There are details around this that are worth understanding though, so instead of me outlining it 
here, I will direct you to [this excellent article][jvmcpu] explaining it all in detail.

## Summary

Modern Java is well suited to run in a container environment, but there are some not-so-obvious details that everybody 
should know to ensure they're getting the best performance from their applications. I hope that the information 
presented here, combined with the excellent [references](#references), helps you accomplish this goal.

## References

* [Nobody Puts Java In A Container](https://jaxenter.com/nobody-puts-java-container-139373.html)
* [JVM Memory Settings in a Container Environment](https://medium.com/adorsys/jvm-memory-settings-in-a-container-environment-64b0840e1d9e)
* [JDK 8u181 release notes regarding container support][jvmrelnotes]
* [Docker support in Java 8 — finally!](https://blog.softwaremill.com/docker-support-in-new-java-8-finally-fd595df0ca54)
* [How to correctly size containers for Java 10 applications](https://banzaicloud.com/blog/java10-container-sizing/)
* [Docker and the JVM](https://www.javacodegeeks.com/2018/12/docker-jvm.html)
* [Java 8: From PermGen to Metaspace](https://dzone.com/articles/java-8-permgen-metaspace)
* [Analyzing Java memory usage in a Docker container](http://trustmeiamadeveloper.com/2016/03/18/where-is-my-memory-java/)
* [Kubernetes throwing OOM for pods running a JVM](https://stackoverflow.com/questions/52596383/kubernetes-throwing-oom-for-pods-running-a-jvm)
* [CPU considerations for Java applications running in Docker and Kubernetes][jvmcpu]
* [Kubernetes: Managing Compute Resources for Containers][k8slim]
* [Docker: Runtime options with Memory, CPUs, and GPUs][dockerlim]

---

[^1]: Note In the 64 GB / 16 GB JVM example, this does not mean that the JVM process automatically consumes 16 GB of RAM for the heap... just that it can potentially grow that large before being considered out of memory. Additionally, there is no downward pressure on the garbage collector. The heap can then easily grow to consume more memory than the container can support before the GC is triggered due to miscalculated max heap space. This would inevitable cause application issues (e.g. OOM errors) or worse (e.g. OOM-killed, crash).
[^2]: For information on setting hard limits in Docker and Kubernetes, check out the following links: [1][dockerlim], [2][k8slim] 
[^3]: See this [StackOverflow](https://stackoverflow.com/a/54297753) thread where this is discussed.

[vpa]: https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler
[dockerlim]: https://docs.docker.com/config/containers/resource_constraints
[k8slim]: https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container
[jvmcpu]: https://medium.com/@christopher.batey/cpu-considerations-for-java-applications-running-in-docker-and-kubernetes-7925865235b7
[jvmrelnotes]: https://www.oracle.com/technetwork/java/javase/8u191-relnotes-5032181.html#JDK-8146115
