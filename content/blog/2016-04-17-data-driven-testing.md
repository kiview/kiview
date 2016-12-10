---
author: Kevin Wittek
categories:
- Testing
date: 2016-04-17T21:11:12Z
dsq_thread_id:
- "4755061354"
image: /wp-content/uploads/2016/04/dilbert-quality.png
title: Data Driven Testing
---

In my last blog post I've committed myself to writing one blog post every week and it seems I've already failed this goal, since there wasn't any post last week :(. But let us not hesitate in the face of yesterdays defeat and instead continue our quest towards a glorious test-driven string padding library in service of the whole interweb.

The first Spock blog post introduced some of the general concept but now I'd like to show you the really good stuff. One of my favorite Spock features is something called Data Driven Testing. This features allows to run the same tests for different input and expected output data - really useful to easily follow the DRY convention in your test classes and still cover all the edge cases of your production code. There are different ways to use this feature, but I think the easiest and in general most applicable one is to use the Data Tables feature.

Back in the [first Spock post](http://groovy-coder.com/?p=30) I've already suggested to extend our test method with additional input and output data in order to test drive the production code towards the real solution. The test method inside `PadServiceSpec` with applied Data Tables looks like this:


~~~groovy
@Unroll
def "padding string with content #nonPadded to width #totalWidth will result in '#padded' with width #padded.length()"() {
  given: "the service"
  PadService padService = new PadService()

  when: "given string is left padded to total width"
  String padResult = padService.leftPad(nonPadded, totalWidth)

  then: "resulting string has been padded with whitespace characters reach expected width"
  padResult == padded

  where:
  nonPadded | totalWidth || padded
  "foo"     | 6          || "   foo"
  "foobar"  | 6          || "foobar"
  "foobar"  | 8          || "  foobar"
  "foobar"  | -5         || "foobar"
  "baz"     | 2          || "baz"
  "baz"     | 0          || "baz"
  null      | 2          || null
}
~~~
All you have to do is define input and expected output in a table and Spock will automatically generate multiple distinct test cases for you, sweet! Also take a look at the `@Unroll` annotation - this annotation will tell Spock to report each parameterized run as a distinct result, substituting the `#variable` in the method name with the used parameter values.

If run with the old production code all tests but the first one will fail of course (I think I've defined the most useful permutations). So the next step is to use these tests as the specification for the production code and get into coding the feature until all tests pass.

The following code makes the tests results light up as green as the WindowsXP default wallpaper meadows:
```java
public class PadService {

  public String leftPad(String nonPadded, int totalWidth) {
      if (nonPadded == null) {
          /*
           * let's start a discussion if we should even
           * allow null values as valid input
           */
          return null;
      }

      StringBuilder builder = new StringBuilder(nonPadded);
      while (builder.length() < totalWidth) {
          builder.insert(0, " "); // workaround for missing prepend method
      }

      return builder.toString();
  }
}
```

And since IntelliJ IDEA has quite a good support for Spock, this is how the test report looks like inside this marvelous IDE:
![alt text](http://groovy-coder.com/wp-content/uploads/2016/04/2016-04-17-Test-Results.png "please overlook the last test message...")

Like the last time, the source code is available on [GitHub](https://github.com/kiview/java-testing-with-spock) and should be runnable directly after checkout, thanks to the cool included Gradle-Wrapper (And maybe I should upload this library into Maven-Central to make it includable as an external dependency in a wide variety of projects and other libraries :P).

PS:
Sorry for the missing syntax highlighting. It seems like the last WordPress (or Jetpack?) update broke the *SyntaxHighlighter Evolved* plugin. Any recommendations for good syntax highlight in Wordpress?