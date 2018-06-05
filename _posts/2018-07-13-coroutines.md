---
title: "Modern concurrency on Android with Kotlin"
excerpt: "Quick introduction to Kotlin coroutines based on practical examples on how it could drastically improve Android applications and suppress callback hells"
categories:
 - code
tags:
 - vlc
 - android
---

Current Java/Android concurrency framework leads to callback hells and blocking states because we do not have any other simple way to guarantee thread safety.

With coroutines, kotlin brings a very efficient and complete framework to manage concurrency in a more performant and simple way.

{% include toc icon="code-fork" title="Coroutines way" %}

# Suspending vs blocking

Coroutines do not replace threads, it's more like a framework to manage it.  
Its philosophy is to define an execution context which allows to **wait** for background operations to complete, without blocking the original thread.

The goal here is to avoid callbacks and make concurrency easier.

## Basic usage

Very simple first example, we launch a coroutine in the `UI` context. In it, we retrieve an image from the `IO` one, and process it back in `UI`.  

```kotlin
launch(UI) {
    val image = withContext(IO) { getImage() } // Get from IO context
    imageView.setImageBitmap(image) // Back on main thread
}
```

Staightforward code, like a single threaded function. And while `getImage` runs in `IO` dedicated thread, the main thread is free for any other job!
`withContext` function suspends the current coroutine while its action (`getImage()`) is running. As soon as `getImage()` returns and main looper is available, coroutine resumes on main thread, and `imageView.setImageBitmap(image)` is called.

Second example, we now want 2 background works done to use them. We will use the `async`/`await` duo to make them run in parallel and use their result in main thread as soon as both are ready:

```kotlin
val job = launch(UI) {
    val deferred1 = async { getFirstValue() }
    val deferred2 = async(IO) { getSecondValue() }
    useValues(deferred1.await(), deferred2.await())
}

job.join() // suspends current coroutine until job is done
```

`async` is similar to `launch` but returns a `deferred` (which is the Kotlin equivalent of `Future`), so we can get its result with `await()`. Called with no parameter, it runs in `CommonPool` context.

And once again, the main thread is free while we are waiting for our 2 values.

As you can see, `launch` funtion returns a `Job` that can be used to wait for the operation to be over, with the `join()` function. It works like in any other language, except that it **suspends the coroutine instead of blocking the thread**.

## Dispatch

Dispatching is a key notion with coroutines, it's the action to 'jump' from a thread to another one.

Let's look at our current java equivalent to `UI` dispatching, which is `runOnUiThread`:

```java
public final void runOnUiThread(Runnable action) {
    if (Thread.currentThread() != mUiThread) {
        mHandler.post(action); // Dispatch
    } else {
        action.run(); // Immediate execution
    }
}
```

Android implementation of `UI` context is a dispatcher based on a Handler. So this really is the matching implementation:

```kotlin
launch(UI) { ... }
        vs
launch(UI, CoroutineStart.UNDISPATCHED) { ... }
```

`launch(UI)` posts a `Runnable` in a `Handler`, so its code execution is not immediate.  
`launch(UI, CoroutineStart.UNDISPATCHED)` will immediately execute its lambda expression in the current thread.

`UI` guarantees that **coroutine is dispatched on main thread when it resumes**, and it uses a `Handler` as the native Android implementation to post in the application event loop.

See its actual implementation:

```kotlin
val UI = HandlerContext(Handler(Looper.getMainLooper()), "UI")
```

To get a better understanding of Android dispatching, you can read this blog post on [Understanding Android Core: Looper, Handler, and HandlerThread](https://blog.mindorks.com/android-core-looper-handler-and-handlerthread-bd54d69fe91a).

## Coroutine context

A couroutine context (aka coroutine dispatcher) defines on which thread its code will execute, what to do in case of thrown exception and refers to a parent context, to propagate cancellation.

```kotlin
val job = Job()
val exceptionHandler = CoroutineExceptionHandler {
    coroutineContext, throwable -> whatever(throwable)
}

launch(CommonPool+exceptionHandler, parent = job) { ... }
```

`job.cancel()` will cancel all coroutines that have `job` as a parent. And `exceptionHandler` will receive all thrown exceptions in these coroutines.

## Notes

- Coroutines limit Java interoperability
- Confine mutablility to avoid locks
- Coroutines are for ~~threading~~ **waiting**
  - Avoid I/O in CommonPool (and UI…)
  - SharedPool dispatcher coming soon to improve this
- Threads are expensive, so are single-thread contexts
- `CommonPool` is based on a ForkJoinPool on Android 5+
- Coroutines can be used via Channels

`CommonPool` is a threadpool, aimed to be intensively used. If you perform I/O tasks in it, you could get all its threads blocked at the same time and any coroutine relying on it will be waiting.  
JetBrains is adressing this issue and will probably release a shared pool guarantying that at least one thread is always free from I/O operations.  
For now, it's important to keep it free from long tasks and execute them in dedicated threads/contexts, like:
{: .notice--warning}

```kotlin
val IO = ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L,
        TimeUnit.SECONDS, SynchronousQueue<Runnable>()
        ).asCoroutineDispatcher()
```

# Callbacks and locks elimination with channels

Channel definition from JetBrain documentation:

A `Channel` is conceptually very similar to `BlockingQueue`. One key difference is that instead of a blocking put operation it has a suspending `send`, and instead of a blocking take operation it has a suspending `receive`.

## Actors

Let's start with a simple tool to use Channels, the `Actor`.

We already saw it in this blog with the [DiffUtil kotlin implementation](/code/diffutil-threading/#the-kotlin-way).

`Actor` is, yet again, very similar to `Handler`: we define a coroutine context (so, the tread where to execute actions) and it will execute it in a sequencial order.  

Difference is it uses coroutines of course :), we can specify a capacity and **executed code can suspend**.

An `actor` will basically forward any order to a coroutine `Channel`. It will **guaranty the order execution and confine operations in its context**. It greatly helps to remove `synchronize` calls and keep all threads free!

```kotlin
protected val updateActor by lazy {
    actor<Update>(UI, capacity = Channel.UNLIMITED) {
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
suspend fun filter(query: String?) = updateActor.offer(Filter(query))
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

(Sample shamefully copied from JetBrain's [Guide to UI programming with coroutines](https://github.com/Kotlin/kotlinx.coroutines/blob/master/ui/coroutines-guide-ui.md))

Actors can be profitable for Android UI management too, they can ease tasks cancellation and prevent overloading of the UI thread.

Let's first declare a `JobHolder` interface, which will be applied to our `Activity`. This job will be used as a parent for any user triggered task, and will allow their cancellation.

```kotlin
interface JobHolder {
    val job: Job
}
```

Let's implement it and call `job.cancel()` when activity is destroyed.

```kotlin
class MyActivity : AppCompatActivity(), JobHolder {
    override val job: Job = Job() // the instance of a Job for this activity

    override fun onDestroy() {
        super.onDestroy()
        job.cancel() // cancel the job when activity is destroyed
    }
}
```

A bit better, with an [extension function](https://kotlinlang.org/docs/reference/extensions.html#extension-functions), we can make this `Job` accessible from any `View` of a `JobHolder`

```kotlin
val View.contextJob: Job
    get() = (context as? JobHolder)?.job ?: NonCancellable
```

We can now combine all this, `setOnClick` function creates a conflated `actor` to manage its `onClick` actions. In case of multiple clicks, intermediates actions will be ignored, preventing any **ANR**, and these actions will be executed in a context with `contextJob` as a parent. So it will be cancelled when `Activity` is destroyed 😎

```kotlin
fun View.setOnClick(action: suspend () -> Unit) {
    // launch one actor as a parent of the context job
    val eventActor = actor<Unit>(context = UI,
                start = CoroutineStart.UNDISPATCHED,
                capacity = Channel.CONFLATED,
                parent = contextJob) {
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
launch(UI, parent = untilDestroy) { /* amazing things happen here! */ }
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
        launch(UI) {
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

// usage (within UI coroutine context)
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
