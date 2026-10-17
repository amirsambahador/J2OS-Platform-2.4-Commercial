# ai.dl.sound.effect

*J2OS Platform 2.4*

A voice message, a game character, a podcast intro, a prank in a chat room: sometimes a recorded voice should sound different. Higher, deeper, metallic, as if spoken in a hall. This is done not with intelligence but with arithmetic on the sound itself, which is **digital signal processing**.

**ai.dl.sound.effect** (`org.j2os.platform.ai.dl.sound.effect`) brings four such effects to J2OS. It has one class, `VoiceEffect`, with one method and two forms: *take a WAV, apply an effect, give back a WAV*. Three properties set it apart:

- **No model, no server.** The effects are small formulas in plain Java. Nothing is downloaded, installed or started.
- **File in, file out, or bytes out.** The result can be written to a path (safely, even over the input) or returned as bytes for a web response.
- **No false promises.** The effects change tone and pitch. They do **not** turn one speaker into another and do not hide who is speaking.

The whole library in a few lines:

```java
VoiceEffect.apply("voice.wav", "robot.wav", Effect.ROBOT);                   // 1. file to file

byte[] echo = VoiceEffect.apply(new File("voice.wav"), Effect.ECHO);        // 2. file to bytes, nothing written
```

This guide goes from the first effect (chapter 2) through the four effects (3), files and bytes (4) and getting the input right (5) to a complete small workflow (6). The appendices work as a reference.

## Contents

1. [How ai.dl.sound.effect works](#1-how-aidlsoundeffect-works)
2. [The first effect](#2-the-first-effect)
3. [The four effects](#3-the-four-effects)
4. [Files and bytes](#4-files-and-bytes)
5. [Getting the input right](#5-getting-the-input-right)
6. [Putting it together](#6-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The programs of the guide are complete: each has one `main` method. They make their own small recordings, tones of a known pitch or silence, in the working folder, so no audio file is needed to try them.

---

## 1. How ai.dl.sound.effect works

First, a few terms, because they come up throughout the guide:

- **WAV.** A simple audio file: a small header and then the raw sound.
- **Sample.** One measurement of the sound wave. A recording is a long list of them.
- **Sample rate.** How many samples there are per second, such as 8000 or 16000. It decides the length of the recording.
- **16-bit mono PCM.** The one format the module accepts: one channel, each sample a 16-bit whole number.
- **Pitch.** How high a voice sounds. It follows the speed of the samples: played faster, the voice goes higher.

The classes a program meets:

| Class | Role |
|-------|------|
| `VoiceEffect` | Applies an effect to a WAV file. Two `apply` forms |
| `Effect` | The four effects: `HIGHER_PITCH`, `LOWER_PITCH`, `ROBOT`, `ECHO` |
| `VoiceEffectFailedException` | The one checked exception of the module |

Keep these three points in mind:

- **The input must be 16-bit mono WAV.** Stereo, 8-bit and other formats are refused with a clear message. Chapter 5 shows how to convert.
- **The result is always a 16-bit mono WAV** with the same sample rate as the input and a plain 44-byte header. Other information stored in the input file, such as tags, is not carried over.
- **The module is stateless.** `VoiceEffect` has only static methods and keeps nothing between calls, so many threads can use it at once.

## 2. The first effect

`VoiceEffect.apply` reads a WAV, applies an effect and writes the result:

```java
import java.io.*;
import java.nio.file.*;
import javax.sound.sampled.*;
import org.j2os.platform.ai.dl.sound.effect.Effect;
import org.j2os.platform.ai.dl.sound.effect.VoiceEffect;

public class First {
    public static void main(String[] args) throws Exception {
        // a one-second 440 Hz tone: 16-bit, mono, 8000 samples per second
        byte[] pcm = new byte[16000];
        for (int i = 0; i < 8000; i++) {
            short v = (short) (12000 * Math.sin(2 * Math.PI * 440 * i / 8000.0));
            pcm[2 * i] = (byte) v;
            pcm[2 * i + 1] = (byte) (v >> 8);
        }
        AudioFormat format = new AudioFormat(8000, 16, 1, true, false);
        AudioSystem.write(new AudioInputStream(new ByteArrayInputStream(pcm), format, 8000),
                AudioFileFormat.Type.WAVE, new File("voice.wav"));

        VoiceEffect.apply("voice.wav", "out/robot.wav", Effect.ROBOT);

        System.out.println("input : " + Files.size(Path.of("voice.wav")) + " bytes");
        System.out.println("output: " + Files.size(Path.of("out/robot.wav")) + " bytes");
    }
}
```

```text
input : 16044 bytes
output: 16044 bytes
```

- **The input.** The program first makes a one-second tone, 8000 samples of 16 bits, which is 16000 bytes of sound plus a 44-byte header.
- **`apply(input, output, effect)`.** Takes two paths and an effect. If the folder `out` does not exist, it is created.
- **The size.** The robot voice has the same length as the original, so the file has the same size.

## 3. The four effects

The effects are simple. This program applies each to a 440 Hz tone and measures what came out:

```java
import java.io.*;
import java.nio.*;
import javax.sound.sampled.*;
import org.j2os.platform.ai.dl.sound.effect.Effect;
import org.j2os.platform.ai.dl.sound.effect.VoiceEffect;

public class Effects {
    public static void main(String[] args) throws Exception {
        byte[] pcm = new byte[16000];
        for (int i = 0; i < 8000; i++) {
            short v = (short) (12000 * Math.sin(2 * Math.PI * 440 * i / 8000.0));
            pcm[2 * i] = (byte) v;
            pcm[2 * i + 1] = (byte) (v >> 8);
        }
        AudioSystem.write(new AudioInputStream(new ByteArrayInputStream(pcm), new AudioFormat(8000, 16, 1, true, false), 8000),
                AudioFileFormat.Type.WAVE, new File("voice.wav"));

        System.out.println("effect        seconds  peak   pitch");
        for (Effect effect : Effect.values()) {
            byte[] wav = VoiceEffect.apply(new File("voice.wav"), effect);       // WAV bytes, no file written
            ByteBuffer b = ByteBuffer.wrap(wav, 44, wav.length - 44).order(ByteOrder.LITTLE_ENDIAN);
            int n = (wav.length - 44) / 2, peak = 0, crossings = 0;
            short previous = 0;
            for (int i = 0; i < n; i++) {
                short s = b.getShort();
                peak = Math.max(peak, Math.abs(s));
                if (previous < 0 && s >= 0) crossings++;
                previous = s;
            }
            double seconds = n / 8000.0;
            System.out.printf("%-12s  %5.2f  %6d  %4.0f Hz%n", effect, seconds, peak, crossings / seconds);
        }
    }
}
```

```text
effect        seconds  peak   pitch
HIGHER_PITCH   0.67   12000   659 Hz
LOWER_PITCH    1.33   12000   329 Hz
ROBOT          1.00   11970   460 Hz
ECHO           1.00   18000   439 Hz
```

- **`HIGHER_PITCH`.** Plays the sound 1.5 times faster. The recording becomes shorter (0.67 s instead of 1 s) and the pitch goes up by the same factor: 440 Hz becomes 660 Hz.
- **`LOWER_PITCH`.** Plays the sound at 0.75 of the speed. The recording becomes longer (1.33 s) and the pitch goes down to 330 Hz. Speed and pitch cannot be separated in this method.
- **`ROBOT`.** Multiplies the sound by a slow 30 Hz wave, so the volume rises and falls 60 times a second. The length is unchanged, the loudness is a little lower, and the voice gets its metallic flutter. The simple pitch counter in the program is confused by this flutter, so the `460 Hz` is not a pitch change.
- **`ECHO`.** Adds a copy of the sound, 0.2 seconds late and half as loud. The length is unchanged, so the very end of the echo, the last 0.2 seconds of the copy, is cut. The peak grows to 18000 here because the 0.2-second echo of a 440 Hz tone is exactly in phase with the original, so the amplitudes add up. If the sum goes beyond the 16-bit range, it is clipped instead of wrapping around.
- **Fixed settings.** The factors above are built in. `apply` has no parameters to change them.

## 4. Files and bytes

Both forms of `apply` are safe to use on real recordings:

```java
import java.io.*;
import java.nio.file.*;
import javax.sound.sampled.*;
import org.j2os.platform.ai.dl.sound.effect.Effect;
import org.j2os.platform.ai.dl.sound.effect.VoiceEffect;

public class InPlace {
    public static void main(String[] args) throws Exception {
        byte[] pcm = new byte[16000];                                   // one second of silence is enough here
        AudioSystem.write(new AudioInputStream(new ByteArrayInputStream(pcm), new AudioFormat(8000, 16, 1, true, false), 8000),
                AudioFileFormat.Type.WAVE, new File("voice.wav"));
        System.out.println("original      " + Files.size(Path.of("voice.wav")));

        // the output may be the input: the new file replaces the old one in a single step
        VoiceEffect.apply("voice.wav", "voice.wav", Effect.LOWER_PITCH);
        System.out.println("lower once    " + Files.size(Path.of("voice.wav")));

        // so effects can be stacked
        VoiceEffect.apply("voice.wav", "voice.wav", Effect.LOWER_PITCH);
        System.out.println("lower twice   " + Files.size(Path.of("voice.wav")));

        // missing folders are created
        VoiceEffect.apply("voice.wav", "out/deep/echo.wav", Effect.ECHO);
        try (var names = Files.list(Path.of("out/deep"))) {
            names.forEach(p -> System.out.println("in out/deep   " + p.getFileName()));
        }
    }
}
```

```text
original      16044
lower once    21376
lower twice   28486
in out/deep   echo.wav
```

- **The output can be the input.** The result is first written to a temporary file in the target folder and then moved over the old file in one step. A reader never sees a half-written file, and if processing fails the original stays untouched.
- **Stacking effects.** The second `LOWER_PITCH` works on the output of the first one: each pass makes the recording a third longer.
- **Folders.** Missing folders on the output path are created, and the temporary file is gone afterwards: only `echo.wav` is in the folder.
- **The original.** Because writing over the input is allowed, do it only when the original is not needed. To keep it, write to a new path:

```java
// keep the original recording: write the effect to a new file
VoiceEffect.apply("uploads/msg-17.wav", "uploads/msg-17-robot.wav", Effect.ROBOT);
```

- **Bytes.** `apply(File, Effect)` returns the WAV as a `byte[]` and writes nothing. It is the form for a web response or a message that is sent straight on, as in chapter 6.

## 5. Getting the input right

The module is strict about its input, and the messages say what is wrong:

```java
import java.io.*;
import java.nio.file.*;
import javax.sound.sampled.*;
import org.j2os.platform.ai.dl.sound.effect.Effect;
import org.j2os.platform.ai.dl.sound.effect.VoiceEffect;

public class Checks {
    interface Step { void run() throws Exception; }

    static void attempt(String label, Step step) {
        try {
            step.run();
            System.out.println(label + " -> ok");
        } catch (Exception e) {
            System.out.println(label + " -> " + e.getClass().getSimpleName() + ": "
                    + e.getMessage().replace(System.getProperty("user.dir"), "."));
        }
    }

    public static void main(String[] args) throws Exception {
        Files.writeString(Path.of("notes.wav"), "this is not audio at all");
        AudioSystem.write(new AudioInputStream(new ByteArrayInputStream(new byte[32000]),
                new AudioFormat(8000, 16, 2, true, false), 8000), AudioFileFormat.Type.WAVE, new File("stereo.wav"));

        attempt("missing file", () -> VoiceEffect.apply("nope.wav", "o.wav", Effect.ECHO));
        attempt("not a WAV", () -> VoiceEffect.apply("notes.wav", "o.wav", Effect.ECHO));
        attempt("stereo", () -> VoiceEffect.apply("stereo.wav", "o.wav", Effect.ECHO));
        attempt("no effect", () -> VoiceEffect.apply("stereo.wav", "o.wav", null));
    }
}
```

```text
missing file -> VoiceEffectFailedException: java.io.FileNotFoundException: Audio file does not exist: ./nope.wav
not a WAV -> VoiceEffectFailedException: Not a valid RIFF/WAVE file: ./notes.wav
stereo -> VoiceEffectFailedException: Only 16-bit mono WAV is supported (got 2 channel(s), 16-bit)
no effect -> NullPointerException: effect must not be null
```

- **A missing file.** The path is not a file.
- **Not a WAV.** The file does not start with the `RIFF` and `WAVE` marks.
- **Stereo, or another bit depth.** The message states what the file has. The sound is not converted for you.
- **No effect.** A `null` argument is a `NullPointerException` that names the argument.
- **Other damage.** A file with no `data` part, a damaged `fmt` part, or a `fmt` part after the `data` part is refused as "Malformed WAV file". Unknown extra parts, such as tags between the other parts, are skipped.

Recordings from a microphone or a phone are often stereo or have another depth. Convert them first with the Java Sound API; the sample rate can stay as it is:

```java
// the module reads 16-bit mono only: convert other recordings first (same sample rate here)
AudioFormat target = new AudioFormat(8000, 16, 1, true, false);
try (AudioInputStream in = AudioSystem.getAudioInputStream(new File("stereo.wav"));
     AudioInputStream mono = AudioSystem.getAudioInputStream(target, in)) {
    AudioSystem.write(mono, AudioFileFormat.Type.WAVE, new File("mono.wav"));
}
```

## 6. Putting it together

A chat service lets users send a voice message with an effect. The effect arrives as a name; the answer is the processed audio. Several messages are handled at the same time:

```java
import java.io.*;
import java.util.*;
import java.util.concurrent.*;
import javax.sound.sampled.*;
import org.j2os.platform.ai.dl.sound.effect.Effect;
import org.j2os.platform.ai.dl.sound.effect.VoiceEffect;

public class Workflow {
    // a message arrives with a chosen effect name; the answer is the processed audio
    static byte[] process(File upload, String effectName) throws Exception {
        Effect effect = Effect.valueOf(effectName.trim().toUpperCase().replace(' ', '_'));
        return VoiceEffect.apply(upload, effect);
    }

    public static void main(String[] args) throws Exception {
        AudioSystem.write(new AudioInputStream(new ByteArrayInputStream(new byte[16000]),
                new AudioFormat(8000, 16, 1, true, false), 8000), AudioFileFormat.Type.WAVE, new File("upload.wav"));
        File upload = new File("upload.wav");

        List<String> requests = List.of("robot", "higher pitch", "echo", "lower pitch", "whisper");
        ExecutorService pool = Executors.newFixedThreadPool(4);
        Map<String, Future<byte[]>> results = new LinkedHashMap<>();
        for (String r : requests) results.put(r, pool.submit(() -> process(upload, r)));

        for (var e : results.entrySet()) {
            try {
                System.out.println(e.getKey() + " -> " + e.getValue().get().length + " bytes");
            } catch (ExecutionException ex) {
                System.out.println(e.getKey() + " -> " + ex.getCause().getClass().getSimpleName() + ": " + ex.getCause().getMessage());
            }
        }
        pool.shutdown();
    }
}
```

```text
robot -> 16044 bytes
higher pitch -> 10710 bytes
echo -> 16044 bytes
lower pitch -> 21376 bytes
whisper -> IllegalArgumentException: No enum constant org.j2os.platform.ai.dl.sound.effect.Effect.WHISPER
```

- **Names to effects.** `Effect.valueOf` turns the name into an effect after the program cleaned it. A name that does not exist, such as `whisper`, fails with an `IllegalArgumentException` that only that request sees:

```java
// from a request parameter such as "higher_pitch" to an effect
Effect effect = Effect.valueOf(name.trim().toUpperCase());
```

- **Bytes out.** The service never writes a file; each request returns the sound ready to send.
- **Four threads.** No lock is needed: the module keeps no state.
- **Sizes.** The sizes follow chapter 3: `higher pitch` is shorter than the original, `lower pitch` longer, and `robot` and `echo` unchanged.
- **Large uploads.** The module reads the whole file into memory. Limit the size of uploads in the service.

## Appendix A. Quick reference

### A.1 Calls

| Call | Meaning |
|------|---------|
| `VoiceEffect.apply(inputPath, outputPath, effect)` | Reads a WAV, writes the result safely. The output may equal the input |
| `VoiceEffect.apply(inputFile, effect)` | Returns the result as WAV bytes. Writes nothing |

Both throw `VoiceEffectFailedException` for a bad file and `NullPointerException` for a `null` argument.

### A.2 Effects

| Effect | What happens | Length of the result |
|--------|--------------|----------------------|
| `HIGHER_PITCH` | 1.5 times faster, higher voice | About 2/3 of the original |
| `LOWER_PITCH` | 0.75 of the speed, deeper voice | About 4/3 of the original |
| `ROBOT` | Multiplied by a 30 Hz wave, metallic flutter | The same |
| `ECHO` | A copy 0.2 s late and half as loud is added | The same (the echo tail is cut) |

### A.3 Accepted input

| Property | Requirement |
|----------|-------------|
| Container | WAV with `RIFF` and `WAVE` marks |
| Channels | 1 |
| Bits per sample | 16 |
| Sample rate | Any |
| Order of parts | `fmt ` before `data`; other parts are skipped |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| The input path is not a file | `VoiceEffectFailedException` caused by a `FileNotFoundException` |
| The file is not RIFF/WAVE | `VoiceEffectFailedException` |
| No `data` part, no `fmt ` part before `data`, or a damaged `fmt ` part | `VoiceEffectFailedException` |
| More than one channel, or not 16-bit | `VoiceEffectFailedException` stating what was found |
| The file cannot be read or the output cannot be written | `VoiceEffectFailedException` caused by the `IOException` |
| A `null` path, file or effect | `NullPointerException` naming the argument |
| An empty recording | Accepted; the result has a header and no sound |
| A failure while writing | The temporary file is removed and the old output stays |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Believing it hides the speaker.** Pitch and tone change, but the way a person speaks stays recognizable. Do not use these effects for anonymity.
- **Stereo recordings.** Phones and browsers often record stereo or 8-bit audio. Check and convert before calling.
- **Writing over the original.** Allowed, and then the original is gone. Keep the original when it may be needed.
- **Stacking pitch effects.** Each pass changes the length. Two `HIGHER_PITCH` passes give 2.25 times the speed.
- **Expecting a bigger echo.** The echo tail is cut at the end of the recording. Add a second of silence to the input if the tail matters.
- **Loud sounds.** `ECHO` adds up to half of the signal again; the sum is clipped at the largest 16-bit value, which sounds harsh. Lower the volume first when the recording is already loud.
- **Large files.** The whole recording is held in memory, and the result is built next to it.

### C.2 Behavior worth knowing

- **The sample rate is kept.** Only the number of samples changes, so the recording gets shorter or longer.
- **Linear interpolation.** The speed change estimates samples between two neighbors. It is simple and fast, with a slight roughness at high factors.
- **The header is rebuilt.** The result has the plain 44-byte header. Tags and other extra parts of the input are not kept.
- **Odd data.** A trailing half sample is ignored, and a `data` part that claims more than the file holds is read up to the end of the file.
- **Thread-safe.** Static methods, no shared state.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Input format | 16-bit mono WAV |
| Output format | 16-bit mono WAV |
| Effects | Four, with fixed settings |
| Speed and pitch | Changed together |
| Echo | One repeat, 0.2 s, half volume |
| Size | Whole file in memory |
| Speaker identity | Not changed |

### C.4 Tuning and practical advice

- **Process once, store the result.** An effect always gives the same output for the same input, so keep the result instead of recomputing it.
- **Limit the upload size.** Memory use grows with the recording.
- **Convert early.** Convert recordings to 16-bit mono when they are saved, not when an effect is asked for.
- **Pick the effect by name safely.** Catch the `IllegalArgumentException` of `valueOf`, or offer a fixed list.
- **Test by ear.** Try each effect on a real voice before offering it to users; short, clear recordings suit them best.

## Appendix D. Message reference

### D.1 Reading

| Message | Exception | Cause |
|---------|-----------|-------|
| `java.io.FileNotFoundException: Audio file does not exist: <path>` | `VoiceEffectFailedException` | The input path is not a file |
| `Not a valid RIFF/WAVE file: <path>` | `VoiceEffectFailedException` | The file has no `RIFF`/`WAVE` header |
| `Malformed WAV file: no 'data' chunk found` | `VoiceEffectFailedException` | The file ends before a `data` part |
| `Malformed WAV file: invalid 'fmt ' chunk size %d` | `VoiceEffectFailedException` | The `fmt ` part is smaller than 16 or larger than 1024 bytes |
| `Malformed WAV file: truncated 'fmt ' chunk` | `VoiceEffectFailedException` | The file ends inside the `fmt ` part |
| `Malformed WAV file: no 'fmt ' chunk found before 'data'` | `VoiceEffectFailedException` | The `data` part comes first |
| `Only 16-bit mono WAV is supported (got %d channel(s), %d-bit)` | `VoiceEffectFailedException` | Wrong channels or bit depth |
| the text of the cause | `VoiceEffectFailedException` | An `IOException` while reading or writing |

### D.2 Argument checks

| Message | Exception | Cause |
|---------|-----------|-------|
| `<name> must not be null` | `NullPointerException` | `inputWavPath`, `outputWavPath`, `effect` or `inputWavFile` is `null` |
