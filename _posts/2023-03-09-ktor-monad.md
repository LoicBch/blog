---
layout: post
title: Clean Ktor error handling with monads 
excerpt_separator:  <!--more-->
subtitle: A clean implementation of http layer using Ktor and monad pattern that let you cleanly handle multiple error sources and abstracting all complexity in extension and monad mapping function.
--- 

![Ice cream crash](blog/assets/images/failed.png)

Recently, I had the opportunity to start a small project from scratch. The project's purpose is to streamline user issue management and handle the administrative aspects that hosts encounter during the events in which Park4night participates.

This provided an opportunity to build from the ground up and aim to implement a reactive HTTP layer that adheres to the best architectural practices and clean code.
To do that i'll use the monad design pattern and Ktor-client as the HTTP library because it's one of the few libraries available for Kotlin Multiplatform Mobile (KMM) that is entirely written in Kotlin.


### What's a monad ?

A Monad is a functional programming concept that originated from Haskell and is often used for handling exceptions. It helps manage objects that can have multiple types, and it can be thought of as a container and wrapper for different types.

For example, the Java implementation of an `Either` Monad is a class that contains:

  - An optional object of generic type `A`
  - An optional object of generic type `B`
  Every object of type `Either<A, B>` can be either of type `A` or type `B`. The key point here is that we don't need to know the type until we unwrap the `Either` object and check the type it holds.

This pattern is often used with asynchronous operations, giving us the ability to perform operations that can either fail or succeed without directly handling exceptions by surrounding our code with try/catch blocks. It fits perfectly with Kotlin code because Sealed classes make it easy to represent Monads, and unlike Java, the JetBrains team decided not to enforce exception handling


### Ktor client

Ktor is an open-source Kotlin framework designed for application development on both the server-side and client-side. The Ktor client module is specifically tailored for creating HTTP clients using Kotlin. It enables the creation of HTTP clients to make requests to remote servers, whether it's for RESTful requests, data retrieval from web APIs, file downloads, and more.

As a reminder, in KMM projects, we can't use libraries that are written in Java. The good news is that Ktor client is written entirely in Kotlin, so we can use it in all our KMM projects.


### Starting from the most basic solution

The most basic approach of API error handling is often to have an object to 
encapsulate the result of the request :

{% highlight kotlin %}
data class CallResult(
    val error: String?,
    val successValue: Any?,
    val success: Boolean
)
{% endhighlight %}

And when the request result is received the instantiation of the object would look like :

{% highlight kotlin %}
override suspend fun getProduct(): CallResult {
    return try {
        val result = client.get("https://google.com")
        CallResult(success = true, 
                   successValue = result.body(), 
                   error = null)
    } catch (e: Exception) {
        CallResult(success = false, 
                   successValue = null, 
                   error = "An error occurred")
    }
}
{% endhighlight %}

This solution is relatively simple to implement, but it falls short of adhering to clean code principles. It lacks flexibility and can lead to the propagation of poor coding practices throughout the entire codebase. The primary issues with this basic approach are as follows:

* The solution is not type-safe because we don't know the type that the request will return. The 'value' is of type 'Any,' which forces us to perform type casting when unwrapping our value.

* Both success and failure are managed within the same object, and the only distinguishing factor is the boolean `success`. This approach doesn't adhere to the separation of concerns principle and doesn't impose proper limitations. There's nothing preventing our object from having `success` as `false` while having a value or having `success` as `true` with a non-null error.

* Error origin is not properly identified in our example. We don't differentiate between errors originating from I/O, the server, serialization, or other sources. It would be beneficial to categorize error sources so that we can handle them differently.

* In the case of success, every time we want to modify the value of our object, we'll need to check whether the object represents success or error and then modify our `value` variable. This approach leads to code repetition and is error-prone.

We can improve our object by addressing each of these disadvantages one by one.


### Making it typesafe

To resolve the type problem we can make our class generic and have our class receive two type argument, 
one for the success result value and one for the error representation :

{% highlight kotlin %}
data class CallResult<T, E>(
    val error: E?,
    val successValue: T?,
    val success: Boolean
)
{% endhighlight %}

The instanciation :

{% highlight kotlin %}
override suspend fun getProduct(): CallResult<Product, ErrorMessage> {
    return try {
        val result = client.get("https://google.com").body<Product>()
        CallResult(success = true, 
                   successValue = result, 
                   error = null)
    } catch (e: Exception) {
        CallResult(success = false, 
                   successValue = null, 
                   error = ErrorMessage(e.message ?: "Error")))
    }
}
{% endhighlight %}

Because of Kotlin-Serialization if our `Product` data class is annotated with `@Serializable` the json representation of `Product` reveived in the request will be automaticaly serialized into a `Product` object.

### Separation of concern

The next and most significant problem is the need to clearly distinguish between success and error cases. This is where the implementation of the monad pattern comes into play, as an API call can either result in success with a result or in an error with an error cause.

In Kotlin, sealed classes are perfect for representing a monad, we just need to refactor our previous `callResult` object into a sealed class with sub-types to represent success and failure cases:

{% highlight kotlin %}
sealed class CallResult<out T, out E> {
    data class Success<T>(val value: T) : CallResult<T, Nothing>()
    data class Failure<E>(val code: Int, val value: E) : CallResult<Nothing, E>()
}
{% endhighlight %}

instanciation :

{% highlight kotlin %}
override suspend fun getProduct(): CallResult<Product, ErrorMessage> {
    return try {
        val result = client.get("https://google.com").body<Product>()
        CallResult.Success(result)
    } catch (e: Exception) {
        CallResult.Failure(0, ErrorMessage(e.message ?: "Unknown error"))
    }
}
{% endhighlight %}

We kept the generic right type in our sub-class and we assign `Nothing` type if the type just can't exist here.
This approach ensures that we have a clear and structured way to handle success and error cases, making our code more robust and maintainable.


### Errors source

The next goal is to represent multiple error sources. We can achieve this by making our `Failure` class also a sealed class. This allows us to enumerate the list of error sources we want to handle:

{% highlight kotlin %}
sealed class CallResult<out T, out E> {
    data class Success<T>(val value: T) : CallResult<T, Nothing>()
    sealed class Failure<E> : CallResult<Nothing, E>() {
        data class HttpError<E>(val code: Int, val errorBody: E) : Failure<E>()
        data class SerializationError(val message: ErrorMessage) : Failure<Nothing>()
        object NetworkError : Failure<Nothing>()
    }
}
{% endhighlight %}

instanciation :

{% highlight kotlin %}
 override suspend fun getProduct(): CallResult<Product, SimpleError> {
    return try {
        val result = client.get("https://google.com").body<Product>()
        CallResult.Success(result)
    } catch (e: ClientRequestException) {
        val code = e.response.status.value
        CallResult.Failure.HttpError(code, e.errorBody())
    } catch (e: SerializationException) {
        CallResult.Failure.SerializationError(ErrorMessage(e.message.toString()))
    } catch (e: IOException) {
        CallResult.Failure.NetworkError
    } catch (e: JsonConvertException) {
        CallResult.Failure.SerializationError(ErrorMessage(e.message.toString()))
    }
}

suspend inline fun <reified E> ResponseException.errorBody(): E? =
    try {
        response.body()
    } catch (e: SerializationException) {
        null
    }
{% endhighlight %}

This allow us to have different body for different error sources, for example it doesnt make senses to have a errorCode if the error comes from I/O or serialization so we just have an object for network errors and we just have a message for the serialization errors.

The `errorBody` extension function is just here to handle Serialization exception when serializing error object of http errors.

When querying the request, we can now cleanly perform differents operations in the ui layers depending of the error source :

{% highlight kotlin %}
fun getProduct() {
        viewModelScope.launch {
            when (api.fetchEvents()) {
                is ResultWrapper.Success -> 
                    //Do something with result 
                is ResultWrapper.Failure.HttpError -> 
                    //Show popup Error
                is ResultWrapper.Failure.NetworkError -> 
                    //Show connection error
                is ResultWrapper.Failure.SerializationError -> 
                    //Show serialization error
            }
        }
    }
{% endhighlight %}

Because the `when` expression forces us to handle all cases of the sealed classes, there might be situations where we only want to handle specific error sources while ignoring others. In such cases, we can extract the `when` expression into an extension function on `Result.Failure` to obtain the correct code and error message directly, without the need to check the error source each time:

{% highlight kotlin %}
fun ResultWrapper.Failure<*>.message() = when (this) {
    is ResultWrapper.Failure.HttpError<*> -> 
        if (this.errorBody is ErrorMessage) this.errorBody.message else "An http error occurred"
    is ResultWrapper.Failure.SerializationError -> this.error.message
    is ResultWrapper.Failure.NetworkError -> this.message
}
{% endhighlight %}

And use it like so :

{% highlight kotlin %}
fun getProduct() {
        viewModelScope.launch {
            when (val result = api.fetchEvents()) {
                is ResultWrapper.Success -> //Do something with result
                is ResultWrapper.Failure -> Log.d("Show", result.message())
            }
        }
    }
{% endhighlight %}


### Modification on the fly

At this stage if we want to modify the value of our object after instanciation we would have to unwrapp our monad, check if the type is a success or failure and modify the value of the success object.
This is repetitive and it force to have this type of code block at each modification :

{% highlight kotlin %}
return when (response){
        is CallResult.Success -> CallResult.Success(//new Value)
        is CallResult.Failure -> response
    }
{% endhighlight %}

To avoid this, we can define an extension function `map` that take the new value to assign and do this under the hood : 

{% highlight kotlin %}
fun <T, E, U> CallResult<T, E>.map(
    transform: (T) -> U
): CallResult<U, E> {
    return when (this) {
        is CallResult.Success -> {
            value?.let {
                CallResult.Success(transform(it))
            } ?: CallResult.Success(null as U)
        }

        is CallResult.Failure -> this
    }
}
{% endhighlight %}

Here we can modify the value of result WITHOUT handling the case of a failure.
If the value is here we modify it else we don't change the current monad and we only be able to see if it fail or succeed at the end of the road. 

We have three generic :
- T : the current type of success
- E : the current type of error
- U : the new type after the modification

It takes the `transform` lambdas in parameter.
We could have the same function `mapError` to map error case
In the same way we could also handle the case in wich we has to perform another api request that return a monad inside our `transform` lambdas, this would look like :

{% highlight kotlin %}
suspend fun <T, E, U> CallResult<T, E>.andThen(
    transform: suspend (T) -> CallResult<U, E>
): CallResult<U, E> {
    return when (this) {
        is CallResult.Success -> {
            value?.let {
                transform(it)
            } ?: CallResult.Success(null as U)
        }
        is CallResult.Failure -> this
    }
}
{% endhighlight %}

We added `suspend` keyword and changed return type of transform function, we also then just return the result of the transform function as it alerady return a monad. 
These map functions are similar to those used in functional asynchronous libraries, such as `flatMap` in RxJava. 
These asynchronous libraries in Java follow the same functional pattern rather than traditional Java exception handling. 
They gained popularity because they provide a more reactive approach for performing asynchronous operations because it let's you choose WHEN you want to handle exception, a liberty that was not really easy to have in java because of forced exception.

A real life example of the map and thenCall function from another project :

{% highlight kotlin %}
return api.fetchEventById(eventId)
            .map { event ->
                val eventDays = createInitialEventArray(event)
                val productsFormatted = formatEventProducts(event)
                productsWithDays = associateProductsToDays(eventDays, productsFormatted)
                event
            }
            .andThen {
                api.fetchPayments(it.dateStart, it.dateEnd, eventId)
            }
            .map { payments ->
                productsWithDays = associatePaymentToProducts(payments, productsWithDays)
                formatTitleDateOfDays(productsWithDays)
            }
{% endhighlight %} 

Here in the first map function i'm modifying the object i get from the first request, 
then im using `thenCall` function to make another api request with the modifyng result of the first call.
Finaly my usecase is returning the modified result of the second apicall.

### The final Solution

#### Our `CallResult` object

{% highlight kotlin %}
sealed class CallResult<out T, out E> {
    data class Success<T>(val value: T) : CallResult<T, Nothing>()
    sealed class Failure<E> : CallResult<Nothing, E>() {
        data class HttpError<E>(val code: Int, val errorBody: E?) : Failure<E>()
        data class SerializationError(val message: ErrorMessage) : Failure<Nothing>()
        object NetworkError : Failure<Nothing>()
    }
}
{% endhighlight %}

#### The object instantiation

The last step to simplify our previous instantiation in a single function that will be an extension function of Ktor HttpClient :   

{% highlight kotlin %}
suspend inline fun <reified T, reified E> HttpClient.safeGet(block: HttpRequestBuilder.() -> Unit): CallResult<T, E> {
    return try {
        val response = get { block() }
        if (response.status.isSuccess()) {
            CallResult.Success(response.body())
        } else {
            val code = response.status.value
            CallResult.Failure.HttpError(code, response.body())
        }
    } catch (e: ClientRequestException) {
        val code = e.response.status.value
        CallResult.Failure.HttpError(code, e.errorBody())
    } catch (e: SerializationException) {
        CallResult.Failure.SerializationError(ErrorMessage(e.message.toString()))
    } catch (e: IOException) {
        CallResult.Failure.NetworkError
    } catch (e: JsonConvertException) {
        CallResult.Failure.SerializationError(ErrorMessage(e.message.toString()))
    }
}
{% endhighlight %}

the function take the lambdas that return an object `T` in argument
and we can now have a function for each Http method safeGet, safePost etc by just replacing the 
`get { block() }` by `post { block() }`

With these extension functions all the complexity of creating the monad is abstracted away and the api calls in our dataLayer become really clean and simple to use :

{% highlight kotlin %}
override suspend fun getProduct(): CallResult<Product, SimpleError> 
= client.safeGet {
        url(Constants.Api.PRODUCT)
    }
{% endhighlight %}

1 clean expression, perfect !

#### The ApiCall from viewmodel

{% highlight kotlin %}
fun getProduct() {
        viewModelScope.launch {
            when (api.fetchEvents()) {
                is ResultWrapper.Success ->  
                is ResultWrapper.Failure.HttpError -> 
                is ResultWrapper.Failure.NetworkError -> 
                is ResultWrapper.Failure.SerializationError -> 
            }
        }
    }
{% endhighlight %}

