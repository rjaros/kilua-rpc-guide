# Overview

This is a short overview of how to use Kilua RPC. It contains just the basic concepts and ideas. You can find more information in the following chapters.

Let's assume we want to create an encoder application, that gets some text from the user and encodes it on the server with a chosen algorithm.

### Common source set

We start by defining our data model and service interface in the common (shared) source set. We have to declare our "business" `encode` method as suspending. We annotate the service interface with `@RpcService`, which will allow Kilua RPC do all its "magic".

{% code title="Common.kt" %}
```kotlin
import dev.kilua.rpc.annotations.RpcService
import kotlinx.serialization.Serializable

@Serializable
enum class EncodingType {
    BASE64, URLENCODE, HEX
}

@RpcService
interface IEncodingService {
    suspend fun encode(input: String, encodingType: EncodingType): String
}
```
{% endcode %}

### Frontend source set

Kilua RPC compiler plugin will automatically generate `EncodingService` class, which implements `IEncodingService` interface. To use this class in the frontend application it is recommended to use `getService()` function (you can also just use the name of the generated class, but in such case your project will not compile in IDE until compiler plugin task is executed).

{% code title="FrontendApp.kt" %}
```kotlin
import dev.kilua.rpc.getService

val service = getService<IEncodingService>()

launch {
    val result = service.encode("Lorem ipsum", EncodingType.BASE64)
    console.log(result) // outputs TG9yZW0gaXBzdW0K
}
```
{% endcode %}

All asynchronous operations are hidden by the framework. We only have to use a coroutine builder (`launch` in this case) to run a suspending function.

### Backend source set

_(For convenience we will use Ktor module in this chapter)_

To create the implementation of our `EncodingService` we just have to implement the methods of the service interface.

{% code title="Backend.kt" %}
```kotlin
import java.net.URLEncoder
import acme.Base64Encoder
import acme.HexEncoder

class EncodingService : IEncodingService {
    override suspend fun encode(input: String, encodingType: EncodingType): String {
        return when (encodingType) {
                EncodingType.BASE64 -> {
                    Base64Encoder.encode(input)
                }
                EncodingType.URLENCODE -> {
                    URLEncoder.encode(input, "UTF-8")
                }
                EncodingType.HEX -> {
                    HexEncoder.encode(input)
                }
        }
    }
}
```
{% endcode %}

Finally, we initialize routing and register services in the main application function.

{% code title="Main.kt" %}
```kotlin
import dev.kilua.rpc.applyRoutes
import dev.kilua.rpc.getAllServiceManagers
import dev.kilua.rpc.initRpc
import dev.kilua.rpc.registerService
import io.ktor.server.application.*
import io.ktor.server.routing.*

fun Application.main() {
    routing {
        getAllServiceManagers().forEach { applyRoutes(it) }
    }
    initRpc {
        registerService<IEncodingService> { EncodingService() }
    }
}

```
{% endcode %}

When we run our application everything will work automatically - a call on the client side will run the code on the server and the result will be sent back to the caller.

That's all - our first, fullstack application is ready!

You can find "encoder-fullstack-ktor", a complete KVision application based on this overview (with GUI components), in the [kvision-examples](https://github.com/rjaros/kvision-examples) repository on GitHub.
