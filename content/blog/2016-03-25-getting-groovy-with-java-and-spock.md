---
author: Kevin Wittek
categories:
- Testing
date: 2016-03-25T16:24:29Z
dsq_thread_id:
- "4692793614"
image: /wp-content/uploads/2016/03/Leonard_Nimoy_Mr._Spock_Star_Trek-e1458922795206.jpg
tags:
- Gradle
- Groovy
- Java
- Spock
title: Getting Groovy with Java and Spock
---

Bug-Driven-Testing
------------------

Nowadays I simply love testing and my work colleagues call me nicknames such as *MC Test*. But I haven't always been this kind of a test zealot.
When I was introduced to Unit-Testing for the first time, back in university, I couldn't wrap my head around it. Why test the code I just wrote? I just wrote the code, I knew it was working. And I verified that it was working by using a truck load full of print statements and checking field values inside the debugger!

It became even more mysterious once the concept of Test-Driven-Development was introduced. Writing tests before writing the code? This was even more insane!
And so I remember many years of writing production code without a single line of test code...or at least without a single line of useful test code, of course there was the sporadic Unit-Test for my getters and setters.

But the urge to follow good development practices kept on itching and I remember writing my first more or less useful Unit- and Integration-Tests for a Grails Web-Application using JUnit and Grails' mocking and stubbing facilities (the difference between mocking and stubbing was completely unknown to me back then and my tests reflected this as well). And once the first bugs started to come ashore the TDD approach seemed to resonate with me. I started to adopt a technique I've coined Bug-Driven-Testing. Every time a bug was discovered I'd write a test that would evoke the buggy behaviour and so I could bugfix the code by developing against this new test.
It was at that point my testing spark was lit and not for long after this eye-opening experience, I discovered [Spock](https://spockframework.github.io/spock/docs/1.0/index.html) mentioned inside the Grails docs. Since then I've used Spock almost exclusively for all my testing needs and while my test-fu matured, so did Spock (which is available as version 1.0 since 2015-03-02).

Testing Java production code with Spock
---------------------------------------

Although Spock is a Groovy testing framework it's perfectly capable to test your Java code as well. In this post I'd like to give a small introduction into Spock.

Let's assume we want to implement a small library that provides some String padding functions ([this seems to be considered a really useful library inside the JavaScript community](http://www.theregister.co.uk/2016/03/23/npm_left_pad_chaos/)). As the build system I'd like to use [Gradle](http://gradle.org/). Normally I'd use Maven, since it still seems a bit more mature and sane, but since Groovy is a fist class citizen inside the Gradle ecosystem, setting up the project with Gradle is a tad bit easier (also the Gradle support in IntelliJ is getting more awesome with every release, while Maven is treated like a poor cousin).

The `build.gradle` file looks like this:
```groovy
group 'com.groovycoder'
version '1.0-SNAPSHOT'

apply plugin: 'java'
apply plugin: 'groovy'

repositories {
    mavenCentral()
}

dependencies {
    testCompile "org.codehaus.groovy:groovy-all:2.4.6"
    testCompile "org.spockframework:spock-core:1.0-groovy-2.4"
}
```

If you're already familiar with Gradle this is self-explanatory. If not, just take this for granted now. This config simply pulls in all Spock dependencies and allows you to combine Java production and Groovy test code. Next we want to get going with our first test, TDD style. The tests are called `Specification` in Spock and your test classes normally have a `*Spec.groovy` suffix. Assuming we want to code a class called `PadService`, this is what our first test case in `PadServiceSpec` might look like:
```groovy
class PadServiceSpec extends Specification {

    def "should add padding character to given string"() {
        given: "a string"
        String nonPadded = "foo"

        and: "the total width of the padded string"
        int totalWidth = 6

        and: "the service"
        PadService padService = new PadService()

        when: "given string is left padded to total width"
        String padded = padService.leftPad(nonPadded, totalWidth)

        then: "resulting string has been padded with whitespace characters until reaching expected width"
        padded == "   foo"
    }

}
```
In a glimpse you can identify this code as Groovy code, no semicolons, weee!
Spock follows a BDD-style test definition, mapping the arrange-act-assert test phases to given-when-then. Statements inside the `then` block are automatically evaluated as test conditions.

In order to make this test pass, we can write the simplest class possible like this (this time in Java):
```java
public class PadService {

    public String leftPad(String nonPadded, int totalWidth) {
        return "   foo";
    }
}
```

But of course, this code will fail for every other parameter except the ones used in the Spock test. Now we could start adding additional tests, which will perform the same operations with different values. Or we could wrap our tests with a for-loop and initialize an array of different input and expected output values (please don't!). Instead of having to do such cumbersome work, Spock provides a great solution for this use case, called [Data Driven Testing](https://spockframework.github.io/spock/docs/1.0/data_driven_testing.html). In my next blog post I'll show how to use Data Driven Testing in order to test drive the `PadService` to a working version. The source code (the little bit that is already existing...) is available at [GitHub](https://github.com/kiview/java-testing-with-spock).

So long, and happy hacking!
