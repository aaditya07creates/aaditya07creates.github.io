---
layout: post
title: Quotelet
subtitle: A YouTube channel that runs itself
cover-img: /assets/img/Quotelet_banner.png
thumbnail-img: /assets/img/Quotelet_logo.png
share-img: /assets/img/path.jpg
tags: [Youtube, Python, AI]
author: Aaditya Bhave
---
<br />

## A channel that runs itself ##

[Quotelet](https://www.youtube.com/@Quotelet1/shorts) is a YouTube Shorts channel that generates and posts its own videos, with almost no input from me once it's set up. You run one Python script, tell it how many videos you want, and it writes them, builds them, and uploads them straight to YouTube.

Honest origin story: I made it because I was lazy, lol. I liked the *idea* of running a YouTube channel a lot more than the idea of actually sitting down and making videos for one. So instead of doing the boring repetitive part, I spent way more effort building a program to do it for me. That's the good kind of lazy, the kind where you'd rather build a machine to do your chores than just do the chores.

Quick distinction, because people mix these up: **AI-generated** means the AI *thinks up* the content (a script, a quote, a thumbnail), while **automation** means the *process* runs itself. Quotelet is really an automation project with AI bolted on where it helps, not an "AI channel" in the buzzword sense. This project was me exploring exactly where each one earns its place.
<br />

## The one real tradeoff
Here's the thing I kept running into: with a project like this you're always trading between **automation, AI, and quality**, and you rarely get all three. Fully automate a step and the quality often drops. Insist on quality and you have to do that step by hand. Almost every decision below is me picking a side of that triangle on purpose, and I think the picks are what made the channel actually watchable.

The topic is **quotes**, and I stuck to Shorts to keep it tight.

{: .box-note}
**Note:** Picking one clear theme and sticking to it matters. Returning viewers and the algorithm both reward a channel that's *about* something.

Every video comes down to the same pipeline: find a quote, find a background clip, find fitting audio, assemble it, and upload it. Let's walk it.
<br />

## Step 1: the quote
I let AI write the quotes. You *can* just open ChatGPT and ask, but I wanted this callable from code, so I went the **API** route (an API lets a script talk to a model directly, using a "key" as its password).

First I tried ChatGPT's API. That died fast, most models charge per "token," the free trial credits had just been discontinued, and I wasn't about to set up billing for a quotes bot. So I switched to a **Hugging Face** model that gave me a free API (with a daily limit). My prompt was simple:

~~~
Give me a quote that is inspirational/sad/heartwarming along with the author's name.
It should be one line long.
~~~

The problem was the *shape* of the answers. The model would reply with "Here is your quote…" and a follow-up question, which is useless if a script has to drop it straight into a video. I fought the prompt for a while, then made the pragmatic call: **quality over automation here.** I had ChatGPT generate a clean **CSV of 100 quotes** up front, in an exact format, and worked from that list instead of hitting a model per video:

```
"Quote text", "Author", "n"
```

The `"n"` is a used/unused flag, I'll come back to it.
<br />

## Step 2: the background
Inspirational quotes need a calm, eye-catching background so people don't scroll past, so I went with video over a static image. I looked for stock-footage APIs and found a few good free ones, **Pexels** and **Pixabay** among them. Pixabay refused to return footage for reasons I never figured out; **Pexels** worked, letting me pull clips by keyword at the 9:16 Shorts ratio.

But the auto-pulled clips were hit and miss, off-theme or too long. Same crossroads, same answer: **quality over automation.** I spent a few hours hand-picking **29 non-copyrighted clips** that actually fit the vibe, and the channel is better for it.
<br />

## Step 3: the audio
By now I was tired of hunting for APIs, and automated audio selection would've brought its own headaches (wrong mood, wrong length, wrong volume). So I just hand-picked a small set of tracks. Not glamorous, but reliable.
<br />

## Step 4: assembling the video
This is where the real automation lives. Using **MoviePy** (editing, text, audio, compositing), **Pillow** (drawing the text overlay), **NumPy** (MoviePy's data backend) and **os** (file handling), the script builds a finished Short on its own. The core pieces:

- **`wrap_text`** breaks a long quote across lines by measuring each word and folding when it won't fit.
- **`create_text_clip`** draws the quote, author, and watermark onto a semi-transparent dark panel so the text stays readable over any background, then hands it back as a MoviePy clip.
- **`create_short_video`** is the engine: pick a random unused quote, pick a random background and strip its audio, pick a random track, loop and trim the video to the audio's length, composite the text on top, and render at 60fps across 16 CPU threads.
- **`update_quote_status`** flips that quote's flag from `"n"` to `"y"` so it's never reused.

Run it and it just asks how many to make:

~~~
How many videos would you like to generate and upload? 5
~~~

{: .box-warning}
**Warning:** Run this straight from a real terminal, not the PyCharm or IDLE console, otherwise rendering starts but never finishes.
<br />

## Step 5: uploading itself (the newest piece)
For a long time the script stopped at a folder full of finished videos, and I'd still upload them by hand, which meant it wasn't *really* automated. The latest version fixes that: it uploads to YouTube on its own using the **YouTube Data API**.

I authorise the channel once with an OAuth credentials file, and after that `upload_video_to_youtube` posts each finished clip as a **public Short**, with a title, a big keyword-loaded description for reach, and the "not made for kids" flag set. That was the missing link. Now the loop is genuinely end to end: tell it a number, and it *writes, builds, and publishes* that many videos in one run, no hand-holding.
<br />

## What I got out of it
Quotelet taught me a surprising amount, real API work, OAuth and the YouTube upload flow, video processing with MoviePy, and above all a feel for that automation-vs-AI-vs-quality tradeoff and when it's worth doing a step by hand. The setup takes a while, but once it's built I can produce videos in bulk with one command. See the results [here](https://www.youtube.com/@Quotelet1/shorts).
<br />

## Code

{% highlight python linenos %}
from moviepy.editor import *
import random
from PIL import Image, ImageDraw, ImageFont
import numpy as np
import os
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
from google.oauth2.credentials import Credentials

description = "#ShortVideo #InspirationalVideo #MotivationalVideo #QuoteVideo #Quotelet #MotivationalQuotes #InspirationalQuotes #DailyQuotes #QuoteOfTheDay #WordsOfWisdom #PositiveVibes #LifeQuotes #SuccessQuotes"

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
    img = Image.new("RGBA", size, (30, 30, 30, 180))  # Dark gray background
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
    ai_text = "An automated A.I. channel"

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
    video = VideoFileClip(video_path).without_audio()  # Remove the video's audio

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
