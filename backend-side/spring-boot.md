# Spring Boot

[Spring Boot](https://spring.io/projects/spring-boot) is an open source Java-based framework made to ease the bootstrapping and development of the Spring applications. It is developed by Pivotal, and is a part of the Spring Framework. It gives you easy access to rich Spring ecosystem, with many enterprise-grade libraries and technologies. Spring Boot applications can be written in Kotlin and the language is officially supported by the framework.

## Build configuration

Kilua RPC provides a single module for Spring Boot, `kilua-rpc-spring-boot`, which uses Spring dependency injection to access services implementations. You need to add this module to your project.

```kotlin
val commonMain by getting {
    dependencies {
        implementation("dev.kilua:kilua-rpc-spring-boot:$kiluaRpcVersion")
    }
}
```

## Service implementation

### Service class

The implementation of the service class comes down to implementing required interface methods and making it a Spring component by using a `@Service` annotation with a "prototype" scope.

```kotlin
@Service
@Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE)
class AddressService : IAddressService {
    override suspend fun getAddressList(search: String?, sort: Sort) {
        return listOf()
    }
    override suspend fun addAddress(address: Address) {
        return Address()
    }
    override suspend fun updateAddress(id: Int, address: Address) {
        return Address()
    }
    override suspend fun deleteAddress(id: Int) {
        return false
    }
}
```

### Injecting server objects

Spring IoC (Inversion of Control) allows you to inject resources and other Spring components into your service class. You can use standard Spring `@Autowired` annotation or constructor parameter injection.

Kilua RPC allows you to inject `ServerRequest` and `HeadersBuilder<BodyBuilder>` objects, which give you access to the request parameters, user session and response headers.

<pre class="language-kotlin"><code class="lang-kotlin"><strong>@Service
</strong>@Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE)
class AddressService : IAddressService {

    @Autowired
    lateinit var serverRequest: ServerRequest

    override suspend fun getAddressList(search: String?, sort: Sort) {
        println(serverRequest.uri())
        println(serverRequest.session().awaitSingle().id)
        return listOf()
    }
}
</code></pre>

{% hint style="info" %}
Note: The new instance of the service class will be created by Spring for every server request. Use session or request objects to store your state with appropriate scope.
{% endhint %}

### **Blocking code**

Since Spring WebFlux architecture is asynchronous and non-blocking, you should **never** block a thread in your application code. If you have to use some blocking code (e.g. blocking I/O, JDBC) always use the dedicated coroutine dispatcher.

```kotlin
@Service
@Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE)
class AddressService : IAddressService {
    override suspend fun getAddressList(search: String?, sort: Sort) {
        return withContext(Dispatchers.IO) {
            retrieveAddressesFromDatabase(search, sort)
        }
    }
}
```

## Application configuration

### The application class

To allow Kilua RPC work with Spring Boot you have to pass all instances of the `RpcServiceManager` objects (defined in common code) to the Spring environment. You do this by defining a provider method for the `List<RpcServiceManager<Any>>` instance in the main application class. You can use `getAllServiceManagers()` method to simplify your code.

```kotlin
import dev.kilua.rpc.getAllServiceManagers
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.context.annotation.Bean

@SpringBootApplication
class RpcApplication {
    @Bean
    fun getManagers() = getAllServiceManagers()
}

fun main(args: Array<String>) {
    runApplication<RpcApplication>(*args)
}
```

### Security

You can use standard Spring WebFlux Security configuration, and with a help of `serviceMatchers` extension function, you can automatically select endpoints that should be secured.

```kotlin
@EnableWebFluxSecurity
@Configuration
class SecurityConfiguration {

    @Bean
    fun securityWebFilterChain(http: ServerHttpSecurity): SecurityWebFilterChain {
        return http.authorizeExchange {
            it.serviceMatchers(getServiceManager<IAddressService>(), getServiceManager<IProfileService>())
                .authenticated().pathMatchers("/**").permitAll()
        }.csrf {
            it.disable()
        }.exceptionHandling {
            it.authenticationEntryPoint { exchange, _ ->
                val response = exchange.response
                response.statusCode = HttpStatus.UNAUTHORIZED
                exchange.mutate().response(response)
                Mono.empty()
            }
        }.formLogin {
            it.loginPage("/login")
                .authenticationSuccessHandler(RedirectServerAuthenticationSuccessHandler().apply {
                    this.setRedirectStrategy { exchange, _ ->
                        Mono.fromRunnable {
                            val response = exchange.response
                            response.statusCode = HttpStatus.OK
                        }
                    }
                }).authenticationFailureHandler(RedirectServerAuthenticationFailureHandler("/login").apply {
                    this.setRedirectStrategy { exchange, _ ->
                        Mono.fromRunnable {
                            val response = exchange.response
                            response.statusCode = HttpStatus.UNAUTHORIZED
                        }
                    }
                })
        }.logout {
            it.logoutUrl("/logout")
                .requiresLogout(ServerWebExchangeMatchers.pathMatchers(HttpMethod.GET, "/logout"))
                .logoutSuccessHandler(RedirectServerLogoutSuccessHandler().apply {
                    setLogoutSuccessUrl(URI.create("/"))
                })
        }.build()
    }
}
```

