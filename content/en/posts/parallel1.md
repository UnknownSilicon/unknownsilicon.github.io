---
title: "ParallelUtils - Part 1: Modular Minecraft Modding Madness"
date: 2026-10-29T15:21:08-04:00
author: "UnknownSilicon"
cover: ""
tags: ["minecraft", "tech", "coding"]
keywords: ["", ""]
description: ""
showFullContent: false
readingTime: false
hideComments: false
table-of-contents: false
toc-auto-numbering: false

draft: true
---

Aat the start of the COVID-19 pandemic, some friends, at the time friends-of-friends, invited me to join a public minecraft server they started called [Parallel](https://parallelmc.org) (also noted as ParallelMC). Parallel is a survival-multiplayer (SMP) server with lore, a narrative, and very welcoming community. At the time, it was more or less a purely vanilla [Paper](https://papermc.io) server. It didn't have custom features, and only had a few plugins for QOL and server administration, but they wanted something more to stand out and fit the lore.

The server owner Diamondback (Aka "DB") talked about wanting custom mobs, custom items, and generally more flexibility in what the server did. I offered to help, since I had quite a significant amount of programming experience and had at least looked into Minecraft modding in the past. And so, what would become known as ParallelUtils (PU) was born.

PU was initially intended to only add custom mobs. At the time, there weren't easy APIs or free plugins that could provide this functionality