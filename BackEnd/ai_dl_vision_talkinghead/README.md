# ai.dl.vision.talkinghead

*J2OS Platform 2.4*

A lesson video needs a presenter, a welcome screen needs a friendly face, a game needs a character that speaks its lines. In each case you have a voice and need a face: you want a **video in which the face talks**.

**ai.dl.vision.talkinghead** (`org.j2os.platform.ai.dl.vision.talkinghead`) brings it to J2OS. You give it a face and a WAV file with speech, and it writes an MP4 video. The mouth moves with the words, the eyes blink and the head sways a little. The face can be one of two kinds:

- **An avatar.** A cartoon face that the library draws itself. There are six ready-made people and you can change their hair, skin, glasses and more.
- **A photo.** A real portrait that you give. The library finds the face in the picture and animates it.

Three properties set the module apart:

- **Everything runs on the processor.** No graphics card and no online service are needed. The picture and the sound stay on your machine.
- **One class does the work.** `JTalkingHead` starts videos, and each video is a *job* that you can follow, wait for or cancel.
- **It protects your server.** Jobs wait in a queue of limited size, and only a limited number run at the same time.

The whole library in a few lines:

```java
try (JTalkingHead head = JTalkingHead.builder().workingDirectory(Path.of("work")).build()) {
    Path video = head.fromAvatar(AnimatedAvatar.preset(AvatarPreset.SARA))   // 1. choose a face
            .audio(Path.of("hello.wav"))                                     // 2. give it a voice
            .output(Path.of("hello.mp4"))
            .generate();                                                     // 3. get a video
}
```

This guide goes step by step: how the module works (chapter 1), the first video (2), choosing a face (3), video settings (4), a talking photo (5), jobs and progress (6), many videos at once (7), mistakes (8), a web service (9) and a complete small program (10). The appendices are for quick reference.

## Contents

1. [How ai.dl.vision.talkinghead works](#1-how-aidlvisiontalkinghead-works)
2. [The first video](#2-the-first-video)
3. [Choosing a face](#3-choosing-a-face)
4. [Video settings](#4-video-settings)
5. [A talking photo](#5-a-talking-photo)
6. [Jobs and progress](#6-jobs-and-progress)
7. [Many videos at once](#7-many-videos-at-once)
8. [Mistakes](#8-mistakes)
9. [A web service](#9-a-web-service)
10. [Putting it together](#10-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The programs of the guide are complete: each has one `main` method. They expect a folder `assets` with a speech file `hello.wav` and, for chapter 5, a portrait photo. Videos go to a folder named `out`.

---

## 1. How ai.dl.vision.talkinghead works

First, a few words that come up throughout the guide:

- **Avatar.** A drawn character instead of a real person. Here it is a cartoon face with hair, eyes, glasses, a shirt and a background.
- **WAV.** A simple audio file. The speech you give must be a WAV file.
- **MP4.** The common video file format. It plays on phones, in browsers and in most programs.
- **Frame and FPS.** A video is a fast series of pictures. Each picture is a *frame*; *FPS* (frames per second) tells how many are shown every second. 25 is smooth enough for talking faces.
- **Resolution.** The size of the picture in pixels. Bigger looks sharper but takes longer.
- **Landmarks.** Small marks that a program puts on a face: the corners of the eyes, the edge of the lips, the line of the jaw. With landmarks the program knows where the mouth is, so it can move it.
- **Renderer.** The part that draws the frames and writes the video. Here it is a Python script. The library writes it for you and runs it.
- **Job.** One video request. You hand it in, it waits for its turn, it runs, and it ends as finished, failed or cancelled.
- **Queue.** The waiting line of jobs that are accepted but not yet running.

The module has two **engines**, and the engine follows from where the face comes from:

| Engine | Face | How it works |
|--------|------|--------------|
| `AVATAR_CARTOON` | `fromAvatar(...)` | The cartoon face is drawn by code. No model is downloaded. |
| `PHOTO_LANDMARK` | `fromPhoto(...)` | A landmark model finds the face. The lower face is bent to follow the voice, the inside of the mouth is painted, the eyes blink and the picture sways. |

It helps to know what the photo engine really is: it is a light, fast animation, not a deep-learning generator. The result looks like a talking photo, not like a real filmed person. This is also why it runs on a processor.

The classes a program meets:

| Class | Role |
|-------|------|
| `JTalkingHead` | The entry point; one shared instance per program |
| `TalkingRequestBuilder` | Describes one video: audio, output, size, speed, strength |
| `TalkingHeadJob` | One running or waiting video: status, progress, result, cancel |
| `AnimatedAvatar`, `AvatarPreset`, `HairStyle`, `BeardStyle` | The look of an avatar |
| `VideoResolution`, `TalkingEngine`, `JobStatus` | Small enums |
| `TalkingProgress` | A progress report: stage name and percent |
| `TalkingHeadException` and its subclasses | The failures; all unchecked |

You need Python 3.9 or newer on the machine. The library builds a private Python environment inside its working directory, so it never changes the Python you already use. The working directory looks like this:

```text
work/
├── environment/
│   ├── venv/            (a private Python with numpy, Pillow, ffmpeg and, for photos, OpenCV)
│   └── models/          (the landmark model for photos, downloaded once)
├── scripts/1.0.0/       (the renderer scripts that the library writes for you)
├── jobs/                (scratch space of running jobs, removed afterwards)
└── outputs/             (videos without an explicit output(...) path)
```

Keep these three points in mind:

- **The first video prepares the environment.** The library creates the virtual environment and installs a few packages. This needs the internet and takes some minutes, but happens only once. For photos, a small landmark model is also downloaded once.
- **The working directory must use ASCII letters only.** Choose a path like `work` or `D:/talking`, not one with Persian letters or other special characters. Some of the tools inside cannot read such paths.
- **Create one `JTalkingHead` and close it at the end.** It keeps a queue and some threads. Using `try-with-resources`, as the programs below do, closes it correctly.

## 2. The first video

`JTalkingHead.builder()` describes the library; only `workingDirectory` is required. `fromAvatar` starts a video request, `audio` adds the speech, `output` says where to write, and `generate` runs it and waits:

```java
import java.nio.file.Path;
import org.j2os.platform.ai.dl.vision.talkinghead.*;

public class T01Quick {
    public static void main(String[] args) {
        try (JTalkingHead head = JTalkingHead.builder().workingDirectory(Path.of("work")).build()) {
            Path video = head.fromAvatar(AnimatedAvatar.preset(AvatarPreset.SARA))
                    .audio(Path.of("assets/hello.wav"))
                    .output(Path.of("out/hello.mp4"))
                    .generate();
            System.out.println("Video written to " + video);
        }
    }
}
```

The program prints:

```text
Video written to /home/user/talk/out/hello.mp4
```

`generate()` waits until the video is ready and returns its path. If you do not call `output(...)`, the video is written to the `outputs` folder inside the working directory.

Three more facts:

- **The audio is copied when the request is made.** You may delete or reuse your own file right after `submit()` or `generate()` starts.
- **An existing output file is replaced.** Two jobs that write to the same path at the same time are not allowed; the second is refused.
- **The video is exactly as long as the audio.** A three-second WAV gives a three-second video.

> **Advice.** The first call is slow because of the one-time setup. Call `prepareEnvironment` at start-up (chapter 7) so that no visitor has to wait for it.

## 3. Choosing a face

### 3.1 The six presets

`AvatarPreset` offers six ready-made people:

| Preset | Look |
|--------|------|
| `SARA` | Long dark-brown hair, red shirt |
| `NINA` | Auburn bob, glasses, purple shirt |
| `LILY` | Curly dark hair, deep skin tone, yellow shirt |
| `ALI` | Short black hair, stubble, blue shirt |
| `KAVEH` | Bald, full beard, glasses, green shirt |
| `OMID` | Short light-brown hair, green eyes, navy shirt |

`AnimatedAvatar.preset(AvatarPreset.SARA)` gives the avatar of a preset.

### 3.2 Changing the look

`AnimatedAvatar` is **immutable**: it never changes after it is created. Every method that "changes" something returns a *new* avatar, so you chain the calls and keep the result. The methods have the names of the fields:

```java
import org.j2os.platform.ai.dl.vision.talkinghead.exception.InvalidInputException;
import org.j2os.platform.ai.dl.vision.talkinghead.*;

public class T04Custom {
    public static void main(String[] args) {
        AnimatedAvatar presenter = AnimatedAvatar.preset(AvatarPreset.ALI)
                .hairColor("#4b3621")
                .hairStyle(HairStyle.SHORT)
                .beardStyle(BeardStyle.NONE)
                .glasses(true)
                .shirtColor("#8e44ad");
        System.out.println(presenter.hairStyle() + ", glasses=" + presenter.glasses() + ", shirt=" + presenter.shirtColor());
        try {
            presenter.hairColor("brown");
        } catch (InvalidInputException wrong) {
            System.out.println("Rejected: " + wrong.getMessage());
        }
    }
}
```

```text
SHORT, glasses=true, shirt=#8e44ad
Rejected: hairColor must look like #RRGGBB but was: brown
```

- **Colors** are text in the form `#RRGGBB`, like the colors on a web page. The six color fields are `skinColor`, `hairColor`, `eyeColor`, `shirtColor`, `lipColor` and `backgroundColor`.
- **`hairStyle`** takes `SHORT`, `LONG`, `BOB`, `CURLY` or `BALD`.
- **`beardStyle`** takes `NONE`, `STUBBLE` or `FULL`.
- **`glasses`** takes `true` or `false`.

A wrong color is refused at once, as the second line of the output shows, so you find the mistake where you wrote it and not in the middle of a video.

> **Advice.** Start from the preset that is closest to what you want and change only a few things. It is easier than building a face from nothing.

## 4. Video settings

All of these are optional methods of the request. The defaults are good for most videos.

| Method | Meaning | Default |
|--------|---------|---------|
| `output(path)` | The file to write; must end in `.mp4` | `outputs/<id>.mp4` |
| `videoResolution(r)` | The size, from `VideoResolution` | `P512` |
| `framesPerSecond(n)` | Frames per second, 10 to 60 | 25 |
| `mouthMovementStrength(x)` | How wide the mouth moves, 0 to 2 | 1 |
| `headMovementStrength(x)` | How much the head sways, 0 to 2 | 1 |
| `blinking(b)` | Whether the eyes blink | `true` |
| `timeout(d)` | Time limit of this job | the library's `jobTimeout` |
| `onProgress(f)` | Receives progress reports (chapter 6) | none |

The values of `VideoResolution`:

| Value | Edge in pixels |
|-------|----------------|
| `P256` | 256 |
| `P360` | 360 |
| `P480` | 480 |
| `P512` | 512 |
| `P720` | 720 |
| `P1080` | 1080 |

An avatar video is a square with this edge. A photo video keeps the shape of the photo and the value is its longer edge.

An example of a calm, slow speaker: less head movement, a little more mouth, a larger picture:

```java
head.fromAvatar(presenter)
        .audio(Path.of("speech.wav"))
        .output(Path.of("out/speech.mp4"))
        .videoResolution(VideoResolution.P720)
        .framesPerSecond(30)
        .mouthMovementStrength(1.2)
        .headMovementStrength(0.6)
        .generate();
```

> **Advice.** Larger pictures and more frames per second take longer. Test with `P256`, which is fast, and make the final video with the size you need.

## 5. A talking photo

`fromPhoto` works like `fromAvatar`; only the face comes from a picture file:

```java
try (JTalkingHead head = JTalkingHead.builder().workingDirectory(Path.of("work")).build()) {
    Path video = head.fromPhoto(Path.of("portrait.jpg"))
            .audio(Path.of("hello.wav"))
            .output(Path.of("portrait-talking.mp4"))
            .videoResolution(VideoResolution.P720)
            .generate();
    System.out.println("Video written to " + video);
} catch (InvalidInputException problem) {
    System.err.println("The photo or the audio cannot be used: " + problem.getMessage());
}
```

It prints, as in chapter 2, the path of the video:

```text
Video written to /home/user/talk/portrait-talking.mp4
```

A good photo makes a big difference:

- **One face, looking at the camera.** The face should be clearly visible and not turned far to the side.
- **A normal size.** The photo must be between 1 byte and 25 MB, and it must be a JPEG, PNG, BMP or WebP image.
- **Good light, no hands or microphones over the mouth.** The mouth is repainted, so it must be visible.

If the library cannot use the picture, you get an `InvalidInputException`. Its message says what is wrong, for example that the file does not exist, is not an image, or that no face was found. The photo is checked when you call `submit()` or `generate()`, not when you call `fromPhoto`.

The photo engine needs one more download: a small face landmark model. It is fetched once and its checksum is verified. If the download fails, the message tells you where to get the file and where to save it.

## 6. Jobs and progress

`generate()` waits. When you do not want to wait, use `submit()`. It returns a `TalkingHeadJob` at once, and you follow the video while it is made:

```java
import java.nio.file.Path;
import org.j2os.platform.ai.dl.vision.talkinghead.*;

public class T02Progress {
    public static void main(String[] args) throws InterruptedException {
        try (JTalkingHead head = JTalkingHead.builder().workingDirectory(Path.of("work")).build()) {
            TalkingHeadJob job = head.fromAvatar(AnimatedAvatar.preset(AvatarPreset.ALI))
                    .audio(Path.of("assets/hello.wav"))
                    .output(Path.of("out/progress.mp4"))
                    .onProgress(p -> System.out.println("  " + p.stageName() + " " + p.percent() + "%"))
                    .submit();
            System.out.println("Submitted. Status: " + job.status());
            Path video = job.await();
            System.out.println("Status: " + job.status() + ", file: " + video.getFileName());
        }
    }
}
```

```text
Submitted. Status: QUEUED
  Starting 0%
  Preparing environment 0%
  Rendering 1%
  Drawing frames 10%
  Drawing frames 40%
  Animating the mouth 70%
  Encoding the video 95%
Status: COMPLETED, file: progress.mp4
```

Read the output from the top:

- The job starts in the `QUEUED` state: it is accepted and waits for its turn.
- The progress reports arrive in order. The first three (`Starting`, `Preparing environment`, `Rendering`) come from the library. The next ones come from the renderer, which names its own stages.
- When the video is written, the status is `COMPLETED`.

The progress function is called on a library thread, not on yours. Keep the function short and do not wait inside it.

A `TalkingHeadJob` can answer these questions at any time:

| Method | Answer |
|--------|--------|
| `identifier()` | A unique text id of the job |
| `engine()` | `AVATAR_CARTOON` or `PHOTO_LANDMARK` |
| `status()` | `QUEUED`, `RUNNING`, `COMPLETED`, `FAILED` or `CANCELLED` |
| `isFinished()` | `true` when the job is completed, failed or cancelled |
| `progressPercent()` | A number from 0 to 100 |
| `currentStageName()` | The name of the current stage; `Queued` at first |
| `resultVideoPath()` | An `Optional` with the path when the job is completed |
| `failure()` | An `Optional` with the reason when the job failed |
| `cancel()` | Asks the job to stop |
| `await()` | Waits and returns the path, or throws the failure |
| `future()` | The same result as a `CompletableFuture<Path>` |

### 6.1 Cancelling

`cancel()` stops a job. A job that still waits in the queue is dropped immediately. A job that is already running has its renderer process killed:

```java
import java.nio.file.Path;
import org.j2os.platform.ai.dl.vision.talkinghead.*;
import org.j2os.platform.ai.dl.vision.talkinghead.exception.JobCancelledException;

public class T03Cancel {
    public static void main(String[] args) throws InterruptedException {
        try (JTalkingHead head = JTalkingHead.builder().workingDirectory(Path.of("work")).build()) {
            TalkingHeadJob job = head.fromAvatar(AnimatedAvatar.preset(AvatarPreset.NINA))
                    .audio(Path.of("assets/hello.wav"))
                    .output(Path.of("out/cancel.mp4"))
                    .submit();
            Thread.sleep(1500);
            System.out.println("Cancel accepted: " + job.cancel());
            try {
                job.await();
            } catch (JobCancelledException cancelled) {
                System.out.println("Stopped: " + cancelled.getMessage());
            }
            System.out.println("Final status: " + job.status());
        }
    }
}
```

```text
Cancel accepted: true
Stopped: The job was cancelled.
Final status: CANCELLED
```

`await()` of a cancelled job throws a `JobCancelledException`. Catch it when cancelling is a normal event, for example when a visitor presses a "Stop" button.

## 7. Many videos at once

A video uses the processor heavily. If twenty visitors ask for a video at the same moment and all twenty start together, the machine becomes slow for everybody. The library prevents this with two limits:

- **`maximumConcurrentJobs(engine, n)`.** How many jobs of one engine run at the same time. The default is half of the processor cores, and at least 1.
- **`maximumQueuedJobs(n)`.** How many accepted jobs may wait. The default is 50.

When both are full, a new request is **refused at once** with a `QueueFullException`. Refusing quickly is better than a server that slowly drowns. The program below allows one running job and two waiting jobs, then asks for five videos:

```java
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;
import org.j2os.platform.ai.dl.vision.talkinghead.*;
import org.j2os.platform.ai.dl.vision.talkinghead.exception.QueueFullException;

public class T06Queue {
    public static void main(String[] args) {
        try (JTalkingHead head = JTalkingHead.builder().workingDirectory(Path.of("work"))
                .maximumConcurrentJobs(TalkingEngine.AVATAR_CARTOON, 1).maximumQueuedJobs(2).build()) {
            List<TalkingHeadJob> accepted = new ArrayList<>();
            for (int i = 1; i <= 5; i++) {
                try {
                    accepted.add(head.fromAvatar(AnimatedAvatar.preset(AvatarPreset.LILY))
                            .audio(Path.of("assets/hello.wav")).output(Path.of("out/batch-" + i + ".mp4")).submit());
                    System.out.println("Request " + i + " accepted");
                } catch (QueueFullException full) {
                    System.out.println("Request " + i + " rejected: " + full.getMessage());
                }
            }
            accepted.forEach(TalkingHeadJob::await);
            System.out.println(accepted.size() + " videos finished");
        }
    }
}
```

```text
Request 1 accepted
Request 2 accepted
Request 3 accepted
Request 4 rejected: The queue of the engine AVATAR_CARTOON is full. Try again later.
Request 5 rejected: The queue of the engine AVATAR_CARTOON is full. Try again later.
3 videos finished
```

One request runs and two wait, so three are accepted; the fourth and fifth are refused. A web service answers such a refusal with "try again later".

All the settings of the library, with their defaults:

```java
JTalkingHead head = JTalkingHead.builder()
        .workingDirectory(Path.of("work"))                    // required, ASCII characters only
        .pythonCommand("py", "-3")                            // default: python3 / python / py -3
        .maximumConcurrentJobs(TalkingEngine.AVATAR_CARTOON, 2)   // default: half of the CPU cores
        .maximumQueuedJobs(20)                                // default 50
        .jobTimeout(Duration.ofMinutes(5))                    // default 10 minutes
        .maximumAudioDuration(Duration.ofMinutes(3))          // default 10 minutes
        .jobRetention(Duration.ofMinutes(30))                 // default 1 hour
        .build();
```

- **`pythonCommand`.** Only needed when the library cannot find Python. Give the program and its arguments, for example `"py", "-3"` on Windows.
- **`jobTimeout`.** The time after which a running job is stopped (a `JobTimeoutException`). A request can set its own with `timeout(...)`.
- **`maximumAudioDuration`.** Longer speech is refused before any work starts.
- **`jobRetention`.** How long a finished job stays known to `findJob(id)`. After that it is forgotten, but the video file stays where it is.

To avoid the slow first video, prepare the environment when the program starts:

```java
JTalkingHead head = JTalkingHead.builder().workingDirectory(Path.of("work")).build();
head.prepareEnvironment(TalkingEngine.AVATAR_CARTOON, TalkingEngine.PHOTO_LANDMARK);   // at start-up, not at the first request
```

`prepareEnvironment` does the one-time setup now and returns when everything is ready. If the setup already happened, it returns at once.

## 8. Mistakes

Most mistakes are caught when you hand in the request, before the job is queued. Each is an `InvalidInputException` with a message that tells what to fix. The program makes seven mistakes on purpose:

```java
import java.nio.file.Path;
import org.j2os.platform.ai.dl.vision.talkinghead.*;
import org.j2os.platform.ai.dl.vision.talkinghead.exception.TalkingHeadException;

public class T05Mistakes {
    static void attempt(String title, Runnable action) {
        try {
            action.run();
            System.out.println(title + "\n  accepted");
        } catch (TalkingHeadException problem) {
            System.out.println(title + "\n  " + problem.getClass().getSimpleName() + ": " + problem.getMessage());
        }
    }

    public static void main(String[] args) {
        try (JTalkingHead head = JTalkingHead.builder().workingDirectory(Path.of("work")).build()) {
            AnimatedAvatar avatar = AnimatedAvatar.preset(AvatarPreset.SARA);
            attempt("Text file as audio", () -> head.fromAvatar(avatar).audio(Path.of("assets/notes.txt")).submit());
            attempt("Very short audio", () -> head.fromAvatar(avatar).audio(Path.of("assets/tiny.wav")).submit());
            attempt("Wrong output extension", () -> head.fromAvatar(avatar).audio(Path.of("assets/hello.wav"))
                    .output(Path.of("out/video.avi")).submit());
            attempt("Frame rate 120", () -> head.fromAvatar(avatar).audio(Path.of("assets/hello.wav"))
                    .framesPerSecond(120).submit());
            attempt("Mouth strength 5", () -> head.fromAvatar(avatar).audio(Path.of("assets/hello.wav"))
                    .mouthMovementStrength(5).submit());
            attempt("Missing photo", () -> head.fromPhoto(Path.of("assets/nobody.jpg")).audio(Path.of("assets/hello.wav")).submit());
            attempt("No audio", () -> head.fromAvatar(avatar).submit());
        }
        attempt("Non-ASCII working folder", () -> JTalkingHead.builder().workingDirectory(Path.of("work-\u00e9")).build());
    }
}
```

```text
Text file as audio
  InvalidInputException: The audio file is not a WAV file: notes.txt
Very short audio
  InvalidInputException: The WAV file is shorter than 200 ms.
Wrong output extension
  InvalidInputException: The output file must have the extension .mp4 but was: /home/user/talk/out/video.avi
Frame rate 120
  InvalidInputException: framesPerSecond must be between 10 and 60 but was 120
Mouth strength 5
  InvalidInputException: mouthMovementStrength must be between 0 and 2.0 but was 5.0
Missing photo
  InvalidInputException: The photo file does not exist: assets/nobody.jpg
No audio
  InvalidInputException: audio(...) is required
Non-ASCII working folder
  InvalidInputException: The working directory must use ASCII characters only, but it is: /home/user/talk/work-é
```

Notice that the library looks inside the audio file: the library checks that it is a real WAV, that it is at least 200 milliseconds long and that it is not longer than `maximumAudioDuration`.

All failures are unchecked and extend `TalkingHeadException`, so you can catch the base class once, or the subclasses you care about:

| Exception | Meaning |
|-----------|---------|
| `InvalidInputException` | The request or a file is wrong; fix it and ask again |
| `QueueFullException` | Too many jobs; try again later |
| `LibraryClosedException` | `close()` was called |
| `JobCancelledException` | The job was cancelled |
| `JobTimeoutException` | The renderer ran past its time limit and was stopped |
| `JobFailedException` | The renderer stopped with an error; it has a code and the last output lines |
| `PythonNotFoundException` | No Python 3.9 or newer was found |
| `EnvironmentSetupException` | The private environment could not be prepared |

## 9. A web service

Videos take time, so a web service should not keep a request open until the video is ready. The pattern is: accept, answer with a job id, and let the client ask again. The library supports this directly with `findJob(id)`:

| Request | What the service does |
|---------|-----------------------|
| `POST /videos/avatar?preset=SARA` with a WAV file as the body | Saves the upload, calls `submit()`, answers `202` and `{"jobId": "..."}` |
| `GET /jobs/{id}` | Finds the job with `findJob`, answers its status, percent and stage |
| `GET /jobs/{id}/video` | Sends the MP4 when the status is `COMPLETED`; otherwise answers `409` |
| `DELETE /jobs/{id}` | Calls `cancel()` |

The errors are easy to turn into HTTP codes: `InvalidInputException` becomes `400`, `QueueFullException` becomes `429` with a `Retry-After` header, `LibraryClosedException` becomes `503`, and an unknown id (`findJob` returns an empty `Optional`) becomes `404`. A complete example with the JDK's own `HttpServer` is in the `example` folder (`Example07HttpService`).

Two points for a real service:

- **Limit the upload size.** The example refuses uploads larger than 50 MB.
- **Close the library when the service stops.** `close()` cancels the waiting and running jobs and refuses new ones. Finished videos in the output folder stay.

## 10. Putting it together

Three recordings need one video each, all with the same avatar. The program starts all three at once with `generateAsync()`, which returns a `CompletableFuture<Path>`, and then collects the results:

```java
import java.nio.file.Path;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.concurrent.CompletableFuture;
import org.j2os.platform.ai.dl.vision.talkinghead.*;
import org.j2os.platform.ai.dl.vision.talkinghead.exception.TalkingHeadException;

public class T07Batch {
    public static void main(String[] args) {
        AnimatedAvatar presenter = AnimatedAvatar.preset(AvatarPreset.OMID).glasses(true);
        Map<String, CompletableFuture<Path>> videos = new LinkedHashMap<>();
        try (JTalkingHead head = JTalkingHead.builder().workingDirectory(Path.of("work")).build()) {
            for (String clip : new String[]{"clip1", "clip2", "clip3"}) {
                videos.put(clip, head.fromAvatar(presenter)
                        .audio(Path.of("assets/hello.wav"))
                        .output(Path.of("videos/" + clip + ".mp4"))
                        .videoResolution(VideoResolution.P360)
                        .generateAsync());
            }
            videos.forEach((clip, future) -> {
                try {
                    System.out.println(clip + " -> " + future.join().getFileName());
                } catch (java.util.concurrent.CompletionException problem) {
                    System.out.println(clip + " failed: " + problem.getCause().getMessage());
                }
            });
        }
    }
}
```

```text
clip1 -> clip1.mp4
clip2 -> clip2.mp4
clip3 -> clip3.mp4
```

The library runs as many videos at the same time as the limits allow, and the rest wait. The program does not care about the order: `join()` waits for each result in turn. If one video fails, its `CompletionException` carries the reason, and the other videos are not affected.

In a few lines:

- Pick a face: `fromAvatar(...)` or `fromPhoto(...)`.
- Add the speech with `audio(...)` and, if you want, an `output(...)` path.
- `generate()` waits for the video; `submit()` and `generateAsync()` do not.
- Keep one `JTalkingHead`, prepare it at start-up, and close it at the end.

## Appendix A. Quick reference

### A.1 The library builder

| Call | Meaning | Default |
|------|---------|---------|
| `JTalkingHead.builder()` | Starts the description | — |
| `workingDirectory(path)` | Where everything is kept; required; ASCII characters only | — |
| `pythonCommand(program, args...)` | How to start Python | `python3`, `python`, or `py -3` on Windows; version 3.9 or newer |
| `maximumConcurrentJobs(engine, n)` | Running jobs per engine | Half of the processor cores, at least 1 |
| `maximumQueuedJobs(n)` | Waiting jobs | 50 |
| `jobTimeout(duration)` | Time limit of a running job | 10 minutes |
| `maximumAudioDuration(duration)` | Longest accepted speech | 10 minutes |
| `jobRetention(duration)` | How long finished jobs can be found | 1 hour |
| `build()` | Creates the library | — |

### A.2 `JTalkingHead`

| Call | Meaning |
|------|---------|
| `prepareEnvironment(engines...)` | Does the one-time setup for the engines now |
| `fromAvatar(avatar)` | Starts a request with a cartoon face |
| `fromPhoto(path)` | Starts a request with a photo |
| `findJob(id)` | Returns an `Optional<TalkingHeadJob>` |
| `close()` | Cancels waiting and running jobs; the output folder is not touched |

### A.3 The request (`TalkingRequestBuilder`)

| Call | Meaning |
|------|---------|
| `audio(path)` | The speech WAV; required |
| `output(path)` | The MP4 to write; replaces an existing file |
| `videoResolution(r)` | The size; default `P512` |
| `framesPerSecond(n)` | 10 to 60; default 25 |
| `mouthMovementStrength(x)` | 0 to 2; default 1 |
| `headMovementStrength(x)` | 0 to 2; default 1 |
| `blinking(b)` | Default `true` |
| `timeout(duration)` | Time limit of this job |
| `onProgress(consumer)` | Receives `TalkingProgress` (stage name, percent) |
| `submit()` | Queues the job and returns a `TalkingHeadJob` at once |
| `generateAsync()` | Queues the job and returns a `CompletableFuture<Path>` |
| `generate()` | Queues the job, waits and returns the path |

### A.4 The enums

| Type | Values |
|------|--------|
| `TalkingEngine` | `PHOTO_LANDMARK`, `AVATAR_CARTOON` |
| `JobStatus` | `QUEUED`, `RUNNING`, `COMPLETED`, `FAILED`, `CANCELLED` |
| `VideoResolution` | `P256`, `P360`, `P480`, `P512`, `P720`, `P1080` |
| `AvatarPreset` | `SARA`, `NINA`, `LILY`, `ALI`, `KAVEH`, `OMID` |
| `HairStyle` | `SHORT`, `LONG`, `BOB`, `CURLY`, `BALD` |
| `BeardStyle` | `NONE`, `STUBBLE`, `FULL` |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| The request is wrong (audio, output, numbers) | `InvalidInputException` when the job is handed in |
| The photo does not exist or is not a usable image | `InvalidInputException` when the job is handed in |
| The renderer finds no face or a bad file | `InvalidInputException` with the message of the renderer |
| The queue is full | `QueueFullException` from `submit()` |
| The library is closed | `LibraryClosedException` |
| Python is missing or too old | `PythonNotFoundException` |
| The environment cannot be prepared | `EnvironmentSetupException` |
| The job takes too long | `JobTimeoutException`; the renderer is killed |
| The renderer stops with an error | `JobFailedException` with a code and the last output |
| The job is cancelled or the library closes | `JobCancelledException` from `await()` |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Expecting a filmed person from a photo.** The photo engine is a light animation. Avatars and photos are best for short, friendly videos.
- **A working directory with non-ASCII characters.** It is refused. Choose a plain path.
- **The first video inside a user request.** The one-time setup takes minutes. Call `prepareEnvironment` at start-up.
- **Deleting a video file you still need.** The library never deletes your output files. It deletes only its own scratch space.
- **Two jobs with the same output path.** The second is refused while the first is running.
- **A queue that is too large.** A long queue means long waiting times. It is better to refuse early.
- **Forgetting `close()`.** The library keeps threads. Use `try-with-resources`.

### C.2 Behavior worth knowing

- The environment preparation is protected by a lock, also between several programs that share one working directory.
- Scratch folders in `jobs` are removed after each job; leftovers older than 24 hours are removed at start-up.
- Progress goes from 0 to 100 and ends with `Finished` at 100 for a successful job.
- A finished job is kept in memory for `jobRetention`; the video file is not affected by that.

### C.3 Limits

- Audio must be WAV, at least 200 milliseconds and at most `maximumAudioDuration`.
- Photos: JPEG, PNG, BMP or WebP, 1 byte to 25 MB, with a visible face.
- Frame rate from 10 to 60; strengths from 0 to 2.
- The output file must end in `.mp4`.
- Python 3.9 or newer; internet for the first setup.

### C.4 Tuning and practical advice

- Leave the concurrency at its default first; raise it only if the processor is not busy.
- Use `P256` for tests and drafts, and the final size only for the final video.
- Keep speeches short. A long speech makes a long video and a long wait.
- In a service, show the percent of `progressPercent()` so people can see that something happens.
- Move the working directory to a fast disk; the environment and the scratch files live there.

## Appendix D. Message reference

### D.1 Audio and photo checks

| Message | Cause |
|---------|-------|
| `The audio lasts N seconds but at most M seconds are allowed.` | Longer than `maximumAudioDuration` |
| `The audio file is not a WAV file: <name>` | The file is not WAV |
| `The WAV file is shorter than 200 ms.` | Too short |
| `The WAV file has no audio data: <name>` | The file contains no sound |
| `The WAV file has no valid format description: <name>` | The WAV header is damaged |
| `The photo file does not exist: <path>` | Wrong path |
| `The photo must be between 1 byte and 25 MB.` | The file is empty or too big |
| `The photo file is too short to be an image.` | The file is not a real image |
| `The photo must be a JPEG, PNG, BMP or WebP image.` | Another image type |
| `The input files could not be read: ...` | A file could not be copied for the job |

### D.2 Request and settings checks

| Message | Cause |
|---------|-------|
| `audio(...) is required` | The request has no audio |
| `The output file must have the extension .mp4 but was: <path>` | Wrong extension |
| `The output path is a directory: <path>` | The path is a folder |
| `Another running job already writes to <path>` | Two jobs, one output path |
| `framesPerSecond must be between 10 and 60 but was N` | Frame rate out of range |
| `<name> must be between 0 and 2.0 but was V` | A strength out of range |
| `timeout must be a positive duration` | Zero or negative time |
| `<field> must look like #RRGGBB but was: <value>` | A wrong color |
| `workingDirectory(...) is required` | The library has no working directory |
| `The working directory must use ASCII characters only, but it is: <path>` | Non-ASCII path |
| `maximumConcurrentJobs must be at least 1 but was N` | A value below 1 |
| `maximumQueuedJobs must be at least 1 but was N` | A value below 1 |
| `<name> must be a positive duration` | A duration setting is zero or negative |

### D.3 Running jobs

| Message | Exception |
|---------|-----------|
| `The queue of the engine <E> is full. Try again later.` | `QueueFullException` |
| `The library was closed.` | `LibraryClosedException` |
| `The job was cancelled before it started.` | `JobCancelledException` |
| `The job was cancelled.` | `JobCancelledException` |
| `The job was interrupted because the library is closing.` | `JobCancelledException` |
| `The renderer did not finish within N seconds and was stopped.` | `JobTimeoutException` |
| `No Python 3.9 or newer was found. Tried: '...'. Install Python from https://www.python.org or pass its command with pythonCommand(...).` | `PythonNotFoundException` |

### D.4 Environment

| Message | Cause |
|---------|-------|
| `The renderer scripts could not be written to <folder>` | The scripts folder is not writable |
| `The Python virtual environment could not be created with Python X. On Debian or Ubuntu install the package python3-venv. Output: ...` | `venv` is missing or failed |
| `Installing the Python packages [...] failed. Check the internet connection and that your Python version has wheels for them. Output: ...` | Package installation failed |
| `The installed packages for <engine> do not work. Delete the folder '<folder>' and try again. Output: ...` | The check after installing failed |
| `The landmark model could not be downloaded from <url> (...). Download it manually and save it as <file> (SHA-256 ...).` | The photo model download failed three times |
| `Waiting for another process that prepares the environment took too long.` | Another program holds the setup lock for over an hour |
