---
title: "Work is in progress on VLC!"
excerpt_separator: <!--summary-->
comments: false
categories:
  - features
tags:
  - vlc
  - android
---

Not so much activity on Play Store since v2.0.6 release.  
I've been quite busy this summer but VLC keeps going on with very interesting evolutions!  
<!--summary-->
This post will only deal with features, not their implementation. I will later publish other posts, more interesting for developers, about it.

{% include toc icon="gears" title="Incoming features" %}

2.1.x version name will be used for beta, we'll decide later which version the stable release will be.  
For now, let me introduce you to the new VLC for Android.

## UX evolutions

### Presentation
Video cards have been refactored, we now show you video informations over its cover picture. This is a nicer presentation and gives room for displaying more videos at once.
Others lists view have been reworked too, like audio media, browsers and history. We got rid of cardview pattern, and got back to a more flat and clean design.

Audio lists have been a bit lifted too but this change is lighter.

Info page has also been redesigned with a fancy collapse effect for thumbnail or cover.

![video grid](/assets/images/v2.1/video_grid.png){: .align-center}

![info panel](/assets/images/v2.1/songs_list.png)
![info panel](/assets/images/v2.1/screen_info.png)

### New Audio Player Style

Audio player background is now a blurred version of current art cover if available.

![player list](/assets/images/v2.1/screen_player_list.png)
![player cover](/assets/images/v2.1/screen_player_cover.png)

### Dynamic UI

#### Scrolling

The action bar will now hide when your scroll your media lists, to save the more useful space possible while you're looking for your media.

![list scroll](/assets/images/v2.1/scroll.gif){: .align-center}

Album view has been revamped to become more #Material and now serves for playlists too

![video grid](/assets/images/v2.1/album.gif){: .align-center}

#### Updating content

Thanks to the *DiffUtil* tool from Appcompat library, grids/lists updates are now animated and more efficient.  
One disturbing side effect is when you refresh and there's no change in your media library. You won't see any update, not even any flickering.  
But the cool thing is whith actual content update, like during media scan or filtering the current view with [the new search UX](#search), insertions/deletions and position changes are animated:

![video grid](/assets/images/v2.1/sort.gif){: .align-center}

### TV design

TV interface had its own lifting too, nothing really impressive here.
Colors are less depressive and we make long media titles scroll, I heard your (justified) frustration :)

The interesting new feature on TV is the Picture-In-Picture (aka PIP) mode, available for all Android TVs powered by Android 7+.  
It's really like our popup mode in classic phone/tablet interface (so we used the very same icon in *video advanced options* for PIP).  
You can still watch your video while using another application.

![video grid](/assets/images/v2.1/tv_pip.png){: .align-center}

![audio player](/assets/images/v2.1/tv_audio_player.png){: .align-center}

## 360° Videos support

VLC now supports 360° videos, you can change viewpoint by swiping or with remote control arrows.

![video 360](/assets/images/v2.1/360.png){: .align-center}

Cardboard/VR mode is not available yet but we are working on it.

## Search
Search has been splitted in two modes:
- First step, the text you type in triggers a **filtering** in the current view. Exaclty like in current playlist view.

![video grid](/assets/images/v2.1/filter.gif){: .align-center}

- Then, if you want to do a global search, click on the `SEARCH IN ALL MEDIALIBRARY` button to show the new search activity. Which will bring detailed results grouped by video/artist/album/songs/genres/playlist

![video grid](/assets/images/v2.1/search.gif){: .align-center}

Bonus:
VLC will now be compatible with voice search.  
Asking Google Now *"Search game in VLC"* will trigger a *game* search and show you this new search result screen.



![video grid](/assets/images/v2.1/search.png){: .align-center}

## Android Auto

This release will bring Android Auto compatibility.  
You'll be able to use VLC as your travel music player with easy browsing into you audio library, with the minimum possible distraction from driving.

![video grid](/assets/images/v2.1/auto_main.png){: .align-center}

![video grid](/assets/images/v2.1/auto_menu.png){: .align-center}

## Action mode

You can now select multiple items by long press on them (classic context menu is still available with the three dots icon) and enjoy actions like *play it all* or *add selection to playlist*  
Actions available depend on media type and selection count.

This is very handy for starting or creating a playlist.

![video grid](/assets/images/v2.1/actionmode.png){: .align-center}

## Miscellaneous new features

* DayNight mode integration
* Restored double/long click on remote play to skip songs
* Removed sound lowering on notification
* Force previous song on audioplayer swipe
* Fix audioplayer layout for black theme and RTL
* Save subtitles delay and optionally audio delay for each file.
* Support for LG devices with 18:9 aspect ratio

## Under the hood

### MediaLibrary

That's the most important change in this update, because it affects the whole application, but you should barely notice it...

VLC now uses [medialibrary](https://code.videolan.org/videolan/medialibrary) like VLC for Tizen, others VLC ports will follow.  
It's a C++ library, written by [Hugo Beauzée-Luyssen](http://www.beauzee.fr/), which parses storages for video/audio files and manages a sqlite database to model your media library (with album, artist, genre classification, etc..). It replaces the old system we had on Android which just saved media files with their metadata, we had no proper structure for media library.  
Until v2.0.x, the *artists* list was generated at runtime by grouping all audio media by ther artist meta, same for *albums* and *genres*. Our database has now a correct and complete structure, *artists* list is obtained by a simple sqlite request. So these categories listing are now faster.  
Beside this speed improvement, one of the main benefits of this medialibrary is to provide better [search results](#search)

For now we are focusing on the first scan performance to make it at least as fast as the previous system.  

So, this library is aimed to be a common module of all VLC ports, wich means all debugging, performance and any improvement will benefit other platforms.  

Next steps for this library will be media scrapping, and network scan:
* Medialibrary will get informations and illustrations for your media, so we'll be able to present you a nice collection, and not files list anymore. We will also group media by shows/seasons et genre/release year/whatever
* You will be able to scan your NAS content to access your media easily.

### Playback performance & formats support

VLC core team worked hard too to bring performance improvements and some new features. Here are some highlights:
* 360° videos support
* Adaptive (HLS/Dash) & TS playback improved
* OpenGLES 2.0 is now used to render video (for software decoders & mediacodec)
* Support for VP8/9/10 in MP4
* e-AC3 HDMI passthrough

## Future

We also plan to implement a feature to **download media on your device**, in order to sync your series episodes or songs from your NAS to your device.

We'd like to support videos playlists like we do with videos grouped by their common name prefix.

As previously stated, medialibrary will allow to make VLC a real **media center** with fancy movies/tv shows presentation, and better artists/albums illustrations.

At last, I started an **extension API** in order for everyone to develop android applications which will provide content to VLC.
For example, we will release (with sources of course) extensions bringing support of podcasts subscriptions and Google Drive access.
