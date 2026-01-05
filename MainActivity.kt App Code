package com.example.smartglasses

import android.Manifest
import android.content.Intent
import android.content.pm.PackageManager
import android.os.Bundle
import android.speech.RecognitionListener
import android.speech.RecognizerIntent
import android.speech.SpeechRecognizer
import android.speech.tts.TextToSpeech
import android.util.Log
import android.view.MotionEvent
import android.widget.Button
import android.widget.Toast
import androidx.activity.ComponentActivity
import androidx.core.app.ActivityCompat
import androidx.core.content.ContextCompat
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.RequestBody.Companion.toRequestBody
import org.json.JSONArray
import org.json.JSONObject
import java.util.Locale

/**
 * MainActivity
 *
 * What this app does (high level):
 * 1) User holds down a button -> Android SpeechRecognizer starts listening (speech-to-text).
 * 2) When the user releases the button -> we stop listening and wait for final results.
 * 3) We send the recognized text to Gemini via HTTP (OkHttp).
 * 4) We speak Gemini's response aloud using Android TextToSpeech (TTS).
 *
 * Key design choices:
 * - "Hold to Talk" UX: ACTION_DOWN starts listening, ACTION_UP stops listening.
 * - Basic "in flight" lockout: while a Gemini request is running, we ignore touches.
 * - Keep prompts small: we truncate user speech to ~30 tokens (rough estimate).
 * - Keep responses small: maxOutputTokens is set to 80.
 */
class MainActivity : ComponentActivity(), TextToSpeech.OnInitListener {

    /**
     * Request code used when asking Android for microphone permission.
     * This is how we identify the permission result callback.
     */
    private val REQ_RECORD_AUDIO = 1001

    // ---------------------------
    // Gemini networking state
    // ---------------------------

    /**
     * OkHttpClient is the HTTP engine. We reuse one instance for performance and best practice.
     */
    private val client = OkHttpClient()

    /**
     * Prevent overlapping Gemini calls + prevents starting a new listen while we are waiting.
     * When true: we ignore button touches.
     * When false: user can hold to talk.
     */
    private var isApiCallInFlight = false

    // ---------------------------
    // Speech-to-text state (Android SpeechRecognizer)
    // ---------------------------

    /**
     * SpeechRecognizer performs speech-to-text on-device/Google services depending on device.
     * It is lifecycle-sensitive; we create it in setupSpeechRecognizer() and destroy it in onDestroy().
     */
    private var speechRecognizer: SpeechRecognizer? = null

    /**
     * Intent describing how we want Android speech recognition to behave (free form, language, etc.)
     * This is passed to startListening().
     */
    private var recognizerIntent: Intent? = null

    // ---------------------------
    // Text-to-speech state (Android TTS)
    // ---------------------------

    /**
     * TextToSpeech engine used to speak Gemini response out loud.
     * Initialized in onCreate(), becomes usable once onInit() reports SUCCESS.
     */
    private var tts: TextToSpeech? = null

    /**
     * True once the TTS engine successfully initialized and language is supported.
     * If false, speak() becomes a no-op.
     */
    private var ttsReady = false

    // ---------------------------
    // Activity lifecycle
    // ---------------------------

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Load UI layout (activity_main.xml) which must contain a Button with id "buttonRecord".
        setContentView(R.layout.activity_main)

        // Initialize TTS. Android will call onInit(status) when the engine is ready.
        tts = TextToSpeech(this, this)

        // Grab the record button and set its initial label.
        val button = findViewById<Button>(R.id.buttonRecord)
        button.text = "Hold to Talk"

        // Set up Android SpeechRecognizer + intent + callbacks.
        setupSpeechRecognizer()

        /**
         * Touch handling:
         * - ACTION_DOWN: start listening (if permission ok).
         * - ACTION_UP/CANCEL: stop listening and show "Processing…"
         *
         * Note: We use setOnTouchListener instead of setOnClickListener
         * because we want "press and hold" behavior, not a single click.
         */
        button.setOnTouchListener { v, event ->

            // If a Gemini request is currently running, ignore touches.
            // Returning true means "we handled it" (so the touch doesn't trigger other behavior).
            if (isApiCallInFlight) return@setOnTouchListener true

            when (event.action) {

                MotionEvent.ACTION_DOWN -> {
                    // If we don't have mic permission, ask for it and stop here.
                    if (!hasMicPermission()) {
                        requestMicPermission()
                        return@setOnTouchListener true
                    }

                    /**
                     * If the phone is currently speaking a previous answer,
                     * stop immediately when the user begins talking again.
                     *
                     * Note: "ttsReady == true" is redundant (ttsReady is Boolean),
                     * but we keep it as-is since you asked not to change code logic.
                     */
                    if (ttsReady == true && tts?.isSpeaking == true) {
                        tts?.stop()
                    }

                    // Update UI state to show user we are listening.
                    button.text = "Listening… (release)"
                    v.isPressed = true

                    // Start speech recognition using the configured intent.
                    speechRecognizer?.startListening(recognizerIntent)
                    true
                }

                MotionEvent.ACTION_UP, MotionEvent.ACTION_CANCEL -> {
                    // User released the button (or touch was canceled).
                    v.isPressed = false

                    // UI indicates we're now processing speech -> text -> Gemini.
                    button.text = "Processing…"

                    // Stop listening and wait for onResults() or onError().
                    speechRecognizer?.stopListening()
                    true
                }

                // Ignore other touch actions (move, etc.).
                else -> false
            }
        }
    }

    // ---------------------------
    // TTS (Text-To-Speech)
    // ---------------------------

    /**
     * Called by Android TextToSpeech engine once initialization finishes.
     * status == TextToSpeech.SUCCESS means we can use TTS *if* the language is supported.
     */
    override fun onInit(status: Int) {
        ttsReady = (status == TextToSpeech.SUCCESS)

        if (ttsReady) {
            // Choose the device default language for speech output.
            // setLanguage returns status codes that tell us if language data is missing/unsupported.
            val result = tts?.setLanguage(Locale.getDefault())

            // If language isn't available on device, disable TTS and warn the user.
            if (result == TextToSpeech.LANG_MISSING_DATA || result == TextToSpeech.LANG_NOT_SUPPORTED) {
                ttsReady = false
                Toast.makeText(this, "TTS language not supported on this device.", Toast.LENGTH_LONG).show()
            }
        } else {
            // Initialization failed completely (no engine / error).
            Toast.makeText(this, "TTS init failed.", Toast.LENGTH_LONG).show()
        }
    }

    /**
     * Speak text aloud (Gemini reply).
     * - QUEUE_FLUSH means: stop any current speech and replace it with this new speech.
     * - "GEMINI_REPLY" is an utterance ID (useful for callbacks if you add them later).
     */
    private fun speak(text: String) {
        if (!ttsReady) return
        tts?.speak(text, TextToSpeech.QUEUE_FLUSH, null, "GEMINI_REPLY")
    }

    // ---------------------------
    // Speech Recognizer (Speech -> Text)
    // ---------------------------

    /**
     * Configure Android's speech recognition.
     *
     * 1) Check if speech recognition is available.
     * 2) Create SpeechRecognizer instance.
     * 3) Create recognizerIntent (free-form language).
     * 4) Attach a RecognitionListener to receive callbacks.
     */
    private fun setupSpeechRecognizer() {
        // Some devices/emulators may not support speech recognition.
        if (!SpeechRecognizer.isRecognitionAvailable(this)) {
            Toast.makeText(this, "Speech recognition not available on this device.", Toast.LENGTH_LONG).show()
            return
        }

        // Create the recognizer instance tied to this Activity context.
        speechRecognizer = SpeechRecognizer.createSpeechRecognizer(this)

        // Build an Intent that tells Android how we want recognition to behave.
        recognizerIntent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {

            // Free form means: conversational speech, not a strict grammar.
            putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)

            // Use the device default language (you could set "en-US", etc., if desired).
            putExtra(RecognizerIntent.EXTRA_LANGUAGE, Locale.getDefault())

            // If true, you can get partial/intermediate transcripts. You're using false for only final results.
            putExtra(RecognizerIntent.EXTRA_PARTIAL_RESULTS, false)
        }

        // Attach callbacks so we know what the recognizer is doing and what it heard.
        speechRecognizer?.setRecognitionListener(object : RecognitionListener {
            override fun onReadyForSpeech(params: Bundle?) {
                // Called when the recognizer is ready to listen.
                // Useful place to update UI if needed.
            }

            override fun onBeginningOfSpeech() {
                // Called when speech is detected.
            }

            override fun onRmsChanged(rmsdB: Float) {
                // Called with sound level changes (volume). Useful for visual meters.
            }

            override fun onBufferReceived(buffer: ByteArray?) {
                // Called with audio buffer (rarely used for typical apps).
            }

            override fun onEndOfSpeech() {
                // Called when the user stops talking.
                // Note: you are also explicitly calling stopListening() on button release.
            }

            /**
             * If recognition fails for any reason, Android provides an error code.
             * We show a toast and reset the button back to "Hold to Talk".
             */
            override fun onError(error: Int) {
                runOnUiThread {
                    Toast.makeText(this@MainActivity, "Speech error: $error", Toast.LENGTH_SHORT).show()
                    resetButton()
                }
            }

            /**
             * Called when final recognition results are ready.
             * We pull out the best string and send it to Gemini.
             */
            override fun onResults(results: Bundle?) {
                // Android returns multiple candidate strings; best guess is usually first.
                val texts = results?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)

                // Take first result, trim spaces; if null, produce empty string.
                val userText = texts?.firstOrNull()?.trim().orEmpty()

                // If we got nothing useful, show message and reset UI.
                if (userText.isBlank()) {
                    runOnUiThread {
                        Toast.makeText(this@MainActivity, "No speech detected.", Toast.LENGTH_SHORT).show()
                        resetButton()
                    }
                    return
                }

                // Debug log of what speech recognition heard.
                Log.d("STT_TEXT", userText)

                // Send recognized text to Gemini.
                callGeminiWithText(userText)
            }

            override fun onPartialResults(partialResults: Bundle?) {
                // Not used because EXTRA_PARTIAL_RESULTS is false.
            }

            override fun onEvent(eventType: Int, params: Bundle?) {
                // Reserved for future events.
            }
        })
    }

    // ---------------------------
    // Token limiting helper
    // ---------------------------

    /**
     * Attempts to truncate text to an approximate token count.
     *
     * Why approximate?
     * - True tokenization depends on the model and tokenizer.
     * - For quick limiting, a rough rule of thumb is ~4 characters per token for English.
     *
     * What it does:
     * 1) Trim ends.
     * 2) Collapse all whitespace into single spaces.
     * 3) Cut off at maxTokens * 4 characters.
     */
    private fun truncateToApproxTokens(text: String, maxTokens: Int): String {
        // Rough estimate: ~4 characters per token.
        val maxChars = maxTokens * 4

        // Normalize whitespace so we don't waste tokens on long spacing/newlines.
        val cleaned = text.trim().replace(Regex("\\s+"), " ")

        // If already short enough, return as-is; otherwise cut to maxChars.
        return if (cleaned.length <= maxChars) {
            cleaned
        } else {
            cleaned.take(maxChars).trimEnd()
        }
    }

    // ---------------------------
    // Gemini request flow (Speech -> Gemini -> Speak)
    // ---------------------------

    /**
     * Takes the recognized user text and:
     * - Disables the button (so user can't spam requests)
     * - Truncates input (~30 tokens)
     * - Adds an instruction ("Reply briefly.")
     * - Calls askGeminiText() which performs the HTTP request
     * - Speaks reply and resets UI
     */
    private fun callGeminiWithText(userText: String) {
        val button = findViewById<Button>(R.id.buttonRecord)

        // Disable UI while calling the API.
        setButtonEnabled(button, false)
        button.text = "Asking Gemini…"

        // Limit what we send (~30 tokens).
        val limitedUserText = truncateToApproxTokens(userText, 30)

        // Prompt instruction goes HERE (this guides Gemini to keep output short).
        val prompt = "$limitedUserText\nReply briefly."

        // Make the actual HTTP call and handle the result.
        askGeminiText(prompt) { reply ->
            // Show the reply in a toast (quick debug/user feedback).
            Toast.makeText(this, reply, Toast.LENGTH_LONG).show()

            // Speak the reply aloud.
            speak(reply)

            // Restore UI to "Hold to Talk".
            resetButton()
        }
    }

    // ---------------------------
    // UI helpers
    // ---------------------------

    /**
     * Restores the record button to its idle state.
     */
    private fun resetButton() {
        val button = findViewById<Button>(R.id.buttonRecord)
        button.text = "Hold to Talk"
        setButtonEnabled(button, true)
    }

    /**
     * Enable/disable button and dim it visually.
     *
     * Also flips isApiCallInFlight:
     * - If enabled == false -> API is in flight (lock out)
     * - If enabled == true  -> ready for user input again
     */
    private fun setButtonEnabled(button: Button, enabled: Boolean) {
        isApiCallInFlight = !enabled
        button.isEnabled = enabled
        button.alpha = if (enabled) 1.0f else 0.6f
    }

    // ---------------------------
    // Permissions (Microphone)
    // ---------------------------

    /**
     * Check if RECORD_AUDIO permission is already granted.
     */
    private fun hasMicPermission(): Boolean {
        return ContextCompat.checkSelfPermission(this, Manifest.permission.RECORD_AUDIO) ==
                PackageManager.PERMISSION_GRANTED
    }

    /**
     * Ask the user for microphone permission.
     * Android will call onRequestPermissionsResult().
     */
    private fun requestMicPermission() {
        ActivityCompat.requestPermissions(
            this,
            arrayOf(Manifest.permission.RECORD_AUDIO),
            REQ_RECORD_AUDIO
        )
    }

    /**
     * Receives the user's response to the microphone permission dialog.
     * If granted, we show a toast; if denied, we show a toast.
     *
     * Note: This code doesn't automatically start listening after granting permission.
     * The user would press and hold again after granting.
     */
    override fun onRequestPermissionsResult(
        requestCode: Int,
        permissions: Array<out String>,
        grantResults: IntArray
    ) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults)

        if (requestCode == REQ_RECORD_AUDIO) {
            val granted = grantResults.isNotEmpty() && grantResults[0] == PackageManager.PERMISSION_GRANTED
            Toast.makeText(
                this,
                if (granted) "Mic permission granted." else "Mic permission denied.",
                Toast.LENGTH_SHORT
            ).show()
        }
    }

    // ---------------------------
    // Gemini call (text only)
    // ---------------------------

    /**
     * Sends a text prompt to Gemini and returns the model's text reply via callback.
     *
     * Implementation notes:
     * - Uses the Gemini REST endpoint with your API key.
     * - Builds a JSON body with "contents" -> "parts" -> { "text": prompt }.
     * - Limits output tokens to 80 to control cost / verbosity.
     * - Runs the network call on a background Thread (not on main UI thread).
     * - Uses runOnUiThread to deliver callback results safely back to UI.
     */
    private fun askGeminiText(prompt: String, onResult: (String) -> Unit) {
        // API key is stored in BuildConfig (usually injected from gradle / local properties).
        val apiKey = BuildConfig.GEMINI_API_KEY

        // REST endpoint: model + method + API key query param.
        val url =
            "https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=$apiKey"

        /**
         * Build request JSON body.
         *
         * Rough structure:
         * {
         *   "contents": [
         *     {
         *       "parts": [
         *         { "text": "..." }
         *       ]
         *     }
         *   ],
         *   "generationConfig": { "maxOutputTokens": 80 }
         * }
         */
        val bodyJson = JSONObject().apply {
            put(
                "contents",
                JSONArray().put(
                    JSONObject().apply {
                        put(
                            "parts",
                            JSONArray().put(
                                JSONObject().put("text", prompt)
                            )
                        )
                    }
                )
            )
            // Keep outputs small to save credits.
            put("generationConfig", JSONObject().put("maxOutputTokens", 80))
        }

        // Convert JSON to request body with content-type application/json.
        val body = bodyJson.toString().toRequestBody("application/json".toMediaType())

        // Build the HTTP request (POST).
        val request = Request.Builder().url(url).post(body).build()

        /**
         * Network call must not run on UI thread.
         * This uses a raw Thread; alternatives could include coroutines, but we keep your code unchanged.
         */
        Thread {
            try {
                // Execute HTTP call synchronously inside this background thread.
                client.newCall(request).execute().use { response ->

                    // Read the raw response body as a string (may be empty).
                    val responseBody = response.body?.string().orEmpty()

                    // Debug logging (helpful while developing).
                    Log.d("GEMINI_RAW", "HTTP ${response.code} ${response.message}")
                    Log.d("GEMINI_RAW", responseBody)

                    // Parse JSON response.
                    val root = JSONObject(responseBody)

                    // If server returned an error object, surface it to the user.
                    if (root.has("error")) {
                        val msg = root.getJSONObject("error").optString("message", "Unknown error")
                        runOnUiThread { onResult("API Error ${response.code}: $msg") }
                        return@use
                    }

                    // Gemini typically returns "candidates" array.
                    val candidates = root.optJSONArray("candidates")
                    if (candidates == null || candidates.length() == 0) {
                        runOnUiThread { onResult("No candidates returned.") }
                        return@use
                    }

                    /**
                     * Extract the text:
                     * candidates[0].content.parts[0].text
                     *
                     * This assumes:
                     * - at least one candidate
                     * - content exists
                     * - at least one part exists
                     * - part[0] contains "text"
                     */
                    val content = candidates.getJSONObject(0).optJSONObject("content")
                    val parts = content?.optJSONArray("parts")
                    val reply = parts?.optJSONObject(0)?.optString("text", null)

                    // Return to main thread and invoke callback.
                    runOnUiThread { onResult(reply ?: "No text reply.") }
                }
            } catch (e: Exception) {
                // Any parsing/network exceptions end up here.
                // Return the error message to the UI thread.
                runOnUiThread { onResult("Error: ${e.message}") }
            }
        }.start()
    }

    // ---------------------------
    // Cleanup
    // ---------------------------

    /**
     * Always release resources to avoid memory leaks:
     * - SpeechRecognizer uses system resources.
     * - TTS holds an engine instance.
     */
    override fun onDestroy() {
        super.onDestroy()

        // Clean up speech recognition resources.
        speechRecognizer?.destroy()
        speechRecognizer = null

        // Clean up TTS resources.
        tts?.stop()
        tts?.shutdown()
        tts = null
    }
}
