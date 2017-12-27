---
layout: post
title:  "Exploring Spark Java"
date:   2017-12-27
categories: programming
---

I wanted to build a home automation project leveraging IoT, Alexa, and whatever else I could think to throw at it. As of 
this post, the project is still in development, but does have complete and working components. The central component of 
this automation is the server that ties it all together. I wanted to use Java, but also wanted to do away with the 
unnecessary "Enterprise" aspects that makes Java development for smaller projects so cumbersome. SparkJava made a strong 
promise: _"A micro framework for creating web applications in Kotlin and Java 8 with minimal effort"_. 

The keywords which sold me on it were "micro framework" and "minimal effort". And from just seeing the examples of how 
easy it was to wire up basic GET requests/responses, I immediately fired up my IDE. This post will explore whether 
SparkJava delivers on that promise, and how my project developed around SparkJava's principles and my own needs.

## But First
I don't intend for this post to be a tutorial, nor do I want readers evaluating different frameworks and hoping to come 
away thinking this post will convince you of SparkJava, nor think that I had deeply compared and contrasted multiple 
frameworks (I really didn't) before trying SparkJava. I do know there is some overlap between how Spring does routing 
and SparkJava does routing. The goal of this post is to just evaluate SparkJava on its claims and my specific 
experiences.

Secondly, my overall project structure and state is not necessarily how to best use SparkJava. My use of the framework 
changed dramatically, as after building some of the components, I heavily disagreed with the architectural opinions 
of the SparkJava authors. I didn't fully move away from their proposed design as I was lazy, and wanted to focus more 
on features and delivering working things, rather than a pure engineering product.

## Request Flow
The flow of a request is fairly simple, and can be at a high level imagined with _Figure 1_.

<br />
{:refdef: style="text-align: center;"}
![Basic HTTP GET Flow]({{site.cdn_path}}/spark/basic_flow.png)
{: refdef}
<p class="caption">Figure 1</p>
<br /> 

## Don't Make Everything Static
At first I was willing to believe this quote from the "Application Structure" tutorial, _"This is probably not what you learned in Java class, but I believe statics are better than dependency injection when dealing with web applications / controllers. Injecting dependencies makes everything a lot more ceremonious..."_. Following this quote is maybe ok if you are building 
a super simple project. For me, this quote meant a large refactoring. Making things static meant I didn't know where 
variables came from, or who owned what, or why. Cohesion was way down and coupling way up. This became apparent as I 
started to introduce components that had to talk to each other with slightly different interfaces or needs, and the 
web of dependencies became large.

Needless to say, switching to Dependency Injection (DI) was the superior choice for my needs. Suddenly I knew where to
import something from and how to reference it, and my IDE was able to helpfully narrow down the options. It was not overly 
ceremonious to declare my dependencies. I leveraged Spring only for the DI. A couple Annotations and everything was much 
cleaner.

The only real oddity was in needing to create an `AnnotationConfigApplicationContext` and a default `ShutdownHook`. None 
of this altered the overall project substantially, and I only need to list out all the components the WebConfig will use 
as it was the entry point to the Application, and needed the beans created for it. This is standard practice, and was 
all done in Java without any XML.