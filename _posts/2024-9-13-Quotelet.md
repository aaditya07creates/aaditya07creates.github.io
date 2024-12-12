---
layout: post
title: Quotelet
subtitle: An automated A.I. channel
cover-img: /assets/img/Quotelet_banner.png
thumbnail-img: /assets/img/Quotelet_logo.png

share-img: /assets/img/path.jpg
tags: [Youtube, Python, AI]
author: Aaditya Bhave
---
<br />

## An automated A.I. channel ##
For the past year we've seen a lot of A.I. generated content on youtube. Now there is a difference between A.I. generated and automation we must understand first, A.I. (artificial intelligence) generated content is the content A.I. thinks of, for example a script for a video, the thumbnail or even the voiceover. Automated on the other hand is making the process autonomous or requiring minimum human intervention. Keep in mind I have written "automated A.i. channel" and not "A.I. automated channel" since A.I. is not the one that is automating the process. Thats enough of saying A.I. for now.

In this project i've explored multiple ways of doing both A.I. integration and automation.

## Project outline
Basically being lazy leads you to want to create a project like this, my goal is to create a youtube channel that doesnt require much or any input from me after it's set up.
When you want to create an automated channel you need to sacrifice either A.I. and automation or quality, i've explored many combinations of both and in the end chose what I liked the most. For this project i've used python to make the script for generating the videos. The terms used in the project are going to be pretty basic for the general public to understand.


## How to make a youtube video

Now first let us discuss the basics, i've chosen to make youtube shorts to keep the project simple and easy. The topic for our youtube channel will be quotes.

{: .box-note}
**Note:**
   Its good to stick to a particular theme and topic to attract returning viewers and to gain subscribers. The youtube algorithm likes these sort of channels.

<br />

To make a youtube video (shorts) about a topic (quotes) there is a process.
* Search for a quote in a book or online
* Look for a background video to make video attractive
* Find a fitting audio
* Put everything together in and editing software
* Upload it to youtube

Well it looks pretty simple, let's get to making our channel.


## Searching for a quote
The first step for a quotes channel is to search for a quote (obviously).
You can read a book of quotes, search online on a website, or look at your social media's good morning messages.
Well I chose to ask A.I. for my quotes. Now there are many LLM models out there like google's Gemini, meta's Llama, openai's ChatGPT. We can directly start a conversation and request a quote from there, but this is long and tiring (for me). So I decided to use an API. An API is like a way to ask one of the LLMs from a script or peice of code without needing to visit the website. It requires a "key" that is like a password to understand who is accessing the model and how much. I decided to go with ChatGPT, that failed terribly. Using a model requires a currency called "tokens" most of the time, and i had none. They used to give free tokens for trials but they stopped recently :c and required billing which i was not ready for. Then i decided to use another LLM hugging face, this model gave me an API for free :D (with a per-day limit of course). I put this into a simple python script after importing some libraries to ask for a quote, here was my prompt.

~~~
Give me a quote that is inspirational/sad/heartwarming along with the author's name.
It should be one line long.
~~~

Its response was something like this.

~~~
Here is your quote
"The only way to do great work is to love what you do"
This quote was given by Steve Jobs at a 2005 commencement speech at Stanford University.
Would you like some more quotes?
~~~

Now if I wanted to automate everything and not look at each step i couldn't have it giving answers like this, it should have had a proper format and be easy to add to the video.
I tried redesigning the prompt alot but it didn't work out, I knew the same would happen with other LLMs too and they might give the same quote twice. So I decided to go for quality here in this step than 100% automation, I asked ChatGPT to give me a list of 100 quotes in the form of a CSV (comma seperated values) file according to my need. This was great, the format was like this:

```
"Quote text", "Author", "n"
```
The quote text is followed by the author name and then a letter "y" or "n" but we will come back to that later.

Since the quote collection was done it was time to move to the next step, a rather complicated one.

## Looking for background videos
Our quotes are inspirational mostly and calming so we need a fitting background to make sure viewers don't scroll past. It could be an image or video but I decided to go for a video to make it a bit more attention grabbing. I searched online for some APIs like we did for the LLMs but this time to retreive videos from the site.

I found a couple of free and awesome APIs like Pexels and Pixbay (a few others too). I began using Pixbay but it wasn't retrieving footage for some reason, i'm still not sure why. Then I looked at Pexels, this API worked well, i was able to download videos based on the keywords and dimensions (a youtube shorts video dimension is 9:16). I came across a problem though, the downloading was working well but the videos didn't always fit the theme and some of them were too long. 

Now we arrive at the crossroads again, do we want quality or automation. I decided to go for quality, by that I mean I spent hours manually choosing and downloading non-copyrighted stock videos that I liked. I collected 29 videos that fit the aspect ratio and asthetics. In the end I feel that it was the best choice because it will help attract even more viewers.

## Finding a fitting audio
Now at this point I was tired of looking for APIs and decided to do manual collection, It would have resulted in problems like long audio selection, audio that doesnt match the theme, or audio that is too loud or too quiet. I looked online and downladed a few songs.

## Putting everything together
Well now the real automation begins. What I plan to do is use python to combine the audio and video and add a text overlay. So I made a basic folder structure on my computer to keep things organized.


## Unfinished blog (WIP)

