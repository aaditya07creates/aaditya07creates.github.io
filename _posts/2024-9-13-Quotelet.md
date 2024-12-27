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
For the past year we've seen a lot of A.I. generated content on youtube. Now there is a difference between A.I. generated and automation we must understand first, A.I. (artificial intelligence) generated content is the content A.I. thinks of, for example a script for a video, the thumbnail or even the voiceover. Automated on the other hand is making the process autonomous or requiring minimum human intervention. Keep in mind I have written "automated A.i. channel" and not "A.I. automated channel" since A.I. is not the one that is automating the process. Thats enough of saying A.I. for now. Check out the channel [here!](https://beautifuljekyll.com/faq/#links-in-project-page)

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

* automatedvidgen
   * audios
   * finishedvideos
   * graphics
      * Banner image
      * Logo image
   * videos
   * quotes.txt
   * quotesreset.txt
   * script.py

With the folder structure done we move onto the main code.

I imported the libraries we needed: 
* MoviePy to handle video editing, adding text, adding audio and combining clips
* PIL (pillow) to create images that contain the text overlays
* Numpy which is required by moviepy to process data
* OS for working with directories and file paths

Let us discuss the functions I wrote now.

### 1. wrap_text(text, font, max_width)
This function is for breaking up a long quote onto multiple lines by iterating through each word and checking if it fits in the line. It finally returns the wrapped text with newline characters to represent split lines.

### 2. create_text_clip(text, author, quote_fontsize, author_fontsize, ai_text_fontsize, color, size, padding, duration)
This chonky function is for creating the text clip that will be added to the video. First i create a dark grey background that will be made transparent to make sure the text is visible when on top of the video. I call the wrap_text function to deal with the text wrapping, then i position all the texts and draw them using the ImageDraw.Draw method. The image is finally converted into a numpy array and returned to be used as a MoviePy clip.

### 3. create_short_video(automatedvideogen_path)
This is our main function which handles the video creation. It reads the quotes from the quotes.txt and selects a quote by random, next it selects a random video from the videos folder and removes the audio. A random audio is selected from the audios file and the video is looped and cut according to the length of the audio automatically. It forms a new clip with the CompositeVideoClip function to combine the clips and audios. The final file is finally compiled using 16 threads of my CPU for speed at 60fps and is stored in the finishedvideos folder with its file name created dynamically using the generate_ouput_file_name function I wrote.

### 4. update_quote_status
This function aimply sets the status of the quote in the quotes.txt to yes indicating we musn't use it in a video again.

~~~
"Quote text", "Author", "y"
~~~
### 5. Finishing up
The program loops the number of time you want it to create how many ever videos you want with a prompt when you run the program.

~~~
How many videos would you like to generate? 5
~~~

{: .box-warning}
**Warning:** Run the program directly in terminal and not in Python IDLE or Pycharm terminal. Running it in here will cause it to begin video generation but the process will never complete and will be stuck or take a lot of time.

## Conclusion
This project was pretty fun, it gave me a chance to learn and apply my knowledge about APIs, automation and also about youtube channels. Initially it takes some time to set-up but after you complete it you can create videos in bulk. You can see the final result [here!](https://beautifuljekyll.com/faq/#links-in-project-page)

## Code

{% highlight javascript linenos %}
from moviepy.editor import *
import random
from PIL import Image, ImageDraw, ImageFont
import numpy as np
import os
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
from google.oauth2.credentials import Credentials

description = "#ShortVideo #InspirationalVideo #MotivationalVideo #QuoteVideo #VideoEditing #TextOnVideo #CinematicVideo #ShortClips #MovieClips #VideoQuotes #BestQuotes #MotivationalQuotes #InspirationalQuotes #FamousQuotes #DailyQuotes #QuoteOfTheDay #VideoArt #VideoCreation #CreativeVideo #ArtisticVideo #InspirationalQuotes #MotivationalQuotes #QuotesFromFamousPeople #LifeQuotes #SuccessQuotes #LoveQuotes #HappinessQuotes #WisdomQuotes #QuoteOnLife #AuthorQuotes #InspirationalSayings #LifeLessons #LifeAdvice #LifeMotivation #PositiveQuotes #PowerfulQuotes #DeepQuotes #ThoughtfulQuotes #EncouragingQuotes #MoviePy #VideoEditor #PythonVideoEditing #CompositeVideo #TextAnimation #VideoClipGenerator #ShortFilm #MoviePyEditor #AIGeneratedVideo #TextClip #InspiringVideo #MotivationDaily #QuotesInMotion #QuoteOverlay #Quotelet #FamousAuthors #QuoteDesign #CreativeEditing #VideoOfTheDay #PositiveVibes #MotivationMonday #ThoughtOfTheDay #QuoteLife #TextOverlay #VideoCreationTool #PythonEditing #VideoMaker #InspirationDaily #TextEffects #QuoteOnScreen #QuoteGraphics #FamousQuoteVideo #MotivationalSpeech #QuoteToLiveBy #QuoteWallpaper #WordsOfWisdom #VideoInspiration #ShortQuoteVideo #VideoMotivation #WisdomInWords #LifeWisdom #EmpowermentQuotes #EncouragementVideo #SuccessMindset #GoalsInLife #PositiveMindset #MoviePyEditing #VideoWithText #PowerfulWords #ShortInspiration #DailyMotivation #SuccessQuotesDaily #SelfGrowth #LifeInspiration #VideoProduction #QuotesThatInspire #InspirationalClip #MindsetMatters #MindsetQuotes #EmpowermentVideo #LifeChangingQuotes #QuotesDaily #InspirationalWords #LifeQuotes #WisdomQuotes #PowerfulQuotes #SuccessDaily #MotivationInMinutes #UpliftingWords #LifeGoals #PositiveThinking #PersonalGrowth #InspirationQuotes #VideoEditing #CreativeVideos #QuoteletVideo #MindsetShift #EncouragingWords #VideoInspo #QuoteCreatives #PositiveQuotes #TextOnScreen #QuotesWithVideo #ShortInspirationalVideo #WisdomOfTheDay #FamousQuotesDaily #QuoteMastery #InspirationalShorts #MotivationalQuotes #SuccessMotivation #AchievementQuotes #GrowthMindset #SelfImprovement #LifeChangingWords #DreamBig #DailyQuotes #InspirationalStory #SuccessJourney #VideoInspiration #LifeChangingInspiration #DreamQuotes #DailyWordsOfWisdom #MindsetGoals #PurposeDrivenLife #TextVideo #QuoteLovers #TextArtVideo #SuccessDriven #EncouragementQuotes #WisdomWords #MotivationBoost #QuotesOfGreatness #PositiveVibesOnly #LifeMantra #GoalOriented #FocusOnSuccess #TextAnimation #MindsetMotivation #WordsOfEncouragement"

# Wrap text function to fit within the specified width
def wrap_text(text, font, max_width):
    lines = []
    words = text.split()
    current_line = ""

    for word in words:
        test_line = f"{current_line} {word}".strip()
        text_bbox = font.getbbox(test_line)
        text_width = text_bbox[2] - text_bbox[0]  # Calculate width from bbox
        if text_width <= max_width:
            current_line = test_line
        else:
            if current_line:  # Avoid appending empty lines
                lines.append(current_line)
            current_line = word

    if current_line:  # Append any remaining text
        lines.append(current_line)
    return "\n".join(lines)



# Create a text clip with a quote, author, AI text, and watermark
def create_text_clip(text, author, quote_fontsize=60, author_fontsize=40, ai_text_fontsize=30, color='white', size=(1280, 720), padding=100, duration=30):
    img = Image.new("RGBA", size, (30, 30, 30, 180)  # Dark gray background
    draw = ImageDraw.Draw(img)

    # Load fonts for text
    quote_font = ImageFont.truetype("arial.ttf", quote_fontsize)
    author_font = ImageFont.truetype("arial.ttf", author_fontsize)
    ai_text_font = ImageFont.truetype("arial.ttf", ai_text_fontsize)

    # Calculate drawing area
    drawing_area_width = size[0] - 2 * padding
    drawing_area_height = size[1] - 2 * padding

    # Wrap the quote text to fit
    wrapped_text = wrap_text(text, quote_font, drawing_area_width)
    author_text = f"~ {author}"
    ai_text = "An automated A.I. channel"  # Updated text here

    # Draw the text onto the image
    quote_bbox = draw.textbbox((0, 0), wrapped_text, font=quote_font)
    text_width, text_height = quote_bbox[2] - quote_bbox[0], quote_bbox[3] - quote_bbox[1]

    text_x = padding
    text_y = max((drawing_area_height - text_height) / 2 + padding, padding)
    draw.text((text_x, text_y), wrapped_text, font=quote_font, fill=color)

    author_y = text_y + text_height + 20
    draw.text((text_x, author_y), author_text, font=author_font, fill=color)

    ai_bbox = draw.textbbox((0, 0), ai_text, font=ai_text_font)
    ai_text_x = (size[0] - (ai_bbox[2] - ai_bbox[0])) / 2
    ai_text_y = author_y + author_fontsize + 80
    draw.text((ai_text_x, ai_text_y), ai_text, font=ai_text_font, fill=color)

    # Add watermark
    watermark_text = "Quotelet"
    watermark_fontsize = 30
    watermark_font = ImageFont.truetype("arial.ttf", watermark_fontsize)
    watermark_x = padding
    watermark_y = size[1] - padding - watermark_fontsize
    draw.text((watermark_x, watermark_y), watermark_text, font=watermark_font, fill=color)

    # Convert to NumPy array for video processing
    img_np = np.array(img)
    img_clip = ImageClip(img_np).set_duration(duration).set_position('center')
    return img_clip

def create_short_video(automatedvidgen_path):
    quotes_file = "C:/Users/aadit/Downloads/automatedvidgen/quotes.txt"
    video_folder = "C:/Users/aadit/Downloads/automatedvidgen/videos"
    audio_folder = "C:/Users/aadit/Downloads/automatedvidgen/audios"
    finished_videos_folder = "C:/Users/aadit/Downloads/automatedvidgen/finishedvideos"

    with open(quotes_file, 'r', encoding='utf-8') as f:
        quotes = f.readlines()

    # Get a list of available quotes and their original indexes
    available_quotes = [(index, quote) for index, quote in enumerate(quotes) if '", "n"' in quote]
    if not available_quotes:
        print("No available quotes left to use.")
        return None

    # Randomly select a quote from the available quotes
    selected_quote_index, selected_quote = random.choice(available_quotes)

    parts = selected_quote.split('", "')
    if len(parts) == 3:
        quote_text = parts[0][1:]
        author = parts[1]
    else:
        print(f'Skipping line due to incorrect format: {selected_quote}')
        return None

    video_files = [f for f in os.listdir(video_folder) if f.endswith(('.mp4', '.mov'))]
    if not video_files:
        print("No video files found.")
        return None

    selected_video = random.choice(video_files)
    video_path = os.path.join(video_folder, selected_video)
    video = VideoFileClip(video_path).without_audio()  # Remove the video’s audio

    # Create text overlay for the video
    txt_clip = create_text_clip(quote_text, author, quote_fontsize=70, author_fontsize=40, color='white', size=video.size)

    # Select random audio file
    audio_files = [f for f in os.listdir(audio_folder) if f.endswith(('.mp3', '.wav'))]
    if not audio_files:
        print("No audio files found.")
        return None

    selected_audio = random.choice(audio_files)
    audio_path = os.path.join(audio_folder, selected_audio)
    audio_clip = AudioFileClip(audio_path)

    # Loop the video to match the audio duration if necessary
    if video.duration < audio_clip.duration:
        video = video.loop(duration=audio_clip.duration)

    # Create video with overlay
    video_with_overlay = CompositeVideoClip([video, txt_clip]).set_audio(audio_clip)

    # Trim or extend the video to match the audio length
    final_video = video_with_overlay.subclip(0, audio_clip.duration)

    # Save the final video
    output_file = generate_output_file_name(finished_videos_folder, "short_video", ".mp4")
    final_video.write_videofile(output_file, fps=60, codec='libx264', threads=16)

    return output_file, selected_quote_index


# Generate a unique output file name
def generate_output_file_name(folder, base_name, extension):
    for i in range(1, 1001):
        file_name = f"{base_name}_{i}{extension}"
        file_path = os.path.join(folder, file_name)
        if not os.path.exists(file_path):
            return file_path
    raise FileExistsError("All file names are taken.")

# Update the status of the quote to "used"
def update_quote_status(quotes_file, selected_quote_index, quotes):
    selected_quote = quotes[selected_quote_index].strip()
    if '", "n"' in selected_quote:
        updated_quote = selected_quote.replace('", "n"', '", "y"')
        quotes[selected_quote_index] = updated_quote + "\n"
        with open(quotes_file, 'w', encoding='utf-8') as f:
            f.writelines(quotes)

# Upload video to YouTube
def upload_video_to_youtube(video_path, credentials_path):
    credentials = Credentials.from_authorized_user_file(credentials_path, [
        "https://www.googleapis.com/auth/youtube.upload"
    ])

    youtube = build('youtube', 'v3', credentials=credentials)

    title = os.path.basename(video_path).split('.')[0]
    body = {
        'snippet': {
            'title': title,
            'description': description
        },
        'status': {
            'privacyStatus': 'public',
            'madeForKids': False  # Automatically set to "Not Made for Kids"
        }
    }

    media = MediaFileUpload(video_path, mimetype='video/mp4', resumable=True)

    request = youtube.videos().insert(part=','.join(body.keys()), body=body, media_body=media)

    response = request.execute()
    print(f"Video uploaded successfully: {response['id']}")



# Main program to generate and upload videos
automatedvidgen_path = "C:\\Users\\aadit\\Downloads\\automatedvidgen"
credentials_path = "C:\\Users\\aadit\\QUOTELETTOKEN.json"
quotes_file = "C:\\Users\\aadit\\Downloads\\automatedvidgen\\quotes.txt"


try:
    if not os.path.exists(quotes_file):
        print(f"Quotes file not found at: {quotes_file}")
        input("Press Enter to exit...")
        exit()

    while True:
        try:
            num_videos = int(input("How many videos would you like to generate and upload? "))
            if num_videos > 0:
                break
            else:
                print("Please enter a positive number.")
        except ValueError:
            print("Invalid input. Please enter a valid number.")

    for i in range(num_videos):
        print(f"Generating video {i + 1}/{num_videos}...")
        result = create_short_video(automatedvidgen_path)
        if result is not None:
            output_file, selected_quote_index = result
            with open(quotes_file, 'r', encoding='utf-8') as f:
                quotes = f.readlines()
            update_quote_status(quotes_file, selected_quote_index, quotes)

            print(f"Uploading video {i + 1}/{num_videos} to YouTube...")
            upload_video_to_youtube(output_file, credentials_path)
        else:
            print(f"Skipping video {i + 1} due to an error.")

except Exception as e:
    print(f"An error occurred: {e}")

input("Press Enter to exit...")

{% endhighlight %}