Part 16: Where to Go From Here
============================================================

As I have already hinted at in the previous article, the Neophyte’s Guide to Scala is coming to an end. Over the last five months, we have delved into numerous language and library features of Scala, hopefully deepening your understanding of those features and their underlying ideas and concepts.

As such, I hope this guide has served you well as a supplement to whatever introductory resources you have been using to learn Scala, whether you attended the Scala course at Coursera or learned from a book. I have tried to cover all the quirks I stumbled over back when I learned the language – things that were only mentioned briefly or not covered at all in the books and tutorials available to me – and I hope that especially those explanations were of value to you.

As the series progressed, we ventured into more advanced territory, covering ideas like type classes and path-dependent types. While I could go on writing about more and more arcane features, I felt like this would go against the idea of this series, which is clearly targeted at aspiring neophytes.

Hence, I will conclude this series with some suggestions of where to go from here if you want more. Rest assured that I will continue blogging about Scala, just not within the context of this series.

How you want to continue your journey with Scala, of course, depends a lot on your individual preferences: Maybe you are now at a point where you would like to teach Scala to others, or maybe you are excited about Scala’s type system and would like to explore some of the language’s more arcane features by delving into type-level programming.

More often than not, a good way to really get comfortable with a new language and its whole ecosystem of libraries is to use it for creating something useful, i.e. a real application that is more than just toying around. Personally, I have also gained a lot from contributing to open source projects early on.

In the following, I will elaborate those four paths, which, of course, are not mutually exclusive, and provide you with numerous links to highly recommendable additional resources.

Teaching Scala
------------------------------------------------------

Having followed this series, you should be familiar enough with Scala to be able to teach the basics. Maybe you are in a Java or Ruby shop and want to get your coworkers excited about Scala and functional programming.

Great, then why not organize a workshop? A nice way of introducing people to a new language is to not do a talk with lots of slides, but to teach by example, introducing a language in small steps by solving tiny problems together. Active participation is key!

If that’s something you’d like to do, the Scala community has you covered. Have a look at Scala Koans, a collection of small lessons, each of which provides a problem to be solved by fixing an initially failing test. The project is inspired by the Ruby Koans project and is a really good resource for teaching the language to others in small collaborative coding sessions.

Another amazing project ideally suited for workshops or other events is Scalatron, a game in which bots fight against each other in a virtual arena. Why not teach the language by developing such a bot together in a workshop, that will then fight against the computer? Once the participants are familiar enough with the language, organize a tournament, where each participant will develop their own bot.


Mastering arcane powers
-------------------------------------------------------

We have only seen a tiny bit of what the Scala type system allows you to do. If this small hint at the high wizardry that’s possible got you excited and you want to master the arcane powers of type-level programming, a good starting resource is the blog series Type-Level Programming in Scala by Mark Harrah.

After that, I recommend to have a look at Shapeless, a library in which Miles Sabin explores the limits of the Scala language in terms of generic and polytypic programming.

Creating something useful
-------------------------------------------------------

Reading books, doing tutorials and toying around with a new language is all fine to get a certain understanding of it, but in order to become really comfortable with Scala and its paradigm and learn how to think the Scala way, I highly recommend that you start creating something useful with it – something that is clearly more than a toy application (this is true for learning any language, in my opinion).

By tackling a real-world problem and trying to create a useful and usable application, you will also get a good overview of the ecosystem of libraries and get a feeling for which of those can be of service to you in specific situations.

In order to find relevant libraries or get updates of ones you are interested in, you should subscribe to implicit.ly and regularly take a look at Scala projects on GitHub.

### It’s all about the web

These days, most applications written in Scala will be some kind of server applications, often with a RESTful interface exposed via HTTP and a web frontend.

If the actor model for concurrency is a good fit for your use case and you hence choose to use the Akka toolkit, an excellent choice for exposing a REST API via HTTP is Spray Routing. This is a great tool if you don’t need a web frontend, or if you want to develop a single-page web application that will talk to your backend by means of a REST API.

If you need something less minimalistic, of course, Play, which is part of the Typesafe stack, is a good choice, especially if you seek something that is widely adopted and hence well supported.

### Living in a concurrent world

If after our two parts on actors and Akka, you think that Akka is a good fit for your application, you will likely want to learn a lot more about it before getting serious with it.

While the Akka documentation is pretty exhaustive and thus serves well as a reference, I think that the best choice for actually learning Akka is Derek Wyatt’s book Akka Concurrency, a preliminary version of which is already available as a PDF.

Once you have got serious with Akka, you should definitely subscribe to Let It Crash, which provides you with news and advanced tips and tricks and regarding all things Akka.

If actors are not your thing and you prefer a concurrency model allowing you to leverage the composability of Futures, your library of choice is probably Twitter’s Finagle. It allows you to modularize your application as a bunch of small remote services, with support for numerous popular protocols out of the box.

Contributing
-----------------------------------------------------

Another really great way to learn a lot about Scala quickly is to start contributing to one or more open source projects – preferably to libraries you have been using while working on your own application.

Of course, this is nothing that is specific to Scala, but still I think it deserves to be mentioned. If you have only just learned Scala and are not using it at your day job already, it’s nearly the only choice you have to learn from other, more experienced Scala developers.

It forces you to read a lot of Scala code from other people, discovering how to do things differently, possibly more idiomatically, and you can have those experienced developers review your code in pull requests.

I have found the Scala community at large to be very friendly and helpful, so don’t shy away from contributing, even if you think you’re too much of a rookie when it comes to Scala.

While some projects might have their own way of doing things, it’s certainly a good idea to study the Scala Style Guide to get familiar with common coding conventions.

Connecting
-----------------------------------------------------

By contributing to open source projects, you have already started connecting with the Scala community. However, you may not have the time to do that, or you may prefer other ways of connecting to like-minded people.

Try finding a local Scala user group or meetup. Scala Tribes provides an overview of Scala communities across the globe, and the Scala topic at Lanyrd keeps you up-to-date on any kind of Scala-related event, from conferences to meetups.

If you don’t like connecting in meatspace, the scala-user mailing list/Google group and the Scala IRC channel on Freenode may be good alternatives.

Other resources

Regardless of which of the paths outlined above you follow, there are a few resources I would like to recommend:

* Functional Programming in Scala by Paul Chiusano and Rúnar Bjarnason, which is currently available as an Early Access Edition and teaches you a lot more about functional programming and thinking about problems with a functional mindset.

* The Scala Documentation Site, which, for some reason, is not linked very prominently from the main Scala website, especially the available guides and tutorials.
    
*Resources for Getting Started With Functional Programming and Scala by Kelsey Innis contains a lot more helpful links about some of the topics we covered in this series.

Conclusion
------------------------------------------------------

I hope you have enjoyed this series and that I could get you excited about Scala. While this series is coming to an end, I seriously hope that it’s just the beginning of your journey through Scala land. Let me know in the comments how your journey went so far and where you think it will go from here.




