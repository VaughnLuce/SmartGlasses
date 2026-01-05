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

### Project Files Modified Outside MainActivity

- `local.properties` – Stores Gemini API key (not committed)
- `AndroidManifest.xml` – App permissions
- `app/build.gradle.kts` – API key injection via BuildConfig
- `activity_main.xml` – UI layout

## Step-by-Step Setup (Use the **Project** View in Android Studio)

> **Important:** These instructions assume you are using Android Studio’s **Project** dropdown (not the Android view).  
> In Android Studio (top-left panel), change the dropdown to: **Project**.

---

### 1) Create the Android Studio Project
1. Open **Android Studio**
2. Click **New Project**
3. Choose **Empty Views Activity**
4. Language: **Kotlin**
5. Finish and let Gradle sync

---

### 2) Switch to the Correct File Browser View
1. In the left sidebar (Project panel), open the dropdown that usually says **Android**
2. Select: **Project**

You should now see folders like:
- `.gradle/`
- `app/`
- `build.gradle(.kts)` files

---

### 3) Create `local.properties` and Store Your API Key
**Project panel click path:**
- **(Project Root)** → `local.properties`

If `local.properties` does not exist:
1. Right click **(Project Root)**  
2. **New** → **File**
3. Name it: `local.properties`

Add your key (example format):
- `GEMINI_API_KEY=YOUR_KEY_HERE` (no spaces)

---

### 4) Edit the App Module Gradle File (`:app`)
**Project panel click path:**
- **(Project Root)** → `app` → `build.gradle.kts`

In this file, you will:
- Enable `BuildConfig` (if needed)
- Read `local.properties`
- Inject `GEMINI_API_KEY` into `BuildConfig`

CHECK build.gradle.kts CODE FOR CLARIFICATION

import java.util.Properties //(very top of file)

buildConfigField("String", "GEMINI_API_KEY", "\"$geminiApiKey\"")
    // ✅ One buildFeatures block only
    buildFeatures {
        buildConfig = true
        compose = true
    }

### 5) Add Permissions in `AndroidManifest.xml`
**Project panel click path:**
- **(Project Root)** → `app` → `src` → `main` → `AndroidManifest.xml`

Add permissions for:
- Microphone recording
- Internet access

CHECK AndroidManifest.xml CODE FOR CLARIFICATION... include these
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <uses-permission android:name="android.permission.INTERNET" />

---

### 6) Create the UI Layout (Button) in `activity_main.xml`
**Project panel click path:**
- **(Project Root)** → `app` → `src` → `main` → `res` → `layout` → `activity_main.xml`

This is where the main button is defined (the one used to trigger listening / AI requests).
COPY THIS CODE
---
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:gravity="center"
    android:padding="16dp">

    <Button
        android:id="@+id/buttonRecord"
        android:layout_width="175dp"
        android:layout_height="242dp"
        android:text="Press to Speak" />

</LinearLayout>

---

### 7) Create Helper Package Folders (Inside Your App Package)
**Project panel click path:**
- **(Project Root)** → `app` → `src` → `main` → `java` → `com` → `example` → `smartglasses`

Inside `smartglasses`, create these folders (packages):
- `speech`
- `gemini`
- `tts`

How to create each folder:
1. Right click `smartglasses`
2. **New** → **Package**
3. Name it (example): `speech`
4. Repeat for `gemini` and `tts`

---

### 8) Add the Helper Files Into Those Packages
**Project panel click path (examples):**
- `.../smartglasses/speech/` → add your speech-to-text file
- `.../smartglasses/gemini/` → add your Gemini API client file
- `.../smartglasses/tts/` → add your text-to-speech helper file

How to add a Kotlin file:
1. Right click the package folder (ex: `speech`)
2. **New** → **Kotlin Class/File**
3. Paste your code into the new file

---

### 9) Paste/Use Your Main Code
**Project panel click path:**
- **(Project Root)** → `app` → `src` → `main` → `java` → `com` → `example` → `smartglasses` → `MainActivity.kt`

Paste your main app logic here (button control → speech-to-text → Gemini → text-to-speech).

---

### 10) Run the App
1. Connect a physical Android phone (recommended) OR start an emulator
2. Press **Run ▶**
3. Accept the microphone permission prompt
4. Press the button and test the full pipeline (speech → AI → spoken reply)

---

## Quick Reference: File Locations (Project View)

- **API Key File:**  
  `Project Root → local.properties`

- **App Gradle (Module :app):**  
  `Project Root → app → build.gradle.kts`

- **Manifest:**  
  `Project Root → app → src → main → AndroidManifest.xml`

- **UI Layout (Button):**  
  `Project Root → app → src → main → res → layout → activity_main.xml`

- **Kotlin Source Code:**  
  `Project Root → app → src → main → java → com → example → smartglasses`

