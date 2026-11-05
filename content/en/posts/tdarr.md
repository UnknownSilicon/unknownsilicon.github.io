---
title: "Trans Coding Transcoding"
date: "2025-02-01T10:00:00-03:00"
author: "UnknownSilicon"
cover: ""
tags: ["homelab", "tech"]
keywords: ["", ""]
description: ""
showFullContent: false
readingTime: false
hideComments: false
table-of-contents: false
toc-auto-numbering: false

draft: true
---

Having a media server is fun. You can put all your content in one place, stream it to anywhere, run into storage space issues, and experience the same sort of lag you get trying to watch a Youtube video on a dial up internet connection, all from your own network!

Jokes aside, storage space and audio/video codec compatability issues are some of the most frustrating parts of my media server. Aquired media tends to have a huge variation on media codecs, containers, and formatting. I've largely fixed the issue through a wonderful programmable distributed transcoder called [Tdarr](https://tdarr.io/). I happen to have a pretty powerful desktop PC (Ryzen ..., NVIDIA RTX 1050, 64GB DDR4 RAM), which outspecs my media server by a considerable amount, so taking advantage of it was a must. 

# File Sizes

Older video codecs are significantly less efficient than the newer H.264 (AVC) and H.265 (HEVC) codecs. These days, pretty much every device supports H.264 and most devices also support H.265. There's very little reason to use anything other than these two codecs, but many ripped or original media files are still in older codecs. 

To take advantage of this, I use Tdarr to automatically re-encode media streams with older codecs into H.264. Due to its slight compatability issues and the occasional visible decrease in quality from transcoding from older formats to H.265, I only use it when the original source provides it. 

<!-- Show flow diagram. If format is not H.264 or H.265, transcode to H.264.-->

In addition to codecs, many media files contain several video or audio streams. We typically only need one or two of each: An "ideal" video stream, plus one for compatability, and an English audio stream, potentially alongside an original language audio stream. Some media containers can also embed subtitles, which similarly only need English and original language formats. 

These changes alone tend to reduce file sizes a moderate amount. Transcoding everything to H.265 would for sure help more, but due to the listed issues above it's not hugely worth it given the storage space I have access to. Unfortunately, these changes alone do not fix all of the problems with media files. In fact, these file size optimizations simply compensate for the increased file sizes associated with the changes needed to enable Direct Streaming.

# The Direct Streaming Problem

Jellyfin can stream media in three ways: Direct Stream, Direct Play, and transcoded media. 
