---
layout: post
title: Spring + Hibernate + EhCache Caching
comments: true
---

In this post, we are going to demonstrate the Spring cache + EhCache feature on an example Spring Boot project. Caching will be defined as data queried from a relational database (example configurations prepared for H2 and PostgreSQL database engines).

## Application
Let's consider the database layer and application layer.

### Database Layer
The below diagram shows relationships between data tables. Our main object type is Company, which we will want to cache. Company is related to many other tables; some of the relationships are OneToMany, so querying the whole structure might be a time-consuming operation. 

![database-schema](https://raw.githubusercontent.com/adamzareba/company-structure-hibernate-cache/master/src/main/docs/db_schema.png)

### Application Layer

The test application is developed in Spring Boot + Hibernate + Flyway with an exposed REST API. To demonstrate data company operations, the following endpoints were created:

```java
@RestController
@RequestMapping("/company")
public class CompanyController {
    @Autowired
    private CompanyService companyService;
    @RequestMapping(method = RequestMethod.GET, produces = MediaType.APPLICATION_JSON_VALUE)
    @ResponseStatus(value = HttpStatus.OK)
    public @ResponseBody
    List<Company> getAll() {
        return companyService.getAll();
    }
    @RequestMapping(value = "/{id}", method = RequestMethod.GET, produces = MediaType.APPLICATION_JSON_VALUE)
    @ResponseStatus(value = HttpStatus.OK)
    public @ResponseBody
    Company get(@PathVariable Long id) {
        return companyService.get(id);
    }
    @RequestMapping(value = "/filter", method = RequestMethod.GET, produces = MediaType.APPLICATION_JSON_VALUE)
    @ResponseStatus(value = HttpStatus.OK)
    public @ResponseBody
    Company get(@RequestParam String name) {
        return companyService.get(name);
    }
    @RequestMapping(method = RequestMethod.POST, produces = MediaType.APPLICATION_JSON_VALUE)
    @ResponseStatus(value = HttpStatus.OK)
    public ResponseEntity<?> create(@RequestBody Company company) {
        companyService.create(company);
        HttpHeaders headers = new HttpHeaders();
        ControllerLinkBuilder linkBuilder = linkTo(methodOn(CompanyController.class).get(company.getId()));
        headers.setLocation(linkBuilder.toUri());
        return new ResponseEntity<>(headers, HttpStatus.CREATED);
    }
    @RequestMapping(method = RequestMethod.PUT, produces = MediaType.APPLICATION_JSON_VALUE)
    @ResponseStatus(value = HttpStatus.OK)
    public void update(@RequestBody Company company) {
        companyService.update(company);
    }
    @RequestMapping(value = "/{id}", method = RequestMethod.DELETE, produces = MediaType.APPLICATION_JSON_VALUE)
    @ResponseStatus(value = HttpStatus.OK)
    public void delete(@PathVariable Long id) {
        companyService.delete(id);
    }
}
```

## Cache Configuration
Let's see how to enable caching and work with cache regions.

### Enable caching

To enable the annotation-driven cache management capability in your Spring application, we need to add @EnableCaching annotation to configuration class. This annotation registers CacheInterceptor or AnnotationCacheAspect, which will detect cache annotations like @Cacheable, @CachePut, and @CacheEvict.

Spring comes with the cache manager interface org.springframework.cache.CacheManager, so we need to provide concrete implementation for cache storage. There’re [multiple implementations](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-caching.html#_supported_cache_providers), such as:

* Generic
* JCache
* EhCache 2.x
* Hazelcast
* Infinispan
* Couchbase
* Redis
* Caffeine

In this post, we are going to use the EhCache provider. The below example shows cache enabling with EhCache-related beans in a separate configuration class. Overriding these two beans is not needed if you want to stay with the default definition, but we wanted to make cache transactions aware to synchronize put/evict operations with ongoing Spring-managed transactions.

```java
@Configuration
@EnableCaching(mode = AdviceMode.ASPECTJ)
public class CacheConfiguration {
    @Bean
    public EhCacheManagerFactoryBean ehCacheManagerFactory() {
        EhCacheManagerFactoryBean cacheManagerFactoryBean = new EhCacheManagerFactoryBean();
        cacheManagerFactoryBean.setConfigLocation(new ClassPathResource("ehcache.xml"));
        cacheManagerFactoryBean.setShared(true);
        return cacheManagerFactoryBean;
    }
    @Bean
    public EhCacheCacheManager ehCacheCacheManager() {
        EhCacheCacheManager cacheManager = new EhCacheCacheManager();
        cacheManager.setCacheManager(ehCacheManagerFactory().getObject());
        cacheManager.setTransactionAware(true);
        return cacheManager;
    }
}
```

### Cache Regions
Cached data could be stored in separate regions and we can define individual configurations for cache items with an XML file. Let’s define two regions:

1. Storing companies by ID.
2. Storing companies by name.
For both regions, the maximum elements kept in memory is 10,000 and the maximum time for the company before it's invalidated is 60 minutes (for more detailed configuration, go to [EhCache Official Reference](http://www.ehcache.org/ehcache.xml)).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE ehcache>
<ehcache>
    <diskStore path="java.io.tmpdir"/>
    <cache name="company.byId"
           maxElementsInMemory="10000" eternal="false" timeToIdleSeconds="600"
           timeToLiveSeconds="3600" overflowToDisk="true"/>
    <cache name="company.byName"
           maxElementsInMemory="10000" eternal="false" timeToIdleSeconds="600"
           timeToLiveSeconds="3600" overflowToDisk="true"/>
</ehcache>
```

## Cache Operations
Let's look at populating with @Cacheable, invaldating with @CacheEvict, and updating with @CachePut.

### Populate: @Cacheable
The [@Cacheable](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/cache/annotation/Cacheable.html) annotation indicates that the result of invoking a method (or all methods in a class) can be cached. Each time an advised method is invoked, the caching behavior will be applied, checking whether the method was already invoked for the given arguments.
   
In the example below, we want to cache Company objects in the company.byId cache region. The key in our region is the Company ID field. To handle something other than cache example test data (name starting with test), we can add a condition based on the result object (object returned as a result of a method).

```java
@Cacheable(value = "company.byId", key = "#id", unless = "#result != null and #result.name.toUpperCase().startsWith('TEST')")
public Company find(Long id) {
    CriteriaBuilder builder = entityManager.getCriteriaBuilder();
    CriteriaQuery<Company> query = builder.createQuery(Company.class);
    Root<Company> root = query.from(Company.class);
    root.fetch(Company_.cars, JoinType.LEFT);
    Fetch<Company, Department> departmentFetch = root.fetch(Company_.departments, JoinType.LEFT);
    Fetch<Department, Employee> employeeFetch = departmentFetch.fetch(Department_.employees, JoinType.LEFT);
    employeeFetch.fetch(Employee_.address, JoinType.LEFT);
    departmentFetch.fetch(Department_.offices, JoinType.LEFT);
    query.select(root).distinct(true);
    Predicate idPredicate = builder.equal(root.get(Company_.id), id);
    query.where(builder.and(idPredicate));
    return DataAccessUtils.singleResult(entityManager.createQuery(query).getResultList());
}
```

To better see what’s happening, we can turn on debug-level logging for Hibernate:

```
logging.level.org.hibernate.SQL=debug
```

...and invoke REST company endpoint:

```
curl http://localhost:9000/company/1
```

Server logs will show the below SQL query only once:

```log
2018-01-19 10:00:34.143 DEBUG 20256 --- [nio-8080-exec-4] org.hibernate.SQL                        : 
    select
        distinct company0_.id as id1_2_0_,
        cars1_.id as id1_1_1_,
        department2_.id as id1_3_2_,
        employees3_.id as id1_4_3_,
        address4_.id as id1_0_4_,
        offices5_.id as id1_5_5_,
        company0_.name as name2_2_0_,
        cars1_.company_id as company_3_1_1_,
        cars1_.registration_number as registra2_1_1_,
        cars1_.company_id as company_3_1_0__,
        cars1_.id as id1_1_0__,
        department2_.company_id as company_3_3_2_,
        department2_.name as name2_3_2_,
        department2_.company_id as company_3_3_1__,
        department2_.id as id1_3_1__,
        employees3_.address_id as address_4_4_3_,
        employees3_.department_id as departme5_4_3_,
        employees3_.name as name2_4_3_,
        employees3_.surname as surname3_4_3_,
        employees3_.department_id as departme5_4_2__,
        employees3_.id as id1_4_2__,
        address4_.house_number as house_nu2_0_4_,
        address4_.street as street3_0_4_,
        address4_.zip_code as zip_code4_0_4_,
        offices5_.address_id as address_3_5_5_,
        offices5_.department_id as departme4_5_5_,
        offices5_.name as name2_5_5_,
        offices5_.department_id as departme4_5_3__,
        offices5_.id as id1_5_3__ 
    from
        company company0_ 
    left outer join
        car cars1_ 
            on company0_.id=cars1_.company_id 
    left outer join
        department department2_ 
            on company0_.id=department2_.company_id 
    left outer join
        employee employees3_ 
            on department2_.id=employees3_.department_id 
    left outer join
        address address4_ 
            on employees3_.address_id=address4_.id 
    left outer join
        office offices5_ 
            on department2_.id=offices5_.department_id 
    where
        company0_.id=1
```

The next time the repository method is invoked, data will be gathered from the cache.

The example below shows an implementation for caching data based on the name.

```java
@Cacheable(value = "company.byName", key = "#name", unless = "#name != null and #name.toUpperCase().startsWith('TEST')")
public Company find(String name) {
    CriteriaBuilder builder = entityManager.getCriteriaBuilder();
    CriteriaQuery<Company> query = builder.createQuery(Company.class);
    Root<Company> root = query.from(Company.class);
    root.fetch(Company_.cars, JoinType.LEFT);
    Fetch<Company, Department> departmentFetch = root.fetch(Company_.departments, JoinType.LEFT);
    Fetch<Department, Employee> employeeFetch = departmentFetch.fetch(Department_.employees, JoinType.LEFT);
    employeeFetch.fetch(Employee_.address, JoinType.LEFT);
    departmentFetch.fetch(Department_.offices, JoinType.LEFT);
    query.select(root).distinct(true);
    Predicate idPredicate = builder.equal(root.get(Company_.name), name);
    query.where(builder.and(idPredicate));
    return DataAccessUtils.singleResult(entityManager.createQuery(query).getResultList());
}
```

### Invalidate: @CacheEvict

The [@CacheEvict](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/cache/annotation/CacheEvict.html) annotation indicates that a method (or all methods on a class) triggers a cache evict operation, removing only specific or removing all items from the cache region.

Since the code below removes data from the database, the object needs to be removed from both caches:

```java
@Caching(evict = {@CacheEvict(value = "company.byId", key = "#company.id"), @CacheEvict(value = "company.byName", key = "#company.name")})
public void delete(Company company) {
    entityManager.remove(company);
}
```

### Update: @CachePut
With the [@CachePut](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/cache/annotation/CachePut.html) annotation, it’s possible to update the cache. In the below example, the cache for storing companies by ID is updated. It’s worth noting that since the company name is mutable, we cannot update the cache because we don’t know the old name value for that company. To proceed with that, we remove all entries in the cache for companies by name.

```java
@Caching(evict = {@CacheEvict(value = "company.byName", allEntries = true)},
        put = {@CachePut(value = "company.byId", key = "#result.id", unless = "#result != null and #result.name.toUpperCase().startsWith('TEST')")})
public Company update(Company company) {
    return entityManager.merge(company);
}
```

## Cache Statistics

To preview live cache, statistics it is possible to expose EhCache MBeans through JMX like below:

```java
@Configuration
@Dev
public class CacheMonitoring {
    @Autowired
    private EhCacheCacheManager ehCacheCacheManager;
    @Bean
    public MBeanServer mBeanServer() {
        MBeanServer mBeanServer = ManagementFactory.getPlatformMBeanServer();
        return mBeanServer;
    }
    @Bean
    public ManagementService managementService() {
        ManagementService managementService = new ManagementService(ehCacheCacheManager.getCacheManager(), mBeanServer(), true, true, true, true);
        managementService.init();
        return managementService;
    }
}
```

The following MBeans will be exposed:

* CacheManager
* Cache
* CacheConfiguration
* CacheStatistics

![jconsole](https://dzone.com/storage/temp/7932520-jconsole.png)

## Summary
In this post, we covered basic cache operations like getting, inserting, removing, and updating. Defining these operations and more complex requirements (like conditional caching or cache synchronization) is straightforward with annotations.

The source code for above listings can be found in the GitHub project [company-structure-hibernate-cache](https://github.com/adamzareba/company-structure-hibernate-cache).
