# useSpeechRecognitionAPI

*J2OS Platform 2.4*

A manager dictates the minutes of a meeting instead of typing them. Words should appear on the screen as they are spoken, and at the end the whole text should be saved to a file. The browser can already listen and turn speech into text; the page only has to start it, collect the results and handle the many ways it can stop.

**useSpeechRecognitionAPI** is a small composable of the Nuxt front end of J2OS that wraps the browser's speech recognition in a simple interface. It has three features:

- **Callbacks, not polling.** The page hands over three functions, and the composable calls the matching one for each piece of speech, each error and the end of a session.
- **Persian, live.** The language is Persian (`fa-IR`), the session listens continuously, and half-finished phrases arrive as well as finished ones.
- **A companion transcript.** Every finished phrase is also written into a text kept inside the composable, which can be saved to a file.

In a few lines:

```js
const speech = useSpeechRecognitionAPI()

onMounted(() => {
  speech.init(
    (time, text, isFinal) => console.log(time, text, isFinal),
    (message) => console.warn(message),
    () => console.log('the session ended')
  )
})

// later, from two buttons
speech.start()
speech.stop()
```

This guide goes step by step: chapter 1 explains how the composable works and chapter 2 does the first dictation. Chapter 3 covers state and timing, chapter 4 failures and chapter 5 the saved transcript. Chapter 6 builds a dictation box that restarts itself and cleans up after itself. The appendices are a quick reference.

## Contents

1. [How useSpeechRecognitionAPI works](#1-how-usespeechrecognitionapi-works)
2. [The first dictation](#2-the-first-dictation)
3. [State and timing](#3-state-and-timing)
4. [When things go wrong](#4-when-things-go-wrong)
5. [The transcript](#5-the-transcript)
6. [A dictation box](#6-a-dictation-box)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are complete pages of a Nuxt project: each is one `.vue` file. In Nuxt `ref`, `onMounted`, `onBeforeUnmount` and `useSpeechRecognitionAPI` need no `import`, so none is written.

---

## 1. How useSpeechRecognitionAPI works

First, a few terms, because they come back throughout the guide:

- **Speech recognition.** A feature of the browser that listens to the microphone and returns text. The composable does not recognise anything itself.
- **Session.** One round of listening, from `start()` until that round ends.
- **Interim result.** A phrase that is still being heard. It may change on the next update.
- **Final result.** A phrase that the browser has settled on. It does not change any more.
- **Callback.** A function you give to the composable, which it calls later.

The functions of this composable:

| Piece | Role |
|-------|------|
| `init(onResult, onError, onEnd)` | Checks the browser, prepares the engine and stores the three callbacks |
| `start()` | Starts a session |
| `stop()` | Asks the session to end |
| `isListening()` | Tells whether a session is running |
| `clear()` | Empties the saved transcript |
| `exportFile(fileName)` | Saves the transcript as a text file |

6 points to keep in mind:

- **The browser must have the feature.** If it does not, `init` throws "Please use Chrome." and the other functions do nothing afterwards.
- **Call `init` in `onMounted`.** `init` looks at `window`, which does not exist while Nuxt renders on the server.
- **The settings are fixed.** Language `fa-IR`, continuous listening and interim results are all set by the composable and cannot be changed from outside.
- **Results come through callbacks.** The composable keeps no reactive state; the page keeps its own.
- **Each call of the composable has its own engine and its own transcript.** Two pages that use it share nothing.
- **The details are up to the browser.** It needs the user's permission for the microphone, usually works only on secure (HTTPS) pages, and may send the audio to an online service; that is why a network error exists.

## 2. The first dictation

The manager presses Start, speaks, and expects words to appear. For that the page needs three things: the callbacks given to `init`, a way to start, and a place to show what arrives. The page below has a start button, a stop button and a list of everything the callbacks receive. A session hears two phrases (the first arrives in two steps) and is then stopped:

```vue
<script setup>
const speech = useSpeechRecognitionAPI()

const lines = ref([])
const status = ref('stopped')

onMounted(() => {
  speech.init(
    (time, text, isFinal) => lines.value.push(`${isFinal ? 'final' : 'interim'} | ${time} | ${text}`),
    (message) => lines.value.push('error | ' + message),
    () => { status.value = 'stopped' }
  )
})

function start() {
  speech.start()
  status.value = 'listening'
}
</script>

<template>
  <button id="start" @click="start">Start</button>
  <button id="stop" @click="speech.stop()">Stop</button>
  <p id="status">{{ status }}</p>
  <div v-for="(line, i) in lines" :key="i" class="row">{{ line }}</div>
</template>
```

```text
settings: ['fa-IR', True, True]
status: listening
settings after the first start: ['fa-IR', True, True]
status: stopped
interim | ۱۰:۳۰:۰۰ | سلام
interim | ۱۰:۳۰:۰۰ | سلام دنیا
final | ۱۰:۳۰:۰۰ | سلام دنیا
final | ۱۰:۳۰:۰۰ | خداحافظ
```

Before the first start the engine is set up with `fa-IR`, continuous listening and interim results. Then two interim results arrive. The second contains the first: an interim text is the whole phrase *so far*, so the page should **replace** the previous one, not add to it. A final result follows with the finished phrase, and then a second phrase. The `time` argument is the clock time at which the composable handled the piece, written in Persian digits. It is not the moment the words were spoken, so do not use it to align speech with a recording.

When the session ends, the status goes to "stopped" by itself. `onEnd` is the page's notification; it fires whatever the reason for the end.

The three callbacks look like this:

```js
function onResult(time, text, isFinal) {}   // a piece of speech: when, what, and whether it is settled
function onError(message) {}                // a readable Persian message, not the browser's code
function onEnd() {}                         // the session is over, for any reason
```

Calling `init` again replaces the callbacks and keeps the same engine, so a page that needs different callbacks later can simply call it again.

## 3. State and timing

A button that says "Start" must turn into "Stop" at the right moment, and the first idea is to ask the composable with `isListening()`. The catch is that starting and stopping are not instant: the browser starts and ends sessions in its own time, and the composable reports only the state the browser has announced. It therefore lags behind the calls, as the page below shows:

```vue
<script setup>
const speech = useSpeechRecognitionAPI()

const rows = ref([])
const wait = (ms) => new Promise(resolve => setTimeout(resolve, ms))

onMounted(async () => {
  const log = (label) => rows.value.push(`${label}: ${speech.isListening()}`)

  speech.start()
  log('start() before init(), then isListening')

  speech.init(() => {}, () => {}, () => {})
  log('after init(), nothing started yet')

  speech.start()
  log('right after start()')
  await wait(100)
  log('100 ms later')

  speech.start()
  log('start() again while running')

  speech.stop()
  log('right after stop()')
  await wait(100)
  log('100 ms later')
})
</script>

<template>
  <p>isListening() at each step</p>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
</template>
```

```text
start() before init(), then isListening: false
after init(), nothing started yet: false
right after start(): false
100 ms later: true
start() again while running: true
right after stop(): true
100 ms later: false
engines created: 1
console warnings: ['Speech recognition is already running: InvalidStateError: Failed to ex']
```

Right after `start()`, `isListening()` is still `false`; it becomes `true` a moment later, when the browser announces the start. In the same way, right after `stop()` it is still `true`, and `false` a moment later. `stop()` is only a request, and the browser may deliver a last result before it ends. Two more cases are quiet: `start()` before `init()` does nothing and raises no error (there is no engine yet), and a second `start()` while running throws nothing, because the composable catches the browser's exception and writes a warning to the console.

So a button that must react at once cannot rely on `isListening()`. Keep your own flag, set it when the user clicks and clear it in `onEnd`:

```js
const listening = ref(false)

function start() {
  speech.start()
  listening.value = true          // isListening() is still false at this moment
}

// and in init(): onEnd = () => { listening.value = false }
```

There is no "started" callback. The page's own flag is the state that is shown, and `onEnd` is the one reliable signal that the session is over.

## 4. When things go wrong

Speech recognition fails more often than most features, and in two different places. The browser may have no speech recognition at all, and a session may fail while running. The first shows up in `init`, the second in `onError`:

```vue
<script setup>
const speech = useSpeechRecognitionAPI()

const status = ref('checking')
const note = ref('')

onMounted(() => {
  try {
    speech.init(() => {}, () => {}, () => {})
    status.value = 'ready'
  } catch (e) {
    status.value = 'unsupported: ' + e.message
  }
})

function tryStart() {
  try {
    speech.start()
    note.value = 'start() returned without an error'
  } catch (e) {
    note.value = 'start() threw: ' + e.message
  }
}
</script>

<template>
  <p id="status">{{ status }}</p>
  <button id="start" @click="tryStart">Start</button>
  <p id="note">{{ note }}</p>
</template>
```

```text
with speech recognition: ready
start() returned without an error | engine running: True
without: unsupported: Please use Chrome.
start() returned without an error
```

With the feature, `init` finishes and `start()` begins a session. Without it, `init` throws "Please use Chrome." and `start()` afterwards returns quietly, because there is no engine to start. A page should catch the error from `init` and hide or disable its buttons, instead of offering buttons that do nothing.

Errors during a session reach `onError`. The composable turns the browser's error code into a Persian message first:

```vue
<script setup>
const speech = useSpeechRecognitionAPI()

const messages = ref([])

onMounted(() => {
  speech.init(
    () => {},
    (message) => messages.value.push(message),
    () => {}
  )
  speech.start()
})
</script>

<template>
  <p>Messages received: {{ messages.length }}</p>
  <div v-for="(m, i) in messages" :key="i" class="row">{{ m }}</div>
</template>
```

```text
صدایی شنیده نشد. لطفاً دوباره تلاش کنید.
میکروفون پیدا نشد یا در دسترس نیست.
دسترسی به میکروفون رد شد.
خطای شبکه در ارتباط با سرویس تشخیص گفتار.
تشخیص گفتار لغو شد.
زبان انتخاب‌شده پشتیبانی نمی‌شود.
سرویس تشخیص گفتار در دسترس نیست.
خطای ناشناخته: bad-grammar
```

Seven known codes (no speech, no microphone, permission denied, network, aborted, language not supported, service not allowed) each have a message, and any other code becomes "خطای ناشناخته:" (unknown error) followed by the raw code. The important detail is that **only the message arrives**: the page gets text, not the code, so it cannot reliably tell a harmless "no speech" from a "permission denied" without comparing messages. Show the text to the user, and treat any error as a reason not to restart the session automatically (chapter 6).

An error is not always the end of a session. After an error the browser normally ends the session too, and then `onEnd` runs, but the page that must be sure should handle both.

## 5. The transcript

Dictation is rarely only for display on the screen; the minutes have to be kept. Each final result is added to a text kept inside the composable, one line per phrase: the time in square brackets, the text, and a Windows-style line break. Interim results are not kept, because they are not settled:

```vue
<script setup>
const speech = useSpeechRecognitionAPI()

const shown = ref([])
const fileName = ref('')

onMounted(() => {
  speech.init(
    (time, text, isFinal) => { if (isFinal) shown.value.push(text) },
    () => {},
    () => {}
  )
  speech.start()
})

function clearAll() {
  speech.clear()
  shown.value = []
}
</script>

<template>
  <input id="name" v-model="fileName">
  <button id="export" @click="speech.exportFile(fileName)">Export</button>
  <button id="clear" @click="clearAll">Clear</button>
  <div v-for="(t, i) in shown" :key="i" class="row">{{ t }}</div>
</template>
```

```text
shown: ['سلام دنیا', 'این یک آزمایش است']
empty name -> file name 'فایل_۱۴۰۵_۷_۲۳.txt', content '[۱۰:۳۰:۰۰] سلام دنیا\r\n[۱۰:۳۰:۰۰] این یک آزمایش است\r\n'
custom name -> file name 'meeting.txt', content '[۱۰:۳۰:۰۰] سلام دنیا\r\n[۱۰:۳۰:۰۰] این یک آزمایش است\r\n'
after clear -> file name 'meeting.txt', content ''
```

Only final phrases are in the file: two lines, though four results have arrived. The default file name is the word "فایل", an underscore and today's date in the Persian calendar; a given name is used as it is, so include the extension. The file is plain UTF-8 text, built in the browser and offered as a download; nothing is uploaded.

Two behaviours matter when the page has a "new minutes" button:

- **`clear()` empties the transcript, not the page.** The list on the screen has to be cleared by the page itself, as this page does. In the output, the export after clearing is an empty file.
- **Sessions add up.** The transcript is not emptied by `stop()` or by a new `start()`; it grows until `clear()`. Call `clear()` when a new document begins.

## 6. A dictation box

Everything so far comes together in one realistic page. The written text grows with every final phrase, the phrase being spoken shows below it, and the page keeps listening for as long as the user wants it to, even though the browser may end a session on its own. A permission error stops that loop, and leaving the page stops the engine:

```vue
<script setup>
const speech = useSpeechRecognitionAPI()

const supported = ref(true)
const note = ref('')
const written = ref('')
const heard = ref('')
const wanted = ref(false)
const failed = ref(false)
const restarts = ref(0)

onMounted(() => {
  try {
    speech.init(onResult, onError, onEnd)
  } catch (e) {
    supported.value = false
    note.value = e.message
  }
})

onBeforeUnmount(() => {
  wanted.value = false
  speech.stop()
})

function onResult(time, text, isFinal) {
  if (isFinal) {
    written.value += (written.value ? ' ' : '') + text.trim()
    heard.value = ''
  } else {
    heard.value = text
  }
}

function onError(message) {
  failed.value = true
  wanted.value = false
  note.value = message
}

function onEnd() {
  heard.value = ''
  if (wanted.value && !failed.value) {
    restarts.value++
    speech.start()
  }
}

function begin() {
  failed.value = false
  note.value = ''
  wanted.value = true
  speech.start()
}

function finish() {
  wanted.value = false
  speech.stop()
}
</script>

<template>
  <p v-if="!supported" id="note">{{ note }}</p>
  <template v-else>
    <button id="begin" @click="begin">Dictate</button>
    <button id="finish" @click="finish">Done</button>
    <button id="save" @click="speech.exportFile('notes.txt')">Save</button>
    <button id="clear" @click="speech.clear(); written = ''">Clear</button>
    <p id="written">{{ written }}</p>
    <p id="heard">{{ heard }}</p>
    <p id="note">{{ note }}</p>
    <p id="restarts">{{ restarts }}</p>
  </template>
</template>
```

```text
interim | written='' | heard='سلام' | note='' | restarts=0 | engine running=True
final, then interim | written='سلام دنیا' | heard='این یک' | note='' | restarts=0 | engine running=True
second final | written='سلام دنیا این یک آزمایش است' | heard='' | note='' | restarts=0 | engine running=True
browser ended the session | written='سلام دنیا این یک آزمایش است' | heard='' | note='' | restarts=1 | engine running=True
Done pressed | written='سلام دنیا این یک آزمایش است' | heard='' | note='' | restarts=1 | engine running=False
permission denied | written='سلام دنیا این یک آزمایش است' | heard='' | note='دسترسی به میکروفون رد شد.' | restarts=1 | engine running=False
saved 'notes.txt': '[۱۰:۳۰:۰۰] سلام دنیا\r\n[۱۰:۳۰:۰۰] این یک آزمایش است\r\n'
after Clear | written='' | heard='' | note='دسترسی به میکروفون رد شد.' | restarts=1 | engine running=False
without support: Please use Chrome.
```

Follow the output from the top. Interim text appears in the "heard" line and is cleared when the phrase becomes final and moves to the written text. When the browser ends the session by itself, the page starts a new one (restarts: 1), because the user still wants to dictate and nothing has failed. The key is the page's own *wish*: **Done** sets the wish to false first and then calls `stop()`, so the end that follows does not restart anything; and a permission error sets the failure flag and clears the wish, so the end that follows is only an end. **Save** gives the transcript, and **Clear** empties the written text together with the transcript. Without support, the page shows the message from `init` and draws no buttons.

On leaving the page, stop the engine, so that the microphone is not kept busy:

```js
onBeforeUnmount(() => speech.stop())      // the page is going away
onDeactivated(() => speech.stop())        // a kept-alive page is hidden
```

The page above does the first of the two; use both when the page is kept alive between visits.

---

## Appendix A. Quick reference

### A.1 The functions

| Function | Returns | Notes |
|----------|---------|-------|
| `init(onResult, onError, onEnd)` | Nothing | Throws "Please use Chrome." without browser support; creates the engine once; replaces the callbacks each time |
| `start()` | Nothing | Does nothing before `init`; a start while running only writes a console warning |
| `stop()` | Nothing | Asks the session to end; `onEnd` follows |
| `isListening()` | `true` or `false` | Follows the browser's start and end events, so it lags |
| `clear()` | Nothing | Empties the transcript |
| `exportFile(fileName)` | Nothing | Downloads the transcript as a text file |

### A.2 The callbacks

| Callback | Arguments | When |
|----------|-----------|------|
| `onResult` | `time`, `text`, `isFinal` | For every interim and final piece; `time` is a clock time in Persian digits |
| `onError` | `message` | When the engine reports an error; the message is Persian text |
| `onEnd` | None | When a session ends, for any reason |

### A.3 Fixed settings

| Setting | Value |
|---------|-------|
| Language | `fa-IR` |
| Continuous listening | On |
| Interim results | On |
| Alternatives used | The first one only |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Browser without speech recognition | `init` throws `Error: Please use Chrome.` |
| `start()` or `stop()` before a successful `init` | Nothing happens, no error |
| `start()` while a session runs | A console warning, no exception |
| The engine reports an error | `onError` with a Persian message |
| An error code the composable does not know | `onError` with "خطای ناشناخته:" and the code |
| The session ends | `onEnd`, and `isListening()` becomes `false` |
| `exportFile` with an empty transcript | An empty text file is downloaded |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Calling `init` outside `onMounted`.** On the server `window` does not exist and the call fails.
- **Trusting `isListening()` right after a call.** It lags; keep a flag of your own.
- **Forgetting to stop.** Leaving the page without `stop()` keeps the microphone in use.
- **Adding interim texts together.** Each interim text is the whole phrase so far; replace, do not append.
- **Restarting blindly in `onEnd`.** After a permission error, a restart loops. Restart only when nothing failed.
- **Expecting another language.** The language is fixed to Persian.
- **Treating the transcript as the screen.** `clear()` does not clear the page, and `stop()` does not clear the transcript.
- **Expecting the file to hold interim text.** Only final phrases are saved.
- **Ignoring privacy.** The browser may send the audio to an online service; tell the user.

### C.2 Behaviour worth knowing

- The engine is created on the first `init` and reused after that; later calls only replace the callbacks.
- `time` is taken when the composable handles a piece, not when the words were spoken.
- Every piece of one update is delivered, starting from the first one that changed.
- Only the first alternative of each piece is used.
- Two uses of the composable have two engines and two transcripts.
- The transcript lives in memory only and is lost when the page is closed.
- The download link is created and used at once; nothing stays on the page.
- `start()` catches every exception the browser throws, not only "already running", and reports all of them with the same console warning.

### C.3 Limits

| What | Limit |
|------|-------|
| Languages | One, fixed (`fa-IR`) |
| Alternatives per phrase | One used |
| Browsers | Those that provide speech recognition (the composable names Chrome) |
| Callbacks | One set per composable call |
| Transcript | Final phrases only, in memory |

### C.4 Tuning

- Keep your own "listening" flag, set on click and cleared in `onEnd`.
- Restart in `onEnd` only when the user still wants to listen and no error was seen.
- Show interim text in a separate place from the final text.
- Stop the engine in `onBeforeUnmount` (and `onDeactivated` for kept-alive pages).
- Offer the export as a button, with a file name field, and clear both the transcript and the screen together.
- Tell the user that the microphone and, possibly, an online service are used.

## Appendix D. Message reference

### D.1 Messages passed to `onError`

| Browser code | Message (Persian) | Meaning |
|--------------|-------------------|---------|
| `no-speech` | صدایی شنیده نشد. لطفاً دوباره تلاش کنید. | Nothing was heard |
| `audio-capture` | میکروفون پیدا نشد یا در دسترس نیست. | No microphone found or available |
| `not-allowed` | دسترسی به میکروفون رد شد. | The user or browser refused the microphone |
| `network` | خطای شبکه در ارتباط با سرویس تشخیص گفتار. | The recognition service could not be reached |
| `aborted` | تشخیص گفتار لغو شد. | The recognition was cancelled |
| `language-not-supported` | زبان انتخاب‌شده پشتیبانی نمی‌شود. | The language is not supported |
| `service-not-allowed` | سرویس تشخیص گفتار در دسترس نیست. | The service is not allowed or not available |
| Any other | خطای ناشناخته: and the code | An unknown error |

### D.2 Other messages

| Message | Cause |
|---------|-------|
| `Please use Chrome.` (thrown by `init`) | The browser has neither `SpeechRecognition` nor `webkitSpeechRecognition` |
| `Speech recognition is already running:` (console warning) | The browser's `start` threw, usually because a session was running |
