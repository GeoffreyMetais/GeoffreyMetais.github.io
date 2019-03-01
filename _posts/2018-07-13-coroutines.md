---
title: "Modern concurrency on Android with Kotlin"
excerpt: "Quick introduction to Kotlin coroutines based on practical examples on how it could drastically improve Android applications and suppress callback hells"

categories:
 - code
tags:
 - vlc
 - android

toc: true
toc_label: "Coroutines way"
toc_icon: "code-branch"
---

Current Java/Android concurrency framework leads to callback hells and blocking states because we do not have any other simple way to guarantee thread safety.

With coroutines, kotlin brings a very efficient and complete framework to manage concurrency in a more performant and simple way.

# Suspending vs blocking

Coroutines do not replace threads, it's more like a framework to manage it.  
Its philosophy is to define an execution context which allows to **wait** for background operations to complete, without blocking the original thread.

The goal here is to avoid callbacks and make concurrency easier.

## Basic usage

Very simple first example, we launch a coroutine in the `Main` context (main thread). In it, we retrieve an image from the `IO` one, and process it back in `Main`.  

```kotlin
launch(Dispatchers.Main) {
    val image = withContext(Dispatchers.IO) { getImage() } // Get from IO context
    imageView.setImageBitmap(image) // Back on main thread
}
```

Staightforward code, like a single threaded function. And while `getImage` runs in `IO` dedicated threadpool, the main thread is free for any other job!
`withContext` function suspends the current coroutine while its action (`getImage()`) is running. As soon as `getImage()` returns and main looper is available, coroutine resumes on main thread, and `imageView.setImageBitmap(image)` is called.

Second example, we now want 2 background works done to use them. We will use the `async`/`await` duo to make them run in parallel and use their result in main thread as soon as both are ready:

```kotlin
val job = launch(Dispatchers.Main) {
    val deferred1 = async(Dispatchers.Default) { getFirstValue() }
    val deferred2 = async(Dispatchers.IO) { getSecondValue() }
    useValues(deferred1.await(), deferred2.await())
}

job.join() // suspends current coroutine until job is done
```

`async` is similar to `launch` but returns a `deferred` (which is the Kotlin equivalent of `Future`), so we can get its result with `await()`. Called with no parameter, it runs in current scope default context.

And once again, the main thread is free while we are waiting for our 2 values.

As you can see, `launch` funtion returns a `Job` that can be used to wait for the operation to be over, with the `join()` function. It works like in any other language, except that it **suspends the coroutine instead of blocking the thread**.

## Dispatch

Dispatching is a key notion with coroutines, it's the action to 'jump' from a thread to another one.

Let's look at our current java equivalent of `Main` dispatching, which is `runOnUiThread`:

```java
public final void runOnUiThread(Runnable action) {
    if (Thread.currentThread() != mUiThread) {
        mHandler.post(action); // Dispatch
    } else {
        action.run(); // Immediate execution
    }
}
```

Android implementation of `Main` context is a dispatcher based on a Handler. So this really is the matching implementation:

```kotlin
launch(Dispatchers.Main) { ... }
        vs
launch(Dispatchers.Main, CoroutineStart.UNDISPATCHED) { ... }


// Since kotlinx 0.26:
launch(Dispatchers.Main.immediate) { ... }
```

`launch(Dispatchers.Main)` posts a `Runnable` in a `Handler`, so its code execution is not immediate.  
`launch(Dispatchers.Main, CoroutineStart.UNDISPATCHED)` will immediately execute its lambda expression in the current thread.

`Dispatchers.Main` guarantees that **coroutine is dispatched on main thread when it resumes**, and it uses a `Handler` as the native Android implementation to post in the application event loop.

Its actual implementation looks like:

```kotlin
val Main: HandlerDispatcher = HandlerContext(mainHandler, "Main")
```

To get a better understanding of Android dispatching, you can read this blog post on [Understanding Android Core: Looper, Handler, and HandlerThread](https://blog.mindorks.com/android-core-looper-handler-and-handlerthread-bd54d69fe91a).

## Coroutine context

A couroutine context (aka coroutine dispatcher) defines on which thread its code will execute, what to do in case of thrown exception and refers to a parent context, to propagate cancellation.

```kotlin
val job = Job()
val exceptionHandler = CoroutineExceptionHandler {
    coroutineContext, throwable -> whatever(throwable)
}

launch(Disaptchers.Default+exceptionHandler+job) { ... }
```

`job.cancel()` will cancel all coroutines that have `job` as a parent. And `exceptionHandler` will receive all thrown exceptions in these coroutines.

## Scope

A `coroutineScope` makes errors handling easier:  
If any child coroutine fails, the entire scope fails and all of children coroutines are cancelled.

In the `async` example, if the retrieval of a value failed, the other one continued then we would have a broken state to manage.  
With a `coroutineScope`, `useValues` will be called only if both values retrieval succeeded. Also, if `deferred2` fails, `deferred1` is cancelled.

```kotlin
coroutineScope { 
    val deferred1 = async(Dispatchers.Default) { getFirstValue() }
    val deferred2 = async(Dispatchers.IO) { getSecondValue() }
    useValues(deferred1.await(), deferred2.await())
}
```

We also can "scope" an entire class to define its default `CoroutineContext` and leverage it.

Example of a class implementing `CoroutineScope`:
```kotlin
open class ScopedViewModel : ViewModel(), CoroutineScope {
    protected val job = Job()
    override val coroutineContext = Dispatchers.Main+job

    override fun onCleared() {
        super.onCleared()
        job.cancel()
    }
}
```

Launching coroutines in a `CoroutineScope`:

`launch` or `async` default dispatcher is now the current scope dispatcher. And we can still choose a different one the same way we did before.

```kotlin
launch {
    val foo = withContext(Dispatchers.IO) { … }
    // lambda runs within scope's CoroutineContext
    …
}

launch(Dispatchers.Default) {
    // lambda runs in default threadpool.
    …
}
```

Standalone coroutine launching (outside of any `CoroutineScope`):
```kotlin
GlobalScope.launch(Dispatchers.Main) {
    // lambda runs in main thread.
    …
}
```

We can even define a scope for application with dispatcher `Main` as default:
```kotlin
object AppScope : CoroutineScope by GlobalScope {
    override val coroutineContext = Dispatchers.Main.immediate
}
```

## Notes

- Coroutines limit Java interoperability
- Confine mutablility to avoid locks
- Coroutines are for ~~threading~~ **waiting**
  - Avoid I/O in `Dispatchers.Default` (and `Main`…)
  - `Dispatchers.IO` designed for this
- Threads are expensive, so are single-thread contexts
- `Dispatchers.Default` is based on a ForkJoinPool on Android 5+
- Coroutines can be used via Channels


# Callbacks and locks elimination with channels

Channel definition from JetBrain documentation:

A `Channel` is conceptually very similar to `BlockingQueue`. One key difference is that instead of a blocking put operation it has a suspending `send` (or a non-blocking `offer`), and instead of a blocking take operation it has a suspending `receive`.

## Actors

Let's start with a simple tool to use Channels, the `Actor`.

We already saw it in this blog with the [DiffUtil kotlin implementation](/code/diffutil-threading/#the-kotlin-way).

`Actor` is, yet again, very similar to `Handler`: we define a coroutine context (so, the tread where to execute actions) and it will execute it in a sequencial order.  

Difference is it uses coroutines of course :), we can specify a capacity and **executed code can suspend**.

An `actor` will basically forward any order to a coroutine `Channel`. It will **guaranty the order execution and confine operations in its context**. It greatly helps to remove `synchronize` calls and keep all threads free!

```kotlin
protected val updateActor by lazy {
    actor<Update>(capacity = Channel.UNLIMITED) {
        for (update in channel) when (update) {
            Refresh -> updateList()
            is Filter -> filter.filter(update.query)
            is MediaUpdate -> updateItems(update.mediaList as List<T>)
            is MediaAddition -> addMedia(update.media as T)
            is MediaListAddition -> addMedia(update.mediaList as List<T>)
            is MediaRemoval -> removeMedia(update.media as T)
        }
    }
}
// usage
fun filter(query: String?) = updateActor.offer(Filter(query))
//or
suspend fun filter(query: String?) = updateActor.send(Filter(query))
```

In this example, we take advantage of the Kotlin sealed classes feature to select which action to execute.

```kotlin
sealed class Update
object Refresh : Update()
class Filter(val query: String?) : Update()
class MediaAddition(val media: Media) : Update()
```

And all this actions will be queued, they will never run in parallel. That's a good way to achieve **mutability confinement**.

## Android lifecycle + Coroutines

Actors can be profitable for Android UI management too, they can ease tasks cancellation and prevent overloading of the main thread.

Let's implement it and call `job.cancel()` when activity is destroyed.

```kotlin
class MyActivity : AppCompatActivity(), CoroutineScope {
    protected val job = SupervisorJob() // the instance of a Job for this activity
    override val coroutineContext = Dispatchers.Main.immediate+job


    override fun onDestroy() {
        super.onDestroy()
        job.cancel() // cancel the job when activity is destroyed
    }
}
```

A `SupervisorJob` is similar to a regular Job with the only exception that cancellation is propagated only downwards.  
So we do not cancel all coroutines in the `Activity`, when one fails.
{: .notice--info}

A bit better, with an [extension function](https://kotlinlang.org/docs/reference/extensions.html#extension-functions), we can make this `CoroutineContext` accessible from any `View` of a `CoroutineScope`

```kotlin
val View.coroutineContext: CoroutineContext?
    get() = (context as? CoroutineScope)?.coroutineContext
```

We can now combine all this, `setOnClick` function creates a conflated `actor` to manage its `onClick` actions. In case of multiple clicks, intermediates actions will be ignored, preventing any **ANR**, and these actions will be executed in `Activity`'s scope`. So it will be cancelled when `Activity` is destroyed 😎

```kotlin
fun View.setOnClick(action: suspend () -> Unit) {
    // launch one actor as a parent of the context job
    val scope = (context as? CoroutineScope)?: AppScope
    val eventActor = scope.actor<Unit>(capacity = Channel.CONFLATED) {
        for (event in channel) action()
    }
    // install a listener to activate this actor
    setOnClickListener { eventActor.offer(Unit) }
}
```

In this example, we set the `Channel` as *Conflated* to ignore events when we have too much of them. You can change it to `Channel.UNLIMITED` if you prefer to queue events without missing anyone of them, but still protect your app from *ANR*
{: .notice--warning}

We also can combine coroutines and `Lifecycle` frameworks to automate UI tasks cancellation:

```kotlin
val LifecycleOwner.untilDestroy: Job get() {
    val job = Job()

    lifecycle.addObserver(object: LifecycleObserver {
        @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
        fun onDestroy() { job.cancel() }
    })

    return job
}
//usage
GlobalScope.launch(Dispatchers.Main, parent = untilDestroy) {
    /* amazing things happen here! */
}
```

## Callbacks mitigation (Part 1)

Example of a callback based API use transformed thank to a `Channel`. 

API works like this:

1. `requestBrowsing(url, listener)` triggers the parsing of folder at `url` address.
2. The `listener` receives `onMediaAdded(media: Media)` for each discovered media in this folder.
3. `listener.onBrowseEnd()` is called once folder parsing is done.

Here is the old `refresh` function in VLC browser provider:

```kotlin
private val refreshList = mutableListOf<Media>()

fun refresh() = requestBrowsing(url, refreshListener)

private val refreshListener = object : EventListener{
    override fun onMediaAdded(media: Media) {
        refreshList.add(media))
    }
    override fun onBrowseEnd() {
        val list = refreshList.toMutableList()
        refreshList.clear()
        launch {
            dataset.value = list
            parseSubDirectories()
        }
    }
}
```

How to improve this?

We create a channel, which will be initiated in `refresh`. Browser callbacks will now only forward media to this channel then close it.

`Refresh` function is now easier to understand. It sets the channel, calls the VLC browser then fills a list with the media and processes it.

Instead of the `select` or `consumeEach` functions, we can use `for` to wait for media and it will break once `browserChannel` is closed

```kotlin
private lateinit var browserChannel : Channel<Media>

override fun onMediaAdded(media: Media) {
    browserChannel.offer(media)
}

override fun onBrowseEnd() {
    browserChannel.close()
}

suspend fun refresh() {
    browserChannel = Channel(Channel.UNLIMITED)
    val refreshList = mutableListOf<Media>()
    requestBrowsing(url)
    //Suspends at every iteration to wait for media
    for (media in browserChannel) refreshList.add(media)
    //Channel has been closed
    dataset.value = refreshList
    parseSubDirectories()
}
```

## Callbacks mitigation (Part 2): Retrofit

Second approach, we don't use kotlinx-coroutines at all but the coroutine core framework.  
Let's see how coroutines really work!

`retrofitSuspendCall` function wraps a Retrofit `Call` request to make it a `suspend` function.  
With `suspendCoroutine` we call the `Call.enqueue` method and suspend the coroutine. The provided callback will call `continuation.resume(response)` to resume the coroutine with the server response once received.

Then, we just have to bundle our Retrofit functions in `retrofitSuspendCall` to have a suspending functions returning the requests result.

``` kotlin
suspend inline fun <reified T> retrofitSuspendCall(request: () -> Call<T>
) : Response<T> = suspendCoroutine { continuation ->
    request.invoke().enqueue(object : Callback<T> {
        override fun onResponse(call: Call<T>, response: Response<T>) {
            continuation.resume(response)
        }
        override fun onFailure(call: Call<T>, t: Throwable) {
            continuation.resumeWithException(t)
        }
    })
}

suspend fun browse(path: String?) = retrofitSuspendCall {
    ApiClient.browse(path)
}

// usage (within Main coroutine context)
livedata.value = Repo.browse(path)
```

This way, the network blocking call is done in Retrofit dedicated thread, coroutine is here to wait for the response, and in-app usage couldn't be simpler!

This implementation is inspired by [gildor/kotlin-coroutines-retrofit](https://github.com/gildor/kotlin-coroutines-retrofit) library, which makes it ready to use.  
[JakeWharton/retrofit2-kotlin-coroutines-adapter](https://github.com/JakeWharton/retrofit2-kotlin-coroutines-adapter) is also available with another implementation, for the same result.

# To be continued

`Channel` framework can be used in many other ways, you can look at [BroadcastChannel](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/-broadcast-channel/) for more powerful implementations according to your needs.  
We can also create channels with the [Produce](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.experimental.channels/produce.html) function.  
It can also be useful for communication between UI components: an adapter can pass click events to its Fragment/Activity via a `Channel` or an `Actor` for example.

Related readings:

- [Coroutines guide](https://github.com/Kotlin/kotlinx.coroutines/blob/master/coroutines-guide.md)
- [Guide to UI programming with coroutines](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/coroutines-guide-ui.md)
- [Understanding Android Core: Looper, Handler, and HandlerThread](https://blog.mindorks.com/android-core-looper-handler-and-handlerthread-bd54d69fe91a)
- [Presenter as a Function: Reactive MVP for Android Using Kotlin Coroutines](https://medium.com/@rocketwagon/presenter-as-a-function-reactive-mvp-for-android-using-kotlin-coroutines-442fc4c77119)
- [Sample DiffUtil implementation](https://gist.github.com/GeoffreyMetais/7c97389fed6b1674e4113d10bc656b92)
