---
layout: post
title: Hand gestures
subtitle: Using hand gestures to control my laptop
cover-img: /assets/img/handges.jpg
thumbnail-img: /assets/img/nice.png
share-img: /assets/img/nice.png
tags: [OpenCV]
author: Aaditya Bhave
---
<br />

## Introduction ##

I was going through my laptop settings one day and found and option for eyetracking, sadly it was disabled as my hardware wasn't compattible. So I decided to make my own. Instead of eye tracking I felt that hand gestures would be much cooler and make me feel like iron man :D. My goal was to make a program to capture a video from my laptop camera live and detect hand signs using a hand recognition model to control my laptop without me haing to physically touch the laptop. I did the full project using python.
<br />




## How it works

I used the OpenCV library to capture frames from my camera. I passed the incoming frames into a google's ML mediapipe, it is a python library that can be used for various purposes. After the model dectects a certain gesture, the script uses the pyautogui library to give my laptop keyboard commands. 

I programmed 2 gestures:
* Alt+tab : Align your fingers (including thumb) straight and swipe right or left to change windows according to the direction
* Scrolling : join your pointer finger and thumb and flex all other fingers in a "nice" hand sign. Hold the pointer and thumb together and move your hand to scroll as if you are grabbing the page, move your thumb and pointer apart to stop scrolling and move your hand to its intial position.

Programming more gestures is much easier after getting the basic code done.

## Conclusion
This is just the beginning for all sorts of possibilities, interactive holograms, new air keyboards, immersive and intuitive ui interaction, new ways of gaming and so much more. This is a glimpse into what our future could look like.

## Code

{% highlight javascript linenos %}

import cv2
import mediapipe as mp
import pyautogui
import numpy as np
import time
import threading

# Initialize MediaPipe Hand Detection
mp_hands = mp.solutions.hands
hands = mp_hands.Hands(max_num_hands=1, min_detection_confidence=0.5, min_tracking_confidence=0.5)  # Min confidence thresholds
mp_drawing = mp.solutions.drawing_utils

# Initialize camera with higher resolution and frame rate
cap = cv2.VideoCapture(0)
cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1280)  # Set resolution width
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 720)  # Set resolution height
cap.set(cv2.CAP_PROP_FPS, 60)  # Increase FPS to 60

# Gesture control variables
scroll_smooth_factor = 0.1  # Smooth scroll factor (lower = smoother)
last_scroll_position = None
scrolling = False
last_finger_position = None
last_sway_direction = None
last_sway_time = time.time()
alt_tab_threshold = 30  # Gesture threshold for Alt+Tab (previous method)
alt_tab_direction = None  # To track swipe direction for Alt+Tab (left or right)
last_alt_tab_time = 0  # Time when Alt+Tab was last performed
alt_tab_cooldown = 1.5  # 1.5 seconds cooldown for Alt+Tab

# Threading function for capturing frames
def capture_frame():
    global frame
    while True:
        ret, frame = cap.read()
        if not ret:
            print("Failed to grab frame")
            break

# Check if camera is opened successfully
if not cap.isOpened():
    print("Error: Camera not accessible.")
    exit()

# Start the frame capture in a separate thread
frame_thread = threading.Thread(target=capture_frame)
frame_thread.daemon = True
frame_thread.start()

# Function to calculate the distance between two points
def calculate_distance(p1, p2):
    return np.linalg.norm(np.array([p1.x, p1.y]) - np.array([p2.x, p2.y]))

# Function to handle Alt+Tab based on swipe direction
def alt_tab_switch(direction='right'):
    global last_alt_tab_time  # Make sure to use the global variable
    current_time = time.time()
    if current_time - last_alt_tab_time > alt_tab_cooldown:  # Check if enough time has passed since the last Alt+Tab
        pyautogui.keyDown('alt')  # Hold down the Alt key
        if direction == 'right':
            pyautogui.press('tab')  # Press Tab to switch to the next window
        elif direction == 'left':
            pyautogui.hotkey('alt', 'shift', 'tab')  # Press Alt+Shift+Tab to go to the previous window
        pyautogui.keyUp('alt')  # Release the Alt key
        last_alt_tab_time = current_time  # Update the time of the last Alt+Tab

while True:
    # Wait until the frame is captured
    if 'frame' not in globals() or frame is None:
        print("Waiting for frame...")
        time.sleep(0.1)
        continue

    # Convert to RGB (MediaPipe requires RGB)
    rgb_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
    results = hands.process(rgb_frame)

    # Check if hands are detected
    if results.multi_hand_landmarks:
        for landmarks in results.multi_hand_landmarks:
            # Draw hand landmarks
            mp_drawing.draw_landmarks(frame, landmarks, mp_hands.HAND_CONNECTIONS)
            
            # Extract finger positions (fingertip positions)
            thumb_tip = landmarks.landmark[mp_hands.HandLandmark.THUMB_TIP]
            index_tip = landmarks.landmark[mp_hands.HandLandmark.INDEX_FINGER_TIP]
            middle_tip = landmarks.landmark[mp_hands.HandLandmark.MIDDLE_FINGER_TIP]
            ring_tip = landmarks.landmark[mp_hands.HandLandmark.RING_FINGER_TIP]
            pinky_tip = landmarks.landmark[mp_hands.HandLandmark.PINKY_TIP]
            
            # Gesture 1: Pinching Gesture (Scroll)
            # Pinch thumb and index finger, other fingers extended
            distance = calculate_distance(thumb_tip, index_tip)
            other_fingers_distance = calculate_distance(middle_tip, ring_tip) + calculate_distance(ring_tip, pinky_tip)
            
            # Adjust the scrolling gesture for more sensitivity and smoother scrolling
            if distance < 0.04 and other_fingers_distance > 0.1:
                if last_scroll_position:
                    delta_y = index_tip.y - last_scroll_position[1]
                    if abs(delta_y) > 0.02:  # Reduced threshold for smoother scroll
                        scroll_amount = int(delta_y * 1000)  # Increased scrolling speed for more responsiveness
                        pyautogui.scroll(scroll_amount)
                        scrolling = True  # Set scrolling flag to true
                last_scroll_position = [index_tip.x, index_tip.y]
            else:
                # If the fingers aren't pinched, stop scrolling
                scrolling = False
                last_scroll_position = None

            # Gesture 2: Swipe Gesture (Alt+Tab)
            # Detect hand swipe (only horizontal)
            thumb_y = thumb_tip.y
            index_y = index_tip.y
            middle_y = middle_tip.y

            # Determine the swipe gesture direction (only horizontal)
            swipe_threshold = 0.15
            if abs(thumb_y - index_y) < swipe_threshold and abs(index_y - middle_y) < swipe_threshold:
                if last_finger_position:
                    swipe_distance_x = index_tip.x - last_finger_position[0]

                    # Trigger Alt+Tab only for horizontal swipe (ignore vertical)
                    if abs(swipe_distance_x) > swipe_threshold and not scrolling:  # Don't trigger Alt+Tab if scrolling is active
                        # Determine the direction of swipe
                        if swipe_distance_x < 0:
                            alt_tab_direction = 'left'  # Left swipe
                        else:
                            alt_tab_direction = 'right'  # Right swipe
                        alt_tab_switch(alt_tab_direction)  # Perform Alt+Tab
                        time.sleep(0.5)  # Wait for a moment to prevent rapid switching

                last_finger_position = [index_tip.x, index_tip.y]

    # Show the frame with gesture feedback
    cv2.imshow('Gesture Control', frame)

    # Exit if 'q' is pressed
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

# Release resources
cap.release()
cv2.destroyAllWindows()


{% endhighlight %}