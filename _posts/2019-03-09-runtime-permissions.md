---
title: "Android runtime permissions in one (suspending) function"
excerpt_separator: <!--summary-->
comments: true
categories:
  - code
tags:
  - kotlin
  - android

toc: true
toc_label: "Steps"
toc_icon: "cogs"
---
This post offers a basic implementation of a single suspending function managing the runtime permission process.
<!--summary-->

# Runtime permission API

If you are reading this, you should already know this API. It is only composed of [`requestPermissions`](https://developer.android.com/reference/android/support/v4/app/ActivityCompat.html#requestpermissions) method, and its related callback [`onRequestPermissionsResult`](https://developer.android.com/reference/android/support/v4/app/ActivityCompat.OnRequestPermissionsResultCallback.html#onRequestPermissionsResult(int,%20java.lang.String[],%20int[])). But this is a View-level API, not really convenient.

We still are lucky, this API is accessible from fragments, and that will allow us to make it way easier to use.

# Headless Fragment

The trick is to make it a **headless fragment**, i.e. a fragment without a view. It's still bound to its activity and can fire dialogs, that's what we want.

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    retainInstance = true
    requestPermissions(arrayOf(Manifest.permission.READ_EXTERNAL_STORAGE), PERMISSION_STORAGE_CODE)
}

// No onCreateView() overriding

override fun onRequestPermissionsResult(requestCode: Int, permissions: Array<String>, grantResults: PermissionResults) {
    when (requestCode) {
        PERMISSION_STORAGE_CODE -> {
            if (grantResults.granted()) {
                deferredGrant.complete(true)
                exit()
                return
            } else if (shouldShowRequestPermissionRationale(...)) {
                // We insist
                return
            }
            deferredGrant.complete(false)
            exit()
        }
    }
}
protected fun exit() {
    retainInstance = false
    activity?.run {
        if (!isFinishing) supportFragmentManager
            .beginTransaction()
            .remove(this@BaseHeadlessFragment)
            .commitAllowingStateLoss()
    }
}
```

Our runtime permission implementation has become modular, it can be called from any activity just by starting this fragment. That's a first step!

The permission grant result is handled by `deferredGrant.complete(boolean)`, that's how we can get a `suspend` function controlling our fragment.

# Deferred behavior

We will now leverage it, and create a single function to launch this permission granting process and get its result.  
For this, we use a  [`CompletableDeferred`], which is a [`Deferred`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-deferred/index.html) like the one returned by the [`async`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html) function, but we can complete it by ourselves once we have our result.

All we have to do is to launch our fragment, prepare a [`CompletableDeferred`] and await for it.

Voilà! We have our function:

```kotlin
protected val deferredGrant = CompletableDeferred<Boolean>()

suspend fun awaitGrant() = deferredGrant.await()

companion object {
    suspend fun FragmentActivity.getStoragePermission() : Boolean {
        if (canReadStorage()) return true
        return launchFragment().awaitGrant()
    }
}
```

# Profit

Usage is straghtforward:  
From a coroutine, any activity can call `getStoragePermission()`, and it suspends until user has made its choice.  
No callback anymore, and it's callable from any Activity or Fragment in your app \o/

```kotlin
if (getStoragePermission()) {
    proceed()
} else {
    retreat()
}
```

While the user is asked for the permission you want for your app, coroutine execution is suspended by `getStoragePermission()` condition. Then, the relevant branch of this `if` expression will be executed.

![Permission dialog](/assets/images/permissions/permission-dialog.png){: .align-center}

[`CompletableDeferred`]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-completable-deferred/