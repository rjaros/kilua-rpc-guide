# Javalin

[Javalin](https://javalin.io) is a very popular, simple, lightweight and flexible micro web framework for Java and Kotlin. It's un-opinionated and runs on top of Jetty, one of the most used and stable web-servers for the JVM.

## Build configuration

Kilua RPC provides three different modules for Javalin:

* `kilua-rpc-javalin`, which doesn't use dependency injection (DI) and requires manual services registration,
* `kilua-rpc-javalin-koin`, which uses [Koin](https://insert-koin.io/) dependency injection framework to access services implementations.
* `kilua-rpc-javalin-metro`, which uses [Metro](https://github.com/zacsweers/metro) dependency injection framework to access services implementations.

You need to add one of these modules to your project.

```kotlin
val commonMain by getting {
    dependencies {
        implementation("dev.kilua:kilua-rpc-javalin:$kiluaRpcVersion")
//        implementation("dev.kilua:kilua-rpc-javalin-koin:$kiluaRpcVersion")
//        implementation("dev.kilua:kilua-rpc-javalin-metro:$kiluaRpcVersion")
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

You can use constructor parameters to inject server objects - `Context` , `JavalinState` and `WsContext` (see [Websockets](../websockets.md) chapter) into your service classes. These objects give you access to the application configuration, its state, the current request and the user session.

```kotlin
class AddressService(val ctx: Context) : IAddressService {
    override suspend fun getAddressList(search: String?, sort: Sort) {
        println(ctx.req.remoteAddr)
        println(ctx.req.session.id)
        return listOf()
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
import io.javalin.Javalin

fun main() {
    Javalin.create { config ->
        config.initRpc {
            registerService<IAddressService> { AddressService() }
        }
        config.applyRoutes(getServiceManager<IAddressService>())
    }.start(8080)
}
```

When using Koin, you call `initRpcKoin` to initialize Koin modules. Constructor parameter injection is automatically supported by Koin.

```kotlin
import dev.kilua.rpc.applyRoutes
import dev.kilua.rpc.getAllServiceManagers
import dev.kilua.rpc.initRpcKoin
import io.javalin.Javalin
import org.koin.core.module.dsl.factoryOf
import org.koin.dsl.bind
import org.koin.dsl.module

val addressModule = module {
    factoryOf(::AddressService) bind IAddressService::class
}

fun main() {
    Javalin.create { config ->
        config.initRpcKoin {
            modules(addressModule)
        }
        config.applyRoutes(getServiceManager<IAddressService>())
    }.start(8080)
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
import io.javalin.Javalin

@DependencyGraph(AppScope::class)
interface AppGraph : RpcGraph.Factory

fun main() {
    Javalin.create { config ->
        config.initRpcMetro(createGraph<AppGraph>())
        config.applyRoutes(getServiceManager<IAddressService>())
    }.start(8080)
}
```

### Security

When configuring security with `AccessManager` you can call `applyRoutes` with additional parameter containing roles.

```kotlin
enum class ApiRole : Role { AUTHORIZED, ANYONE }

fun main() {
    Javalin.create { config ->
        config.accessManager { handler, ctx, permittedRoles ->
            when {
                permittedRoles.contains(ApiRole.ANYONE) -> handler.handle(ctx)
                ctx.sessionAttribute<Profile>(SESSION_PROFILE_KEY) != null -> handler.handle(ctx)
                else -> ctx.status(HttpStatus.UNAUTHORIZED).json("Unauthorized")
            }
        }
        config.initRpc {
            modules(ConfigModule().module(), DbModule().module())
        }
        config.applyRoutes(getServiceManager<IAddressService>(), setOf(ApiRole.AUTHORIZED))
        config.applyRoutes(getServiceManager<IProfileService>(), setOf(ApiRole.AUTHORIZED))
        config.applyRoutes(getServiceManager<IRegisterProfileService>(), setOf(ApiRole.ANYONE))

        // Security config
        config.routes.post("/login", { ctx ->
            val username = ctx.formParam("username") ?: ""
            val password = ctx.formParam("password") ?: ""
            transaction {
                UserDao.select {
                    (UserDao.username eq username) and (UserDao.password eq DigestUtils.sha256Hex(password))
                }.firstOrNull()?.let {
                    val profile =
                        Profile(it[UserDao.id], it[UserDao.name], it[UserDao.username].toString(), null, null)
                    ctx.sessionAttribute(SESSION_PROFILE_KEY, profile)
                    ctx.status(HttpStatus.OK)
                } ?: ctx.status(HttpStatus.UNAUTHORIZED)
            }
        }, setOf(ApiRole.ANYONE))
        config.routes.get("/logout", { ctx ->
            ctx.req().session.invalidate()
            ctx.redirect("/", HttpStatus.FOUND)
        }, setOf(ApiRole.AUTHORIZED))
    }.start(8080)
}
```
