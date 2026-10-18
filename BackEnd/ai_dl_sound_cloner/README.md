# ai.dl.sound.cloner

*J2OS Platform 2.4*

Many programs have to speak a lot of different sentences in one particular voice: announcements on a loudspeaker, answers of a voice assistant, the narration of a course. Recording every sentence by hand is slow, and every change of the text means another recording session. A program that can read any new sentence in that voice solves this. That is **voice cloning**: the program learns what a voice sounds like from a short recording, and then speaks new text in that voice.

**ai.dl.sound.cloner** (`org.j2os.platform.ai.dl.sound.cloner`) brings it to J2OS. It has one main class, `VoiceCloner`, with two ideas:

- **Enroll.** Give the program a short recording of a voice and a name. It keeps the recording.
- **Speak.** Give it that name and a text. It writes a WAV file in which the text is spoken with that voice.

Three things set it apart from the rest:

- **Everything runs on your own machine.** The recording and the text never leave it. There is no online service and no key.
- **A short recording is enough.** About 10 to 30 seconds of clean speech works.
- **Little to manage.** The library prepares its own private Python environment and downloads the voice conversion model by itself, the first time it is needed.

Everything in one small example:

```java
VoiceCloner cloner = VoiceCloner.create()
        .piperExecutable("piper/piper")                         // 1. a normal text-to-speech program
        .piperVoiceModel("piper/voice.onnx");
cloner.enroll("narrator", "my-voice.wav");                          // 2. teach it a voice from a short recording
cloner.speak("narrator", "Welcome!", "welcome.wav");                // 3. any text, spoken in that voice
```

A word about responsibility: clone only a voice whose owner has agreed. A cloned voice sounds very much like the real person, and using it without permission can hurt people and is illegal in many places.

Roughly, the guide goes like this: how the module works (chapter 1), the first cloned voice (2), a good recording (3), settings (4), mistakes (5), using it inside an application (6) and a complete small program (7). The appendices are meant for quick lookups.

## Contents

1. [How ai.dl.sound.cloner works](#1-how-aidlsoundcloner-works)
2. [The first cloned voice](#2-the-first-cloned-voice)
3. [A good recording](#3-a-good-recording)
4. [Settings](#4-settings)
5. [Mistakes](#5-mistakes)
6. [Using it inside an application](#6-using-it-inside-an-application)
7. [A realistic example](#7-a-realistic-example)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

None of the programs is a fragment: each has one `main` method. They expect the folder `piper` from chapter 1 next to the program and a recording named `my-voice.wav`.

---

## 1. How ai.dl.sound.cloner works

First, a few words that come up throughout the guide:

- **Text-to-speech (TTS).** A program that turns written text into spoken audio. The voice it uses is generic, not the voice of a particular person.
- **Piper.** A fast, free TTS program that runs offline. It is a separate download, not part of the library. The library starts it for you.
- **Piper voice.** A model file with the extension `.onnx`. It decides the language and the style of the generic speech. A Persian voice reads Persian; an English voice reads English.
- **WAV.** A simple audio file format. Both the recording you give and the audio you get back are WAV files.
- **Enrollment.** Saving a person's recording under a name, so the voice can be used later. The recording is called the *sample*.
- **Tone color conversion.** The cloning step itself. The "tone color" is what makes a voice recognizable: its warmth, its roughness, its pitch. The converter takes the generic speech and changes only this tone color to match the sample. The words and the rhythm stay the same. The library uses a model called OpenVoice for this.
- **Python environment.** The converter is written in Python. The library creates a private Python installation in its own folder, so it never touches the Python you already have.

So cloning is two steps, one after the other:

```text
text ──► Piper ──► neutral.wav ────────────────┐
                                               ├──► tone color converter ──► welcome.wav
narrator.wav (the enrolled sample) ────────────┘
```

The classes a program meets:

| Class | Role |
|-------|------|
| `VoiceCloner` | Enrolls voices and speaks text with them |
| `VoiceCloningException` | The one exception of the module; it is unchecked, so no `try` is forced on you |

Before the first program, download Piper for your system and one Piper voice, and arrange them like this:

```text
piper/
├── piper               (piper.exe on Windows)
└── voice.onnx          (any Piper voice; its language decides the language of the speech)
```

You also need Python 3 installed on the machine, because the library builds its private environment from it.

The three points to carry forward:

- **Nothing is ready at the start.** The first `speak` call prepares the Python environment and downloads the conversion model (about 900 MB). This can take several minutes, but it happens only once.
- **Piper decides the language.** The cloner changes only the tone of the voice. If the Piper voice speaks English, the result is English in the tone of the enrolled voice.
- **Voices are saved on disk.** After `enroll`, the voice is still there the next time the program starts.

## 2. The first cloned voice

`VoiceCloner.create()` starts the description. Two settings are required: where Piper is and which Piper voice to use. `enroll` saves a voice, `speak` uses it:

```java
import org.j2os.platform.ai.dl.sound.cloner.VoiceCloner;

public class C01Speak {
    public static void main(String[] args) {
        VoiceCloner cloner = VoiceCloner.create()
                .piperExecutable("piper/piper")
                .piperVoiceModel("piper/voice.onnx")
                .workspaceDirectory("ws");
        cloner.enroll("narrator", "my-voice.wav");
        System.out.println("Voice saved.");
        System.out.println("Written: " + cloner.speak("narrator", "Welcome to our service.", "welcome.wav"));
    }
}
```

The program prints:

```text
Voice saved.
[VoiceCloner] Checking the environment and the voice conversion model...
Written: /home/user/voice/welcome.wav
```

The line with `[VoiceCloner]` is a progress message. While the setup runs, `speak` also prints its steps. This is what the very first run looks like on a clean machine:

```text
Voice saved.
[VoiceCloner] Checking the environment and the voice conversion model...
[VoiceCloner] Creating the Python environment...
[VoiceCloner] Installing dependencies (large download, can take several minutes)...
[VoiceCloner] Environment is ready.
Written: /home/user/voice/welcome.wav
```

Read these lines as a checklist:

1. *Checking the environment* — the library looks for its private Python and tries to load the model.
2. *Creating the Python environment* and *Installing dependencies* — only when the check failed. This is the long part.
3. *Environment is ready* — the setup is finished and the real work starts.

After this, the setup is skipped. A later run prints only the checking line.

`speak` returns the absolute path of the WAV file, so the result can be passed on directly. Folders in the path are created when they do not exist.

> **Advice.** Run one short `speak` when the application starts, for example in a startup step. Then the long setup happens while nobody waits for an answer.

## 3. A good recording

The quality of the clone depends mostly on the sample. A few simple rules help a lot:

- **10 to 30 seconds.** Shorter samples sound less like the person; longer ones do not help much.
- **One speaker only.** If two people talk, the voice becomes a mixture.
- **A quiet room.** Background music, a fan or an echo is copied together with the voice.
- **Natural speech.** Read a paragraph in the normal way. Do not whisper, shout or sing.
- **A WAV file.** `enroll` accepts only files that end in `.wav`. Convert other formats first.

`enroll` also checks the **name**. A voice name has 1 to 64 characters: letters, digits, `_` and `-`. A space or a dot is not allowed. The name becomes a file name inside the workspace, which is why it is strict.

Two more facts about `enroll`:

- It **copies** the recording. You may delete or move the original afterwards.
- Enrolling an existing name **replaces** the old sample. This is the way to improve a voice: record again and enroll again under the same name.

`enroll` returns the same `VoiceCloner`, so several voices can be enrolled in one chain:

```java
cloner.enroll("narrator", "narrator.wav").enroll("guest", "guest.wav");
```

## 4. Settings

Besides the two required ones, three optional settings exist:

```java
VoiceCloner cloner = VoiceCloner.create()
        .piperExecutable("piper/piper")
        .piperVoiceModel("piper/voice.onnx")
        .workspaceDirectory("D:/voices-workspace")      // default: ~/.j2os/voice-cloner
        .timeoutMinutes(5)                              // limit for one speak call, default 10
        .progressListener(line -> log.info(line));      // default: System.out.println; null = silent
```

- **`workspaceDirectory`.** The folder where the library keeps everything it creates. By default it is `.j2os/voice-cloner` inside the home folder of the user. Use another folder if the home disk is small, because the environment and the model need a few gigabytes.
- **`timeoutMinutes`.** The time limit for **one** `speak` call. The default is 10 minutes. A call that takes longer is stopped. The setup of the first call has its own, longer limit (60 minutes), so a slow download is not cut off by this setting.
- **`progressListener`.** The receiver of the progress lines. The default prints them to the console. Pass your own function to write them into a log, or `null` to stay silent.

This is what the workspace folder contains:

```text
ws/
├── voices/
│   └── narrator.wav             (your enrolled samples, one file per voice)
├── python-environment/          (a private Python with the packages that were installed)
├── model-cache/                 (the downloaded voice conversion model)
└── voice_conversion.py          (the small script that the library writes for you)
```

You can delete `model-cache` and `python-environment` at any time. The library rebuilds them at the next `speak`. Do not delete `voices` unless you still have the original samples.

## 5. Mistakes

Most mistakes are caught before any heavy work starts, and each gives a clear message. The program below makes six of them on purpose:

```java
import org.j2os.platform.ai.dl.sound.cloner.VoiceCloner;
import org.j2os.platform.ai.dl.sound.cloner.exception.VoiceCloningException;

public class C02Mistakes {
    static void attempt(String title, Runnable action) {
        try {
            action.run();
        } catch (VoiceCloningException problem) {
            System.out.println(title + "\n  " + problem.getMessage());
        }
    }

    public static void main(String[] args) {
        VoiceCloner cloner = VoiceCloner.create()
                .piperExecutable("piper/piper")
                .piperVoiceModel("piper/voice.onnx")
                .workspaceDirectory("ws")
                .progressListener(null);
        attempt("Name with a space", () -> cloner.enroll("my voice", "my-voice.wav"));
        attempt("Sample is not a WAV file", () -> cloner.enroll("narrator", "notes.txt"));
        attempt("Empty text", () -> cloner.speak("narrator", " ", "x.wav"));
        attempt("Voice never enrolled", () -> cloner.speak("guest", "Hello", "x.wav"));
        attempt("Piper not set", () -> VoiceCloner.create().speak("narrator", "Hello", "x.wav"));
        attempt("Piper path wrong", () -> VoiceCloner.create().piperExecutable("nope/piper")
                .piperVoiceModel("piper/voice.onnx").speak("narrator", "Hello", "x.wav"));
    }
}
```

```text
Name with a space
  Voice name must be 1-64 characters of letters, digits, _ or -
Sample is not a WAV file
  Sample must be an existing .wav file: notes.txt
Empty text
  Text must not be empty
Voice never enrolled
  Voice is not enrolled: guest
Piper not set
  piperExecutable and piperVoiceModel are required
Piper path wrong
  Piper executable not found: nope/piper
```

Every message tells what to fix. The checks run in this order: the voice name, the two Piper settings, the Piper files, the text, and then whether the voice is enrolled. Only after all of them pass does the library start the setup.

`VoiceCloningException` is unchecked. In a small tool it is fine to let it end the program. In an application, catch it where the request is handled and show the message to the user or write it to the log.

## 6. Using it inside an application

A real program rarely calls `speak` once and exits. These points make it safe:

- **Create one `VoiceCloner` and share it.** The class is thread-safe. Set everything (`piperExecutable`, `piperVoiceModel`, and so on) **before** you give it to other threads.
- **Limit the parallel `speak` calls.** Each call starts two programs and uses a lot of processor time. A small pool of two or three threads is enough for most applications. More threads only make every call slower.
- **Do not make a user wait for the first call.** Prepare the environment at start-up, as the advice in chapter 2 says.
- **Keep generated files.** The same text in the same voice always costs the same time. When the same text is needed again, play the stored file instead of calling `speak` again.
- **Writing the output is safe.** The result is written to a private file first and moved to your path at the end. Two calls with the same output path never mix, and a failed call never leaves a broken file.

## 7. A realistic example

The task: three announcements in one voice. The program has one recording, `my-voice.wav`. It enrolls the voice once and creates one file per announcement:

```java
import java.util.LinkedHashMap;
import java.util.Map;
import org.j2os.platform.ai.dl.sound.cloner.VoiceCloner;
import org.j2os.platform.ai.dl.sound.cloner.exception.VoiceCloningException;

public class C03Announcements {
    public static void main(String[] args) {
        Map<String, String> announcements = new LinkedHashMap<>();
        announcements.put("opening", "The service is now open. Welcome!");
        announcements.put("closing", "The office closes in fifteen minutes.");
        announcements.put("sale", "Today every item costs half price.");

        VoiceCloner cloner = VoiceCloner.create()
                .piperExecutable("piper/piper")
                .piperVoiceModel("piper/voice.onnx")
                .workspaceDirectory("ws")
                .progressListener(null);
        try {
            cloner.enroll("narrator", "my-voice.wav");
            for (Map.Entry<String, String> item : announcements.entrySet()) {
                cloner.speak("narrator", item.getValue(), "announcements/" + item.getKey() + ".wav");
                System.out.println("Ready: announcements/" + item.getKey() + ".wav");
            }
        } catch (VoiceCloningException problem) {
            System.err.println("Could not create the announcements: " + problem.getMessage());
        }
    }
}
```

```text
Ready: announcements/opening.wav
Ready: announcements/closing.wav
Ready: announcements/sale.wav
```

Any audio player can now use `announcements/opening.wav`, `closing.wav` and `sale.wav`. When the text of an announcement changes, run the program again. Nobody has to record anything.

Some ideas for extending it: read the announcements from a file instead of a `Map`; create one file per language by using a different Piper voice for each; let every user enroll their own voice under a different name.

In code, it comes to this:

- Enroll a clean 10 to 30 second recording under a simple name.
- `speak(name, text, path)` writes the WAV and returns its path.
- The first call is slow because of the one-time setup; later calls are fast.
- Share one `VoiceCloner` and limit the calls that run at the same time.

## Appendix A. Quick reference

### A.1 Calls

| Call | Meaning |
|------|---------|
| `VoiceCloner.create()` | Starts a new cloner with the default settings |
| `piperExecutable(path)` | The Piper program; required |
| `piperVoiceModel(path)` | The Piper voice file (`.onnx`); required |
| `workspaceDirectory(path)` | Where everything is kept; default `.j2os/voice-cloner` in the home folder |
| `timeoutMinutes(minutes)` | Time limit of one `speak` call; default 10 |
| `progressListener(consumer)` | Receives the progress lines; `null` means silent; default is the console |
| `enroll(voiceName, sampleWavPath)` | Saves a voice from a `.wav` sample; returns the same cloner |
| `speak(voiceName, text, outputWavPath)` | Writes the text as a WAV in that voice; returns the absolute path |

### A.2 Defaults

| Setting | Default |
|---------|---------|
| Workspace | `.j2os/voice-cloner` in the home folder of the user |
| Time limit of one `speak` | 10 minutes |
| Time limit of the one-time setup | 60 minutes |
| Progress lines | Printed to the console |
| Voice name | 1 to 64 characters: letters, digits, `_`, `-` |
| Recommended sample | One speaker, 10 to 30 seconds, WAV |

## Appendix B. Failure handling

All failures are a `VoiceCloningException`, which is unchecked.

| Situation | What happens |
|-----------|--------------|
| The voice name is empty, too long or has other characters | The exception is thrown at once, before anything is touched |
| The sample does not exist or does not end in `.wav` | The exception is thrown by `enroll` |
| The sample cannot be copied | The exception is thrown by `enroll`, with the cause |
| A Piper setting is missing | The exception is thrown by `speak` |
| The Piper program or the voice file does not exist | The exception is thrown by `speak` |
| The text is `null` or blank | The exception is thrown by `speak` |
| The voice was never enrolled | The exception is thrown by `speak` |
| The setup fails (no internet, no Python, not enough disk) | The exception carries the output of the failing command |
| A command runs longer than its time limit | The command is stopped and the exception says which one |
| The thread is interrupted while waiting | The exception is thrown; the interrupt flag stays set |
| Any step fails during `speak` | The temporary files are deleted; the output path is untouched |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Cloning a voice without permission.** Always ask the owner first.
- **A noisy or very short sample.** The result sounds unlike the person. Record again.
- **Expecting a new language.** The cloner changes the tone, not the language or the accent rules. Choose the Piper voice of the language you need.
- **A first call inside a user request.** It can take many minutes. Prepare the environment at start-up.
- **Too many parallel calls.** They compete for the processor and all become slow.
- **Deleting the `voices` folder.** The enrolled voices are gone. Keep the original samples.
- **Changing a setting after sharing the object.** Configure first, then share.

### C.2 Behavior worth knowing

- The setup runs once per process and is protected by a lock: a second thread that calls `speak` during the setup waits for it.
- The environment is trusted only after the conversion model really loads in it. A broken model cache is cleared and downloaded again automatically.
- The library sets the variables `PYTHONUTF8`, `TQDM_MININTERVAL` and `TTS_HOME` for its Python process. Your own environment is not changed.
- Temporary files are deleted in every case, success or failure.

### C.3 Limits

- A voice needs a WAV sample; other formats must be converted first.
- Voice names are limited to letters, digits, `_` and `-`, up to 64 characters.
- The model download is about 900 MB and the environment needs several more gigabytes of disk.
- No graphics card is needed, but the conversion is slower on weak processors.

### C.4 Tuning and practical advice

- Put the workspace on a fast disk with enough free space.
- Use `timeoutMinutes` for long texts on slow machines; split very long texts into sentences and join the files if you need to.
- Send the progress lines to your log, so a long first setup does not look like a frozen program.
- Cache the results for texts that repeat.
- Test a new voice with a short sentence before you generate many files.

## Appendix D. Message reference

### D.1 Input checks

| Message | Cause |
|---------|-------|
| `Voice name must be 1-64 characters of letters, digits, _ or -` | The name is empty, too long or has other characters |
| `Sample must be an existing .wav file: <path>` | The file does not exist or does not end in `.wav` |
| `Could not enroll voice: <name>` | The sample could not be copied into the workspace |
| `piperExecutable and piperVoiceModel are required` | One of the two required settings is missing |
| `Piper executable not found: <path>` | The Piper program does not exist |
| `Piper voice model not found: <path>` | The `.onnx` file does not exist |
| `Text must not be empty` | The text is `null` or blank |
| `Voice is not enrolled: <name>` | `enroll` was never called for this name |

### D.2 Setup and speaking

| Message | Cause |
|---------|-------|
| `Could not prepare the workspace` | The workspace folder cannot be created or written |
| `Python environment is still not usable after installation:` followed by output | The installation ended but the model still does not load |
| `Command failed (<code>): <command>` followed by output | A started program ended with an error code |
| `Command timed out: <command>` | The command ran past its time limit |
| `Could not run command: <command>` | The program cannot be started, for example Python is not installed |
| `Interrupted while running: <command>` | The thread was interrupted |
| `Could not generate speech` | A file could not be created or moved |
| `Could not clear directory: <folder>` | The broken model cache could not be removed |

### D.3 Progress lines

| Line | Meaning |
|------|---------|
| `[VoiceCloner] Checking the environment and the voice conversion model...` | The first step of every setup |
| `[VoiceCloner] Creating the Python environment...` | The private Python is being created |
| `[VoiceCloner] Installing dependencies (large download, can take several minutes)...` | The packages are being installed |
| `[VoiceCloner] The model could not be loaded; clearing the model cache and downloading again...` | The cache was damaged and is rebuilt |
| `[VoiceCloner] Environment is ready.` | The setup is finished |
