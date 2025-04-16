# Ktor

[Ktor](https://ktor.io/) is the asynchronous web framework for Kotlin created by JetBrains. It is written in Kotlin and its machinery and API is utilizing Kotlin coroutines. It's unopinionated design makes it a very good choice for development of applications composed of many different technologies.

## Build configuration

Kilua RPC provides two different modules for Ktor:

* `kilua-rpc-ktor`, which doesn't use dependency injection (DI) and requires manual services registration,
* `kilua-rpc-ktor-koin`, which uses [Koin](https://insert-koin.io/) dependency injection framework to access services implementations.

You need to add one of these modules to your project.

```kotlin
val commonMain by getting {
    dependencies {
        implementation("dev.kilua:kilua-rpc-ktor:$kiluaRpcVersion")
//        implementation("dev.kilua:kilua-rpc-ktor-koin:$kiluaRpcVersion")
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

You can use `@Factory` annotation, if you use Koin and `koin-annotations` to configure dependency injection.

```kotlin
@Factory
class AddressService : IAddressService {
    // ...
}
```

### Injecting server objects

You can use constructor parameters to inject server objects - `ApplicationCall` and `WebSocketServerSession` into your service classes. These objects give you access to the application configuration, its state, the current request and the user session (if it is configured).

```kotlin
class AddressService(private val call: ApplicationCall) : IAddressService {
    override suspend fun getAddressList(search: String?, sort: Sort) {
        println(call.application.environment.config.propertyOrNull("option1")?.getString())
        println(call.application.attributes)
        println(call.request.uri)
        println(call.request.queryParameters["param1"])
        println(call.sessions)
        return listOf()
    }
}
```

### **Blocking code**

Since Ktor architecture is asynchronous and non-blocking, you should **never** block a thread in your application code. If you have to use some blocking code (e.g. blocking I/O, JDBC) always use the dedicated coroutine dispatcher.

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

This function is the application starting point. It's used to initialize and configure application modules and features. Minimal implementation for Kilua RPC integration contains `initRpc` and `applyRoutes` function calls. `initRpc` function should be executed as last.

When using manual service registration you call `initRpc` with a lambda function, which binds the interface with its implementation. Different overloads of `registerService` function allow injecting server objects into your classes.

```kotlin
import dev.kilua.rpc.applyRoutes
import dev.kilua.rpc.getServiceManager
import dev.kilua.rpc.initRpc
import dev.kilua.rpc.registerService
import io.ktor.server.application.*
import io.ktor.server.plugins.compression.*
import io.ktor.server.routing.*
import io.ktor.server.websocket.*

fun Application.main() {
    install(Compression)
    install(WebSockets)
    routing {
        applyRoutes(getServiceManager<IAddressService>())
    }
    initRpc {
        registerService<IAddressService> { AddressService() }
//        registerService<IAddressService> { call -> AddressService(call) }
    }
}
```

When using Koin you call `initRpc` with a list of Koin modules. Constructor parameter injection is automatically supported by Koin.

```kotlin
import dev.kilua.rpc.applyRoutes
import dev.kilua.rpc.getServiceManager
import dev.kilua.rpc.initRpc
import io.ktor.server.application.*
import io.ktor.server.plugins.compression.*
import io.ktor.server.routing.*
import io.ktor.server.websocket.*
import org.koin.core.annotation.ComponentScan
import org.koin.core.annotation.Module
import org.koin.ksp.generated.module

@Module
@ComponentScan
class AddressModule

fun Application.main() {
    install(Compression)
    install(WebSockets)
    routing {
        applyRoutes(getServiceManager<IAddressService>())
    }
    initRpc(AddressModule().module)
}
```

### Authentication

When using [authentication plugin](https://ktor.io/docs/server-auth.html) you can choose different authentication options for different services.

```kotlin
fun Application.main() {
    install(Authentication) {
        form {
            userParamName = "username"
            passwordParamName = "password"
            validate { credentials ->
                // ...
            }
            skipWhen { call -> call.sessions.get<Profile>() != null }
        }
    }

    routing {
        applyRoutes(getServiceManager<IRegisterProfileService>()) // No authentication needed
        authenticate {
            post("login") {
                // ...
            }
            get("logout") {
                call.sessions.clear<Profile>()
                call.respondRedirect("/")
            }
            applyRoutes(getServiceManager<IAddressService>()) // Authentication needed
            applyRoutes(getServiceManager<IProfileService>()) // Authentication needed
        }
    }
    initRpc {
       // ...
    }
}
```
