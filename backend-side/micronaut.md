# Micronaut

[Micronaut](https://micronaut.io) is a modern, JVM-based, fullstack framework for building modular, easily testable microservice and serverless applications. Micronaut provides a simple compile-time aspect-oriented programming API, which is very similar to Spring, but does not use reflection.

## Build configuration

Kilua RPC provides a single module for Micronaut, `kilua-rpc-micronaut`, which uses Micronaut dependency injection to access services implementations. You need to add this module to your project.

```kotlin
val commonMain by getting {
    dependencies {
        implementation("dev.kilua:kilua-rpc-micronaut:$kiluaRpcVersion")
    }
}
```

## Service implementation

### Service class

The implementation of the service class comes down to implementing required interface methods and making it a Micronaut `@Prototype` component.

```kotlin
@Prototype
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

Micronaut IoC (Inversion of Control) allows you to inject resources and other Micronaut components into your service class. You can use standard `@Inject` annotation or constructor parameter injection.

Kilua RPC allows you to inject `HttpRequest` Micronaut object (which can also give you access to the user session, if it is configured)

```kotlin
@Prototype
class AddressService : IAddressService {

    @Inject
    lateinit var httpRequest: HttpRequest<*>

    override suspend fun getAddressList(search: String?, sort: Sort) {
        println(httpRequest.uri)
        SessionForRequest.find(httpRequest).ifPresent { session ->
            println(session.id)    
        }
        return listOf()
    }
}
```

{% hint style="info" %}
Note: The new instance of the service class will be created by Micronaut for every server request. Use session or request objects to store your state with appropriate scope.
{% endhint %}

### **Blocking code**

Since Micronaut architecture is asynchronous and non-blocking, you should **never** block a thread in your application code. If you have to use some blocking code (e.g. blocking I/O, JDBC) always use the dedicated coroutine dispatcher.

```kotlin
@Prototype
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

To allow Kilua RPC work with Micronaut you have to pass all instances of the `RpcServiceManager` objects (defined in common code) to the Micronaut environment. You do this by defining a provider method for the `RpcManagers` instance in the main application class. You can use `getAllServiceManagers()` method to simplify your code.

```kotlin
@Factory
class RpcApplication {
    @Bean
    fun getManagers() = RpcManagers(getAllServiceManagers())
}

fun main(args: Array<String>) {
    run(*args)
}
```

### Security

To secure your application you can use different Micronaut components and ready to use modules. See [Micronaut Security](https://micronaut-projects.github.io/micronaut-security/latest/guide/) guide for details. You can apply different security settings for different services by defining custom `SecurityRule` using Kilua RPC `matches` helper function.

```kotlin
import dev.kilua.rpc.matches

@Singleton
open class AppSecurityRule(rolesFinder: RolesFinder) : AbstractSecurityRule<HttpRequest<*>>(rolesFinder) {
    override fun check(request: HttpRequest<*>, authentication: Authentication?): Publisher<SecurityRuleResult> {
        return if (request.matches(getServiceManager<IAddressService>(), getServiceManager<IProfileService>())) {
            compareRoles(listOf(SecurityRule.IS_AUTHENTICATED), getRoles(authentication))
        } else {
            Mono.just(SecurityRuleResult.ALLOWED)
        }
    }
}
```

