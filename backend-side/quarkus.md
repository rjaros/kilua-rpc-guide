# Quarkus

[Quarkus](https://quarkus.io/) is a modern, JVM-based, fullstack framework, based on Vert.x HTTP engine.

## Build configuration

Kilua RPC provides a single module for Quarkus, `kilua-rpc-quarkus`, which uses Quarkus dependency injection to access services implementations. You need to add this module to your project.

```kotlin
val commonMain by getting {
    dependencies {
        implementation("dev.kilua:kilua-rpc-quarkus:$kiluaRpcVersion")
    }
}
```

## Service implementation

### Service class

The implementation of the service class comes down to implementing required interface methods and making it a Quarkus `@Dependent` component.

```kotlin
@Dependent
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

Quarkus DI solution (also called ArC) is based on the [Jakarta Contexts and Dependency Injection 4.1](https://jakarta.ee/specifications/cdi/4.1/jakarta-cdi-spec-4.1.html) specification and allows you to inject resources and other Quarkus components into your service class. You can use standard `@Inject` annotation or constructor parameter injection.

Kilua RPC allows you to inject `RoutingContext` (a Vert.x class), which can also give you access to the user session.

```kotlin
@Dependent
class AddressService : IAddressService {

    @Inject
    lateinit var rctx: RoutingContext

    override suspend fun getAddressList(search: String?, sort: Sort) {
        println(rctx.request().remoteAddress().host())
        println(rctx.session().id())
        return listOf()
    }
}
```

{% hint style="info" %}
Note: The new instance of the service class will be created by Quarkus for every server request. Use session or request objects to store your state with appropriate scope.
{% endhint %}

{% hint style="info" %}
Quarkus doesn't support traditional HTTP sessions out of the box, but you can use sessions from Vert.x by registering SessionHandler in the Vert.x router.
{% endhint %}

### **Blocking code**

Since Quarkus architecture is asynchronous and non-blocking, you should **never** block a thread in your application code. If you have to use some blocking code (e.g. blocking I/O, JDBC) always use the dedicated coroutine dispatcher.

```kotlin
@Dependent
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

To allow Kilua RPC work with Quarkus you have to pass all instances of the `RpcServiceManager` objects (defined in common code) to the Quarkus environment. You do this by defining a provider method for the `RpcManagers` instance in the main application class. You can use `getAllServiceManagers()` method to simplify your code.

```kotlin
@ApplicationScoped
class RpcApplication {
    @Produces
    @ApplicationScoped
    fun getManagers() = RpcManagers(getAllServiceManagers())
}
```

The main method need to be located in the `application` subproject sources.

```kotlin
fun main(args: Array<String>) {
    Quarkus.run(*args)
}
```
