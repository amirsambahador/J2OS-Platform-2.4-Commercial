# useScreenRecorder

*J2OS Platform 2.4*

A support team wants to see the problem, not read a description of it. A teacher wants to record a lesson, and a tester wants to attach a short video to a bug report. All of them need the same thing: the visitor shares their screen, the page records it, and a video file ends up on their computer. The browser has the pieces for this, but they are spread over several APIs (screen capture, microphone, audio mixing, recording, download), and each has its own ways to fail.

**useScreenRecorder** is a composable of the Nuxt front end of J2OS that puts them behind four functions. It has three features:

- **A short, orderly flow.** `init` once, then `start`, then `stop`. The recording is saved as a `.webm` video file by the browser.
- **Optional sound.** The visitor's voice, the sound of the shared screen, or both mixed into one track.
- **Clear errors.** Every wrong order of calls and every refusal by the visitor throws an `Error` with a short English sentence that a page can show as it is.

In a few lines:

```vue
<script setup>
const recorder = useScreenRecorder()

onMounted(() => recorder.init('preview'))        // the <video> that shows what is being recorded
</script>

<template>
  <video id="preview" width="320" autoplay muted />
  <button @click="recorder.start()">Start</button>
  <button @click="recorder.stop('lesson.webm')">Stop</button>   <!-- saves lesson.webm -->
</template>
```

This guide goes step by step: chapter 1 explains how the composable works and chapter 2 makes a first recording. Chapter 3 is about the ways a recording ends, chapter 4 about the preview and the page's own state, and chapter 5 about sound. Chapter 6 collects what can go wrong, and chapter 7 builds a bug-report recorder. The appendices are a quick reference.

## Contents

1. [How useScreenRecorder works](#1-how-usescreenrecorder-works)
2. [The first recording](#2-the-first-recording)
3. [How a recording ends](#3-how-a-recording-ends)
4. [The preview and the page's state](#4-the-preview-and-the-pages-state)
5. [Sound](#5-sound)
6. [When something goes wrong](#6-when-something-goes-wrong)
7. [A bug-report recorder](#7-a-bug-report-recorder)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are complete pages of a Nuxt project: each is one `.vue` file. In Nuxt `ref`, `onMounted` and the composables need no `import`, so none is written. The output under a page is what the visitor sees on it as the page is used, together with the files the browser saves.

---

## 1. How useScreenRecorder works

First, a few terms, because they come back throughout the guide:

- **Preview.** A `<video>` element on the page that shows the shared screen while it is recorded. The composable finds it by its id.
- **Sharing.** The visitor's choice, in the browser's own picker, of a screen, a window or a tab. The page cannot choose for them.
- **Recording.** The video (and sound) of what is shared, collected until the recording ends.
- **Saving.** At the end the browser downloads the collected video as a file.

The functions of this composable:

| Function | Role |
|----------|------|
| `init(videoId, captureMic, captureSystemAudio, onEnded)` | Sets the preview element and the options; throws if the element is missing |
| `start()` | Asks the visitor to share a screen and starts recording |
| `stop(fileName)` | Ends the recording and saves it |
| `isRecording()` | `true` while a recording is running |

6 points to keep in mind:

- **`init` comes first** and only stores settings. It is light and can be repeated.
- **`start` is asynchronous** and ends only after the visitor has answered the picker. It must be called from a click or another gesture of the visitor, or the browser refuses.
- **The visitor is always in control.** A browser bar offers a Stop sharing button, and the page has to take that into account.
- **The file is saved by the browser** as a download, as `video/webm`. The page never sees the video data.
- **One recording at a time** for each composable.
- **Cleanup is not automatic.** If the visitor leaves the page while recording, the page has to stop it (chapter 3).

## 2. The first recording

A teacher records a lesson. The page has a title field, two buttons and a status line. While the recording runs, a small preview shows what is being captured. The file takes the lesson's title as its name.

```vue
<script setup>
const recorder = useScreenRecorder()
const title = ref('lesson-1')
const recording = ref(false)
const message = ref('Press Start to share your screen.')

onMounted(() => {
  // the callback runs after every recording that has finished and been saved
  recorder.init('preview', false, false, () => {
    recording.value = false
    message.value = 'Recording saved.'
  })
})

async function start() {
  try {
    await recorder.start()
    recording.value = true
    message.value = 'Recording...'
  } catch (e) {
    message.value = e.message
  }
}

function stop() {
  try {
    recorder.stop(`${title.value}.webm`)
  } catch (e) {
    message.value = e.message
  }
}

onBeforeUnmount(() => {
  if (recorder.isRecording()) stop()
})
</script>

<template>
  <input id="title" v-model="title">
  <video id="preview" width="320" autoplay muted :style="{ display: recording ? 'block' : 'none' }" />
  <button id="start" @click="start">Start</button>
  <button id="stop" @click="stop">Stop</button>
  <p id="message">{{ message }}</p>
</template>
```

```text
Page opens:                                        Press Start to share your screen. Preview is hidden.
Visitor presses Stop before starting:              There is no recording to stop
Visitor presses Start and shares the screen:       Recording... Preview is visible.
Visitor presses Start again:                       A recording is already in progress Preview is visible.
Visitor names the lesson and presses Stop:         Recording saved. Preview is hidden. The browser saves intro-to-sales.webm with picture.
Visitor presses Stop and Start at once:            The previous recording is still being saved, please wait a moment then Recording saved.
Visitor presses the browser's Stop sharing button: Recording saved. Preview is hidden. The browser saves screen-record.webm.
Visitor leaves the page while recording:           the browser saves lesson-1.webm.
```

`init('preview', false, false, ...)` names the `<video>` and specifies that no sound is recorded. The last argument is a function that runs after every recording that has been saved; the page uses it to hide the preview and to show "Recording saved."

Read the output in order. Pressing Stop with nothing recorded gives a sentence and nothing else. After Start the visitor sees the browser's picker, chooses a screen, and the preview appears. A second Start is refused with its own sentence, and the recording goes on undisturbed. Stop with a new title saves `intro-to-sales.webm`, and the file holds a picture.

## 3. How a recording ends

A recording ends in one of three ways, and a page should expect all of them:

```js
// 1. the page stops it, and chooses the file name
recorder.stop(`${title.value}.webm`)

// 2. the visitor presses the browser's own "Stop sharing" button:
//    the recording is saved as screen-record.webm and the onEnded callback runs

// 3. the visitor leaves the page: nothing happens by itself, so stop it here
onBeforeUnmount(() => {
  if (recorder.isRecording()) recorder.stop()
})
```

The table shows the result of each:

| How it ends | File name | `onEnded` |
|-------------|-----------|-----------|
| The page calls `stop('name.webm')` | The name given | Runs after the file is saved |
| The page calls `stop()` | `screen-record.webm` | Runs after the file is saved |
| The visitor presses the browser's Stop sharing button | `screen-record.webm` | Runs after the file is saved |
| The page calls `stop(name)` while leaving | The name given | Runs after the file is saved |

3 details follow from the table:

- **The name is used as given.** The composable does not add or check an extension; write `.webm` yourself.
- **Stopping from the browser bar is a normal ending,** not an error. The file is saved, and `onEnded` is the page's only notification, which is why it should reset the page's state.
- **Saving takes a moment.** Between `stop` and the saved file, a new `start` is refused with "The previous recording is still being saved, please wait a moment". The output shows the sentence appearing and, a short time later, "Recording saved". If the page has a Start button, disable it until `onEnded` has run.

## 4. The preview and the page's state

The preview is a plain `<video>` that the composable fills with the shared screen, without sound. It is not hidden or shown by the composable: the lesson page above shows it only while recording. After the recording, the video element still holds the stream, but its tracks have ended.

`isRecording()` is a plain function, not a ref. A page that wants the Stop button to be disabled at the right moments keeps a ref of its own:

```js
// isRecording() is an ordinary function, not a ref: the page does not update when it changes.
// Keep a ref of your own and change it where recording starts and ends.
const recording = ref(false)

recorder.init('preview', false, false, () => { recording.value = false })   // after every saved recording
await recorder.start()
recording.value = true
```

Notice where the flag changes: it becomes `true` after `start` succeeds and `false` in `onEnded`. `onEnded` covers all three endings of chapter 3, including the browser's Stop sharing button, which is why it is the right place.

## 5. Sound

The second and third arguments of `init` choose the sound:

| `captureMic` | `captureSystemAudio` | The file holds |
|--------------|----------------------|----------------|
| `false` | `false` | picture only |
| `true` | `false` | picture and the visitor's voice |
| `false` | `true` | picture and the sound of the shared screen |
| `true` | `true` | picture and both, mixed into a single track |

The voice needs the microphone permission, which the browser asks for when `start` runs. The sound of the shared screen is not up to the page: the visitor has to tick the "share audio" box in the browser's picker, and not every screen or window can provide sound. If they do not, or if that is not possible, the recording simply has no screen sound, and no error is raised; with the voice switched on, the file then holds the voice alone.

Because `init` only stores settings, a page can offer the choices as checkboxes and call `init` again before each recording:

```js
//                     video id    my voice   sound of the shared screen
recorder.init('capture', withVoice.value, withTabAudio.value, done)

// init() only stores these settings, so call it again before start() when the visitor changes them
```

## 6. When something goes wrong

Every failure throws an `Error` with a short English message. For failures caused by the browser or the visitor, the browser's own error is in `error.cause`:

```js
try {
  await recorder.start()
} catch (e) {
  message.value = e.message            // short sentence in English, safe to show
  console.error(e.cause)               // the browser's own error (NotAllowedError, ...), when there is one
}
```

| Call | Situation | Message |
|------|-----------|---------|
| `init` | No element with that id | `Video with id="..." not found.` (thrown at once) |
| `start` | `init` has not been called | `Preview video is not set up. Call init first.` |
| `start` | A recording is running | `A recording is already in progress` |
| `start` | The last one is still being saved | `The previous recording is still being saved, please wait a moment` |
| `start` | The visitor cancels the picker or the browser refuses | `Screen access was cancelled or failed` |
| `start` | Voice requested and the microphone is refused | `Microphone access was denied` |
| `stop` | Nothing has been started | `There is no recording to stop` |
| `stop` | `stop` was already called | `Recording has already been stopped` |

Two things in the table deserve attention. A refused microphone does not leave a half-started recording behind: the screen sharing that was already granted is released, and the visitor can try again right away, which the bug-report page of the next chapter does. And the messages are written for people, but they are in English; a page that is shown in Persian should translate them by matching the message, or show its own text and log the original.

## 7. A bug-report recorder

A support page asks the visitor to reproduce the problem on screen and record it. It offers two choices (voice, sound of the screen), a Record button and a Finish button, and it explains what has happened after each step. Failures are shown as sentences, and the visitor can change the choices and try again.

```vue
<script setup>
const recorder = useScreenRecorder()
const withVoice = ref(false)
const withTabAudio = ref(false)
const recording = ref(false)
const message = ref('Choose what to record, then press Record.')

function done() {
  recording.value = false
  message.value = 'Thanks! Your recording was downloaded. Attach it to your ticket.'
}

async function record() {
  // init() only stores settings, so it can be repeated with new choices
  recorder.init('capture', withVoice.value, withTabAudio.value, done)
  try {
    await recorder.start()
    recording.value = true
    message.value = 'Recording your screen. Reproduce the problem, then press Finish.'
  } catch (e) {
    message.value = `Could not start: ${e.message}.`
  }
}
</script>

<template>
  <label><input id="voice" v-model="withVoice" type="checkbox"> My voice</label>
  <label><input id="tab" v-model="withTabAudio" type="checkbox"> Sound from the shared screen</label>
  <video id="capture" width="320" autoplay muted />
  <button id="record" :disabled="recording" @click="record">Record</button>
  <button id="finish" :disabled="!recording" @click="recorder.stop('bug-report.webm')">Finish</button>
  <p id="message">{{ message }}</p>
</template>
```

```text
Page opens:                                         Choose what to record, then press Record. Finish is disabled.
Visitor presses Record:                             Recording your screen. Reproduce the problem, then press Finish. Record is disabled, Finish is enabled.
Visitor presses Finish:                             Thanks! Your recording was downloaded. Attach it to your ticket. The browser saves bug-report.webm with picture.
Visitor ticks "My voice" and records again:         the file has sound and picture.
Visitor also ticks "Sound from the shared screen":  the file has sound and picture.
Visitor cancels the screen picker:                  Could not start: Screen access was cancelled or failed. Record is enabled.
Visitor ticks "My voice" but blocks the microphone: Could not start: Microphone access was denied.
Visitor unticks it and tries again:                 Recording your screen. Reproduce the problem, then press Finish.
```

The page combines most of this guide. The two buttons are enabled or disabled by its own `recording` flag, which `done` (the `onEnded` callback) resets. `record` calls `init` with the current choices every time, so the checkboxes take effect on the next recording. The failures are all caught in one place: cancelling the picker and blocking the microphone each give a sentence, and the Record button stays usable, so the last line of the output shows a successful retry.

For your own pages the pattern is the same: keep a state flag of your own, reset it in `onEnded`, catch the errors of `start` and show their messages, name the file yourself, and stop the recording when the page is left.

---

## Appendix A. Quick reference

### A.1 Functions

| Call | Returns | Notes |
|------|---------|-------|
| `init(videoId, captureMic, captureSystemAudio, onEnded)` | | Defaults: `false`, `false`, none; throws if the element is missing |
| `start()` | Promise | Asks for the screen; rejects with an `Error` |
| `stop(fileName)` | | Default name `screen-record.webm`; throws if nothing is recording |
| `isRecording()` | Boolean | Not reactive |

### A.2 Ways a recording ends

| Cause | File name |
|-------|-----------|
| `stop('a.webm')` | `a.webm` |
| `stop()` | `screen-record.webm` |
| Browser's Stop sharing | `screen-record.webm` |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Wrong order of calls | An `Error` with a short sentence |
| Visitor cancels or refuses the screen | `Screen access was cancelled or failed`; nothing is left running |
| Microphone is refused | `Microphone access was denied`; the shared screen is released |
| Visitor stops from the browser bar | A normal ending; the file is saved |
| Page left while recording | The recording keeps running until the page stops it |
| Screen sound not shared by the visitor | No screen sound in the file; no error |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Calling `start` outside a click.** The browser refuses to show the picker.
- **Treating `isRecording()` as reactive.** The page does not update; keep a ref.
- **Not resetting state in `onEnded`.** The browser's Stop sharing button then leaves the page in "recording".
- **Starting again right after `stop`.** The previous recording is still being saved.
- **Forgetting to stop when the page is left.** The recording continues.
- **Passing a file name without an extension.** The browser saves the name as it is.
- **Showing the English messages to Persian readers unchanged.** Translate or replace them.
- **Expecting screen sound without asking the visitor to share it.** The picker has the choice.
- **Calling `init` with an id that does not exist yet.** It throws; call it in `onMounted`.

### C.2 Behaviour worth knowing

- The video is saved as WebM.
- With voice and screen sound together, the two are mixed into one audio track.
- `onEnded` runs after every saved recording and not after a failed start.
- The preview is always muted.
- After a recording every media track the composable opened is stopped.

### C.3 Limits

| What | Limit |
|------|-------|
| Recordings at the same time | One per composable |
| Output format | WebM only |
| Where the file goes | The browser's downloads; the page does not receive the data |
| Choosing the screen | The visitor only |
| Browsers | Those that support screen capture; many mobile browsers do not |

### C.4 Tuning

- Disable Start until `onEnded` has run.
- Name files after what was recorded: lesson title, ticket number, date.
- Offer sound as choices and call `init` again before each recording.
- Catch the errors of `start` and `stop` in one place and show the messages.
- Stop the recording in `onBeforeUnmount`.

## Appendix D. Message reference

### D.1 Errors

| Message | Cause |
|---------|-------|
| `Video with id="..." not found.` | `init` found no element with that id |
| `Preview video is not set up. Call init first.` | `start` before `init` |
| `A recording is already in progress` | `start` while recording |
| `The previous recording is still being saved, please wait a moment` | `start` right after `stop` |
| `Screen access was cancelled or failed` | The picker is cancelled or screen capture fails |
| `Microphone access was denied` | Voice is requested and the microphone is refused |
| `There is no recording to stop` | `stop` with nothing started |
| `Recording has already been stopped` | `stop` twice |

### D.2 Other values

| Value | Meaning |
|-------|---------|
| `screen-record.webm` | The default file name |
| `error.cause` | The browser's own error behind a start failure |
