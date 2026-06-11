# Jooby

[Jooby](https://jooby.io) is a scalable, fast and modular micro web framework for Java and Kotlin. It is very easy to use and has a lot of different, ready to use [modules](https://jooby.io/#modules).&#x20;

## Build configuration

Kilua RPC provides three different modules for Jooby:

* `kilua-rpc-jooby`, which doesn't use dependency injection (DI) and requires manual services registration,
* `kilua-rpc-jooby-koin`, which uses [Koin](https://insert-koin.io/) dependency injection framework to access services implementations.
* `kilua-rpc-jooby-metro`, which uses [Metro](https://github.com/zacsweers/metro) dependency injection framework to access services implementations.

You need to add one of these modules to your project.

```kotlin
val commonMain by getting {
    dependencies {
        implementation("dev.kilua:kilua-rpc-jooby:$kiluaRpcVersion")
//        implementation("dev.kilua:kilua-rpc-jooby-koin:$kiluaRpcVersion")
//        implementation("dev.kilua:kilua-rpc-jooby-metro:$kiluaRpcVersion")
    }
}
```

## Service implementation

### Service class

The implementation of the service class comes down to implementing required interface methods.

```kotlin
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

When using Metro, you need to add some annotations and the `MetroRpcService` interface to your service class.

```kotlin
import dev.kilua.rpc.MetroRpcService
import dev.kilua.rpc.RpcAppScope
import dev.zacsweers.metro.ClassKey
import dev.zacsweers.metro.ContributesIntoMap
import dev.zacsweers.metro.Inject
import dev.zacsweers.metro.binding

@Inject
@ContributesIntoMap(RpcAppScope::class, binding = binding<MetroRpcService>())
@ClassKey(IAddressService::class)
class AddressService : IAddressService, MetroRpcService {
    // ...
}
```

### Injecting server objects

You can use constructor parameters to inject server objects - `Context` and `Kooby` into your service classes. These objects give you access to the application configuration, its state, the current request and the user session.

```kotlin
class AddressService(val ctx: Context, val kooby: Kooby) : IAddressService {
    override suspend fun getAddressList(search: String?, sort: Sort) {
        println(kooby.config.getString("option1") ?: "default")
        println(ctx.remoteAddress)
        println(ctx.session().id)
        return listOf()
    }
}
```

### **Blocking code**

Since Jooby execution model assumes suspending endpoints are non-blocking, you should **never** block a thread in your application code. If you have to use some blocking code (e.g. blocking I/O, JDBC) always use the dedicated coroutine dispatcher.

```kotlin
class AddressService : IAddressService {
    override suspend fun getAddressList(search: String?, sort: Sort) {
        return withContext(Dispatchers.IO) {
            retrieveAddressesFromDatabase(search, sort)
        }
    }
}
```

## Application configuration

### The main function

This function is the application starting point. It's used to initialize and configure application modules and features. Minimal implementation for Kilua RPC integration contains `applyRoutes` function calls and service initialization with an appropriate `initRpc` function.

When using manual service registration, you call `initRpc` with a lambda function, which binds  interfaces with their implementations. Different overloads of `registerService` function allow injecting server objects into your service classes.

```kotlin
import dev.kilua.rpc.applyRoutes
import dev.kilua.rpc.getServiceManager
import dev.kilua.rpc.initRpc
import dev.kilua.rpc.registerService
import io.jooby.kt.runApp

fun main(args: Array<String>) {
    runApp(args) {
        initRpc {
            registerService<IAddressService> { AddressService() }
//            registerService<IAddressService> { ctx -> AddressService(ctx) }
//            registerService<IAddressService> { ctx, kooby -> AddressService(ctx, kooby) }
        }
        applyRoutes(getServiceManager<IAddressService>())
    }
}
```

When using Koin, you call `initRpcKoin` to initialize Koin modules. Constructor parameter injection is automatically supported by Koin.

```kotlin
import dev.kilua.rpc.applyRoutes
import dev.kilua.rpc.getAllServiceManagers
import dev.kilua.rpc.initRpcKoin
import io.jooby.kt.runApp
import org.koin.core.module.dsl.factoryOf
import org.koin.dsl.bind
import org.koin.dsl.module

val addressModule = module {
    factoryOf(::AddressService) bind IAddressService::class
}

fun main(args: Array<String>) {
    runApp(args) {
        initRpcKoin {
            modules(addressModule)
        }
        applyRoutes(getServiceManager<IAddressService>())
    }
}
```

When using Metro, you call `initRpcMetro` to initialize Metro dependency graph. Constructor parameter injection is automatically supported by Metro.

```kotlin
import dev.kilua.rpc.RpcGraph
import dev.kilua.rpc.applyRoutes
import dev.kilua.rpc.getAllServiceManagers
import dev.kilua.rpc.initRpcMetro
import dev.zacsweers.metro.AppScope
import dev.zacsweers.metro.DependencyGraph
import dev.zacsweers.metro.createGraph
import io.jooby.kt.runApp

@DependencyGraph(AppScope::class)
interface AppGraph : RpcGraph.Factory

fun main(args: Array<String>) {
    runApp(args) {
        initRpcMetro(createGraph<AppGraph>())
        applyRoutes(getServiceManager<IAddressService>())
    }
}
```

### Security with Pac4j

You can use [Pac4j](https://jooby.io/modules/pac4j/) security module to configure authentication and authorization in your app. By calling `applyRoutes` function before or after Pac4j module declaration, you apply different security requirements to different services.

```kotlin
fun main(args: Array<String>) {
    runApp(args) {
        initRpc {
            // ...
        }
        applyRoutes(getServiceManager<IRegisterProfileService>()) // No authentication needed
        val config = org.pac4j.core.config.Config()
        config.addAuthorizer("Authorizer") { _, _, _ -> true }
        install(Pac4jModule(Pac4jOptions().apply {
            defaultUrl = "/"
        }, config).client("*", "Authorizer") { _ ->
            FormClient("/") { credentials, context, sessionStore ->
            }
        })
        applyRoutes(getServiceManager<IAddressService>())    // Authentication needed
        applyRoutes(getServiceManager<IProfileService>())    // Authentication needed
    }
}
```
