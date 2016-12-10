---
author: Kevin Wittek
categories:
- Testing
date: 2016-05-16T12:17:28Z
dsq_thread_id:
- "4831153365"
image: /wp-content/uploads/2016/05/5237117618_37f4df665e_b-1-e1463403139965.jpg
title: Using Spring-Boot 1.4 testing features with Spock
---

There was a [great blog post](https://spring.io/blog/2016/04/15/testing-improvements-in-spring-boot-1-4) over at the [spring.io blog](https://spring.io/blog) a couple of days ago, introducing the new testing improvements coming with Spring-Boot 1.4. I was very intrigued by these new upcoming features, but at the same time kind of sad, that Spock wasn't mentioned anywhere in the examples (at least you can find some general mentioning about using Spock for testing Spring-Boot applications in the [official documentation](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-testing-spring-boot-applications-with-spock)). In my humble opinion, using Spock to test your Spring-Boot application is a match made in heaven and since I'm a Spring-Boot as well as Spock fan, I thought I just might provide the examples of using Spock alongside the new Spring-Boot 1.4 testing features the original blog post was lacking.

I'll try to structure this post similar to the original post so you can skip back and forth between the two and checkout the differences in the examples. In order to integrate Spock with Spring (and Spring-Boot) you'll need this dependency:
`'org.spockframework:spock-spring:1.0-groovy-2.4'`

The `build.gradle` config looks like this (there are some additional dependencies for the example project):

https://gist.github.com/kiview/19fa9b7fa65219832a851d272b93d0cc#file-build-gradle

Testing without Spring
----------------------
The original post gave some great advice about unit testing your distinct Spring components: Don't involve Spring into this! Thanks to the magic of TDD and dependency injection this shouldn't be a big problem for the main business components of your application (assuming you've followed the practices of [The Clean Architecture](https://blog.8thlight.com/uncle-bob/2012/08/13/the-clean-architecture.html) and [Hexagonal Architecture](http://alistair.cockburn.us/Hexagonal+architecture)). Let's look at this example of a Spring `@Service` using some other `@Component` (there is no implementation difference between `@Service` and `@Component`, we're talking solely semantics here).

I think it's a shame that many source code examples and tutorials found in the world wide web use artificial and shallow use cases that are as far away from real world usage scenarios as JavaScript is from having a mature build system and so I've tried to come up with a useful example application. A friend once told me the process of cooking a sauce hollandaise is a quite difficult one, because you have to monitor the cooking temperature in a very precise fashion. And so this application represents a temperature monitoring system for sauce hollandaise consisting of a `HollandaiseTemperatureMonitor` service using some `Thermometer` component.

Test driving the class `HollandaiseTemperatureMonitor` I've come up with the following Spock tests (Since I'm an inhabitant of the part of the world which uses the metric system, all temperature units are in degree Celsius &#x1f609;):

https://gist.github.com/kiview/19fa9b7fa65219832a851d272b93d0cc#file-HollandaiseTemperatureMonitorSpec-groovy

This is an example of a pure unit test without involving any Spring dependencies whatsoever. There is still one interesting Spock feature to be found here. We create a Stub `Thermometer` by simply calling `Stub(Thermometer)` and instruct this stub to return the `givenTemperature` afterwards with this line:

`thermometer.currentTemperature() >> givenTemperature`

If you are somehow unfamiliar with the term *Stub* here is a great [article by Martin Fowler](http://martinfowler.com/articles/mocksArentStubs.html) going deep into the differences between Stubs, Mocks, Fakes and so on.

The corresponding production code of `HollandaiseTemperatureMonitor` that will make this tests pass looks like this:

https://gist.github.com/kiview/19fa9b7fa65219832a851d272b93d0cc#file-HollandaiseTemperatureMonitor-java

Integration tests
-----------------

So far we haven't seen any of the new Spring-Boot 1.4 testing features, so let's get to the cool stuff now. When building a Spring-Application I always like to have a really simple smoke test in place, to simply verify that the application starts without errors. This test may look like the following:

https://gist.github.com/kiview/19fa9b7fa65219832a851d272b93d0cc#file-ApplicationSpecWithoutAnnotation-groovy

Here you can see the new `@SpringBootTest` annotation at work, which promises to remove all the integration test boiler plate annotations which you needed prior to Spring-Boot 1.4, alas it is not compatible with Spock right now (see this [issue](https://github.com/spockframework/spock/issues/581)). I've already submitted a [pull request](https://github.com/spockframework/spock/pull/582) to add Spring-Boot 1.4 compatibility to Spock, but for now we have to use a workaround by explicitly using the `@ContextConfiguration` annotation:

https://gist.github.com/kiview/19fa9b7fa65219832a851d272b93d0cc#file-SpringBootSpockTestingApplicationSpecIT-groovy

If Spock finds the `@ContextConfiguration` on a class, it will assume this is a Spring test and will act accordingly and so this test will pass as expected.

Now let's move on to the next new testing feature, *testing application slices.*

Testing Application Slices
--------------------------

Introducing a new use case to our system, we might like to persist some statistic data of our hollandaise cooking process. And what could be a better place to store this data then a relational database? (The answer is nearly everything else, but I wanted some JPA component for the next example and so I had to come up with something...)

Spring-Boot 1.4 introduces some handy shortcuts for integration testing the persistence layer of your application, like the `@DataJpaTest` annotation. This annotation will instruct Spring to only initialize the components which are needed for the interaction with the persistence (specifically JPA) layer and so we might gain a faster startup time for our integrations tests.

I've written a really simply test to demonstrate this feature:

https://gist.github.com/kiview/19fa9b7fa65219832a851d272b93d0cc#file-HistoricTemperatureDataRepositorySpecIT-groovy

Please note that again I've had to add the `@ContextConfiguration` annotation or else Spock wouldn't identify the test as a Spring integration test. This is not mentioned in the official documentation and this behavior stems from the same [issue](https://github.com/spockframework/spock/issues/581) as before.

Summary
-------

Okay, that's it for now. I simply wanted to give some Spock examples for the new Spring-Boot 1.4 testing stuff the original blog post was lacking and I'm quite amused that I've discovered some missing Spock support along the way. I hope the proposed workarounds might help someone wondering why this features aren't working as described inside the documentation. Maybe I'll try to implement a more stable Spring discovery for the Spock-Spring module in the near future (for now I'm still waiting for my pull request to be merged &#x1f609;).

PS:
You can find the source code at [GitHub](https://github.com/kiview/spring-boot-with-spock). I've also changed to display the source code with the help of the [oEmbed Gist Plugin](https://wordpress.org/plugins/oembed-gist/) since syntax highlighting and support of markdown code fencing in Wordpress was living hell and I couldn't stand the broken encodings anymore...
I'm still not totally happy with the small column width of the code examples, I think I need to tweak the Wordpress theme in order to work better with source code examples.
