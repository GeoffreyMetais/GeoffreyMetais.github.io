---
title: "Recyclerview++ with DiffUtil"
excerpt: "DiffUtil tool allowed greatly improving lists/grids updates in VLC, here is how we implemented it."
categories:
 - VLC
 - Android
 - code
tags:
 - vlc
 - android
---

[DiffUtil] is an AppCompat utility class developed to calculate the delta between two different datasets of a recyclerview, and dispatch it to its adapter.  
I.e. you pass it the current list and new one you want to display and it automagically handles all the notify[Item/ItemRange][Inserted/Deleted/Moved/Changed] to provide cool RecyclerView animations or subtles item updates (like a single progressbar update instead of the whole item rebinding).
{% include toc icon="code-fork" title="DiffUtil steps" %}
This is the perfect tool to easily manage recyclerview refresh and get dynamic animations or really discreet view updates for every dataset modification.

## Principles
1. Provide your custom [DiffUtil.Callback] fed with the old and new datasets to the [DiffUtil.calculateDiff] method, and obtain a [DiffUtil.DiffResult] which contains all the necessary operations (additions, deletions and optionnally moves) to get from old list to the new one.
2. Update your dataset
3. Call `diffResult.dispatchUpdatesTo(recyclerviewAdapter)`, your recyclerviewAdapter will receive all the corresponding _notifyItem*_ events and profit shiny recyclerview animations!

It is strongly advised to run **all dataset operations in the main thread, to ensure consistency**. And for now (until next post), DiffUtil calculations will also be run in mainthread for consistency concern.
{: .notice--warning}

## Basic usage

Here is a simple example of DiffUtil powered recyclerview update method in browsers adapter:
```java
@MainThread
void update(final ArrayList<Item> items) {
    final DiffUtil.DiffResult result = DiffUtil.calculateDiff(
                new MediaItemDiffCallback(mMediaList, items), false);
    mMediaList = items;
    result.dispatchUpdatesTo(MyAdapter.this);
}
```
You can see we only have these 3 basic steps.  
With `mMediaList` the current dataset, `items` the new one, and `false` the parameter value for [moves detection](#moves-detection), we don't handle it for now.

The base [DiffUtil.Callback] implementation we use:
```java
public class MediaItemDiffCallback extends DiffUtil.Callback {
    protected Item[] oldList, newList;

    public MediaItemDiffCallback(Item[] oldList, Item[] newList) {
        this.oldList = oldList;
        this.newList = newList;
    }

    @Override
    public int getOldListSize() {
        return oldList == null ? 0 :oldList.length;
    }

    @Override
    public int getNewListSize() {
        return newList == null ? 0 : newList.length;
    }

    @Override
    public boolean areItemsTheSame(int oldItemPosition, int newItemPosition) {
        return oldList[oldItemPosition].equals(newList[newItemPosition]);
    }

    @Override
    public boolean areContentsTheSame(int oldItemPosition, int newItemPosition) {
        return true;
    }
}
```
For now, `areContentsTheSame` always returns `true`, we'll see later how to use it for smarter updates.
Let's focus on `areItemsTheSame`: this is simply a equals*-ish* method to determine if two instances correspond to the same model view.

Now, with this diffResult, for every item insertion and deletion we get the precise adapter notification.  
Typical application is list feeding, all new items are nicely inserted or removed which males fancy animations to happen, like if we had set all the `notifyItemInserted` and `notifyItemDeleted` one by one.

Nicer use case is list/grid filtering, check the result, with that simple `update(newList)` call:

![video grid](/assets/images/v2.1/filter.gif){: .align-center}

## Smarter items updates

In video section, we want to update an item view if media progress or thumbnail has changed, but we don't want to completely rebind its view.  
Until VLC 2.0.6, you could experience a flicker of the video you were watching once getting back from video player.
That's because media progress had changed and we blindly updated the whole view (which triggered thumbnail reloading).

Here is the [DiffUtil.Callback] specific to video grid which not only checks if items are the same, but if their content has changed also:
```java
private class VideoItemDiffCallback extends DiffUtil.Callback {
        List<Item> oldList, newList;
        VideoItemDiffCallback(List<Item> oldList, List<Item> newList) {
            this.oldList = oldList;
            this.newList = newList;
        }

        @Override
        public int getOldListSize() {
            return oldList == null ? 0 : oldList.size();
        }

        @Override
        public int getNewListSize() {
            return newList == null ? 0 : newList.size();
        }

        @Override
        public boolean areItemsTheSame(int oldItemPosition, int newItemPosition) {
            return oldList.get(oldItemPosition).equals(newList.get(newItemPosition));
        }

        @Override
        public boolean areContentsTheSame(int oldItemPosition, int newItemPosition) {
            Item oldItem = oldList.get(oldItemPosition);
            Item newItem = newList.get(newItemPosition);
            return oldItem.getTime() == newItem.getTime() &&
              TextUtils.equals(oldItem.getArtworkMrl(), newItem.getArtworkMrl());
        }

        @Nullable
        @Override
        public Object getChangePayload(int oldItemPosition, int newItemPosition) {
            Item oldItem = oldList.get(oldItemPosition);
            Item newItem = newList.get(newItemPosition);
            if (oldItem.getTime() != newItem.getTime())
                return UPDATE_TIME;
            else
                return UPDATE_THUMB;
        }
    }
```

If `areItemsTheSame` returns `true`, `areContentsTheSame` is called for the same items. `areContentsTheSame` returns `false` if we detect small changes we want to propagate to the UI without totally rebinding the concerned item view.
In our case this is two `Item` instances representing the same video file but artwork URL or time value has changed.

If `areContentsTheSame` returns `false`, `getChangePayload` is called for the same items. We define here the updated value and return it. The dispatch util will call `notifyItemChanged(position, payload)` with this value. Then it's up to the adapter to manage the update:

In the adapter, we override `onBindViewHolder(holder, position, payload)` to achieve this:
```java
@Override
public void onBindViewHolder(ViewHolder holder, int position, List<Object> payloads) {
    if (payloads.isEmpty()) {
        onBindViewHolder(holder, position);
    } else {
        MediaWrapper media = mVideos.get(position);
        for (Object data : payloads) {
          switch ((int) data) {
            case UPDATE_THUMB:
                AsyncImageLoader.loadPicture(holder.thumbView, media);
                break;
            case UPDATE_TIME:
                fillView(holder, media);
                break;
          }
        }
    }
}
```
`onBindViewHolder(holder, position, payloads)` makes the smart move and updates the related view(s), instead of rebinding the whole item view. In our case, we load the thumb in the `ImageView` or we regenerate the progress string and update the progress bar with `fillView()`.  
As the dataSet is already updated, we don't need to pass the values here, so I opted for constants refering to the different actions.

## Moves detection

The `DiffUtil.calculateDiff` algorithm can optionnally do a second pass to look for items movements between the old and new lists. If the sort order doesn't change, it is useless to do it.  
For us it's only interesting in video grid for now, so we call:
```java
DiffUtil.calculateDiff(new VideoItemDiffCallback(oldList, newList), detectMoves);
```

Where `detectMoves` variable is a boolean which is set to `true` only in case of video resorting call, in other cases we spare the second pass. Thanks to it, animations are fancier with only moving cards, without this moves detection we'd get disappearing and reappearing cards.

Resorting videos with moves detection:

![video grid](/assets/images/v2.1/sort.gif){: .align-center}

Without moves detection:

![video grid](/assets/images/diffutil/sort_no_detection.gif){: .align-center}

## Going further

We now master diffutil, but we can do better and get calculation out of UI thread.  
Rendez-vous to the second part of this post to check it out.

[DiffUtil]: https://developer.android.com/reference/android/support/v7/util/DiffUtil.html
[DiffUtil.Callback]: https://developer.android.com/reference/android/support/v7/util/DiffUtil.Callback.html
[DiffUtil.calculateDiff]: https://developer.android.com/reference/android/support/v7/util/DiffUtil.html#calculateDiff(android.support.v7.util.DiffUtil.Callback
[DiffUtil.DiffResult]: https://developer.android.com/reference/android/support/v7/util/DiffUtil.DiffResult.html
