# Frontend side

Kilua RPC makes the process of sending and receiving data fully transparent and invisible in the frontend code. Just get the instance of your Service class using `getService()` function and use it. The only thing you need to consider is  the suspending nature of the remote methods - they have to be run inside a coroutine context (of course your code will not even compile if they are not).

```kotlin
object AddressModel {
    private val addressService = getService<IAddressService>()

    val addresses = mutableListOf<Address>()
    var search: String? = null
    var sort: Sort = Sort.FN

    suspend fun getAddressList() {
        val newAddresses = addressService.getAddressList(search, sort)
        addresses.syncWithList(newAddresses)
    }

    suspend fun deleteAddress(id: Int): Boolean {
        val result = addressService.deleteAddress(id)
        if (result) getAddressList()
        return result
    }
}   
```

&#x20;

## RPC endpoint paths

By default all RPC endpoints are called by using relative paths (`rpc/*`). It should work fine for most simple cases, where the app is deployed on `/` and when it's deployed using some custom path like `/something`. But it doesn't always work. One case is when the backend is published separately from the frontend (even on different hosts). The other case is when the application uses history api client routing (nice urls like `/some/other/route`) - the app will try to call `/some/other/rpc/*` endpoints and it will fail. Such app can also be deployed using a custom `/something` path, which makes things even harder.

The general solution for these problems is setting `rpc_url_prefix` global variable. Its value is read as `globalThis.jsGet("rpc_url_prefix")`, so it can be set in different ways. You can use JavaScript inside the main `index.html` file:

```
<script>window["rpc_url_prefix"] = "some value";</script>
```

or you can use Kotlin code:

```
globalThis.jsSet("rpc_url_prefix", "some value")
```

The value of this variable (if set) is added to the path called by the RPC client as `$urlPrefix/rpc`.  The value can be both absolute and relative.

Going back to typical cases. When the backend is deployed separately, you can use the full URL address (`window["rpc_url_prefix"] = "`[`https://example.com/backend_path`](http://example.com/backend_path)`"`). When the application is using client routing but is deployed on `/` you can use `window["rpc_url_prefix"] = "";` - it will make all RPC calls starting with `/`. When the app is also deployed on a different path (`/something`) you need to know (or guess) this path and set `window["rpc_url_prefix"] = "/something"`.
