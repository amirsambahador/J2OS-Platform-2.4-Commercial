# useMicrophone

*J2OS Platform 2.4*

A clinic wants the doctor to dictate notes instead of typing them. The page needs a button that records, a sign that something is happening, and then the words in a text box. Between the button and the words there is a lot of hidden work: asking for the microphone, recording in whatever format the browser chooses, converting that into the format a speech recognizer wants, uploading it and reading the answer.

**useMicrophone** is a composable of the Nuxt front end of J2OS that does all of this behind a few refs and three functions. It has three features:

- **Record, stop, get text.** `startRecording()`, `stopRecording()` and a ref that holds the recognized text.
- **The right audio.** Whatever the browser recorded is converted to a 16 kHz mono 16-bit WAV, the format the speech recognizer of the J2OS back end expects.
- **Failures do not throw.** A denied microphone, a failed upload or an unreadable answer never throws an exception; the message of each is stored in the `error` ref.

The core of it, in code:

```vue
<script setup>
const { isRecording, recognizedText, error, startRecording, stopRecording } =
  useMicrophone({ endpoint: '/api/speech-to-text/recognize' })

const toggle = () => (isRecording.value ? stopRecording() : startRecording())
</script>

<template>
  <button @click="toggle">{{ isRecording ? 'Stop' : 'Record' }}</button>
  <p v-if="error">{{ error }}</p>
  <p>{{ recognizedText }}</p>
</template>
```

Roughly, the guide goes like this: chapter 1 explains how the composable works. Chapter 2 records and reads the result, chapter 3 looks at the audio that is sent and the contract with the server, chapter 4 saves a recording to the device and chapter 5 covers every failure. Chapter 6 collects mistakes that are easy to make, and chapter 7 builds a dictated note. The appendices are meant for quick lookups.

## Contents

1. [How useMicrophone works](#1-how-usemicrophone-works)
2. [Recording and reading the result](#2-recording-and-reading-the-result)
3. [The audio that is sent](#3-the-audio-that-is-sent)
4. [Saving the recording](#4-saving-the-recording)
5. [When things go wrong](#5-when-things-go-wrong)
6. [Using it carefully](#6-using-it-carefully)
7. [A dictated note](#7-a-dictated-note)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Every example is a complete page of a Nuxt project, and each is a single `.vue` file. In Nuxt `ref`, `computed`, `onBeforeUnmount` and `useMicrophone` need no `import`, so none is written.

---

## 1. How useMicrophone works

To keep later chapters short, here are the terms they rely on:

- **Recording.** The browser captures the microphone in chunks for as long as the recording runs. The format is the browser's own, such as WebM, and the audio is kept in memory.
- **Conversion.** When the recording stops, the audio is decoded, resampled to 16 kHz, mixed down to one channel and written out as a WAV file.
- **Recognition.** The WAV file is uploaded to the endpoint you give, and the server returns the text in its answer.

What the composable exposes:

| Piece | Role |
|-------|------|
| `useMicrophone({ endpoint })` | Creates one recorder with its own state |
| `startRecording()` | Asks for the microphone and starts |
| `stopRecording()` | Stops, converts, uploads, and resolves when all of that is done |
| `downloadRecording(fileName)` | Saves the last WAV to the visitor's device |
| `isRecording` | `true` while the microphone is recording |
| `isProcessing` | `true` while the recording is converted and uploaded |
| `recognizedText` | The text the server answered with |
| `audioBlob` | The converted WAV, as a `Blob` |
| `error` | The message of the last failure, or `null` |

Remember six things:

- **The state is held in refs.** Read `.value` in code; in a template the refs of a destructured result are unwrapped by themselves.
- **No exceptions are thrown.** `startRecording` and `stopRecording` put failures into `error` and carry on.
- **`startRecording` clears the old result.** `error`, `recognizedText` and `audioBlob` are reset whenever a recording starts.
- **`stopRecording` returns a promise** that settles after conversion and upload. `isRecording` becomes `false` a little after the call, not immediately.
- **The WAV is kept even when the upload fails,** so a failed recognition can still be saved or sent again.
- **Each `useMicrophone()` has its own state.** Two recorders on one page share nothing.

## 2. Recording and reading the result

The doctor presses Record, speaks, presses Stop and waits for words. The page's job is to follow that path and to read the result at the right moment. The page below logs the state at each step of one recording:

```vue
<script setup>
const mic = useMicrophone({ endpoint: 'https://stt.test/ok' })
const log = ref([])

const snapshot = () => `recording=${mic.isRecording.value} processing=${mic.isProcessing.value} text=${JSON.stringify(mic.recognizedText.value)} error=${JSON.stringify(mic.error.value)} blob=${mic.audioBlob.value ? 'yes' : 'no'}`
const mark = label => log.value.push(`${label}: ${snapshot()}`)

onMounted(() => mark('at start'))

async function start() { await mic.startRecording(); mark('after startRecording') }
async function stop() {
  const pending = mic.stopRecording()
  mark('right after stopRecording()')
  await pending
  mark('after the promise resolved')
}
</script>

<template>
  <button id="start" @click="start">Start</button>
  <button id="stop" @click="stop">Stop</button>
  <div v-for="(l, i) in log" :key="i" class="row">{{ l }}</div>
</template>
```

```text
at start: recording=false processing=false text="" error=null blob=no
after startRecording: recording=true processing=false text="" error=null blob=no
right after stopRecording(): recording=true processing=false text="" error=null blob=no
after the promise resolved: recording=false processing=false text="hello from the microphone" error=null blob=yes
request: POST /ok multipart/form-data file-field=True filename=True wav=('RIFF/WAVE', (1, 1, 16000, 32000, 2, 16))
```

At the start everything is empty or `false`, and `error` is `null`. After `startRecording` the microphone is open and `isRecording` is `true`. The unexpected part comes at the stop: right after calling `stopRecording()`, `isRecording` is **still `true`**, because the browser stops the recorder asynchronously and the ref changes only when it has stopped. The result is ready only when the promise resolves. By then the text has arrived and the WAV is in `audioBlob`, since conversion and upload both happen inside the promise. One request is sent: a `POST` with a `multipart/form-data` body that holds a file named `audio.wav` in the field `file`.

The rule is simple: **read the result after the `await`, never right after the call.** And decide what to do with an empty answer, because an empty text is not an error:

```js
await mic.stopRecording()

if (mic.error.value) {
  showMessage(mic.error.value)
} else if (mic.recognizedText.value === '') {
  showMessage('Nothing was recognized, please try again.')
} else {
  note.value += ' ' + mic.recognizedText.value
}
```

## 3. The audio that is sent

Speech recognizers are strict about audio. Browsers record in whatever format they choose (usually WebM with a compressed codec), while the recognizer of the J2OS back end wants a plain WAV at 16 kHz. The composable bridges the two: when the recording stops it decodes the audio, resamples it, mixes it down to one channel and writes the WAV itself. The page below records for two seconds and reads the header of the resulting file:

```vue
<script setup>
const mic = useMicrophone({ endpoint: 'https://stt.test/ok' })
const rows = ref([])

const ascii = (view, at, n) => Array.from({ length: n }, (_, i) => String.fromCharCode(view.getUint8(at + i))).join('')

async function describe(blob) {
  const view = new DataView(await blob.arrayBuffer())
  const dataBytes = view.getUint32(40, true)
  return [
    `type: ${blob.type}`,
    `starts with: ${ascii(view, 0, 4)} ${ascii(view, 8, 4)} ${ascii(view, 12, 4)}`,
    `format: ${view.getUint16(20, true) === 1 ? 'PCM' : 'other'}, ${view.getUint16(22, true)} channel, ${view.getUint32(24, true)} Hz, ${view.getUint16(34, true)} bit`,
    `header matches the size: ${view.getUint32(4, true) === blob.size - 8 && dataBytes === blob.size - 44}`,
    `length: about ${Math.round(dataBytes / 2 / view.getUint32(24, true))} s`
  ]
}

async function run() {
  await mic.startRecording()
  await new Promise(resolve => setTimeout(resolve, 2000))
  await mic.stopRecording()
  rows.value = await describe(mic.audioBlob.value)
}
</script>

<template>
  <button id="run" @click="run">Record two seconds</button>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
</template>
```

```text
type: audio/wav
starts with: RIFF WAVE fmt
format: PCM, 1 channel, 16000 Hz, 16 bit
header matches the size: true
length: about 2 s
upload: multipart/form-data, file field True, file name audio.wav True, RIFF/WAVE, (1, 1, 16000, 32000, 2, 16)
```

The format is fixed: PCM, one channel, 16000 Hz, 16 bits per sample, regardless of what format the browser recorded and of the microphone's own rate and channel count. The file is well-formed (the sizes in the header match the size of the blob), its type is `audio/wav`, and the file that is uploaded has the same header.

The size depends on the length. At 16 000 samples per second and 2 bytes each, a minute of speech is about 1.9 MB. The whole recording is held in memory, at least twice (the browser's own recording and the WAV), until the page lets go of it. That is fine for a note, but not suitable for recording a lecture.

The contract with the server is short:

```text
POST  <endpoint>
Content-Type: multipart/form-data

  file  = audio.wav      (16000 Hz, mono, 16-bit PCM)

200 OK
{ "text": "the recognized words" }
```

The J2OS back end provides the server side too: a Spring controller around the `SpeechToText` module:

```java
@PostMapping("/recognize")
public ResponseEntity<Map<String, String>> recognize(@RequestParam("file") MultipartFile file)
        throws SpeechToTextFailedException, IOException {
    String text = speechToText.recognize(file.getInputStream());
    return ResponseEntity.ok(Map.of("text", text));
}
```

Three parts of the contract decide how the page is built:

- **`endpoint` is whatever address you pass.** It can be a relative path (`/api/speech-to-text/recognize`) or a full address; if the origin is different, the server must allow it (CORS).
- **The answer must be JSON,** and only its `text` is used. If `text` is missing, the result is an empty string and no error.
- **The upload adds no headers.** If the server needs a token, the composable has no place to put one; put the endpoint behind a same-origin route that adds it.

## 4. Saving the recording

A long dictation is valuable, and a server error should not lose it. The converted WAV stays in `audioBlob` even when recognition fails, and `downloadRecording` saves it on the visitor's device. It does nothing when there is no recording yet:

```vue
<script setup>
const mic = useMicrophone({ endpoint: 'https://stt.test/ok' })
const note = ref('')

async function record() {
  await mic.startRecording()
  await new Promise(resolve => setTimeout(resolve, 800))
  await mic.stopRecording()
}
</script>

<template>
  <button id="save-early" @click="mic.downloadRecording('early.wav')">Save before recording</button>
  <button id="record" @click="record">Record</button>
  <button id="save" @click="mic.downloadRecording('note.wav')">Save</button>
  <button id="save-default" @click="mic.downloadRecording()">Save with the default name</button>
  <p>{{ note }}</p>
</template>
```

```text
save before any recording: no file was created
save -> file name note.wav, WAVE, 16000 Hz
save-default -> file name recording.wav, WAVE, 16000 Hz
```

Before any recording, `downloadRecording` has no effect and no file is created. With a name, the file is saved under that name; without one it is `recording.wav`. The saved file is the converted 16 kHz WAV, not the browser's original recording, and nothing is uploaded, since saving works from the data already in the page.

This is why a "Save audio" button next to the recognized text is a useful fallback: when recognition fails, the visitor keeps the audio and can send it again later.

## 5. When things go wrong

A page that records must expect failures, and the composable makes them easy to handle: the message of everything is stored in `error`, as text, and no exception is thrown. The page below runs six situations, each with a fresh recorder, and the last lines list what has reached the server:

```vue
<script setup>
const rows = ref([])
const fresh = endpoint => useMicrophone({ endpoint })

async function attempt(label, endpoint, { stopFirst = false, before } = {}) {
  const mic = fresh(endpoint)
  if (before) before()
  if (!stopFirst) await mic.startRecording()
  await new Promise(resolve => setTimeout(resolve, 400))
  await mic.stopRecording()
  rows.value.push(`${label}: recording=${mic.isRecording.value} text=${JSON.stringify(mic.recognizedText.value)} error=${JSON.stringify(mic.error.value)} blob=${mic.audioBlob.value ? 'yes' : 'no'}`)
}

async function run() {
  await attempt('server answers 500', 'https://stt.test/fail')
  await attempt('answer has no text', 'https://stt.test/empty')
  await attempt('answer is not JSON', 'https://stt.test/html')
  await attempt('server unreachable', 'https://stt.test/drop')
  await attempt('stop without start', 'https://stt.test/ok', { stopFirst: true })
  await attempt('no endpoint given', undefined)
}
</script>

<template>
  <button id="run" @click="run">Run</button>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
</template>
```

```text
server answers 500: recording=false text="" error="Server responded with 500" blob=yes
answer has no text: recording=false text="" error=null blob=yes
answer is not JSON: recording=false text="" error="Unexpected token '<', \"<html>\" is not valid JSON" blob=yes
server unreachable: recording=false text="" error="Failed to fetch" blob=yes
stop without start: recording=false text="" error=null blob=no
no endpoint given: recording=false text="" error="Server responded with 404" blob=yes
request: POST /fail multipart/form-data file-field=True filename=True wav=('RIFF/WAVE', (1, 1, 16000, 32000, 2, 16))
request: POST /empty multipart/form-data file-field=True filename=True wav=('RIFF/WAVE', (1, 1, 16000, 32000, 2, 16))
request: POST /html multipart/form-data file-field=True filename=True wav=('RIFF/WAVE', (1, 1, 16000, 32000, 2, 16))
request: POST /drop multipart/form-data file-field=True filename=True wav=('RIFF/WAVE', (1, 1, 16000, 32000, 2, 16))
```

These six situations fall into three kinds. When the **server answers wrongly**, a bad status puts `Server responded with 500` in `error` (the recording is not lost, `audioBlob` is set) and an answer that is not JSON gives the browser's own parsing message. An answer with no `text` gives no error at all and `recognizedText` stays empty, so an empty text can mean "nothing was recognized" as well as "the server answered something else". When the **request cannot reach the server**, `error` is the browser's own network error (`Failed to fetch` in Chromium). When the **page misuses the composable**, `stopRecording` without a recording resolves at once and changes nothing, and a missing `endpoint` sends the upload to the relative address `undefined`, which usually ends as `Server responded with 404`. Always pass `endpoint`.

Failures to start are stored in `error` in the same way. The two ways a microphone fails to open:

```vue
<script setup>
const mic = useMicrophone({ endpoint: 'https://stt.test/ok' })
const rows = ref([])

async function ask() {
  await mic.startRecording()
  rows.value.push(`recording=${mic.isRecording.value} error=${JSON.stringify(mic.error.value)}`)
}
</script>

<template>
  <button id="ask" @click="ask">Start</button>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
</template>
```

```text
no microphone connected: recording=false error="Requested device not found"
permission refused: recording=false error="Permission denied"
```

With no microphone connected the browser says so (`Requested device not found` in Chromium). When the visitor refuses, the message is the browser's refusal (`Permission denied`); `isRecording` stays `false` and nothing is left open. These texts belong to the browser, so they differ between browsers and languages. Show your own sentence for the cases you care about, and keep the original for the log.

## 6. Using it carefully

The composable is small and does not question the page. It does not guard against three mistakes that are easy to make. The page below starts twice in a row, stops, stops again and records once more:

```vue
<script setup>
const mic = useMicrophone({ endpoint: 'https://stt.test/ok' })
const rows = ref([])
const streams = []

// Remember every microphone stream the browser hands out, to see which ones are still live.
const original = navigator.mediaDevices.getUserMedia.bind(navigator.mediaDevices)
navigator.mediaDevices.getUserMedia = async (constraints) => {
  const stream = await original(constraints)
  streams.push(stream)
  return stream
}

const live = () => streams.map(s => s.getTracks().filter(t => t.readyState === 'live').length)

async function run() {
  await mic.startRecording()
  await mic.startRecording()
  rows.value.push(`two starts: streams=${streams.length}, live tracks per stream=${JSON.stringify(live())}`)
  await new Promise(resolve => setTimeout(resolve, 500))
  await mic.stopRecording()
  rows.value.push(`after stop: live tracks per stream=${JSON.stringify(live())}`)
  const outcome = await Promise.race([
    mic.stopRecording().then(() => 'resolved', e => `rejected (${e.name})`),
    new Promise(resolve => setTimeout(() => resolve('still waiting after 1.5 s'), 1500))
  ])
  rows.value.push(`second stop: ${outcome}`)
  await mic.startRecording()
  await new Promise(resolve => setTimeout(resolve, 500))
  await mic.stopRecording()
  rows.value.push(`record again: text=${JSON.stringify(mic.recognizedText.value)} streams=${streams.length} live=${JSON.stringify(live())}`)
}
</script>

<template>
  <button id="run" @click="run">Run</button>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
</template>
```

```text
two starts: streams=2, live tracks per stream=[1,1]
after stop: live tracks per stream=[1,0]
second stop: still waiting after 1.5 s
record again: text="hello from the microphone" streams=3 live=[1,0,0]
```

**Starting twice leaves a microphone open.** The second start opens a second stream and forgets the first, so after the stop one stream is still live, and the browser's microphone indicator stays on until the page is closed. **Stopping twice never finishes:** the second `stopRecording()` returns a promise that does not settle, even after 1.5 seconds. Recording again works and replaces the old text, error and WAV. The fix for both is to start only when idle and stop only while recording:

```js
const { isRecording, isProcessing, startRecording, stopRecording } = useMicrophone({ endpoint })

// Start only when idle, stop only when recording.
const start = () => { if (!isRecording.value && !isProcessing.value) return startRecording() }
const stop = () => { if (isRecording.value) return stopRecording() }
```

The third mistake is leaving the page. The composable has no automatic cleanup. The page below shows a child component that starts recording when it appears and is then removed; its microphone track is still live afterwards. The first line of the log shows the request the composable makes for the microphone:

```vue
<script setup>
const shown = ref(true)
const rows = ref([])
const streams = []
const constraints = []
const original = navigator.mediaDevices.getUserMedia.bind(navigator.mediaDevices)
navigator.mediaDevices.getUserMedia = async (c) => {
  constraints.push(JSON.stringify(c))
  const s = await original(c)
  streams.push(s)
  return s
}
const Child = {
  setup() {
    const mic = useMicrophone({ endpoint: 'https://stt.test/ok' })
    mic.startRecording()
    return () => h('span', 'recording child')
  }
}
const live = () => streams.map(s => s.getTracks().filter(t => t.readyState === 'live').length)
async function run() {
  await new Promise(r => setTimeout(r, 600))
  rows.value.push(`constraints asked: ${constraints[0]}`)
  rows.value.push(`child on screen: live tracks=${JSON.stringify(live())}`)
  shown.value = false
  await new Promise(r => setTimeout(r, 600))
  rows.value.push(`child removed: live tracks=${JSON.stringify(live())}`)
}
</script>

<template>
  <button id="run" @click="run">Run</button>
  <Child v-if="shown" />
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
</template>
```

```text
constraints asked: {"audio":{"echoCancellation":true,"noiseSuppression":true,"autoGainControl":true}}
child on screen: live tracks=[1]
child removed: live tracks=[1]
```

If the component can go away while recording, stop it yourself:

```js
const mic = useMicrophone({ endpoint })

// The composable does not clean up after itself: leaving the page while
// recording would keep the microphone open.
onBeforeUnmount(() => {
  if (mic.isRecording.value) mic.stopRecording()
})
```

## 7. A dictated note

Putting the pieces together gives a small feature: a note that is written by voice. One button records and stops, shows the state in its label, and is disabled while the recognition runs. The recognized words are appended to a text box the user can edit, a failure appears as a message, and the audio can be saved:

```vue
<script setup>
const mic = useMicrophone({ endpoint: 'https://stt.test/ok' })
const note = ref('')

const label = computed(() => {
  if (mic.isProcessing.value) return 'Recognizing…'
  return mic.isRecording.value ? 'Stop' : 'Record a note'
})

async function toggle() {
  if (mic.isRecording.value) {
    await mic.stopRecording()
    // Only fill in the note when something was recognized; keep what was typed otherwise.
    if (mic.recognizedText.value) note.value = (note.value ? note.value + ' ' : '') + mic.recognizedText.value
  } else {
    await mic.startRecording()
  }
}
</script>

<template>
  <button id="toggle" :disabled="mic.isProcessing.value" @click="toggle">{{ label }}</button>
  <button id="save" :disabled="!mic.audioBlob.value" @click="mic.downloadRecording('note.wav')">Save audio</button>
  <p v-if="mic.error.value" id="error">Could not record: {{ mic.error.value }}</p>
  <textarea id="note" v-model="note" />
</template>
```

```text
start: button 'Record a note', save disabled True
recording: button 'Stop'
sending: button 'Recognizing…'
done: note 'hello from the microphone', save disabled False
second note: 'hello from the microphone hello from the microphone'
```

The button's label follows the state: "Record a note", "Stop", then "Recognizing…" while the promise is pending. After the recognition the words are added to the note and "Save audio" becomes available. A second dictation is added to the first, because the page appends and the composable only holds the last result.

The pattern worth copying is that **the page decides what to keep**. It appends only a non-empty text, so a failed or empty recognition never wipes what the user has typed, and errors are shown in a message line that appears only when `error` has a value. Add the guards of chapter 6 and the endpoint rules of chapter 3, and the note is ready for real use.

---

## Appendix A. Quick reference

### A.1 Creating and using

| Call | Returns | Notes |
|------|---------|-------|
| `useMicrophone({ endpoint })` | The object below | `endpoint` is the upload address; always give it |
| `startRecording()` | Promise | Clears the old result; never rejects |
| `stopRecording()` | Promise | Settles after conversion and upload; resolves at once without a recording |
| `downloadRecording(fileName)` | Nothing | Default name `recording.wav`; does nothing without a recording |

### A.2 State

| Ref | Meaning |
|-----|---------|
| `isRecording` | `true` while recording |
| `isProcessing` | `true` during conversion and upload |
| `recognizedText` | The server's `text`, or an empty string |
| `audioBlob` | The WAV, or `null` |
| `error` | The last failure's message, or `null` |

### A.3 What is sent

| Part | Value |
|------|-------|
| Method | `POST` |
| Body | `multipart/form-data` |
| Field | `file`, with the file name `audio.wav` |
| Audio | WAV, PCM, 16 kHz, mono, 16-bit |
| Expected answer | JSON with `text` |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Microphone refused or missing | `error` holds the browser's message; `isRecording` is `false` |
| Server answers with an error status | `error` is `Server responded with ` and the status; `audioBlob` is set |
| Answer is not JSON | `error` holds the browser's parsing message; `audioBlob` is set |
| Server unreachable | `error` holds the browser's network message; `audioBlob` is set |
| Answer without `text` | No error; `recognizedText` is empty |
| Browser cannot decode its own recording | `error` holds the browser's message; no `audioBlob` |
| `stopRecording` without a recording | Resolves at once |
| `stopRecording` twice | The second promise never settles |
| `startRecording` twice | The first microphone stream stays open |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Forgetting `endpoint`.** The upload goes to a relative address named `undefined`.
- **Reading the result right after `stopRecording()`.** Await the promise first.
- **Starting while recording, or while processing.** Leaves a stream open or mixes results.
- **Stopping twice.** The second promise hangs; a page that awaits it hangs too.
- **Leaving the page while recording.** The microphone stays on.
- **Treating an empty text as definitely an error, or definitely a success.** It can mean nothing was recognized, or the answer had no `text`.
- **Showing the browser's error text to visitors.** It is technical and differs between browsers.
- **Long recordings.** The audio is kept in memory and uploaded in one request.
- **Using it on a plain HTTP page.** Browsers give out the microphone only on secure pages (HTTPS, or `localhost`).
- **Sending a token.** There is no way to add headers to the upload.

### C.2 Behavior worth knowing

- The request for the microphone asks for echo cancellation, noise suppression and automatic gain.
- `isRecording` becomes `true` after the microphone has been opened, and `false` after the recorder has stopped.
- The microphone is released when the recording stops, and also when a start fails.
- The WAV is built in memory with the browser's audio engine, so the page works offline up to the upload.
- Each `useMicrophone()` call keeps its own recorder and refs.
- The composable never throws an exception to the caller; the real risk is a stuck state, not an exception.

### C.3 Limits

| What | Limit |
|------|-------|
| Output audio | 16 kHz, mono, 16-bit WAV only |
| Length | Memory of the device; about 1.9 MB of WAV per minute |
| Requests | One upload per recording, no streaming |
| Headers, cookies | None added |
| Languages | Whatever the server's recognizer supports |
| Browsers | Those with microphone capture, `MediaRecorder` and Web Audio |

### C.4 Tuning

- Pass `endpoint` always, and put it behind your own same-origin route when authentication is needed.
- Guard start and stop with the state refs, as in chapter 6.
- Stop in `onBeforeUnmount`.
- Disable the button while `isProcessing` is `true`.
- Offer "Save audio" next to the result, so a failed upload does not lose the recording.
- Append recognized text instead of replacing what the user typed.
- Translate the common errors into your own sentences.
- Keep recordings short; split long dictations into several.

## Appendix D. Message reference

### D.1 Errors in `error`

| Message | Cause |
|---------|-------|
| `Server responded with 500` (any status) | The endpoint answered with an error status |
| The browser's own message, for example `Permission denied` or `Requested device not found` | The microphone could not be opened |
| The browser's own message, for example `Failed to fetch` | The upload did not reach the server |
| The browser's parsing message | The answer is not JSON |
