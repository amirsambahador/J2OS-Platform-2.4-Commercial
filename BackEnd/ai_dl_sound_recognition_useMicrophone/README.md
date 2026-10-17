# ai.dl.sound.recognition

*J2OS Platform 2.4*

A person speaks and a program should understand the words: a voice note that becomes text, a command given by speaking, a meeting that is written down while it takes place. This is **speech recognition**. It is not magic; it needs a trained model of a language, audio in the right shape, and some care about where each piece runs.

**ai.dl.sound.recognition** (`org.j2os.platform.ai.dl.sound.recognition`) brings it to J2OS on the server side. It is built on Vosk, an offline recognition engine, and has one class, `SpeechToText`, with one habit: *load the model once, recognize as many recordings as you like, close at the end*. On the browser side, the Nuxt composable `useMicrophone` records the voice, puts it in the exact shape the server wants and sends it. Three properties set the whole thing apart:

- **Offline.** The audio never leaves your servers. No online service, no key, no per-minute price.
- **Plain inputs.** `recognize` takes a path, a `File`, a `Path` or an `InputStream` (the file of an upload) and returns the text as a `String`.
- **Safe to share.** One loaded model serves many threads at the same time. Closing waits for the running recognitions.

The whole library in a few lines:

```java
try (SpeechToText speechToText = SpeechToText.load("models/vosk-model-small-fa-rhasspy-0.15")) {  // 1. load once
    String text = speechToText.recognize("audio/greeting.wav");                                    // 2. recognize
    System.out.println(text);
}                                                                                                  // 3. release the model
```

This guide goes from the first recognition (chapter 2) through the ways to give audio (3) and the sample rate (4) to the browser (5), mistakes (6) and a complete small workflow (7). The appendices work as a reference.

## Contents

1. [How ai.dl.sound.recognition works](#1-how-aidlsoundrecognition-works)
2. [The first recognition](#2-the-first-recognition)
3. [Ways to give audio](#3-ways-to-give-audio)
4. [The sample rate](#4-the-sample-rate)
5. [From the browser: useMicrophone](#5-from-the-browser-usemicrophone)
6. [When things go wrong](#6-when-things-go-wrong)
7. [Putting it together](#7-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The programs of the guide are complete: each has one `main` method, a Vosk model in a folder named `models` and short recordings in a folder named `audio`. The recordings are invented and short: a few spoken words each, 16 kHz, mono. The browser programs are written for Node, with a small stand-in for the browser pieces, and they call the Java side for the recognition.

---

## 1. How ai.dl.sound.recognition works

First, a few terms, because they come up throughout the guide:

- **Model.** A folder of files made by training, which holds what the engine knows about one language. Vosk publishes ready models for many languages, Persian included, in small and large sizes. Loading one is slow and takes memory.
- **WAV.** A simple audio file: a small header and then the raw sound.
- **Sample rate.** How many measurements of the sound there are per second. Most models expect 16000.
- **16-bit mono PCM.** The one audio format the module accepts: one channel, plain (uncompressed) 16-bit samples.
- **Utterance.** A stretch of speech between two pauses. The engine decides where one ends; the module joins the utterances with a space.

The pieces a program meets:

| Piece | Role |
|-------|------|
| `SpeechToText` | Loads a model, recognizes WAV audio, releases the model on `close` |
| `SpeechToTextFailedException` | The one checked exception of the module |
| `useMicrophone` | Nuxt composable: records, converts to the required WAV, uploads, shows the text |

Keep these three points in mind:

- **The audio format is strict.** WAV, PCM, 16-bit, mono, and a sample rate equal to the one the instance was loaded with. Anything else is refused with a message that says what to change. The module never converts audio.
- **Loading is slow, recognizing is not.** Load the model once when the application starts and keep the instance.
- **Text is plain.** The answer is the words the engine heard, usually without punctuation or capital letters. Persian text arrives intact; the module sets the encoding that the native engine needs.

## 2. The first recognition

`SpeechToText.load` takes the folder of a model. `recognize` takes a recording:

```java
import org.j2os.platform.ai.dl.sound.recognition.SpeechToText;

public class First {
    public static void main(String[] args) throws Exception {
        try (SpeechToText speechToText = SpeechToText.load("models/vosk-model-small-fa-rhasspy-0.15")) {
            System.out.println(speechToText.recognize("audio/greeting.wav"));
        }
    }
}
```

```text
سلام دنیا
```

- **The model folder.** It must exist; its content is Vosk's business.
- **`try`-with-resources.** `SpeechToText` is `AutoCloseable`. Closing releases the model, which is native memory that the garbage collector does not manage.
- **The recording.** A short WAV of someone saying two words. The answer is those two words, with a space between.

## 3. Ways to give audio

The same recording can come in several shapes, and `recognize` accepts all of them:

```java
import java.io.*;
import java.nio.file.*;
import org.j2os.platform.ai.dl.sound.recognition.SpeechToText;

public class Inputs {
    public static void main(String[] args) throws Exception {
        try (SpeechToText stt = SpeechToText.load("models/vosk-model-small-fa-rhasspy-0.15")) {
            // the same recording, handed over in four ways
            System.out.println("path   : " + stt.recognize("audio/greeting.wav"));
            System.out.println("Path   : " + stt.recognize(Path.of("audio/greeting.wav")));
            System.out.println("File   : " + stt.recognize(new File("audio/greeting.wav")));

            byte[] upload = Files.readAllBytes(Path.of("audio/greeting.wav"));      // e.g. a multipart upload
            System.out.println("stream : " + stt.recognize(new ByteArrayInputStream(upload)));

            // metadata chunks around the sound are fine
            System.out.println("tagged : " + stt.recognize("audio/tagged.wav"));

            // two phrases with a pause between them are joined with a space
            System.out.println("pause  : " + stt.recognize("audio/two-phrases.wav"));

            // nothing said: an empty text, not an error
            System.out.println("silence: [" + stt.recognize("audio/silence.wav") + "]");
        }
    }
}
```

```text
path   : سلام دنیا
Path   : سلام دنیا
File   : سلام دنیا
stream : سلام دنیا
tagged : سلام دنیا
pause  : بله نه
silence: []
```

- **Four forms.** A `String` path, a `Path`, a `File` and an `InputStream`. They all give the same text.
- **The stream.** For an uploaded file, pass its stream. The module reads the WAV from it and does **not** close it; whoever opened the stream closes it.
- **Metadata is fine.** Other parts of the WAV file, such as tags before or after the sound, are skipped. Only the `fmt ` and `data` parts matter.
- **Pauses.** A long pause ends an utterance. The utterances are joined with a space, so the answer is one string.
- **Silence.** A recording with nobody speaking gives an empty string, not an error.
- **Independent calls.** Each call starts a fresh recognizer, so one recording never influences the next.

## 4. The sample rate

The model and the recording must agree on the sample rate. The default is 16000. Phone-quality audio is often 8000:

```java
import org.j2os.platform.ai.dl.sound.recognition.SpeechToText;

public class Rates {
    public static void main(String[] args) throws Exception {
        String model = "models/vosk-model-small-fa-rhasspy-0.15";

        try (SpeechToText stt = SpeechToText.load(model)) {                  // 16000 Hz is the default
            try {
                stt.recognize("audio/phone-8k.wav");
            } catch (Exception e) {
                System.out.println(e.getMessage());
            }
        }

        try (SpeechToText stt = SpeechToText.load(model, 8000)) {            // tell the module the rate of your audio
            System.out.println(stt.recognize("audio/phone-8k.wav"));
        }
    }
}
```

```text
java.io.IOException: WAV sample rate (8000 Hz) does not match the model's expected sample rate (16000 Hz) — resample the audio, or load the model with SpeechToText.load(modelDirectory, 8000) instead
نه
```

- **The mismatch.** Audio at 8000 Hz against the default 16000 is refused. The message names both rates and the two ways out.
- **Way 1: resample the audio** to the rate of the model. This is the right choice when the model was trained for 16 kHz speech, as most are.
- **Way 2: tell the module.** `load(modelDirectory, 8000)` states the rate of your audio. Use it only when the model really handles that rate; the module does not check the model.
- **Not converted.** The module compares numbers; it never resamples. Audio in another format or another channel layout can be converted outside, for example:

```text
ffmpeg -i message.m4a -ar 16000 -ac 1 -sample_fmt s16 message.wav
```

## 5. From the browser: useMicrophone

Recorded audio from a page is rarely in the right shape: browsers record compressed audio (WebM or similar), often at 44.1 or 48 kHz. The composable `useMicrophone` closes that gap. The page only sees a few reactive values and two functions:

```html
<script setup>
const { isRecording, isProcessing, recognizedText, error, startRecording, stopRecording } =
  useMicrophone({ endpoint: '/api/speech-to-text/recognize' })

const toggle = () => (isRecording.value ? stopRecording() : startRecording())
</script>

<template>
  <button :disabled="isProcessing" @click="toggle">
    {{ isRecording ? 'Stop' : 'Record' }}
  </button>
  <p v-if="error">{{ error }}</p>
  <p>{{ recognizedText }}</p>
</template>
```

What happens behind the buttons:

- **Record.** `startRecording` asks the browser for the microphone, with echo cancellation, noise suppression and automatic gain turned on, and starts a recorder.
- **Stop.** `stopRecording` stops the recorder, releases the microphone, decodes the recording, resamples it to **16000 Hz mono** and writes it as a 16-bit PCM WAV. Then it uploads the WAV and waits for the answer. The returned promise resolves when everything is done, whether it worked or not.
- **Upload.** One `POST` with a form field named `file` (the file is called `audio.wav`) to the address given as `endpoint`. The answer must be JSON with a `text` field.
- **State.** `isRecording` and `isProcessing` show where the work is; `recognizedText` holds the text; `error` holds the message of any failure; `audioBlob` holds the WAV that was sent.
- **Saving.** `downloadRecording('name.wav')` saves the last WAV on the user's device. Nothing is sent anywhere.

The whole path, run in Node with a stand-in browser (`env` is the small test harness that records what the browser asked for and what the server received). The "server" hands the upload to the Java module of this guide:

```js
const mic = useMicrophone({ endpoint: '/api/speech-to-text/recognize' })
const state = () => ({ recording: mic.isRecording.value, processing: mic.isProcessing.value, text: mic.recognizedText.value })

console.log('idle      ', state())
await mic.startRecording()
console.log('recording ', state())
await mic.stopRecording()
console.log('finished  ', state())

// what the browser asked for, and what the server received
console.log('constraints', JSON.stringify(env.lastConstraints))
console.log('upload     ', env.uploaded.url, '| field "file" named', env.uploaded.name, '| starts with', env.uploaded.header)

// the WAV that was sent: read its header
const header = new DataView(await mic.audioBlob.value.arrayBuffer())
console.log('wav header  channels', header.getUint16(22, true), '| rate', header.getUint32(24, true), '| bits', header.getUint16(34, true))

mic.downloadRecording('my-voice.wav')
console.log('saved as   ', env.downloads)
```

```text
idle       { recording: false, processing: false, text: '' }
recording  { recording: true, processing: false, text: '' }
finished   { recording: false, processing: false, text: 'سلام دنیا' }
constraints {"audio":{"echoCancellation":true,"noiseSuppression":true,"autoGainControl":true}}
upload      /api/speech-to-text/recognize | field "file" named audio.wav | starts with RIFF
wav header  channels 1 | rate 16000 | bits 16
saved as    [ 'my-voice.wav' ]
```

- **Constraints.** The three audio options are exactly the ones above.
- **The upload.** A WAV (it starts with `RIFF`) in the field `file`.
- **The header.** One channel, 16000 Hz, 16 bits: the format `SpeechToText` demands, which is why the default model rate fits.
- **The answer.** The Java module recognized the upload; the page shows it in `recognizedText`.

On the server, a small controller receives the upload and calls the module. The model is loaded when the controller is created, and closed when the application stops:

```java
@RestController
@RequestMapping("/api/speech-to-text")
public class SpeechController {
    private final SpeechToText speechToText;

    public SpeechController(@Value("${sound.model-directory}") String modelDirectory)
            throws SpeechToTextFailedException {
        this.speechToText = SpeechToText.load(modelDirectory);       // once, not per request
    }

    @PostMapping("/recognize")
    public Map<String, String> recognize(@RequestParam("file") MultipartFile file) throws Exception {
        return Map.of("text", speechToText.recognize(file.getInputStream()));
    }

    @PreDestroy
    public void stop() {
        speechToText.close();
    }
}
```

The server side of the same upload, as a program:

```java
import java.io.*;
import org.j2os.platform.ai.dl.sound.recognition.SpeechToText;

public class Upload {
    public static void main(String[] args) throws Exception {
        try (SpeechToText stt = SpeechToText.load("models/vosk-model-small-fa-rhasspy-0.15");
             InputStream upload = new FileInputStream("audio/from-browser.wav")) {     // what the page sent
            System.out.println(stt.recognize(upload));
        }
    }
}
```

```text
سلام دنیا
```

- **Same format, same answer.** The file the page produced is accepted without any conversion on the server.
- **Size.** 16 kHz mono 16-bit audio is 32 000 bytes per second, about 1.9 MB per minute. Set an upload limit that fits the longest message you accept:

```text
spring.servlet.multipart.max-file-size=5MB
spring.servlet.multipart.max-request-size=5MB
```

## 6. When things go wrong

On the server the messages tell what to fix:

```java
import org.j2os.platform.ai.dl.sound.recognition.SpeechToText;

public class Checks {
    interface Step { void run() throws Exception; }

    static void attempt(String label, Step step) {
        try {
            step.run();
            System.out.println(label + " -> ok");
        } catch (Exception e) {
            System.out.println(label + " -> " + e.getClass().getSimpleName() + ": "
                    + String.valueOf(e.getMessage()).replace(System.getProperty("user.dir"), "."));
        }
    }

    public static void main(String[] args) throws Exception {
        attempt("no such model", () -> SpeechToText.load("models/nope"));
        attempt("empty folder", () -> SpeechToText.load("audio"));

        SpeechToText stt = SpeechToText.load("models/vosk-model-small-fa-rhasspy-0.15");
        attempt("stereo", () -> stt.recognize("audio/stereo.wav"));
        attempt("missing file", () -> stt.recognize("audio/nope.wav"));
        stt.close();
        attempt("after close", () -> stt.recognize("audio/greeting.wav"));
    }
}
```

```text
no such model -> SpeechToTextFailedException: java.io.FileNotFoundException: Model directory does not exist: ./models/nope
empty folder -> SpeechToTextFailedException: java.io.IOException: Failed to create a model
stereo -> SpeechToTextFailedException: java.io.IOException: Unsupported WAV channel count: expected mono (1), got 2 — convert the audio to mono before recognizing it
missing file -> SpeechToTextFailedException: java.io.FileNotFoundException: Audio file does not exist: ./audio/nope.wav
after close -> SpeechToTextFailedException: This SpeechToText instance has been closed
```

- **A missing model folder.** The path is not a directory.
- **A folder that is not a model.** The engine itself refuses it.
- **Stereo.** The message says to convert to mono. Other format mistakes are refused the same way (Appendix B).
- **A missing recording.** The path is not a file.
- **After close.** Every recognition fails with a plain message.

In the browser, nothing is thrown at the page: every failure ends in `error`:

```js
const mic = useMicrophone({ endpoint: '/api/speech-to-text/recognize' })

console.log('--- microphone not allowed ---')
env.denied = true
await mic.startRecording()
console.log('recording:', mic.isRecording.value, '| error:', mic.error.value)
env.denied = false

console.log('--- server answers 500 ---')
env.serverStatus = 500
await mic.startRecording()
await mic.stopRecording()
console.log('error:', mic.error.value, '| text: "' + mic.recognizedText.value + '" | processing:', mic.isProcessing.value)
env.serverStatus = 200

console.log('--- stop without start ---')
const other = useMicrophone({ endpoint: '/x' })
await other.stopRecording()
console.log('returned normally, error:', other.error.value)

console.log('--- download without a recording ---')
other.downloadRecording()
console.log('downloads:', env.downloads.length)
```

```text
--- microphone not allowed ---
recording: false | error: Permission denied
--- server answers 500 ---
error: Server responded with 500 | text: "" | processing: false
--- stop without start ---
returned normally, error: null
--- download without a recording ---
downloads: 0
```

- **Microphone not allowed.** `startRecording` catches the refusal, releases anything it opened and puts the browser's message in `error`. `isRecording` stays `false`.
- **Server error.** The upload is rejected with a status; the message is `Server responded with 500`, the text stays empty and `isProcessing` is `false` again.
- **Stop without start.** `stopRecording` simply returns.
- **Download without a recording.** Nothing happens.
- **Stop only while recording.** Calling `stopRecording` a second time on a recording that already ended makes the browser's recorder complain, because it is no longer recording. Guard the call with `isRecording`:

```js
// stopRecording only while a recording is running
if (isRecording.value) await stopRecording()
```

## 7. Putting it together

A service receives uploads from many users at the same time. The model is loaded once; each request thread recognizes its own file; at shutdown the service closes the model:

```java
import java.nio.file.*;
import java.util.*;
import java.util.concurrent.*;
import org.j2os.platform.ai.dl.sound.recognition.SpeechToText;

public class Workflow {
    public static void main(String[] args) throws Exception {
        // load once; every request thread shares the model
        SpeechToText stt = SpeechToText.load("models/vosk-model-small-fa-rhasspy-0.15");

        List<String> uploads = List.of("greeting.wav", "two-phrases.wav", "silence.wav", "tagged.wav", "stereo.wav", "greeting.wav");
        ExecutorService pool = Executors.newFixedThreadPool(3);
        List<Future<String>> answers = new ArrayList<>();
        for (String name : uploads) {
            answers.add(pool.submit(() -> stt.recognize(Path.of("audio", name))));
        }
        for (int i = 0; i < uploads.size(); i++) {
            try {
                System.out.println(uploads.get(i) + " -> [" + answers.get(i).get() + "]");
            } catch (ExecutionException e) {
                System.out.println(uploads.get(i) + " -> " + e.getCause().getMessage());
            }
        }

        pool.shutdown();
        stt.close();                                    // at shutdown: waits for running recognitions
        try {
            stt.recognize("audio/greeting.wav");
        } catch (Exception e) {
            System.out.println("after shutdown -> " + e.getMessage());
        }
    }
}
```

```text
greeting.wav -> [سلام دنیا]
two-phrases.wav -> [بله نه]
silence.wav -> []
tagged.wav -> [سلام دنیا]
stereo.wav -> java.io.IOException: Unsupported WAV channel count: expected mono (1), got 2 — convert the audio to mono before recognizing it
greeting.wav -> [سلام دنیا]
after shutdown -> This SpeechToText instance has been closed
```

- **Three threads, one model.** No lock is needed in the program: the instance guards itself.
- **Per-request failures.** The stereo file fails alone with its message; the other requests are not affected.
- **Empty is a result.** The silent recording gives `[]`, an empty text, which a page can show as "nothing heard".
- **Shutdown.** `close` waits until running recognitions are finished, then releases the model. Later calls fail with the message "This SpeechToText instance has been closed".

## Appendix A. Quick reference

### A.1 SpeechToText

| Call | Meaning |
|------|---------|
| `SpeechToText.load(modelDirectory)` | Loads the model for audio at 16000 Hz |
| `SpeechToText.load(modelDirectory, sampleRate)` | The same, for audio at the given rate |
| `recognize(String path)`, `recognize(Path)`, `recognize(File)` | Recognizes a WAV file |
| `recognize(InputStream)` | Recognizes a WAV stream; the stream stays open |
| `close()` | Waits for running recognitions, then releases the model. Safe to call twice |

### A.2 useMicrophone

| Member | Meaning |
|--------|---------|
| `useMicrophone({ endpoint })` | Creates the composable. `endpoint` is the upload address |
| `isRecording` | `true` while the microphone is recording |
| `isProcessing` | `true` while converting and uploading |
| `recognizedText` | The text from the server; empty until it arrives |
| `audioBlob` | The last WAV sent (16000 Hz, mono, 16-bit) |
| `error` | The message of the last failure, or `null` |
| `startRecording()` | Clears the previous result, opens the microphone, starts recording |
| `stopRecording()` | Stops, converts, uploads. The promise resolves when done |
| `downloadRecording(fileName)` | Saves the last WAV locally. Default name `recording.wav` |

### A.3 Accepted audio

| Property | Requirement |
|----------|-------------|
| Container | WAV with `RIFF` and `WAVE` marks |
| Format | PCM (format code 1) |
| Channels | 1 |
| Bits per sample | 16 |
| Sample rate | Equal to the rate the instance was loaded with (default 16000) |
| Order of parts | `fmt ` before `data`; other parts are skipped |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| The model folder does not exist | `SpeechToTextFailedException` caused by a `FileNotFoundException` |
| The folder is not a usable model | `SpeechToTextFailedException` caused by an `IOException` from the engine ("Failed to create a model") |
| The audio file does not exist | `SpeechToTextFailedException` caused by a `FileNotFoundException` |
| The audio is not RIFF/WAVE, or its parts are malformed | `SpeechToTextFailedException` caused by an `IOException` |
| Not PCM, not mono, not 16-bit | `SpeechToTextFailedException`, with the format found and what to do |
| The sample rate differs from the model's | `SpeechToTextFailedException`, with both rates and two ways out |
| A `fmt ` part that claims 2 GB or more (a corrupt header) | A bare `IllegalArgumentException` ("len < 0"), not wrapped in `SpeechToTextFailedException` |
| A stream that cannot be read | `SpeechToTextFailedException` caused by the `IOException` |
| `recognize` after `close` | `SpeechToTextFailedException` |
| A `null` argument | `NullPointerException` naming the argument |
| Nobody speaks | An empty `String`, no exception |
| Browser: microphone refused or missing | `error` holds the message; `isRecording` stays `false` |
| Browser: the recording cannot be decoded | `error` holds the browser's message |
| Browser: the server answers with a failure status | `error` is `Server responded with <status>` |
| Browser: the server answers without `text` | `recognizedText` is an empty string |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Loading per request.** Reading a model takes seconds and much memory. Load once.
- **Forgetting to close.** The model is native memory. Close it when the application stops.
- **The wrong sample rate.** Declare the rate of your audio, or convert it. Declaring a rate the model was not trained for gives bad text, not an error.
- **Compressed or stereo audio.** Phones and browsers record MP3, AAC, WebM or stereo. Convert before the module, or use `useMicrophone`, which already does.
- **A huge upload.** The module reads the audio piece by piece, but your web layer may hold the whole upload. Limit its size.
- **Expecting punctuation.** The text is plain words. Add punctuation yourself if the display needs it.
- **Trusting every word.** Recognition makes mistakes, especially with noise, accents, names and numbers. Let the user correct the text.
- **Calling `stopRecording` twice.** Guard with `isRecording`.
- **Confusing the two recognizers.** The browser also has its own speech recognition that sends audio to an online service. `useMicrophone` is the one that stays on your servers.

### C.2 Behavior worth knowing

- **Many threads are fine.** Recognitions run side by side on one instance, each with its own recognizer.
- **`close` is polite.** It waits for running recognitions to finish, then releases the model; later calls fail cleanly.
- **The stream stays open.** `recognize(InputStream)` never closes the stream it reads. The file and path forms open and close their own streams.
- **Chunked reading.** The sound is fed to the engine in 4096-byte pieces; the size field of the `data` part limits what is read, so parts after the sound are ignored.
- **UTF-8.** Unless you set it yourself, the module sets `jna.encoding` to UTF-8 so that Persian text is not damaged.
- **Streamed WAV.** A `data` part whose size field is the largest possible value, as some encoders write when the length is not known in advance, is read to the end of the stream.
- **Utterances are joined.** Each finished utterance and the final remainder are appended with a single space between them.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Audio format | WAV, PCM, 16-bit, mono |
| Sample rate | One per instance; default 16000 |
| Languages per instance | One model, one language |
| Output | Plain text; no timing, no alternatives, no confidence |
| Streaming recognition | None; a complete recording at a time |
| Browser conversion | 16000 Hz mono 16-bit WAV, fixed |
| Browser upload | One request, field `file`, answer `{ "text": ... }` |

### C.4 Tuning and practical advice

- **Pick the model for the job.** A small model loads fast and uses little memory; a large one makes fewer mistakes. Try both on your own recordings.
- **Load at start-up.** Fail early if the model is missing, instead of at the first user.
- **Keep recordings short.** Voice messages of a few seconds to a minute give the best experience; split longer audio.
- **Record well.** A close microphone and a quiet room matter more than any setting. The composable already asks the browser for noise suppression and echo cancellation.
- **Show the text, then let people edit it.**
- **Match the two ends.** The composable produces 16000 Hz; keep the default `load(modelDirectory)` on the server unless you change the composable.

## Appendix D. Message reference

### D.1 Loading

| Message | Exception | Cause |
|---------|-----------|-------|
| `java.io.FileNotFoundException: Model directory does not exist: <path>` | `SpeechToTextFailedException` | The model path is not a directory |
| `java.io.IOException: Failed to create a model` | `SpeechToTextFailedException` | The folder is not a usable model (message from the engine) |

### D.2 Recognizing

| Message | Exception | Cause |
|---------|-----------|-------|
| `java.io.FileNotFoundException: Audio file does not exist: <path>` | `SpeechToTextFailedException` | The audio path is not a file |
| `java.io.IOException: Not a valid RIFF/WAVE file` | `SpeechToTextFailedException` | The data does not start with `RIFF`/`WAVE` |
| `java.io.IOException: Malformed WAV file: no 'data' chunk found` | `SpeechToTextFailedException` | The data ends before a `data` part |
| `java.io.IOException: Malformed WAV file: 'data' chunk appeared before 'fmt ' chunk` | `SpeechToTextFailedException` | The parts are in the wrong order |
| `java.io.IOException: Malformed WAV file: 'fmt ' chunk is too short` | `SpeechToTextFailedException` | The `fmt ` part has fewer than 16 bytes |
| `java.io.IOException: Unsupported WAV audio format: expected PCM (1), got %d ...` | `SpeechToTextFailedException` | Compressed or floating-point audio |
| `java.io.IOException: Unsupported WAV channel count: expected mono (1), got %d ...` | `SpeechToTextFailedException` | More than one channel |
| `java.io.IOException: Unsupported WAV bit depth: expected 16-bit, got %d` | `SpeechToTextFailedException` | Not 16-bit |
| `java.io.IOException: WAV sample rate (%d Hz) does not match the model's expected sample rate (%d Hz) ...` | `SpeechToTextFailedException` | The rates differ |
| `This SpeechToText instance has been closed` | `SpeechToTextFailedException` | `recognize` after `close` |

### D.3 Argument checks

| Message | Exception | Cause |
|---------|-----------|-------|
| `<name> must not be null` | `NullPointerException` | `modelDirectory`, `audioFile`, `audioPath` or `audioInputStream` is `null` |

### D.4 Browser

| Message | Where | Cause |
|---------|-------|-------|
| the browser's own message, such as `Permission denied` | `error` | The microphone was refused or is missing |
| `Server responded with %d` | `error` | The upload got a failure status |
| the browser's own message | `error` | The recording could not be decoded, or the network failed |
