# useTranslatorAPI

*J2OS Platform 2.4*

An admin types a product title in Persian, and the shop also needs it in English. A support agent wants to read a customer's message in another language. A quick machine translation, even if rough, saves a trip to another tab.

**useTranslatorAPI** is a small composable of the Nuxt front end of J2OS that translates a piece of text through a free public service, MyMemory. It has three main features:

- **One function.** `translate(text, from, to)` takes the text and two language codes and returns the translated text.
- **No setup.** No key, no account and no server code: the browser calls the service directly.
- **Plain failures.** A problem becomes an ordinary exception with a readable message, for the page to catch.

Short version:

```js
const translator = useTranslatorAPI()

const english = await translator.translate('سلام دنیا', 'fa', 'en')   // 'Hello world'
```

Roughly, the guide goes like this: chapter 1 explains how the composable works and chapter 2 shows the first translation. Chapter 3 covers what is sent, chapter 4 errors, chapter 5 fast typing and chapter 6 long texts. Chapter 7 builds a title field with a translate button. The appendices are meant for quick lookups.

## Contents

1. [How useTranslatorAPI works](#1-how-usetranslatorapi-works)
2. [The first translation](#2-the-first-translation)
3. [What is sent](#3-what-is-sent)
4. [When an error occurs](#4-when-an-error-occurs)
5. [Typing and outdated answers](#5-typing-and-outdated-answers)
6. [Long texts](#6-long-texts)
7. [In practice](#7-in-practice)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are full pages from a Nuxt project, and each is a single `.vue` file. In Nuxt `ref` and `useTranslatorAPI` need no `import`, so none is written.

---

## 1. How useTranslatorAPI works

These terms come up in every chapter, so here they are once:

- **Language code.** A short code for a language, such as `fa` for Persian and `en` for English. The service accepts ISO codes and RFC 3066 tags.
- **Language pair.** The source and the target together, written as `from|to`.
- **Quota.** How much text the service translates for free in a day.

Its one function:

| Function | Role |
|----------|------|
| `translate(text, from, to)` | Returns a promise of the translated text |

The whole interface is this one function. Five points to keep in mind:

- **The text leaves your system.** It travels over the internet to a third-party service. Do not send private or confidential text.
- **One call, one request.** Every call asks the service once. There is no cache, no retry and no time limit.
- **Empty text costs nothing.** An empty (or missing) text returns `''` without sending any request.
- **The answer is not checked.** The result is whatever the service sends back. Show it with `{{ }}`; do not put it into `v-html`.
- **The service has limits.** According to the service's documentation, a request may carry at most 500 bytes of text, and anonymous use is limited to 5000 characters a day. The composable sends no email or key, so it always counts as anonymous.

## 2. The first translation

Using the composable needs three things: a text, a source language and a target language. The call returns a promise of the translated text, so it belongs in an `async` function and is `await`ed. The page below has a box, a button and a place for the result:

```vue
<script setup>
const translator = useTranslatorAPI()
const text = ref('سلام دنیا')
const result = ref('')

async function run() {
  result.value = await translator.translate(text.value, 'fa', 'en')
}
</script>

<template>
  <input id="text" v-model="text">
  <button id="go" @click="run">Translate</button>
  <p id="result" dir="ltr">{{ result }}</p>
</template>
```

```text
result: Hello world
request: ['https://api.mymemory.translated.net/get?q=%D8%B3%D9%84%D8%A7%D9%85%20%D8%AF%D9%86%DB%8C%D8%A7&langpair=fa|en']
special characters, request: ['https://api.mymemory.translated.net/get?q=a%26b%3Dc%3Fd%23e%20%D9%88%20%DB%8C&langpair=fa|en']
result: [fa|en] a&b=c?d#e و ی
```

The result is a plain string. The composable keeps nothing between calls: no history, no current language, no cache. The page decides where the answer goes and how long it is kept, so the composable can be used in any component.

## 3. What is sent

It helps to know what leaves the page, both to understand the limits and to debug a wrong answer. Each call sends one `GET` request:

```text
GET https://api.mymemory.translated.net/get?q=<text>&langpair=<from>|<to>
```

The text and the two codes are encoded for the address, so `&`, `=`, `?` and `#` in the text are harmless, as the second request above shows. The pipe between the codes is not encoded.

The service answers with JSON, and the composable reads three fields of it:

```json
{
  "responseData": { "translatedText": "Hello world" },
  "responseStatus": 200,
  "responseDetails": ""
}
```

- **`responseStatus`** must be `200`. The comparison is not strict, so the string `"200"` is accepted too.
- **`responseDetails`** becomes the error message when the status is not `200`. This is where the service explains why it refused.
- **`responseData.translatedText`** is the result the call returns.

The language codes are passed as given, and the composable does not check them. An unknown or mistyped code is rejected by the service, and its message comes back as the error, so a wrong code shows up on the first try.

## 4. When an error occurs

Anything can go wrong between the page and a third-party service: the network, the server, or a refusal such as a used-up daily quota. The composable reports all of them as exceptions, so the call belongs in a `try`. The page below triggers each kind of error on purpose:

```vue
<script setup>
const translator = useTranslatorAPI()
const lines = ref([])

async function tryText(text) {
  try {
    const out = await translator.translate(text, 'fa', 'en')
    lines.value.push(`${JSON.stringify(text)} => ${JSON.stringify(out)}`)
  } catch (e) {
    lines.value.push(`${JSON.stringify(text)} => ${e.name}: ${e.message}`)
  }
}

async function run() {
  for (const t of ['', undefined, 'http500', 'quota', 'forbidden', 'stringstatus', 'malformed', 'network']) await tryText(t)
}
</script>

<template>
  <button id="go" @click="run">Run</button>
  <ul><li v-for="l in lines" :key="l" class="row">{{ l }}</li></ul>
</template>
```

```text
"" => ""
undefined => ""
"http500" => Error: Failed to fetch translation
"quota" => Error: DAILY LIMIT REACHED (mock)
"forbidden" => Error: Translation failed
"stringstatus" => "ok-string"
"malformed" => SyntaxError: Failed to execute 'json' on 'Response': Unexpected token '<', "<html>" is not valid JSON
"network" => TypeError: Failed to fetch
requests sent: 6
```

The messages tell the causes apart:

- **An empty or missing text is not an error.** It returns `''` and sends no request at all: eight calls make only six requests. Nothing is spent on blank text.
- **An HTTP error** (here 500) gives "Failed to fetch translation".
- **A refusal inside a normal answer** (a status other than 200 in the JSON) gives the service's own `responseDetails`, such as a daily-limit message. When the details are empty, the text is "Translation failed".
- **A network failure** gives the browser's own `TypeError` ("Failed to fetch"); the request never reached the service.
- **A body that is not JSON** gives a `SyntaxError`.

The page can tell a daily-limit refusal from any other refusal only by reading the message. So show the message to the user instead of a fixed text, or write it to a log. A user who sees "daily limit reached" understands what happened, but one who sees "something went wrong" will try again and get the same error again.

## 5. Typing and outdated answers

A tempting idea is to translate while the user types. Then several requests are in flight at once, and they do not always finish in the order they started: a long text is often slower than a short one. A naive page shows whichever answer arrives last, and that answer may belong to an older text:

```vue
<script setup>
const translator = useTranslatorAPI()
const text = ref('')
const naive = ref('')
const guarded = ref('')
let latest = 0

watch(text, async (value) => {
  const mine = ++latest
  const out = await translator.translate(value, 'fa', 'en')
  naive.value = out
  if (mine === latest) guarded.value = out
})
</script>

<template>
  <input id="text" v-model="text">
  <p id="naive">Naive: {{ naive }}</p>
  <p id="guarded">Guarded: {{ guarded }}</p>
</template>
```

```text
Naive: [fa|en] a long sentence typed first
Guarded: [fa|en] b
requests: ['a long sentence typed first', 'b']
```

In the output, the second text is typed after the first, but the first answer arrives later and overwrites the result in the "naive" line. The "guarded" line is correct, because it keeps a counter and ignores answers to older requests:

```js
let latest = 0

async function show(text) {
  const mine = ++latest
  const out = await translator.translate(text, 'fa', 'en')
  if (mine === latest) result.value = out     // ignore answers to older questions
}
```

The composable cannot cancel a request that is already on its way, so ignoring the answer has to be done in the page. Two more habits keep a live translation sensible: wait a moment after the last key before calling, and call only for text that is complete enough to translate. Every call costs quota.

## 6. Long texts

The service accepts a limited amount of text per request: 500 bytes. A byte is not a letter. In UTF-8 each Persian letter takes two bytes, so about 250 Persian letters fill a request. Counting bytes takes one line of code:

```js
const bytes = text => new TextEncoder().encode(text).length
bytes('hello')      // 5
bytes('سلام')       // 8: each Persian letter takes two bytes
```

Longer text must be cut into parts that are each under the limit, preferably at sentence ends, and the answers must be joined. The page below translates a long text this way:

```vue
<script setup>
const translator = useTranslatorAPI()
const result = ref('')
const direct = ref('')

const bytes = s => new TextEncoder().encode(s).length

function split(text, limit = 450) {
  const parts = []
  let current = ''
  for (const sentence of text.split(/(?<=[.!?؟])\s+/)) {
    if (current && bytes(current + ' ' + sentence) > limit) {
      parts.push(current)
      current = sentence
    } else {
      current = current ? current + ' ' + sentence : sentence
    }
  }
  if (current) parts.push(current)
  return parts
}

const long = Array.from({ length: 12 }, (_, i) => `این جملهٔ شمارهٔ ${i + 1} از متن بلند است.`).join(' ')

async function run() {
  try {
    await translator.translate(long, 'fa', 'en')
  } catch (e) {
    direct.value = `${bytes(long)} bytes: ${e.message}`
  }
  const pieces = split(long)
  const out = []
  for (const piece of pieces) out.push(await translator.translate(piece, 'fa', 'en'))
  result.value = `${pieces.length} requests of ${pieces.map(bytes).join(', ')} bytes; ${out.length} answers`
}
</script>

<template>
  <button id="go" @click="run">Run</button>
  <p id="direct">{{ direct }}</p>
  <p id="result">{{ result }}</p>
</template>
```

```text
one request: 746 bytes: QUERY TOO LONG (mock)
2 requests of 433, 312 bytes; 2 answers
```

The long text is refused as one request and accepted as two. The parts are sent one after another, which is gentle with the service and keeps the order. Use a limit a little below 500 (here 450) to leave a safe margin, because a count that is off by a few bytes would make the whole request fail.

## 7. In practice

We add a field for the English title next to the Persian one. The button translates the Persian title and fills the English field, which the editor can still change by hand. The page does four things the composable leaves to it: it ignores blank text, locks the button during the request, keeps answers and shows errors without losing what is already typed:

```vue
<script setup>
const translator = useTranslatorAPI()
const titleFa = ref('کفش ورزشی')
const titleEn = ref('')
const busy = ref(false)
const problem = ref('')
const cache = new Map()

async function fill() {
  const text = titleFa.value.trim()
  if (!text || busy.value) return
  problem.value = ''
  if (cache.has(text)) {
    titleEn.value = cache.get(text)
    return
  }
  busy.value = true
  try {
    const out = await translator.translate(text, 'fa', 'en')
    cache.set(text, out)
    titleEn.value = out
  } catch (e) {
    problem.value = 'Could not translate: ' + e.message
  } finally {
    busy.value = false
  }
}
</script>

<template>
  <input id="fa" v-model="titleFa">
  <button id="go" :disabled="busy" @click="fill">{{ busy ? 'Translating...' : 'Translate' }}</button>
  <input id="en" v-model="titleEn">
  <p id="problem">{{ problem }}</p>
</template>
```

```text
while waiting: button='Translating...'
English: 'Sports shoes' | requests: 1
same text again: English='Sports shoes' | requests: 0
failure: 'Could not translate: Failed to fetch' | English kept: 'Sports shoes'
blank text: requests 0
```

What each part does for the page:

- **The button locks while waiting,** so a double click sends one request.
- **The answer fills the English field,** and a second press with the same Persian title uses the stored answer: no request, no quota spent.
- **An error puts a message under the field** and leaves the English text as it was, so the editor loses nothing.
- **Blank text never reaches the service.** Trimming first matters: a text of only spaces is not empty, and would otherwise be sent.

---

## Appendix A. Quick reference

### A.1 The function

| Call | Returns | Notes |
|------|---------|-------|
| `translate(text, from, to)` | Promise of a string | `''` for empty or missing text; throws an exception on error |

### A.2 What is sent and read

| Direction | Content |
|-----------|---------|
| Request | A `GET` to `https://api.mymemory.translated.net/get` with `q` (the text) and `langpair` (the `from` code, a pipe character, the `to` code) |
| Read from the answer | `responseStatus`, `responseDetails`, `responseData.translatedText` |

### A.3 Service limits (from the service's documentation)

| Limit | Value |
|-------|-------|
| Text per request | 500 bytes |
| Anonymous use per day | 5000 characters |
| With a contact email (`de` parameter) | 50000 characters; the composable does not send one |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Empty or missing text | `''`, no request |
| HTTP status outside 200 to 299 | `Error: Failed to fetch translation` |
| `responseStatus` not 200 | `Error` with `responseDetails`, or "Translation failed" |
| Status 200 but no `responseData` in the answer | `TypeError` while reading `translatedText` |
| No network, blocked request | The browser's `TypeError` ("Failed to fetch") |
| The answer is not JSON | `SyntaxError` |
| Two answers out of order | Both arrive; the page must ignore the older one |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Sending private text.** The text goes to a public third-party service.
- **A text over 500 bytes.** The service does not accept it. Count bytes, not letters.
- **Relying on the free quota.** The daily allowance is small.
- **Using the result in `v-html`.** Show it with `{{ }}`, or sanitize it first (see `useXssProtector`).
- **Text of only spaces.** Spaces do not count as empty. Trim before calling.
- **Unchecked language codes.** A mistyped code gives a service error.
- **Out-of-order answers.** Fast typing needs the counter from chapter 5.
- **No offline mode.** Without the internet every call fails with a network error.
- **Treating the translation as final.** It is machine output, often rough; let the editor correct it.

### C.2 Behavior worth knowing

- The composable creates no state; each call stands alone, and it can be used from any component.
- Nothing is cached. Calling twice with the same text asks the service twice.
- There is no timeout. A request that hangs stays pending until the browser gives up.
- The request goes straight from the visitor's browser to the service, not through your server.
- Cookies are not sent to the service.

### C.3 Limits

| What | Limit |
|------|-------|
| Text per call | 500 bytes (service rule) |
| Free use per day | 5000 characters (service rule, anonymous) |
| Parallel calls | Not limited by the composable |
| Languages | Whatever the service supports |

### C.4 Tuning

- Store answers in the page, as chapter 7 does, so repeated texts cost nothing.
- Translate on a button press rather than on every key.
- Split long text at sentence ends, and send the parts one by one.
- If the quota matters, consider a service of your own behind your server, and keep this composable for light use.

## Appendix D. Message reference

### D.1 Error messages

| Message | Cause |
|---------|-------|
| `Failed to fetch translation` | The service answers with an HTTP error status |
| The service's `responseDetails` | The answer says `responseStatus` is not 200 |
| `Translation failed` | `responseStatus` is not 200 and there are no details |
| `Failed to fetch` (`TypeError`) | The browser cannot reach the service |
| `SyntaxError` | The answer is not valid JSON |
