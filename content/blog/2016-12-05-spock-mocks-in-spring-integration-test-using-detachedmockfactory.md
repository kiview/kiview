---
author: Kevin Wittek
categories:
- Spring
- Testing
date: 2016-12-05T15:30:31Z
dsq_thread_id:
- "5356804595"
title: Spock Mocks in Spring Integration-Test using DetachedMockFactory
---

Spring-Boot introduced some cool testing improvements in Version 1.4 - sadly not all readily available for usage in Spock. A nice feature, that allows for easy mocking and stubbing of `Beans` inside of integration tests is the new `@MockBean` annotation. This annotation will automagically replace the annotated bean with a Mockito mock. All you need to do is annotate a field with the corresponding bean type inside your `@SpringBootTest` annotated test class, just like this:

```java
@MockBean
private MyBean bean;
``` 

There was a lot of [discussion](https://github.com/spockframework/spock/issues/624) inside Spock's issue tracker about using this feature with Spock, but it seems, we won't be able to make this work inside the _spock-spring_ plugin (the best solution seems to write a custom Spock extension which replicates this feature inside the Spock environment - any volunteers?)

But don't give way to despair yet, there is indeed an okayish workaround, using the new `DetachedMockFactory` which was introduced in Spock 1.1. If used in conjunction with the new `@TestConfiguration` annotation from Spring, you get some really nice mocking capabilities for your integration testing needs. Let's look at some example code now!

This time I've thought it would be nice to use everyone's favorite and totally close to reality example application - the famous book store. This is the production code of our `BookService` class, which implements our business logic (the `listAvailableBooks()` method):

```java
@Service
public class BookService {

    private final BookRepository bookRepository;

    public BookService(BookRepository bookRepository) {
        this.bookRepository = bookRepository;
    }

    public Collection<Book> listAvailableBooks() {
        Collection<Book> existingBooks = bookRepository.findAll();

        Collection<Book> availableBooks = existingBooks.stream()
                .filter(Book::isAvailable).collect(Collectors.toList());

        return availableBooks;
    }

}
```

Now let's assume we want to test this class inside an integration test, but we don't want to hit the real persistence layer and so need to stub away the `BookRepository` bean (in the real world, using an integration test for this, does not make any sense at all, we could test the class using solely unit tests just fine). In order to inject the stubbed bean into our system under test, we can simply define a nested configuration class:

```groovy
@TestConfiguration
static class Config {
    private DetachedMockFactory factory = new DetachedMockFactory()

    @Bean
    BookRepository bookRepository() {
        factory.Stub(BookRepository, name: "bookRepository")
    }
}
```

Afterwards, we can use the `BookRepository` bean inside our Spock tests just like any other stub object:

```groovy
def "returns available books"() {
    given: "some books"
    stubBookRepository.findAll() >> [
            new Book("The Color of Magic", "T. P.", true),
            new Book("There and Back Again", "B. B.", true),
            new Book("Do Androids Dream of Electric Sheep?", "P. K. D.", false),
    ]

    when: "retrieving available books"
    def availableBooks = bookService.listAvailableBooks()

    then: "there a some books available"
    availableBooks.size() == 2
}
```

A curious person might ask, why we need to use a `DetachedMockFactory` after all? Couldn't we simply use some code like `BookRepository repoStub = Stub(BookRepository)`? In order to understand why this doesn't work, you need to understand how Spock's mocking/stubbing works. I'm not sure if _I_ really understand how mocking/stubgging works in Spock, but in this case it's enough to know, that you'll only be able to create mock/stub objects if you are inside a specification. This would lead to problems when using the `@SpringBootTest` annotation in order to initialize the Spring context, since Spring will try to boot up before any specification code has been run. Gladly we now have the capabilities of `DetachedMockFactory` at our fingertips, which allows us to create mocks/stub outside a specification. Here is the complete test class:

```groovy
@SpringBootTest
class BookServiceTests extends Specification {

    @Autowired
    BookService bookService

    @Autowired
    BookRepository stubBookRepository

    def "returns no books if no books available"() {
        given: "no books"
        stubBookRepository.findAll() >> []

        when: "retrieving available books"
        def availableBooks = bookService.listAvailableBooks()

        then: "there a no books available"
        availableBooks.isEmpty()
    }

    def "returns available books"() {
        given: "some books"
        stubBookRepository.findAll() >> [
                new Book("The Color of Magic", "T. P.", true),
                new Book("There and Back Again", "B. B.", true),
                new Book("Do Androids Dream of Electric Sheep?", "P. K. D.", false),
        ]

        when: "retrieving available books"
        def availableBooks = bookService.listAvailableBooks()

        then: "there a some books available"
        availableBooks.size() == 2
    }

    @TestConfiguration
    static class Config {
        private DetachedMockFactory factory = new DetachedMockFactory()

        @Bean
        BookRepository bookRepository() {
            factory.Stub(BookRepository, name: "bookRepository")
        }
    }

}

```

I hope this solution will lift some of the sadness which might have risen inside the hearts of passionate Spock users once they've realised some of the new Spring-Boot 1.4 testing improvements won't be easily available for them. And of course you should prefer to use unit tests after all (an environment in which Spock shines even more).

The source code of the example project is available at [Github](https://github.com/kiview/spring-spock-mock-beans-demo).