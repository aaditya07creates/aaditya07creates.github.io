---
layout: post
title: Tug of war game
subtitle: A button dexterity game with esp32
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [esp32, webserver]
author: Aaditya Bhave
---
<br />

## Tug of war game ##
The tug of war game is a button dexterity game. A 2 player game with induidual controllers, that you can spectate through a web interface.
<br />

## Overview of the game

The goal of the game is simple:

* Player 1 and Player 2 each press their respective button to pull the bar towards their side.
* Each button press moves the tug a little bit in the direction of the player’s side.
* The game ends when one player reaches the end of their side, and a winner is declared.
<br />

## Components used

* ESP32 Microcontroller - This handles all the logic and connects the game to the web interface.
* Two Push Buttons - These serve as the inputs for Player 1 and Player 2.
* Web Interface - Displays the game’s status, tug position, and countdown timer.
* 3 Breadboards - To mount the esp32 and act as controllers.

<br />

## How it works
<br />
* **Wifi connection:**\
  The esp32 connects to a specified wifi network so that the game can be viewable from a browser. The wifi network can be changed at the start of the code.

  ~~~
  #include <WiFi.h>
  #include <WebServer.h>

  const char* ssid = "your ssid here (caps sensitive)";
  const char* password = "your password here";
  ~~~

<br />

* **Web interface:**\
  The web interface displays the tug of war's progress and results, it is designed with HTML, CSS and JavaScript. The website refreshes automatically to show a dynamic tug of war game that can be viewed on any device connected to the network. A simple web server is set up using the webserver library. It listens on port 80.

  ~~~
  WebServer server(80);

  server.on("/", []() {
  server.send(200, "text/html", index_html);
  });

  server.on("/status", []() {
  String json = "{\"tugPosition\":" + String(tugPosition) + ", \"message\":\"" + message + "\"}";
  server.send(200, "application/json", json);
  });

  ~~~
<br />

* **Game logic:**\
  The esp32 checks the status of the buttons every 10ms, when the button is pressed the bar shifts indicating the player's tugging towards a side. The tug's position is displayed in real-time on the web interface with a refresh rate of 50ms. If one player reaches the extreme end of the bar, the game ends and the winner is displayed.

  ~~~
  if (digitalRead(button1Pin) == LOW && !button1Pressed) {
  tugPosition -= moveAmount;
  button1Pressed = true;
  delay(10);
  }
  ~~~
<br />

* **Game reset:**\
  To start the game again, the players have to simply click on the "reset" button on the esp32.

## Features to add

The game is far from complete, you can add anything you want! Here are some of the things that could make the game even better.

* Game timer
* Leaderboard system
* Player colour customisation
* Controller cases and components (like lights and buttons)
* Multiplayer support
* Better UI
* Animations
* Button press count

## Code

{% highlight javascript linenos %}
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "ssid here";
const char* password = "password here";

const int button1Pin = 12;  // Player 1 button
const int button2Pin = 13;  // Player 2 button

int tugPosition = 50;
const int winPosition = 100;
const int moveAmount = 2;
bool button1Pressed = false;
bool button2Pressed = false;

WebServer server(80);

bool gameActive = true;

const char index_html[] = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
  <title>ESP32 Tug of War</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    html, body { height: 100%; display: flex; justify-content: center; align-items: center; font-family: Arial, sans-serif; background-color: #121212; color: #ffffff; }
    #container { text-align: center; width: 90vw; max-width: 800px; }
    h1 { font-size: 3vw; margin-bottom: 10px; }
    p { font-size: 1.5vw; margin-bottom: 20px; }
    #progress-container { width: 100%; height: 40px; background-color: #333; border-radius: 10px; position: relative; display: flex; border: 2px solid #666; }
    #progress-bar-left { height: 100%; background-color: #ff5555; border-top-left-radius: 10px; border-bottom-left-radius: 10px; }
    #progress-bar-right { height: 100%; background-color: #5555ff; border-top-right-radius: 10px; border-bottom-right-radius: 10px; }
    #tugDisplay { font-size: 2vw; margin-top: 20px; }
    #result { font-size: 2.5vw; color: #00ff00; }
  </style>
  <script>
    async function refreshTug() {
      const response = await fetch('/status');
      const data = await response.json();
      const tugPosition = data.tugPosition;

      // Update widths of red and blue sections based on tug position
      document.getElementById('progress-bar-left').style.width = tugPosition + '%';
      document.getElementById('progress-bar-right').style.width = (100 - tugPosition) + '%';

      document.getElementById('tugDisplay').innerText = 'Tug Position: ' + tugPosition;
      document.getElementById('result').innerText = data.message;
    }
    
    window.onload = () => {
      setInterval(refreshTug, 50);
    };
  </script>
</head>
<body>
  <div id="container">
    <h1>ESP32 Tug of War</h1>
    <p>Press your button to pull the tug toward your side!</p>
    <div id="progress-container">
      <div id="progress-bar-left"></div>
      <div id="progress-bar-right"></div>
    </div>
    <div id="tugDisplay">Tug Position: 50</div>
    <div id="result"></div>
  </div>
</body>
</html>
)rawliteral";

void setup() {
  Serial.begin(115200);

  pinMode(button1Pin, INPUT_PULLUP);
  pinMode(button2Pin, INPUT_PULLUP);

  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConnected to WiFi");
  Serial.println(WiFi.localIP());

  server.on("/", []() {
    server.send(200, "text/html", index_html);
  });

  server.on("/status", []() {
    String message = "";
    if (tugPosition <= 0) {
      message = "Player 1 Wins!";
      gameActive = false;
    } else if (tugPosition >= 100) {
      message = "Player 2 Wins!";
      gameActive = false;
    } else {
      message = "Keep Tugging!";
    }
    String json = "{\"tugPosition\":" + String(tugPosition) + ", \"message\":\"" + message + "\"}";
    server.send(200, "application/json", json);
  });

  server.begin();
  Serial.println("Server started");
}

void loop() {
  server.handleClient();

  // Game logic to handle button presses only if game is active
  if (gameActive) {
    if (digitalRead(button1Pin) == LOW && !button1Pressed) {
      tugPosition -= moveAmount;
      tugPosition = max(0, tugPosition);
      button1Pressed = true;
      delay(10);
    } else if (digitalRead(button1Pin) == HIGH) {
      button1Pressed = false;
    }

    if (digitalRead(button2Pin) == LOW && !button2Pressed) {
      tugPosition += moveAmount;
      tugPosition = min(100, tugPosition);
      button2Pressed = true;
      delay(10);
    } else if (digitalRead(button2Pin) == HIGH) {
      button2Pressed = false;
    }
  }
}

{% endhighlight %}


## Conclusion

This game is really fun to play with friends. By incorporating physical buttons with a web interface, the tactile clicking experience is complemented by an awesome way to spectate the game. You can view the site on your smart TV, a laptop or even a mobile. The exciting thing is that you can customise it however you like!

