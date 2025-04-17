# Vert.x

[Eclipse Vert.x](https://vertx.io) is a toolkit for building reactive applications on the JVM. It's event driven, non-blocking, lightweight and fast.

## Build configuration

Kilua RPC provides two different modules for Vert.x:

* `kilua-rpc-vertx`, which doesn't use dependency injection (DI) and requires manual services registration,
* `kilua-rpc-vertx-koin`, which uses [Koin](https://insert-koin.io/) dependency injection framework to access services implementations.

You need to add one of these modules to your project.

```kotlin
val commonMain by getting {
    dependencies {
        implementation("dev.kilua:kilua-rpc-vertx:$kiluaRpcVersion")
//        implementation("dev.kilua:kilua-rpc-vertx-koin:$kiluaRpcVersion")
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

You can use constructor parameters to inject server objects - `RoutingContext` , `Vertx` and `ServerWebSocket` (see [Websockets](../websockets.md) chapter) into your service classes. These objects give you access to the application configuration, its state, the current request and the user session.

```kotlin
class AddressService(val rctx: RoutingContext) : IAddressService {
    override suspend fun getAddressList(search: String?, sort: Sort) {
        println(rctx.request().remoteAddress().host())
        println(rctx.session().id())
        return listOf()
    }
}
```

### **Blocking code**

Since Vert.x architecture is asynchronous and non-blocking, you should **never** block a thread in your application code. If you have to use some blocking code (e.g. blocking I/O, JDBC) always use the dedicated coroutine dispatcher.

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

### The main verticle

Vert.x services are deployed as "verticles". Main verticle is the application starting point. It's used to initialize and configure application modules and features. Minimal implementation for Kilua RPC integration contains `initRpc` and `applyRoutes` function calls.

When using manual service registration, you call `initRpc` with a lambda function, which binds  interfaces with their implementations. Different overloads of `registerService` function allow injecting server objects into your service classes.

```kotlin
import dev.kilua.rpc.applyRoutes
import dev.kilua.rpc.getServiceManager
import dev.kilua.rpc.initRpc
import io.vertx.core.AbstractVerticle
import io.vertx.ext.web.Router

class MainVerticle : AbstractVerticle() {
    override fun start() {
        val router = Router.router(vertx)
        vertx.initRpc(router) {
            registerService<IAddressService> { AddressService() }
//            registerService<IAddressService> { rctx -> AddressService(rctx) }
//            registerService<IAddressService> { rctx, vertx -> AddressService(rctx, vertx) }
//            registerService<IAddressService> { rctx, vertx, sws -> AddressService(rctx, vertx, sws) }
        }
        vertx.applyRoutes(router, getServiceManager<IAddressService>())
        vertx.createHttpServer().requestHandler(router).listen(8080)
    }
}
```

When using Koin, you call `initRpc` with a list of Koin modules. Constructor parameter injection is automatically supported by Koin.

```kotlin
import dev.kilua.rpc.applyRoutes
import dev.kilua.rpc.getAllServiceManagers
import dev.kilua.rpc.getServiceManager
import dev.kilua.rpc.initRpc
import io.vertx.core.AbstractVerticle
import io.vertx.ext.web.Router
import org.koin.core.annotation.ComponentScan
import org.koin.core.annotation.Module
import org.koin.ksp.generated.module

@Module
@ComponentScan
class AddressModule

// val addressModule = module {             // manual Koin module declaration
//    factoryOf(::AddressService)
// }

class MainVerticle : AbstractVerticle() {
    override fun start() {
        val router = Router.router(vertx)
        val server = vertx.createHttpServer()
        vertx.initRpc(router, server, getAllServiceManagers(), null, AddressModule().module)
        vertx.applyRoutes(router, getServiceManager<IAddressService>())
        vertx.createHttpServer().requestHandler(router).listen(8080)
    }
}
```

If you are using websockets, you need to use a special version of `initRcp` function, and pass a `HttpServer` instance and a list of all services declaring websocket channel connections.

```kotlin
class MainVerticle : AbstractVerticle() {
    override fun start() {
        val router = Router.router(vertx)
        val server = vertx.createHttpServer()
        vertx.initRpc(router, server, getAllServiceManagers()) {
            registerService<IAddressService> { AddressService() }
        }
        server.requestHandler(router).listen(8080)
    }
}
```

### Security

You can use standard Vert.x configuration to add authentication and authorization to your application. You can use `serviceRoute` extension function to apply your `AuthHandler` to the selected Kilua RPC services.

```kotlin
import dev.kilua.rpc.serviceRoute

const val SESSION_PROFILE_KEY = "com.example.profile"

class MainVerticle : AbstractVerticle() {
    override fun start() {
        val router = Router.router(vertx)
        vertx.initRpc(router) {
            // ..
        }
        router.route().handler(SessionHandler.create(LocalSessionStore.create(vertx)))
        val authHandler = MyAuthHandler()
        router.serviceRoute(getServiceManager<IAddressService>(), authHandler)
        router.serviceRoute(getServiceManager<IProfileService>(), authHandler)
        vertx.applyRoutes(router, getServiceManager<IAddressService>())
        vertx.applyRoutes(router, getServiceManager<IProfileService>())
        vertx.applyRoutes(router, getServiceManager<IRegisterProfileService>())
        router.route(HttpMethod.POST, "/login").handler(BodyHandler.create(false)).blockingHandler { rctx ->
            val username = rctx.request().getFormAttribute("username") ?: ""
            val password = rctx.request().getFormAttribute("password") ?: ""
            transaction {
                UserDao.select {
                    (UserDao.username eq username) and (UserDao.password eq DigestUtils.sha256Hex(password))
                }.firstOrNull()?.let {
                    val profile =
                        Profile(it[UserDao.id], it[UserDao.name], it[UserDao.username].toString(), null, null)
                    rctx.session().put(SESSION_PROFILE_KEY, profile)
                    rctx.response().setStatusCode(200).end()
                } ?: rctx.response().setStatusCode(401).end()
            }
        }
        router.route(HttpMethod.GET, "/logout").handler { rctx ->
            rctx.clearUser()
            rctx.session().destroy()
            rctx.response().putHeader("Location", "/").setStatusCode(302).end()
        }
        vertx.createHttpServer().requestHandler(router).listen(8080)
    }
}
```
