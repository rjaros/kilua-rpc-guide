# Common code

The `common` sources set is the place where you define how your remote services should work. You can define as many services as you wish, and they can have as many methods as you need. It's a good practice to split your services based on their context and functions.

{% hint style="info" %}
Note: When using authentication on the server side, you can usually apply different security options to different services.
{% endhint %}

### Declaring an interface

Designing the interface is probably the most important step, and during this process you have to stick to some important rules.

#### An interface must be annotated with `@RpcService` annotation.

Kilua RPC compiler plugin will generate common, backend and frontend code based on the interface name.

#### A method must be suspending

Kotlin coroutines allow the framework to translate asynchronous calls into synchronous-like code.

#### A method must have from zero to six parameters

This is the restriction of the current version of the library. It may change in the future.

#### A method can't return nullable value

`Unit` return type is not supported as well.

#### Method parameters and return value must be of supported types

* all basic Kotlin types (`String`, `Boolean`, `Int`, `Long`, `Short`, `Char`, `Byte`, `Float`, `Double`)
* `Enum` classes defined in common code and annotated with `@Serializable` annotation
* All date and time types from `kotlinx-datetime` library
* A `dev.kilua.rpc.types.Decimal` type, which is automatically mapped to `Double` on the frontend side and `java.math.BigDecimal` on the backend side
* any class defined in common code with a `@Serializable` annotation
* a `List<T>`, where T is one of the above types
* a `T?`, where T is one of the above types (allowed only as method parameters - see previous rule)
* a `Result<T>`, where T is one of the above types, can be used as a method return value.

{% hint style="info" %}
Note: Default parameters values are supported.
{% endhint %}

Even with the above restrictions, the set of supported types is quite rich and you should be able to model almost any use case for your applications. You can always wrap any data structure into a serializable data class. It's also a simple way to pass around the parameters count limit.

With an interface defined in the common code, the type safety of your whole application is forced at compile time. Any incompatibility between the frontend and the backend code will be marked as a compile-time error.

```kotlin
import dev.kilua.rpc.annotations.RpcService

@Serializable
data class Address(
    val id: Int? = 0,
    val firstName: String? = null,
    val lastName: String? = null,
    val email: String? = null,
)

@Serializable
enum class Sort {
    FN, LN, E
}

@RpcService
interface IAddressService {
    suspend fun getAddressList(search: String?, sort: Sort): List<Address>
    suspend fun addAddress(address: Address): Address
    suspend fun updateAddress(id: Int, address: Address): Address
    suspend fun deleteAddress(id: Int): Boolean
}
```

### Using annotations to change HTTP method and/or route name

By default Kilua RPC will use HTTP POST server calls and automatically generated route names. But you can use `@RpcBinding`, `@RpcBindingMethod` and `@RpcBindingRoute` annotations on any interface method, to change the default values.

```kotlin
@RpcService
interface IAddressService {
    @RpcBindingRoute("get_address_list")
    suspend fun getAddressList(search: String?, sort: Sort): List<Address>
    @RpcBinding(Method.PUT, "add_address")
    suspend fun addAddress(address: Address): Address
    @RpcBindingRoute("update_address")
    suspend fun updateAddress(id: Int, address: Address): Address
    @RpcBinding(Method.DELETE, "delete_address")
    suspend fun deleteAddress(id: Int): Boolean
}
```

{% hint style="info" %}
Note: All Kilua RPC endpoint names (even those with user defined names) are prefixed with "/rpc/" to avoid potential conflicts with other endpoints.
{% endhint %}
