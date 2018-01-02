---
layout: post
title:  "Exploring SparkJava"
date:   2018-1-1
categories: programming
---

I wanted to build a home automation project leveraging IoT, Alexa, and whatever else I could think to throw at it. As of 
this post, the project is still in development, but does have complete and working components. The central component of 
this automation is the server that ties it all together. I wanted to use Java, but also wanted to do away with the 
unnecessary "Enterprise" aspects that makes Java development for smaller projects so cumbersome. SparkJava made a strong 
promise: _"A micro framework for creating web applications in Kotlin and Java 8 with minimal effort"_. 

The keywords which sold me on it were "micro framework" and "minimal effort". And from just seeing the examples of how 
easy it was to wire up basic GET requests/responses, I decided to give it a try. This post will explore whether 
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
At first I was willing to believe this quote from the "Application Structure" tutorial, _"This is probably not what you 
learned in Java class, but I believe statics are better than dependency injection when dealing with web applications / 
controllers. Injecting dependencies makes everything a lot more ceremonious..."_. Following this quote is maybe ok if 
you are building a super simple project.

For me, this quote meant a large refactoring. Making things static meant I didn't
know where variables came from, or who owned what, or why. Cohesion was way down and coupling way up. This became apparent as I 
started to introduce components that had to talk to each other with slightly different interfaces or needs, and the 
web of dependencies became large.

Needless to say, switching to Dependency Injection (DI) was the superior choice for my needs. With DI, I knew where to
import something from and how to reference it, and my IDE was able to helpfully narrow down the options. It was not overly 
ceremonious to declare my dependencies. I leveraged Spring only for DI; a couple Annotations later and everything 
was much cleaner.

The only real oddity was in needing to create an `AnnotationConfigApplicationContext`. None of this altered the overall
project substantially, and I only need to list out all the components the WebConfig will use as it was the entry point
to the Application, and needed the beans created for it. Nothing out of the norm otherwise, and was all done in Java
without XML.

## Routes
I had to deviate slightly again from the official practices. The documentation recommends statically importing so the
Spark natives can be directly referenced. This is fine so long as your application only needs to listen to one port. As
I wanted to support HTTPS, and redirect HTTP to HTTPS, I needed to define two Service's (one for each port).

Otherwise, defining routes was simple and clean. With lambda, routes could be defined like the below

{% highlight java %}
service.get("/home/hello", (req, res) -> {
    return "Hello World";
});
{% endhighlight %}

For the API oriented paths, they tended to look a lot like this

{% highlight java %}
service.get(Path.LIGHTS_ON, lightController.turnOn, json());

service.get(Path.AC_ON, acController.turnOn, json());
service.post(Path.AC_ACTION, acController.action, json());
{% endhighlight %}

For HTML pages, they looked like this

{% highlight java %}
service.get("/home", (req, res) -> Rythm.render("index.rythm",
        deviceHttpControllerFacade.getAllDeviceStatus(),
        deviceHttpControllerFacade.getDeviceCapabilityAsMap(),
        deviceHttpControllerFacade.getAllDevices(),
        acController.getAllAcStates(),
        ambientHttpControllerFacade.getMostRecentAmbientStatesAsMap())
);
{% endhighlight %}

This is a little boilerplate, but the next level up would be for each "feature" to define its own REST interface
through a provided Service object. This would decouple the WebConfig class from the features and their specific interfaces.
Since my API is fairly simple and small, I kept everything within the WebConfig class to keep things straightforward.

## Rythm - HTML Rendering
This was definitely not a good choice. Rythm has poor documentation and tooling support. While SparkJava works with
any rendering engine, I chose Rythm for its apparently simpler syntax and richer features. I could not find anything 
like Pug, which is what I used for NodeJS, and was very disappointed as Pug was an absolute pleasure to work with.

Rythm is not simple to use; the exceptions/errors are difficult to understand and generally unhelpful. I don't have
much experience designing web pages, or particularly good looking ones. However, I did want a kind of "card" like
overview page of all the devices which provided a summarized view into each device's state.

<br />
{:refdef: style="text-align: center;"}
![Basic HTTP GET Flow]({{site.cdn_path}}/spark/device_card_example.png)
{: refdef}
<p class="caption">Figure 1</p>
<br /> 

A snippet of some of the code to generate that looked like this

{% highlight java %}
@if(deviceType.equals("AMB_SEN")) {
    @{AmbientState state = ambientStates.get(device.getDeviceId());}
    @if(state != null){
        @state.getTemperature()C @state.getHumidity()% and @state.getLightLevel()
    }
    else{
        No data
    }
}
{% endhighlight %}

What is a mixed pro/con is the fourth line where I am printing the device temperature, humidity, and light level. The Java
code is seamlessly integrated with the HTML. However, the if-statements and variable declaration lines are very awkward.
It isn't always clear when I need to encase something in a Java block with the `@{}`, or to just directly reference with
a leading `@`. Rythm understands how to translate the above into appropriate calls into Java objects with the same
method interface.

However, my IDE had no way to intelligently hint to me what the method signatures, or even the types,
of objects I was referencing; let alone Java-specific syntax highlighting. This made building crude dynamic pages 
painfully slow, with lots of back and forth reading esoteric Rythm parsing errors and exceptions. Combined with how I had
to restart the web application each time to see a change, the launch time of SparkJava (a couple seconds) started to 
increase test time significantly.

I definitely plan to switch away from Rythm to a more mature and well known framework, even if it isn't as streamlined.

## Auth Layer
Defining a crude authentication layer to prevent fiddling with my site was fairly easy. What was difficult was excluding
some pages from having the "not authenticated" redirect *not* applied to it. SparkJava provides a 
`before( (req, res) -> {})` construct that will happen before all (or some path-restricted) paths. I used this to detect
a session, and if none exists, then to redirect to the login page. However, the login page also doesn't have a session token, 
so it would redirect to login page, and so on.

This could get ugly fast. By default I would rather want auth enforced on all resources lest I forget to tag a resource as protected.
But, then I must maintain a list of resources that don't need auth applied and explicitly check the requested resource 
against the list. I was surprised that Spark had no built in authentication handling, and instead relied on external
libraries. I wanted to keep things simple and "in-house" to reduce dependency overload, but this wasn't an ideal solution. 

That said, I only have two resources that don't need auth, so they were easy to exclude.

## HTTPS
This was tricky, and I really didn't enjoy figuring out all the little details. I chose Let's Encrypt as my certificate
provider and was pleasantly surprised at just how wonderful the process of getting Let's Encrypt configured and working
was. What was not so wonderful was getting that into Java/SparkJava.

Java prefers to handle certificates through a `KeyStore`, which bundles certificates into a single file protected by a
password. SparkJava consumes this just fine, however Let's Encrypt needs to rotate the certificate every 90 days, and while
multiple methods for doing so are available, the most practical was to expose a hidden folder through my website for the 
Let's Encrypt script to do its work for refreshing the certificate.

This was not easy. At first I tried just exposing the hidden folder and seeing if SparkJava could serve files from there: nope. 
Next, I tried searching around to see if SparkJava could be configured to expose hidden directories: nope. Instead, I 
tried cheating and just marking the hidden folder as an external static file location: nope.

Eventually, I gave up and decided to just manually return the exposed file through a mapped GET route.

{% highlight java %}
http.get("/.well-known/*", (req, res) -> {
    res.type("text/plain");
    res.raw().getOutputStream().write(Files.readAllBytes(Paths.get(System.getProperty("user.dir") + req.uri())));
    res.raw().getOutputStream().flush();
    res.raw().getOutputStream().close();
    return res.raw();
});
{% endhighlight %} 

This is awful and very user unfriendly. Worse, this is only for updating the certificate. To get SparkJava to use the
new certificate, my only solution was to create a post-deploy hook from the `certbot` tool from Let's Encrypt that
would rebuild the `KeyStore`, replace the old one SparkJava was using, and then restart the whole service.

This isn't a great way to handle certificate rotation, if only because it required me to configure and build multiple
parts to ensure a certificate is rotated.

If there was one thing I would want SparkJava to do differently, it would be this.

## Logging
This was definitely something I had no idea what I was doing. I have used SLF4J before, but was lost as how to 
configure it correctly, do what I wanted, or really anything other than what SparkJava had shipped with by default. This
is somewhat embarrassing to admit, but I completely gave up on this part. It should have been simple, from both SparkJava's
own documentation and SLF4J's documentation, but I just could not get my own configuration to work.

Perhaps this is the failure of one, or many, of the tools involved, but I don't have the confidence to state which. For
now, SparkJava uses sane defaults, so I can log as needed for debugging purposes. 

## Overall
This post reflected a lot on the negative experiences of SparkJava I had, but there weren't that many issues that I had
faced in adopting SparkJava. All the above issues aside, SparkJava has been great. Defining a REST API is simple, quick 
to stand-up, and more advanced features such as HTTPS, authentication, and more weren't overly difficult to setup.

In large part, SparkJava delivers on its promise of quickly getting an HTTP service up and running, without much hassle
or boilerplate, and with a lot of flexibility from the framework itself. Aside from some of the quirks noted above, I
never felt like SparkJava was in my way or I couldn't understand what was happening. In the few cases where I messed
something up, digging through the documentation and online communities resolved my problems without much fuss.

I recommend peering through the example code on my [IoT project](https://github.com/grnt426/HomeServer) for how I have
leveraged SparkJava.