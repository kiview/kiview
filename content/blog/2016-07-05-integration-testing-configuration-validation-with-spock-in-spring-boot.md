---
author: Kevin Wittek
categories:
- Testing
date: 2016-07-05T20:28:14Z
dsq_thread_id:
- "4963288794"
tags:
- Groovy
- Java
- Spock
title: Integration testing configuration validation with Spock in Spring-Boot
---

This is something I stumbled upon during my regular workday. We wanted to test-drive the validation of our externalized Spring-Boot configuration. It was obvious that we needed an integration test for this feature, since the Spring-Boot framework, as well as the file system, are heavy involved in this kind of test. But what condition did we want to check in order to determine if the test had successfully passed? We decided that our application should not start if misconfigured and as a consequence, we wanted the test to ensure, that the application hasn't started.

Our first approached included using a regular Spring-Boot integration test, by using the `@SpringApplicationConfiguration` annotation. Sadly this route proved to be a blind alley, since Spock's Spring integration tries to initialize the `ApplicationContext` prior to testing and this rightfully failed!

And so we had to run the test inside a regular unit test and manually initialize the Spring application inside the method. To be honest, I wasn't really sure about how to initialize the `ApplicationContext` or how to feed the configuration into the test. Luckily we've stumbled about a very good official Spring-Boot [sample project](https://github.com/spring-projects/spring-boot/tree/master/spring-boot-samples/spring-boot-sample-property-validation). Based on this code, we came up with an solution like the following.

This is the class backing the external configuration:
```groovy
@Component
@ConfigurationProperties("sample")
public class SampleConfigurationProperties {

    @Min(0)
    private int size;

    public int getSize() {
        return size;
    }

    public void setSize(int size) {
        this.size = size;
    }
}
```

And this is a possible integration test:

```groovy
class SampleConfigurationPropertiesSpecIT extends Specification {

    def "does crash if validation of external properties fails"() {

        given: "the context"
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext()
        context.register(SpringBootConfigurationValidationTestingApplication)

        when: "setting invalid config"
        EnvironmentTestUtils.addEnvironment(context, "sample.size:-1")
        context.refresh()

        then: "application crashes"
        thrown Exception
    }

}
```

As you can see, this is a normal Spock unit test (although it's semantically still an integration test). The `ApplicationContext` is initialized as an `AnnotationConfigApplicationContext` and the `EnvironmentTestUtils` are used to inject an invalid external configuration into the `ApplicationContext`.

There might be other use cases in which one might like to manually initialize the `ApplicationContext` instead of relying on Spock to setup the integration test environment, so I thought this post could provide to be usefull for future generation of Spring-Boot hackers to come. You can find my complete sample project on [GitHub](https://github.com/kiview/spring-boot-configuration-validation-testing).
