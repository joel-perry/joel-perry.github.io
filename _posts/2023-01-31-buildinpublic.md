---
layout: post
title:  Build In Public - SongTool2
date:   2023-01-31 02:02:40 -0600
tags:   [buildinpublic songtool2]
---
After starting December with a little time on my hands (while job-hunting) and a desire to dig deeper into SwiftUI, I decided to resurrect an old project. SongTool was my first real project as a developer and paved the way for a newfound career in iOS development. I envisioned a new version of my old friend, with some added features and capabilities, as a playground to try out ideas on effectively structuring a SwiftUI application. What came about was [SongTool2](https://github.com/joel-perry/songtool2).

### The Original
When playing music for a living, it is not unusual to have to learn upwards of 50 songs for a new gig, often in different keys from the recordings. I couldn't find an app that did all I needed, so I set out to build one. With no idea what I was doing, I started learning Objective-C and Core Audio on OSX, eventually producing an app for my own use. SongTool let me build playlists from my iTunes library and local files, control the playback speed and pitch, and loop sections of songs on-demand. Some of my friends saw it and wanted to use it, so I self-published (this was well before the mac App Store). By 2014, I had a small group of musicians and dancers using SongTool and an iOS version seemed like a good idea. SongTool appeared in the App Store in 2014 and had a pretty good run until file access changes around Apple Music kneecapped it. I finally pulled SongTool from the App Store in 2020.

### The New Guy
I had wanted to revisit SongTool and this seemed like a good time to do it, so I started thinking about what SongTool2 would look like. Any new version had to have the core features of SongTool: on-demand looping and speed/pitch shifting. It also needed some upgrades: better playlists, better EQ, and iCloud sync. As of today, I have built out most of the services to drive the app. The user interface is flat right now, but I want to try some new ideas to bring it to life. I still need to incorporate CloudKit, accessibility, and unified error messaging. There is much to be done, but it's a good start and I'm learning a lot in the process, so win-win!