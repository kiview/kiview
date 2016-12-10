---
author: Kevin Wittek
categories:
- Testing
date: 2016-05-30T20:29:06Z
dsq_thread_id:
- "4870268742"
image: /wp-content/uploads/2016/05/24452135_d025b8002f_z.jpg
title: DataJpaTest in my Pocket
---

In my [last post](http://groovy-coder.com/?p=111) I've written about how to use the new *Spring-Boot* 1.4 test annotations in combination Spock for more boilerplate free test code. I've received some great feedback for this post (and it was even featured on some other blogs like [the official Spring blog](https://spring.io/blog/2016/05/17/this-week-in-spring-may-17th-2016), Petri Kainulainen's [great Spring and testing blog](http://www.petrikainulainen.net/weekly/java-testing-weekly-21-2016/), as well as Jacob Aae Mikkelsen's [Grails blog](http://grydeske.net/news/show/138)), for which I'm very grateful!) and I was also asked if I could provide an example of how to use `@DataJpaTest` in conjunction with multiple configured datasources. And so I thought great, that'll be the topic of my next post.

Little Pocket Monster Shop of Horrors
-------------------

Again I've tried to come up with some real world code examples, so let's assume we have some kind of online shop in which you can buy small monsters in order to let them fight for you. Since cross-platform is all the rage nowadays, it might be a good idea for our shop to offer monsters from different vendors, which might be stored in distinct persistence layers.

We have the following package layout:
![packages](http://groovy-coder.com/wp-content/uploads/2016/05/monstershop.png)

The `pokemon` as well as the `digimon` package are both configured to use *Spring-Data-Jpa* with their own datasource. The `PokemonConfig` looks like this:

```java
@Configuration
@EnableJpaRepositories(basePackageClasses = Pokemon.class,
        entityManagerFactoryRef = "pokemonEntityManager",
        transactionManagerRef = "pokemonTransactionManager")
public class PokemonConfig {

    @Bean
    @ConfigurationProperties(prefix = "datasource.pokemon")
    public DataSource pokemonDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean pokemonEntityManager() {
        HibernateJpaVendorAdapter jpaVendorAdapter = new HibernateJpaVendorAdapter();
        jpaVendorAdapter.setGenerateDdl(true);

        LocalContainerEntityManagerFactoryBean factoryBean = new LocalContainerEntityManagerFactoryBean();

        factoryBean.setDataSource(pokemonDataSource());
        factoryBean.setJpaVendorAdapter(jpaVendorAdapter);
        factoryBean.setPackagesToScan(this.getClass().getPackage().getName());
        factoryBean.setPersistenceUnitName("pokemon");

        return factoryBean;
    }

    @Bean
    public PlatformTransactionManager pokemonTransactionManager() {
        return new JpaTransactionManager(pokemonEntityManager().getObject());
    }

}
```
And the `DigimonConfig` looks really similar:

```java
@Configuration
@EnableJpaRepositories(basePackageClasses = Digimon.class,
        entityManagerFactoryRef = "digimonEntityManager",
        transactionManagerRef = "digimonTransactionManager")
public class DigimonConfig {

    @Bean
    @ConfigurationProperties(prefix = "datasource.digimon")
    public DataSource digimonDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean digimonEntityManager() {
        HibernateJpaVendorAdapter jpaVendorAdapter = new HibernateJpaVendorAdapter();
        jpaVendorAdapter.setGenerateDdl(true);

        LocalContainerEntityManagerFactoryBean factoryBean = new LocalContainerEntityManagerFactoryBean();

        factoryBean.setDataSource(digimonDataSource());
        factoryBean.setJpaVendorAdapter(jpaVendorAdapter);
        factoryBean.setPackagesToScan(this.getClass().getPackage().getName());
        factoryBean.setPersistenceUnitName("digimon");

        return factoryBean;
    }

    @Bean
    public PlatformTransactionManager digimonTransactionManager() {
        return new JpaTransactionManager(digimonEntityManager().getObject());
    }

}
```

I must admit, this was the first time I've used *Spring-Boot* and *Spring-Data-Jpa* in conjunction with multiple datasources and the setup wasn't all smooth sailing for me. The [official docs](http://docs.spring.io/spring-boot/docs/current/reference/html/howto-data-access.html#howto-two-datasources) are by no means bad, but the ride get's pretty bumpy once you've got to manually disable different *Spring-Boot* autoconfigure features (since it is not always 100% percent intuitive on which condition autoconfiguration of a specific feature is disabled automagically - and Oliver Gierke seems to agree on this [issue](https://github.com/spring-projects/spring-boot/issues/5541) as well &#x1f609;) and some aspects mentioned inside the docs - like the usage of `EntityManagerFactoryBuilder`; I somehow could not inject the builder class without having a `@Primary` annotated datasource inside my Spring context - didn't work for me at all. Oliver Gierke's *Spring-Data-Jpa* example project on [GitHub](https://github.com/spring-projects/spring-data-examples/tree/master/jpa/multiple-datasources) was a great help for me, since I could nearly copy-paste the `EntityManager` configuration code. Since I'm still kind of a Spring noob (I've started to use Spring in production with the advent of *Spring-Boot*, beforehand I was a zealous [Grails](https://grails.org/) crusader) it's always a tad bit scary to leave the green meadows of *Spring-Boot* autoconfiguration and venture forth into the dark depths of vanilla Spring. But nevertheless, it's almost consistently an enlightening and worthwhile experience once you've entangled the ominous stacktrace jungle.

@DataJpaTest
------------
After getting the general application setup, the `@DataJpaTest` were so darn straight forward and simple, it even feels a bit awkward to devote a blog post to this topic.

You simply have to annotate your tests with `@DataJpaTest` (and `@ContextConfiguration`, since *Spock* is still lacking full support for new *Spring-Boot* 1.4 test annotations) and you're good to go. Ah, it feels good to have the cozy *Spring-Boot* autoconfiguration magic back.

Here is an example test class:

```groovy
@ContextConfiguration // not mentioned by docs, but had to include this for Spock to startup the Spring context
@DataJpaTest
class PersistenceSpecIT extends Specification {

    @Autowired
    TestEntityManager testEntityManager

    @Autowired
    Pokedex pokedex

    @Autowired
    Digivice digivice

    def "fetches all persisted pokemon"() {
        given: "some pokemon"
        Pokemon bulbasaur = new Pokemon(1, "Bulbasaur", ["Grass", "Poison"])
        Pokemon charmander = new Pokemon(4, "Charmander", ["Fire"])

        and: "they are persisted"
        testEntityManager.persist(bulbasaur)
        testEntityManager.persist(charmander)

        when: "pokemon are fetched"
        List fetchedPokemon = pokedex.findAll().asList()

        then: "all persisted pokemon are returned"
        fetchedPokemon.size() == 2
        fetchedPokemon.containsAll([bulbasaur, charmander])
    }

    def "fetches all persisted digimon"() {
        given: "some digimon"
        Digimon agumon = new Digimon("Agumon", "Rookie", "Reptile", "Vaccine")
        Digimon greymon = new Digimon("Greymon", "Champion", "Dinosaur", "Vaccine")
        Digimon patamon = new Digimon("Patamon", "Rookie", "Mammal", "Data")

        and: "they are persisted"
        testEntityManager.persist(agumon)
        testEntityManager.persist(greymon)
        testEntityManager.persist(patamon)

        when: "digimon are fetched"
        List fetchedDigimon = digivice.findAll().asList()

        then: "all persisted digimon are returned"
        fetchedDigimon.size() == 3
        fetchedDigimon.containsAll([agumon, greymon, patamon])
    }

}
```

I'm not entirely sure why this does work without further configuration, but while digging into the *Spring-Boot* source code I've found the following comment:

> By default, tests annotated with @DataJpaTest will use an embedded in-memory database (replacing any explicit or usually auto-configured DataSource). The @AutoConfigureTestDatabase annotation can be used to override these settings.

So I think it might be safe to assume, these tests don't need to care whether you are using multiple datasources, or a single autoconfigured one, they seem to be quite decoupled from this part of the application configuration and I think we will get further information about this annotation once *Spring-Boot* 1.4 is finally released.

I've uploaded the sample project on [GitHub](https://github.com/kiview/MonsterShopExample/tree/master) and I'm looking forward to any comments and/or questions about this topic.