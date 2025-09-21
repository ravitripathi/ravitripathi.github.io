---
title: Setting up a home server
date: 2025-09-21 22:51:00 +0530
tags: [server, ubuntu, jellyfin]
---

I've been wanting to setup my home server for quite some time now. Mainly to host my pictures and some media content locally. Watched a couple of tutorials for a few months, but rather obtuse. Work obligations have also made it hard to find time for such fun experiments.

Finally, I came across this excellent video by _Kalos Likes Computers_, titled "
[The $0 Home Server](https://www.youtube.com/watch?v=IuRWqzfX1ik)". Now this was exactly what I had been looking for. A guide for installing Jellyfin on an old laptop. 

I've taken cues from this video, and I'm mixing up a few things to my liking. First up, I've installed Lubuntu instead of Ubuntu Server as suggested in the video. Having a GUI would help if I need to debug something physically on the laptop if something went wrong, and I wasn't able to SSH into it. 

I'm also installing Docker and NVIDIA drivers on it. Docker for managing my Jellyfin installation. And NVIDIA drivers for (hopefully) speeding up video transcoding on Jellyfin. Let's see how this experiment goes.