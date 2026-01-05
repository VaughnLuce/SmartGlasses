# Smart Glasses – Voice-Driven AI Assistant

## Project Purpose & End Goal

This project aims to create a **wearable pair of smart glasses** that allows me to **ask questions naturally and receive spoken responses instantly**, without needing to pull out my phone.

The motivation behind this project is **convenience and productivity**. I often have questions pop into my head while working, moving, or focusing on a task. Stopping to take out a phone can be disruptive or impractical, especially in work environments where phone usage is discouraged. These smart glasses are designed to remove that friction by enabling **hands-free, voice-based interaction with AI** wherever I am.

The end goal is a **lightweight, low-power wearable system** that listens only when prompted, processes speech using my phone and cloud AI, and delivers clear audio feedback directly to me.

---

## High-Level System Architecture

Below is a simplified overview of how the system communicates:

### System Communication Overview

1. The user presses a button on the glasses and speaks.
2. Audio is captured by the glasses’ microphone.
3. The audio is transmitted to the phone via Bluetooth.
4. The Android app converts speech to text.
5. The text is sent to the Gemini AI API.
6. Gemini generates a text response.
7. The Android app converts the response to speech.
8. Audio is played back to the user through the glasses’ speaker.


**Key idea:**  
The glasses remain lightweight and power-efficient, while the phone handles speech recognition, AI communication, and text-to-speech.

---

## Development Progress (Steps Completed So Far)

### 1. Android Development Environment
- Installed Android Studio
- Created a Kotlin-based Android project using XML layouts
- Configured minimum SDK and testing environment

### 2. App Permissions & Setup
- Added microphone permission
- Added internet permission
- Implemented runtime permission handling for audio recording

### 3. User Interface
- Created a simple UI with a single button
- Button states indicate:
  - Ready to talk
  - Listening
  - Processing AI request

### 4. Speech Input
- Implemented on-device speech-to-text using Android’s speech services
- App listens only when the user presses the button
- Captured spoken input as plain text

### 5. AI Integration
- Connected the app to the Gemini API
- Sent spoken input as a text prompt
- Limited both input size and output length for efficiency
- Received concise AI-generated text responses

### 6. Audio Output
- Implemented text-to-speech on the phone
- AI responses are spoken aloud to the user
- Voice selection handled through device TTS settings

### 7. API Key Security
- API key stored outside source control
- Injected securely into the app at build time
- Ensured keys are not committed to GitHub

### 8. App Flow Coordination
- Button press → speech capture
- Speech result → AI request
- AI response → spoken audio
- UI resets after each interaction

---

## Current State

At this stage, the **entire voice → AI → spoken response pipeline works on the phone**.  
The next phase of the project will focus on:

- Bluetooth communication between glasses and phone
- Microcontroller integration (ESP32)
- External microphone and speaker hardware
- Power optimization for wearable use

---

## Future Work

- Bluetooth audio or data streaming from glasses to phone
- Custom PCB for wearable hardware
- Wake-word or low-power trigger alternatives
- Optional camera integration
- Improved noise isolation for wearable microphone
- Standalone operation where possible

---

## Project Status

🚧 **In active development**  
This README documents progress so the project can be reproduced, extended, and refined over time.
