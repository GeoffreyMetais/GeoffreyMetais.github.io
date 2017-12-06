---
title: "DiffUtil off the UI thread"
excerpt: "Now we have superb RecyclerView updates with DiffUtil, we want to preserve UI thread from being blocked by potentially heavy calculations"
categories:
 - code
tags:
 - vlc
 - android
---

{% include toc icon="code-fork" title="DiffUtil steps" %}
As stated in the [previous post](/code/diffutil/), we do process all [DiffUtil.DiffResult] calculations in main thread to preserve adapter state consistency.  
But in VLC, we have to deal with potentially HUGE datasets, so calculation could take some time.  

Background calculation is mandatory then, and we have to preserve dataset consistency. I lost a few days trying different techniques then finally chose to stack updates within a queue and use it for all dataset operations, because it provides consistency safetyness.  
I've been inspired by [Jon F Hancock blog post] to get this right.

To achieve background calculation and preserve data consistency, we now have to use our `update()` method for **all** dataset updates or manage the pending queue state manually.
{: .notice--warning}


## Threading

`update(list)` method is now splitted in two, in order to allow queueing and recursivity:  
`update(list)` which is now limited to queueing the new list and triggering `internalUpdate(list)` to do the actual job.

Notice all queue accesses or modifications are done in the main thread (for the same reasons that for dataset changes).
{: .notice--info}

```java
// Our queue with next dataset
private final ArrayDeque<Item[]> mPendingUpdates = new ArrayDeque<>();

@MainThread
void update(final ArrayList<Item> newList) {
    mPendingUpdates.add(newList);
    if (mPendingUpdates.size() == 1)
        internalUpdate(newList); //no pending update, let's go
}

//private method, called exclusively by update()
private void internalUpdate(final ArrayList<Item> newList) {
    VLCApplication.runBackground(new Runnable() {
        @Override
        public void run() {
            final DiffUtil.DiffResult result = DiffUtil.calculateDiff(new MediaItemDiffCallback(mDataset, newList), false);
            //back to main thread for the update
            VLCApplication.runOnMainThread(new Runnable() {
                @Override
                public void run() {
                    mDataset = newList;
                    result.dispatchUpdatesTo(BaseBrowserAdapter.this);
                    //We are done with this dataset
                    mPendingUpdates.remove();
                    //Process the next queued dataset if any
                    if (!mPendingUpdates.isEmpty())
                        internalUpdate(mPendingUpdates.peek());
                }
            });
        }
    });
}
```

For simple actions, like item insertion/removal, we must check the `mPendingUpdates` state. Either we handle it, either we use `update(list)` in order to respect the queue process we just set. So, we have to copy the most recent dataset, add/remove the item then call `update(list)`.

Using `mDataset` as the current reference state can be a mistake, if `mPendingUpdates` is not empty, another dataset will be processed between `mDataset` and our new list with item added or removed. In this case, we have to peek the last list from `mPendingUpdates`.
{: .notice--warning}

```java
@MainThread
void addItem(Item item) {
  ArrayList<Item> newList = new ArrayList<>(mPendingUpdates.isEmpty() ? mDataset : mPendingUpdates.peekLast());
  newList.add(item);
  update(newList);
}
```
For item removal, I'd recommend to just avoid calling it with position only, prefer to pass the item reference. Because the position value is likely to be wrong if there is a pending update at this time.

## Skip queued updates

In case you can receive a bunch of updates while [DiffUtil] is calculating the [DiffUtil.DiffResult], you get a stack of new datasets to process. Let's skip to the last one: as we made sure they are consistent we can do it. That's just factorizing the updates.  
We have to clear the `mPendingUpdates` queue from all its elements but the last one.

Here is our current queue processing:
```java
mPendingUpdates.remove();
if (!mPendingUpdates.isEmpty())
    internalUpdate(mPendingUpdates.peek());
```

Which becomes:
```java
mPendingUpdates.remove();
if (!mPendingUpdates.isEmpty()) {
    if (mPendingUpdates.size() > 1) { // more than one update queued
        ArrayList<Item> lastList = mPendingUpdates.peekLast();
        mPendingUpdates.clear();
        mPendingUpdates.add(lastList);
    }
    internalUpdate(mPendingUpdates.peek());
}
```

## Code factorization

Here is my base adapter class, dedicated to pending queue management. Children classes just need to call `update(newList)` for any update.  
(I chose to not specify `List<T>` because I also use arrays)

```java
public abstract class BaseQueuedAdapter <T, VH extends RecyclerView.ViewHolder> extends RecyclerView.Adapter<VH> {

    protected T mDataset;
    private final ArrayDeque<T> mPendingUpdates = new ArrayDeque<>();
    final Handler mHandler = new Handler(Looper.getMainLooper());

    @MainThread
    public boolean hasPendingUpdates() {
        return !mPendingUpdates.isEmpty();
    }

    @MainThread
    public T peekLast() {
        return mPendingUpdates.isEmpty() ? mDataset : mPendingUpdates.peekLast();
    }

    @MainThread
    public void update(final T items) {
        mPendingUpdates.add(items);
        if (mPendingUpdates.size() == 1)
            internalUpdate(items);
    }

    private void internalUpdate(final T newList) {
        new thread(new Runnable() {
            @Override
            public void run() {
                final DiffUtil.DiffResult result = DiffUtil.calculateDiff(new ItemDiffCallback(mDataList, newList), false);
                mHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        mDataset = newList;
                        result.dispatchUpdatesTo(BaseQueuedAdapter.this);
                        processQueue();
                    }
                });
            }
        }).start();
    }

    @MainThread
    private void processQueue() {
        mPendingUpdates.remove();
        if (!mPendingUpdates.isEmpty()) {
            if (mPendingUpdates.size() > 1) {
                T lastList = mPendingUpdates.peekLast();
                mPendingUpdates.clear();
                mPendingUpdates.add(lastList);
            }
            internalUpdate(mPendingUpdates.peek());
        }
    }
}
```

My adapter class becomes:
```java
public class MyAdapter extends BaseQueuedAdapter<List<Item>, MyAdapter.ViewHolder>
```

That's it, we now have asynchronous and classy RecyclerView updates without extra boilerplate 😎

## The Kotlin way

Because we can do better with Kotlin!  

First of all, we use an [Actor], to easily operate a [Channel], so we can send this one every single update we want.  It's capacity is set to `CONFLATED` which means that the channel will bufffer at most one element: when busy, it will just replace the buffered element by the new one we provided. That's a smarter way to process our waiting queue.

When ready, `eventActor` will call `internalUpdate` in the `CommonPool`, in which we still do the calculation.  
UI dispatch is done in `launch` higher-order function with `UI` as parameter so we can `join()` it:  
In Kotlin, `join()` doesn't block the current thread but it *suspends* it while UI job is done, so no thread blocking!

And as a bonus, we gained thread safety! We can call `update(list)` from any thread now.

```kotlin
abstract class DiffUtilAdapter<D, VH : RecyclerView.ViewHolder> : RecyclerView.Adapter<VH>() {

    protected var mDataset: List<D> = listOf()
    private val diffCallback by lazy(LazyThreadSafetyMode.NONE) { DiffCallback() }
    private val eventActor = actor<List<D>>(capacity = Channel.CONFLATED) { for (list in channel) internalUpdate(list) }

    fun update (list: List<D>) = eventActor.offer(list)

    private suspend fun internalUpdate(list: List<D>) {
        val result = DiffUtil.calculateDiff(diffCallback.apply { newList = list }, false)
        launch(UI) {
            mDataset = list
            result.dispatchUpdatesTo(this@DiffUtilAdapter)
        }.join()
    }

    private inner class DiffCallback : DiffUtil.Callback() {
        lateinit var newList: List<D>
        override fun getOldListSize() = mDataset.size
        override fun getNewListSize() = newList.size
        override fun areContentsTheSame(oldItemPosition : Int, newItemPosition : Int) = true
        override fun areItemsTheSame(oldItemPosition : Int, newItemPosition : Int) = mDataset[oldItemPosition] == newList[newItemPosition]
    }
}
```


[DiffUtil]: https://developer.android.com/reference/android/support/v7/util/DiffUtil.html
[DiffUtil.Callback]: https://developer.android.com/reference/android/support/v7/util/DiffUtil.Callback.html
[DiffUtil.DiffResult]: https://developer.android.com/reference/android/support/v7/util/DiffUtil.DiffResult.html
[Jon F Hancock blog post]: https://medium.com/@jonfhancock/get-threading-right-with-diffutil-423378e126d2
[Actor]: https://github.com/Kotlin/kotlinx.coroutines/blob/master/coroutines-guide.md#actors
[Channel]: https://github.com/Kotlin/kotlinx.coroutines/blob/master/coroutines-guide.md#channels