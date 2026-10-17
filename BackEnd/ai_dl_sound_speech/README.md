# ai.dl.sound.speech

*J2OS Platform 2.4*

An order has shipped and the phone says so aloud. A learning app reads a lesson to someone who cannot look at the screen. A call center plays a greeting in a pleasant voice. In all of these, a program turns written text into spoken audio. This is **text-to-speech**, also called speech synthesis.

**ai.dl.sound.speech** (`org.j2os.platform.ai.dl.sound.speech`) brings it to J2OS. It is built on Piper, a fast offline speech synthesizer, and has one class, `TextToSpeech`, with one method in two forms: *give a text, get a WAV*. Three properties set it apart:

- **Offline.** The text never leaves your servers. No online service, no key, no per-character price.
- **Plain inputs and outputs.** A `String` goes in; a WAV file or a `byte[]` comes out.
- **Simple to run.** Piper is started as its own process for every call. Nothing stays in memory between calls, so there is nothing to warm up and nothing to leak.

The whole library in a few lines:

```java
try (TextToSpeech tts = TextToSpeech.load("piper", "fa_IR-ganji-medium")) {   // 1. find Piper and the voice
    tts.synthesize("سلام، حالت چطوره؟", "out/hello.wav");                       // 2. text to a WAV file
    byte[] wav = tts.synthesize("یک پیام دیگر");                               // 3. or to bytes, no file
}
```

This guide goes from the first speech (chapter 2) through file or bytes (3), text and length (4) and time limits and failures (5) to the web (6) and a complete small workflow (7). The appendices work as a reference.

## Contents

1. [How ai.dl.sound.speech works](#1-how-aidlsoundspeech-works)
2. [The first speech](#2-the-first-speech)
3. [File or bytes](#3-file-or-bytes)
4. [Text and length](#4-text-and-length)
5. [Time limits and failures](#5-time-limits-and-failures)
6. [For the web](#6-for-the-web)
7. [Putting it together](#7-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The programs of the guide are complete: each has one `main` method and finds Piper and a Persian voice in a folder named `piper`, as shown in chapter 1. Their output goes to a folder named `out`.

---

## 1. How ai.dl.sound.speech works

First, a few terms, because they come up throughout the guide:

- **Piper.** A separate command-line program that turns text into audio. It is not part of the library: download it for your system and unpack it. The module starts it and talks to it through its standard input and output.
- **Voice.** A model file with the extension `.onnx` that decides how the speech sounds, in one language and one voice. Piper publishes many voices, Persian ones included. A voice name looks like `fa_IR-ganji-medium`: language and region, speaker, and quality.
- **WAV.** A simple audio file: a small header and then the raw sound. Piper writes 16-bit mono WAV; the sample rate depends on the voice.
- **Synthesis.** The act of turning one text into one audio.

The classes a program meets:

| Class | Role |
|-------|------|
| `TextToSpeech` | Finds Piper and a voice, turns text into WAV on every `synthesize` call |
| `TextToSpeechFailedException` | The one checked exception of the module |

The module expects this layout:

```text
piper/
├── piper                      (piper.exe on Windows)
└── model/
    └── fa_IR-ganji-medium.onnx
```

Keep these three points in mind:

- **`load` only looks.** It checks that the executable and the voice file exist and does not start Piper. A voice file that is damaged is found out at the first `synthesize`.
- **One process per call.** Every `synthesize` starts Piper, sends the text, waits and collects the audio. That costs a start-up time on each call, and it means calls can run side by side.
- **A time limit guards every call.** The default is 60 seconds. A Piper process that runs longer is killed.

## 2. The first speech

`TextToSpeech.load` takes the Piper folder and the name of the voice. `synthesize` takes a text and a path:

```java
import java.nio.file.*;
import org.j2os.platform.ai.dl.sound.speech.TextToSpeech;

public class First {
    public static void main(String[] args) throws Exception {
        try (TextToSpeech tts = TextToSpeech.load("piper", "fa_IR-ganji-medium")) {
            tts.synthesize("سلام، امروز حالت چطوره؟", "out/hello.wav");
        }
        long size = Files.size(Path.of("out/hello.wav"));
        System.out.println(size + " bytes, about " + Math.round((size - 44) / 2.0 / 22050 * 10) / 10.0 + " seconds");
    }
}
```

```text
79418 bytes, about 1.8 seconds
```

- **`load("piper", "fa_IR-ganji-medium")`.** Looks for `piper.exe` or `piper` in the folder, and for `model/fa_IR-ganji-medium.onnx` inside it.
- **`synthesize(text, path)`.** Writes the audio to the path. The folder `out` did not exist and is created.
- **The result.** A WAV file. Its size follows the length of the speech: 16-bit samples, 22050 per second for this voice, plus a 44-byte header.
- **`try`-with-resources.** `TextToSpeech` is `AutoCloseable` so that it fits next to the other sound classes, but `close` does nothing: no resource is held between calls.

## 3. File or bytes

Both forms of `synthesize` are safe to use in a server:

```java
import java.nio.file.*;
import java.util.stream.*;
import org.j2os.platform.ai.dl.sound.speech.TextToSpeech;

public class Output {
    public static void main(String[] args) throws Exception {
        TextToSpeech tts = TextToSpeech.load("piper", "fa_IR-ganji-medium");

        // to a file: folders are created, an existing file is replaced in one step
        tts.synthesize("یک پیام", "out/deep/msg.wav");
        tts.synthesize("یک پیام دیگر و کمی بلندتر", "out/deep/msg.wav");

        // to bytes: nothing stays on disk
        byte[] wav = tts.synthesize("سلام");
        System.out.println("bytes: " + wav.length + ", starts with " + new String(wav, 0, 4));

        try (Stream<Path> files = Files.list(Path.of("out/deep"))) {
            System.out.println("in out/deep: " + files.map(p -> p.getFileName().toString()).collect(Collectors.toList()));
        }
        System.out.println("msg.wav: " + Files.size(Path.of("out/deep/msg.wav")) + " bytes");
    }
}
```

```text
bytes: 17682, starts with RIFF
in out/deep: [msg.wav]
msg.wav: 97054 bytes
```

- **To a file.** Missing folders are created. The audio is first written to a temporary file in the target folder and then moved over the target in one step, so a reader never sees a half-written file, and the second call above replaced the first one cleanly.
- **To bytes.** `synthesize(text)` returns the WAV as a `byte[]` and leaves no file behind. It is the form for a web response or for processing in memory.
- **A clean folder.** After both calls the folder holds only `msg.wav`; the temporary files are gone.
- **A failed call.** If Piper fails, the temporary file is removed and an existing target file stays as it was.

## 4. Text and length

The audio is as long as the speech takes. Longer text means a longer audio and a longer wait:

```java
import org.j2os.platform.ai.dl.sound.speech.TextToSpeech;

public class Length {
    public static void main(String[] args) throws Exception {
        TextToSpeech tts = TextToSpeech.load("piper", "fa_IR-ganji-medium");
        for (String text : new String[]{"سلام", "سلام، حالت چطوره؟", "سلام، امروز حالت چطوره؟ هوا خیلی خوب است."}) {
            byte[] wav = tts.synthesize(text);
            double seconds = (wav.length - 44) / 2.0 / 22050;
            System.out.printf("%2d words -> %.1f s  %s%n", text.split("\\s+").length, seconds, text);
        }
    }
}
```

```text
 1 words -> 0.4 s  سلام
 3 words -> 1.3 s  سلام، حالت چطوره؟
 8 words -> 3.3 s  سلام، امروز حالت چطوره؟ هوا خیلی خوب است.
```

- **Time grows with the text.** The more words, the longer the audio and the longer Piper runs for it.
- **Persian text.** The text is sent to Piper as UTF-8, so Persian letters and punctuation arrive intact.
- **Limit what users can send.** A long text keeps one Piper process busy for a long time. For text that comes from users, set a maximum length (chapter 6).
- **Split long documents.** Speak a long text sentence by sentence or paragraph by paragraph and join the files, so that each call is short and a failure costs little.

## 5. Time limits and failures

`load` reports what it can see at once:

```java
import java.time.Duration;
import org.j2os.platform.ai.dl.sound.speech.TextToSpeech;

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
        attempt("no piper folder", () -> TextToSpeech.load("nowhere", "fa_IR-ganji-medium"));
        attempt("no such voice", () -> TextToSpeech.load("piper", "en_US-lessac"));
        attempt("zero timeout", () -> TextToSpeech.load("piper", "fa_IR-ganji-medium", Duration.ZERO));

        TextToSpeech tts = TextToSpeech.load("piper", "fa_IR-ganji-medium");
        attempt("null text", () -> tts.synthesize(null));
    }
}
```

```text
no piper folder -> TextToSpeechFailedException: java.io.FileNotFoundException: piper executable not found under: ./nowhere (expected piper.exe or piper)
no such voice -> TextToSpeechFailedException: java.io.FileNotFoundException: Voice model not found: ./piper/model/en_US-lessac.onnx
zero timeout -> IllegalArgumentException: timeout must be positive, was PT0S
null text -> NullPointerException: text must not be null
```

- **No Piper.** The folder holds neither `piper.exe` nor `piper`. The message shows where it looked.
- **No voice.** `model/<name>.onnx` is missing.
- **A time limit that is zero or negative.** An `IllegalArgumentException` at `load`.
- **No text.** A `null` text is a `NullPointerException` that names the argument.

Problems that only show when Piper runs come later, at `synthesize`:

```java
import java.time.Duration;
import java.nio.file.*;
import org.j2os.platform.ai.dl.sound.speech.TextToSpeech;

public class Failures {
    interface Step { void run() throws Exception; }

    static void attempt(String label, Step step) {
        try {
            step.run();
            System.out.println(label + " -> ok");
        } catch (Exception e) {
            System.out.println(label + " -> " + e.getClass().getSimpleName() + ": "
                    + String.valueOf(e.getMessage()).replace(System.getProperty("user.dir"), ".").strip());
        }
    }

    public static void main(String[] args) throws Exception {
        // the file exists, but Piper cannot use it as a voice
        TextToSpeech broken = TextToSpeech.load("piper", "broken");
        attempt("broken voice", () -> broken.synthesize("سلام"));

        // a time limit that is too short for the text
        TextToSpeech hurried = TextToSpeech.load("piper", "fa_IR-ganji-medium", Duration.ofSeconds(1));
        String longText = "این یک متن بلند است که ساختن صدایش بیشتر از یک ثانیه طول می‌کشد.";
        attempt("too slow", () -> hurried.synthesize(longText, "out/late.wav"));
        System.out.println("late.wav exists: " + Files.exists(Path.of("out/late.wav")));
    }
}
```

```text
broken voice -> TextToSpeechFailedException: piper exited with code 1: Error: failed to load voice model piper/model/broken.onnx
too slow -> TextToSpeechFailedException: piper did not finish within 1 seconds
late.wav exists: false
```

- **A damaged voice.** The file is there, so `load` accepts it, but Piper cannot use it and exits with an error code. The exception carries the code and everything Piper printed, which usually explains the problem.
- **Too slow.** When Piper runs past the limit it is killed and the exception says so. The limit covers the whole run, and no partial file is left behind.
- **Whole seconds.** The time is written in whole seconds, rounded down; use limits of at least a second.

To change the limit, give `load` a duration:

```java
// allow a minute and a half per call, for long texts on a slow machine
TextToSpeech tts = TextToSpeech.load("piper", "fa_IR-ganji-medium", Duration.ofSeconds(90));
```

## 6. For the web

A web page asks for speech and plays it. The server turns the text into bytes and returns them as audio, with a limit on the length of the text:

```java
@GetMapping(value = "/api/speech", produces = "audio/wav")
public byte[] speak(@RequestParam String text) throws TextToSpeechFailedException {
    if (text.length() > 500) {
        throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "text too long");
    }
    return textToSpeech.synthesize(text);
}
```

The page fetches the audio and plays it:

```js
const response = await fetch('/api/speech?text=' + encodeURIComponent(text))
const url = URL.createObjectURL(await response.blob())
await new Audio(url).play()
```

- **Bytes, not files.** The controller returns the `byte[]` straight from `synthesize`. No file needs cleaning up.
- **A limit.** The text length is checked before the call, because every call costs a process and some seconds.
- **A cache.** If the same texts are asked for again and again, such as notifications, keep the audio in memory (chapter 7).
- **Waiting.** A call returns when the speech is complete. For a long text, answer the request quickly and let the page poll for the result, or split the text.

## 7. Putting it together

A shop speaks short notifications. Four requests arrive at the same time; each repeated notification is spoken once and served from memory afterwards; one voice message gets the robot effect from the sound effect module, which works on exactly this kind of WAV:

```java
import java.nio.file.*;
import java.util.*;
import java.util.concurrent.*;
import org.j2os.platform.ai.dl.sound.effect.Effect;
import org.j2os.platform.ai.dl.sound.effect.VoiceEffect;
import org.j2os.platform.ai.dl.sound.speech.TextToSpeech;
import org.j2os.platform.ai.dl.sound.speech.exception.TextToSpeechFailedException;

public class Workflow {
    static final Map<String, byte[]> cache = new ConcurrentHashMap<>();

    // announcements are spoken once, then served from memory
    static byte[] announce(TextToSpeech tts, String text) {
        return cache.computeIfAbsent(text, t -> {
            try {
                return tts.synthesize(t);
            } catch (TextToSpeechFailedException e) {
                throw new IllegalStateException(e);
            }
        });
    }

    public static void main(String[] args) throws Exception {
        TextToSpeech tts = TextToSpeech.load("piper", "fa_IR-ganji-medium");
        List<String> texts = List.of("سفارش شما ارسال شد", "پرداخت انجام شد", "سفارش شما ارسال شد", "خوش آمدید");

        ExecutorService pool = Executors.newFixedThreadPool(4);       // each call starts its own Piper process
        List<Future<byte[]>> results = new ArrayList<>();
        for (String t : texts) results.add(pool.submit(() -> announce(tts, t)));
        for (int i = 0; i < texts.size(); i++) {
            System.out.println(texts.get(i) + " -> " + results.get(i).get().length + " bytes");
        }
        pool.shutdown();

        // the output is 16-bit mono WAV, so the voice effects of the sound module can follow
        Files.write(Path.of("welcome.wav"), announce(tts, "خوش آمدید"));
        VoiceEffect.apply("welcome.wav", "welcome-robot.wav", Effect.ROBOT);
        System.out.println("robot voice: " + Files.size(Path.of("welcome-robot.wav")) + " bytes");
    }
}
```

```text
سفارش شما ارسال شد -> 68394 bytes
پرداخت انجام شد -> 55164 bytes
سفارش شما ارسال شد -> 68394 bytes
خوش آمدید -> 35322 bytes
robot voice: 35322 bytes
```

- **Four threads, one object.** `TextToSpeech` holds no state, so one instance serves every thread. Each call runs its own Piper process.
- **A cache.** The repeated notification was spoken once; the second request got the same bytes. `computeIfAbsent` makes sure that two simultaneous requests for the same text do not both start Piper.
- **Effects.** Piper writes 16-bit mono WAV, the format that `VoiceEffect` accepts, so the two modules can be chained without any conversion.
- **Load once.** Create the `TextToSpeech` at start-up; a missing file shows up early, and keeping the object costs nothing.

## Appendix A. Quick reference

### A.1 Calls

| Call | Meaning |
|------|---------|
| `TextToSpeech.load(piperDirectory, modelName)` | Finds Piper and the voice; time limit 60 seconds |
| `TextToSpeech.load(piperDirectory, modelName, timeout)` | The same, with a time limit per call |
| `synthesize(text, outputAudioPath)` | Writes a WAV file safely; creates folders |
| `synthesize(text)` | Returns the WAV as bytes; leaves no file |
| `close()` | Does nothing |

### A.2 Layout

| Path | Meaning |
|------|---------|
| `<piperDirectory>/piper.exe` | Tried first |
| `<piperDirectory>/piper` | Tried when `piper.exe` is missing |
| `<piperDirectory>/model/<modelName>.onnx` | The voice file |

### A.3 Defaults

| Setting | Default |
|---------|---------|
| Time limit of one call | 60 seconds |
| Text encoding sent to Piper | UTF-8 |
| Output format | WAV, 16-bit mono, sample rate of the voice |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| Neither `piper.exe` nor `piper` in the folder | `TextToSpeechFailedException` caused by a `FileNotFoundException` |
| The voice file is missing | `TextToSpeechFailedException` caused by a `FileNotFoundException` |
| A zero or negative time limit | `IllegalArgumentException` at `load` |
| A `null` argument | `NullPointerException` naming the argument |
| Piper cannot be started | `TextToSpeechFailedException` caused by an `IOException` |
| Piper exits with a non-zero code | `TextToSpeechFailedException` with the code and Piper's output |
| Piper runs past the time limit | `TextToSpeechFailedException`; the process is killed |
| The text cannot be sent to Piper (and Piper exits normally) | `TextToSpeechFailedException` caused by an `IOException` |
| The output cannot be written or moved | `TextToSpeechFailedException` caused by an `IOException` |
| The thread is interrupted while waiting | `TextToSpeechFailedException`; the interrupt flag is kept; the process is killed |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Assuming `load` tested the voice.** It only checks that the files exist. Run one short synthesis at start-up to be sure.
- **Unlimited text from users.** Each call keeps a process busy. Limit the length and the number of calls at the same time.
- **A time limit shorter than the text needs.** Long texts on slow machines need a longer limit; the call is killed otherwise.
- **Forgetting that every call starts a process.** Many parallel calls use many processes and much memory. Limit the number of threads that speak at once.
- **Large outputs in memory.** The bytes form holds the whole WAV in memory. For very long audio, write to a file.
- **A wrong voice for the text.** A voice speaks one language. Text in another language comes out badly.
- **Expecting Persian text to be fixed up.** The text goes to Piper as written. Clean it (numbers, symbols, abbreviations) before speaking if the voice reads them badly.
- **Mixing up the sample rate.** The audio has the sample rate of the voice, such as 22050 Hz, which is not the 16000 Hz that the speech recognition module expects by default.

### C.2 Behavior worth knowing

- **Thread-safe.** The instance is immutable; each call has its own process and its own temporary file.
- **Safe writing.** The output is written to a temporary file next to the target and moved into place when complete.
- **The bytes form** uses the system temporary folder for a moment and deletes the file again.
- **Piper's messages are kept.** Standard output and errors of Piper are collected, and shown in the exception when Piper fails.
- **The process is always cleaned up.** A Piper process still running when a call ends, for any reason, is killed.
- **`close` is empty.** Closing a `TextToSpeech` has no effect and can be called any number of times.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Engine | Piper, installed separately |
| Voices | The `.onnx` files in the `model` folder |
| Output | WAV, one text per call |
| Streaming | None; the audio is complete when the call returns |
| Speed, pitch, volume | Not settable in this module |
| Time per call | 60 seconds unless changed |
| Parallel calls | Any number; each is a process |

### C.4 Tuning and practical advice

- **Keep texts short.** A sentence or two per call gives quick answers.
- **Cache repeated speech.** Notifications and greetings do not change; speak them once.
- **Warm the installation at start-up.** One short synthesis finds a damaged voice early.
- **Choose the voice by listening.** Try the voices of your language and pick by ear; higher quality voices are slower.
- **Limit concurrency.** A small thread pool for speech protects the machine.
- **Post-process with effects.** The WAV can go straight into the sound effect module.

## Appendix D. Message reference

### D.1 Loading

| Message | Exception | Cause |
|---------|-----------|-------|
| `java.io.FileNotFoundException: piper executable not found under: <folder> (expected piper.exe or piper)` | `TextToSpeechFailedException` | Neither executable exists |
| `java.io.FileNotFoundException: Voice model not found: <path>` | `TextToSpeechFailedException` | The `.onnx` file does not exist |
| `timeout must be positive, was <duration>` | `IllegalArgumentException` | A zero or negative time limit |

### D.2 Synthesizing

| Message | Exception | Cause |
|---------|-----------|-------|
| `piper exited with code %d: <output>` | `TextToSpeechFailedException` | Piper stopped with an error |
| `piper did not finish within %d seconds` | `TextToSpeechFailedException` | The time limit passed |
| the text of the cause | `TextToSpeechFailedException` | An `IOException`: starting Piper, sending the text, writing or moving the file; or an interrupt |

### D.3 Argument checks

| Message | Exception | Cause |
|---------|-----------|-------|
| `<name> must not be null` | `NullPointerException` | `piperDirectory`, `modelName`, `timeout`, `text` or `outputAudioPath` is `null` |
