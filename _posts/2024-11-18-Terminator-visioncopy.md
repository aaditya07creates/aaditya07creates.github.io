---
layout: post
title: Terminator Vision
subtitle: Seeing everything as terminator does
cover-img: /assets/img/termvis.png
thumbnail-img: /assets/img/crosshair.jpg
share-img: /assets/img/crosshair.jpg
tags: [Raspberry pi, OpenCV]
author: Aaditya Bhave
---
<br />

## INTRODUCTION ##

The terminator is a robot from a movie that time travels into the past to save someone. I love watching robot movies and got goosebumbs after watching terminator. I decided to make a device that can let me see things like terminator does.
<br />

## Basic idea ##

My idea was to mount a camera and raspberry pi onto my universal VR headset and cast the input from the camera onto a phone through wifi inside the headset after applying face recognition, hand recognition and cool overlays.
<br />


## Components used

* Webcam - For taking video input.
* Raspberry pi - This takes the camera input, applies the overlays and casts to the phone
* A VR headset - To see the output.
* A smartphone - to be put inside the vr headset as a display.
* A Powerbank - To power the contraption.

![Setup](/assets/img/vrphone.jpg)

<br />

## How it works

The raspberry pi takes input from a webcam whose footage is passed through a python script, the script then applies overlays after detecting faces and hands with mediapipe and a pretained face recognition model.

The modified frames are streamed onto a website on the local wifi network through a library called flask. The frames are updated live

# Unfinished blog (wip)