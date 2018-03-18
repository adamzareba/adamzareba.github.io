---
layout: post
title: Cache OAuth 2 in Spring With Redis
comments: true
---

In this post, we are going to demonstrate the caching mechanism for Spring + OAuth2 using [Redis](https://redis.io/) for cache storage. Before you start, you should familiarize yourself with OAuth2 fundamentals. 

## Application
In my previous post, [Secure Spring REST With Spring Security and OAuth2](https://adamzareba.github.io/Secure-Spring-REST-With-Spring-Security-and-OAuth2/), we developed simple Spring Boot application with OAuth 2 that we’re going to use as a starting point for this post (for the purposes of this post, I modified a little bit of the application by defining the relationship between entities like User, Role, and Permission, so we are not storing direct relationships between User and GrantedAuthority in the database anymore).

In our example application, we have the following structure for our business data:

![db-schema](https://dzone.com/storage/temp/8350382-rel.png)

We are able to access this structure via secured endpoints:

```java
@RestController
@RequestMapping("/secured/company")
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

## Collect Possible Performance Issues
First of all, we will have to check oauth2-related database operations for endpoints:

* /oauth/token (endpoint for getting access token)
* /secured/company/{companyId} (example of a secured endpoint)

### Get Access Token
If we run our endpoint to get an access token:

```text
curl -X POST \
  http://localhost:8080/oauth/token \
  -H 'authorization: Basic c3ByaW5nLXNlY3VyaXR5LW9hdXRoMi1yZWFkLXdyaXRlLWNsaWVudDpzcHJpbmctc2VjdXJpdHktb2F1dGgyLXJlYWQtd3JpdGUtY2xpZW50LXBhc3N3b3JkMTIzNA==' \
  -H 'content-type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW' \
  -F grant_type=password \
  -F username=admin \
  -F password=admin1234 \
  -F client_id=spring-security-oauth2-read-write-client
```

The above code result int the following queries being executed:

```sql
select client_id, client_secret, resource_ids, scope, authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity, additional_information, autoapprove from oauth_client_details where client_id = ?
select client_id, client_secret, resource_ids, scope, authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity, additional_information, autoapprove from oauth_client_details where client_id = ?
select client_id, client_secret, resource_ids, scope, authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity, additional_information, autoapprove from oauth_client_details where client_id = ?
select client_id, client_secret, resource_ids, scope, authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity, additional_information, autoapprove from oauth_client_details where client_id = ?
select token_id, token from oauth_access_token where authentication_id = ?
select client_id, client_secret, resource_ids, scope, authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity, additional_information, autoapprove from oauth_client_details where client_id = ?
select client_id, client_secret, resource_ids, scope, authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity, additional_information, autoapprove from oauth_client_details where client_id = ?
select client_id, client_secret, resource_ids, scope, authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity, additional_information, autoapprove from oauth_client_details where client_id = ?
select token_id, token from oauth_access_token where token_id = ?
insert into oauth_access_token (token_id, token, authentication_id, user_name, client_id, authentication, refresh_token) values (?, ?, ?, ?, ?, ?, ?)
insert into oauth_refresh_token (token_id, token, authentication) values (?, ?, ?)
```

As you can see, the same query was executed 7 times in one request:

```sql
select client_id, client_secret, resource_ids, scope, authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity, additional_information, autoapprove from oauth_client_details where client_id = ?
```

JdbcClientDetailsService.loadClientByClientId(String clientId) is a service that executes queries to the database. Since it’s an implementation that uses the database, we should prepare a more generic solution that would cache data even if the source of that data is not a SQL database. To proceed with that, we can cache results of the interface that needs to be implemented to load the client by its id:

* [ClientDetailsService.loadClientByClientId(String clientId)](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/provider/ClientDetailsService.html#loadClientByClientId(java.lang.String))

### Get Access Token Based on Refresh Token
To get access_token based on refresh_token we use the following code:

```text
curl -X POST \
  http://localhost:8080/oauth/token \
  -H 'authorization: Basic c3ByaW5nLXNlY3VyaXR5LW9hdXRoMi1yZWFkLXdyaXRlLWNsaWVudDpzcHJpbmctc2VjdXJpdHktb2F1dGgyLXJlYWQtd3JpdGUtY2xpZW50LXBhc3N3b3JkMTIzNA==' \
  -H 'content-type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW' \
  -F grant_type=refresh_token \
  -F refresh_token=REFRESH_TOKEN
```

The following queries are then executed:

```sql
select client_id, client_secret, resource_ids, scope, authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity, additional_information, autoapprove from oauth_client_details where client_id = ?
select client_id, client_secret, resource_ids, scope, authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity, additional_information, autoapprove from oauth_client_details where client_id = ?
select client_id, client_secret, resource_ids, scope, authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity, additional_information, autoapprove from oauth_client_details where client_id = ?
select client_id, client_secret, resource_ids, scope, authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity, additional_information, autoapprove from oauth_client_details where client_id = ?
select token_id, token from oauth_refresh_token where token_id = ?
select token_id, authentication from oauth_refresh_token where token_id = ?
delete from oauth_access_token where refresh_token = ?
select client_id, client_secret, resource_ids, scope, authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity, additional_information, autoapprove from oauth_client_details where client_id = ?
select token_id, token from oauth_access_token where token_id = ?
insert into oauth_access_token (token_id, token, authentication_id, user_name, client_id, authentication, refresh_token) values (?, ?, ?, ?, ?, ?, ?)
```

The below query was executed 5 times in one request:

```sql
select client_id, client_secret, resource_ids, scope, authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity, additional_information, autoapprove from oauth_client_details where client_id = ?
```

However, we have already covered the caching of this query.

### Get a Secured Resource
To get an example of a secured resource, we have to send an authentication token:

```text
curl -X GET \
  http://localhost:8080/secured/company/1 \
  -H 'authorization: Bearer ACCESS_TOKEN'
```

During token validation, the following queries are executed for each request:

```sql
select token_id, token from oauth_access_token where token_id = ?
select token_id, authentication from oauth_access_token where token_id = ?
```

Since token_id is usually the same for a long time, we might cache these values. Below are implementations of these respective queries:

* JdbcTokenStore.readAccessToken(String tokenValue)
* JdbcTokenStore.readAuthentication(String token)

Like in the previous example, we should cache the invocation of methods on the interface level:

* [TokenStore.readAccessToken(String tokenValue)](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/provider/token/TokenStore.html#readAccessToken(java.lang.String))
* [TokenStore.readAuthentication(String token)](https://docs.spring.io/spring-security/oauth/apidocs/org/springframework/security/oauth2/provider/token/TokenStore.html#readAuthentication(java.lang.String))

### Refresh Client Data
We are not going to demonstrate all the cache operations that should be taken into account during the caching of OAuth2 data. We should remember to properly handle token invalidation or client deletion.

## Cache Service Data
Since we want to define caching on OAuth2 library level services, we would like to inject caching without touching the OOTB code. We can use AOP to define pointcuts where a caching mechanism should be injected.

### Enable Caching
To enable cache management capabilities in a Spring application, we need to add the @EnableCaching annotation to the configuration class. In this post, we are going to use a Redis provider to store the cache. The below example shows cache enabling with Redis related beans in a separate configuration class. Overriding these beans is not needed if you want to stay with a default Spring Boot setup.

```java
@Configuration
@EnableCaching
@PropertySource("classpath:/redis.properties")
public class CacheConfiguration {
    @Value("${redis.host}")
    private String redisHost;
    @Value("${redis.port}")
    private int redisPort;
    @Bean
    public JedisConnectionFactory redisConnectionFactory() {
        JedisConnectionFactory redisConnectionFactory = new JedisConnectionFactory();
        redisConnectionFactory.setHostName(redisHost);
        redisConnectionFactory.setPort(redisPort);
        return redisConnectionFactory;
    }
    @Bean
    public RedisTemplate<?, ?> redisTemplate() {
        RedisTemplate<?, ?> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory());
        return template;
    }
    @Bean
    public RedisCacheManager redisCacheManager() {
        RedisCacheManager cacheManager = new RedisCacheManager(redisTemplate());
        cacheManager.setUsePrefix(true);
        cacheManager.setDefaultExpiration(240);
        return cacheManager;
    }
}
```

### Cache Operations
We are going to use declarative XML-based caching to do this, rather than modifying our OAuth2 classes. We will create a separate configuration that imports the XML file.

```java
@Configuration
@ImportResource({"classpath:oauth2-cache.xml"})
public class OAuth2CacheConfig {
}
```

In the XML file, we have to define caching behaviors. In order to cache following methods:

* loadClientByClientId
* readAuthentication
* readAccessToken
The below definitions need to be created:

```xml
<!-- define caching behavior -->
<cache:advice id="clientById" cache-manager="redisCacheManager">
    <cache:caching cache="OAuthClientDetailsServiceCache">
        <cache:cacheable method="loadClientByClientId" key="#clientId"/>
    </cache:caching>
</cache:advice>
<cache:advice id="authByTokenId" cache-manager="redisCacheManager">
    <cache:caching cache="OAuthTokenStoreReadAuthenticationCache">
        <cache:cacheable method="readAuthentication" key="#token.toString()" />
    </cache:caching>
</cache:advice>
<cache:advice id="tokenById" cache-manager="redisCacheManager">
    <cache:caching cache="OAuthTokenByIdCache">
        <cache:cacheable method="readAccessToken" key="#tokenValue" unless="#result == null"/>
    </cache:caching>
</cache:advice>
```

For the above definition, we have to apply a cacheable behavior to all implementations of the below interfaces:

```xml
<!-- apply the cacheable behavior to all interface methods -->
<aop:config>
    <aop:advisor advice-ref="clientById" pointcut="execution(* org.springframework.security.oauth2.provider.ClientDetailsService.*(..))"/>
    <aop:advisor advice-ref="authByTokenId" pointcut="execution(* org.springframework.security.oauth2.provider.token.TokenStore.*(..)) "/>
    <aop:advisor advice-ref="tokenById" pointcut="execution(* org.springframework.security.oauth2.provider.token.TokenStore.*(..))"/>
</aop:config>
```

## Summary
In this blog post, we showed OAuth2 caching with Spring and Redis.

The source code for the above listings can be found in the GitHub project [company-structure-spring-security-oauth2-cache](https://github.com/adamzareba/company-structure-spring-security-oauth2-cache) (in the ehcache branch, you will find the configuration for storing cache in EhCache).
