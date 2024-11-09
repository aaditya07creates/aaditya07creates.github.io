---
layout: post
title: ESP32 macropad
subtitle: My own customisable bluetooth deck
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [esp32, buttons]
author: Aaditya Bhave
---
<br />

## ESP32 MACROPAD ##
A macropad is a physical device with buttons that are mapped to some customizable functions. It is used to enhance productivity, for navigating software, playing games or all of these at once. I have made my own version of it using the blekeyboard library and the esp32. 
<br />


## Components used

* ESP32 Microcontroller - This handles all the logic and connects to the computer with bluetooth.
* 6 Push Buttons - These are the triggers for the functions.
* A breadboard - To mount the esp32 and pushbuttons.

<br />

## How it works


* **Connecting to bluetooth:**\
  The esp32 connects to the computer/mobile through bluetooth to send information.

  ~~~
  Serial.println("Starting BLE work!");
  bleKeyboard.begin();
  ~~~

<br />

* **Main loop:**\
  When a device connects to the esp32, the main loop begins running. When a button is pressed, the assigned function is carried out like opening an app or increasing the volume. These functions are fully customisable. For example this function opens up task manager:

  ~~~
      if (digitalRead(button3Pin) == LOW) {
      Serial.println("Sending Ctrl + Shift + Esc...");
      bleKeyboard.press(KEY_LEFT_CTRL);    
      bleKeyboard.press(KEY_LEFT_SHIFT);  
      bleKeyboard.press(KEY_ESC);         
      delay(100);
      bleKeyboard.releaseAll();            
    }
  ~~~

  The functions that i have assigned to my keyboard are:
  * ALT + TAB to switch windows
  * Windows + D to minimize all windows
  * CTRL + SHIFT + ESC to open or focus task manager
  * Command to open browser with windows search
  * Volume up
  * Volume down

  We dont have to set single commands either we can make it carry out an entire list as well. For example, the 4th button on my macropad triggers the windows button, types opera into the keyboard and triggers the enter button to open a new window all with the push of a button. There are endless possibilities for this sort of customization, We can open entire workflows too.
<br />

{: .box-note}
**Note:**
  The device is powered by a 10000mAh power bank, this could power the esp32 for a long time despite the resource consuming bluetooth communication.


* **Some issues**\
  The project didnt go very smooth, i encountered a few problems. On searching for the right library for the project, i came across the ESP32-BLE-Keyboard in github. The library didn't work at first, it turns out that it was using std::str in its code instead of String and substr instead of substring. So i edited the library and the program started working. To make sure you don't run into the same issues change the text i did in both BleKeyboard files inside the library folder with the help of notepad or some other text editor.

<br />


## Features to add

The device has so much room for improvement some of the features we can add to make it better are:

* Casing
* Oled display or LEDs
* App or website for customization
* More inputs like potentiometer
* Smaller size
* Better buttons


### Warning

{: .box-warning}
**Warning:**
  I had to power the esp32 through the micro-usb port and not through the dedicated vin and gnd pins as one side of the board was cut off due to the size of the esp32 and the small breadboard. Some say this is not recommended and can damage it, so copy the method with caution.

## Code

{% highlight javascript linenos %}
#include <BleKeyboard.h>

// Button pins
const int button1Pin = 15;  
const int button2Pin = 2;   
const int button3Pin = 4;   
const int button4Pin = 21;   
const int button5Pin = 22;  
const int button6Pin = 23;

BleKeyboard bleKeyboard;

void setup() {

  Serial.begin(115200);
  

  pinMode(button1Pin, INPUT_PULLUP);  // Alt + Tab
  pinMode(button2Pin, INPUT_PULLUP);  // Minimize all windows
  pinMode(button3Pin, INPUT_PULLUP);  // Task Manager
  pinMode(button4Pin, INPUT_PULLUP);  // Open Browser
  pinMode(button5Pin, INPUT_PULLUP);  // Volume Up
  pinMode(button6Pin, INPUT_PULLUP);  // Volume Down

  // Initialize BLE keyboard
  Serial.println("Starting BLE work!");
  bleKeyboard.begin();
}

void loop() {
  if (bleKeyboard.isConnected()) {
    if (digitalRead(button1Pin) == LOW) {
      Serial.println("Sending Alt + Tab...");
      bleKeyboard.press(KEY_LEFT_ALT); 
      bleKeyboard.press(KEY_TAB);     
      delay(80);                     
      bleKeyboard.releaseAll();       
      delay(150);
    }

    if (digitalRead(button2Pin) == LOW) {
    Serial.println("Minimizing all windows...");
    bleKeyboard.press(KEY_LEFT_GUI);    
    bleKeyboard.press('d');            
    delay(100);
    bleKeyboard.releaseAll();          
    }


    if (digitalRead(button3Pin) == LOW) {
      Serial.println("Sending Ctrl + Shift + Esc...");
      bleKeyboard.press(KEY_LEFT_CTRL);    
      bleKeyboard.press(KEY_LEFT_SHIFT);  
      bleKeyboard.press(KEY_ESC);         
      delay(100);
      bleKeyboard.releaseAll();            
    }

  
    if (digitalRead(button4Pin) == LOW) {
      Serial.println("Sending command to open browser");
      bleKeyboard.write(KEY_LEFT_GUI);   
      delay(45);
      bleKeyboard.print("opera");
      delay(50);
      bleKeyboard.write(KEY_RETURN);       
      delay(50);
      bleKeyboard.releaseAll(); 
    }

    if (digitalRead(button5Pin) == LOW) {
      Serial.println("Sending Volume Up...");
      bleKeyboard.press(KEY_MEDIA_VOLUME_DOWN);  
      delay(10);  
    }


    if (digitalRead(button6Pin) == LOW) {
      Serial.println("Sending Volume Down...");
      bleKeyboard.press(KEY_MEDIA_VOLUME_UP);  
      delay(10);  
    }
  }

  delay(50);
}
{% endhighlight %}


## Conclusion

This project was pretty fun to make, i can use it in my daily life. Macropads are usually very expensive from 30 to 100 dollars, so my friends are pretty jealous of me.

