# useQR

*J2OS Platform 2.4*

A product page needs a QR code that opens the same product on a phone. A ticket needs one that carries its number. A visitor should be able to save the code as an image and print it. All of this is a small job: turn a short text into a picture, put it somewhere on the page, and hand it over as a file.

**useQR** is a small composable of the Nuxt front end of J2OS that draws a QR code into an element of the page and saves it as an image. It comes in two forms that behave the same way:

- **`useQR`** finds the element by its `id`.
- **`useQRRefBased`** takes a Vue `ref` on the element instead of an id.

Both have three features:

- **Two functions.** `render` draws a code from a text and `download` saves the drawn code.
- **Fixed look.** The code is drawn 256 by 256 pixels, black on white, with the highest level of error correction.
- **Plain failures.** A missing element or a missing picture throws an ordinary `Error` with a readable message.

The idea in a few lines:

```js
const qr = useQR()

qr.render('box', 'https://shop.example/items/42')   // draws into <div id="box">
qr.download('box')                                  // saves QR.png

// the same with a ref instead of an id
const { render, download } = useQRRefBased()

render(box, 'https://shop.example/items/42')        // box = ref() on the element
download(box)
```

In order, the chapters cover: chapter 1 explains how the composable works and chapter 2 draws the first code. Chapter 3 covers drawing again, chapter 4 saving the image, chapter 5 the ref form and chapter 6 how much text fits. Chapter 7 builds a "share this item" card. The appendices are meant for quick lookups.

## Contents

1. [How useQR works](#1-how-useqr-works)
2. [The first code](#2-the-first-code)
3. [Drawing again](#3-drawing-again)
4. [Saving the image](#4-saving-the-image)
5. [The ref form](#5-the-ref-form)
6. [How much fits](#6-how-much-fits)
7. [Everything on one page](#7-everything-on-one-page)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Nothing here is a fragment; each example is a page of a Nuxt project, and each is a single `.vue` file. In Nuxt `ref`, `onMounted`, `useQR` and `useQRRefBased` need no `import`, so none is written.

---

## 1. How useQR works

First the vocabulary, which comes back all the time:

- **QR code.** A square picture that stores a short text. A phone camera reads it back.
- **Container.** The element of the page (usually a `div`) in which the code is drawn.
- **Error correction.** Extra data inside the code, so that a dirty or damaged code still reads. The composable asks for the highest level, `H`, which tolerates the most damage and takes the most room.
- **Data URL.** The picture written as text (`data:image/png;base64,...`). The downloaded file is built from it.

Its functions at a glance:

| Piece | Role |
|-------|------|
| `render(tagId, text)` and `render(tagRef, text)` | Empties the container and draws the code for `text` |
| `download(tagId)` and `download(tagRef)` | Saves the drawn code as `QR.png` |

The first form of each belongs to `useQR`, the second to `useQRRefBased`. Six facts shape the rest of the guide:

- **It does not draw by itself.** The composable calls a global `QRCode` object, the interface of the qrcodejs library. The page must have that library loaded before `render` runs; the composable does not load it.
- **`render` clears first.** Whatever was in the container is removed, then the new code is drawn. If drawing fails, the container stays empty.
- **The picture appears a moment later.** The library puts the image into the container shortly after `render` returns. A `download` called at the same instant finds nothing (chapter 4).
- **`download` always names the file `QR.png`.** The name cannot be changed.
- **The look cannot be changed.** Size, colors and error correction are fixed inside the composable.
- **The input must be text.** Other values do not produce a useful error (chapter 6).

## 2. The first code

Drawing a code takes two things: a container with an id, and a call to `render` once the container exists. So the call belongs in `onMounted`, because the element is not on the page before that:

```vue
<script setup>
const qr = useQR()

onMounted(() => {
  qr.render('box', 'https://shop.example/items/42')
})
</script>

<template>
  <div id="box" />
</template>
```

```text
inside: canvas (hidden), img; picture: 256x256; a reader finds: https://shop.example/items/42
```

The library puts two elements into the container: a hidden `canvas`, and an `img` that is the picture the visitor sees and the one `download` uses later. The picture is 256 by 256 pixels. The last part of the output line comes from a QR reader that decodes the picture and finds exactly the text that was given, which shows that the code is usable and not only pretty.

## 3. Drawing again

A page often changes the code: another item, another address. `render` can simply be called again on the same container. It clears first and draws second, so the old code never remains:

```vue
<script setup>
const qr = useQR()

const text = ref('https://shop.example/items/42')
const shown = ref('')
const problem = ref('')

function draw() {
  try {
    qr.render('box', text.value)
    shown.value = text.value
    problem.value = ''
  } catch (e) {
    problem.value = e.message
  }
}

onMounted(draw)
</script>

<template>
  <input id="text" v-model="text">
  <button id="draw" @click="draw">Draw</button>
  <p id="shown">{{ shown }}</p>
  <p id="problem">{{ problem }}</p>
  <div id="box" />
</template>
```

```text
first: inside: canvas (hidden), img; picture: 256x256; a reader finds: https://shop.example/items/42
second: inside: canvas (hidden), img; picture: 256x256; a reader finds: https://shop.example/items/77?ref=کمپین (with an invisible byte order mark in front)
empty text: inside: canvas, img (hidden); picture: none; problem: ''
1500 letters: inside: canvas, img (hidden); picture: none; problem: "Cannot read properties of undefined (reading '3')"
```

After the second call the container still holds one canvas and one image, not two. Three situations need care:

- **A non-ASCII character** in the text, such as a Persian word inside an address, makes the library start the text with an invisible byte-order mark (BOM). Most readers drop it, but some show it. The fix is simple: percent-encode such parts before drawing (chapter 7 shows how) and the problem does not appear.
- **An empty text** draws nothing and raises no error. The old code is removed, so the container is empty. Check for an empty value yourself if a blank box would confuse the visitor.
- **A text that is too long** makes the library raise an error, and the container is left empty too, because `render` empties it before drawing. Show the visitor a message and decide whether to draw something else (chapter 6 gives the limits).

The composable puts nothing else on the page, but the library sets a `title` on the container with the text, so hovering over the code shows it.

## 4. Saving the image

A code is often wanted as a file: to print on a poster, to send in a message. `download` takes the picture, makes a link from it and clicks the link, and the browser saves a file:

```vue
<script setup>
const qr = useQR()

const url = 'https://shop.example/items/42'
const problems = ref([])
const wait = (ms) => new Promise(resolve => setTimeout(resolve, ms))

function save(id) {
  try {
    qr.download(id)
  } catch (e) {
    problems.value.push(`${id}: ${e.message}`)
  }
}

function atOnce() {
  qr.render('late', url)
  save('late')
}

async function afterAMoment() {
  qr.render('late', url)
  await wait(100)
  save('late')
}

onMounted(() => qr.render('full', url))
</script>

<template>
  <button id="save" @click="save('full')">Save the code</button>
  <button id="save-empty" @click="save('empty')">Save the empty box</button>
  <button id="save-missing" @click="save('nowhere')">Save a missing box</button>
  <button id="at-once" @click="atOnce">Draw and save at once</button>
  <button id="later" @click="afterAMoment">Draw, wait, save</button>
  <div v-for="(p, i) in problems" :key="i" class="row">{{ p }}</div>
  <div id="full" />
  <div id="empty" />
  <div id="late" />
</template>
```

```text
file 'QR.png', PNG signature True, size 256x256
content: https://shop.example/items/42
after a moment: file 'QR.png'
empty: QR code not found; render it first.
nowhere: Element with id="nowhere" not found.
late: QR code not found; render it first.
```

The file is always named `QR.png`, it is a PNG of 256 by 256 pixels, and a QR reader finds the original text in it. When there is nothing to save, `download` says why in its error message: a container with no code in it gives "QR code not found; render it first.", and a container that does not exist gives `Element with id="nowhere" not found.` with the id repeated in the message.

One case is rarely expected: **drawing and saving in the same click fails** with "QR code not found; render it first.", although the code has just been drawn. The picture is not in the container yet at that moment, and `download` looks only at the image and does not fall back to the hidden canvas. The fix is to wait a moment first. A small helper does this by watching for the image's source for up to half a second:

```js
// Waits until the picture of the code is really there (up to half a second).
async function whenDrawn(el, tries = 20) {
  for (let i = 0; i < tries; i++) {
    const img = el?.querySelector('img')
    if (img && img.src) return true
    await new Promise(resolve => setTimeout(resolve, 25))
  }
  return false
}
```

On a normal page this rarely matters, because the visitor presses "Save" long after the code has been drawn. It matters for a button that does both jobs and for automatic tests.

## 5. The ref form

`useQRRefBased` does the same with a `ref` on the element instead of an id. The advantage is practical: two copies of a component on one page cannot clash over an id, and the element is found by Vue and not by a search of the page:

```vue
<div ref="box" />                 <!-- ref variant: no id needed -->
<div id="box" />                  <!-- id variant: the id is the address -->
```

```vue
<script setup>
const { render, download } = useQRRefBased()

const box = ref()
const visible = ref(true)
const text = ref('https://shop.example/items/42')
const notes = ref([])

const attempt = (label, run) => {
  try { run(); notes.value.push(`${label}: ok`) } catch (e) { notes.value.push(`${label}: ${e.message}`) }
}

// Too early: the element does not exist yet while setup() runs.
attempt('in setup', () => render(box, text.value))

onMounted(() => attempt('in onMounted', () => render(box, text.value)))
</script>

<template>
  <button id="toggle" @click="visible = !visible">Show or hide</button>
  <button id="draw" @click="attempt('on click', () => render(box, text))">Draw</button>
  <button id="save" @click="attempt('download', () => download(box))">Save</button>
  <div v-for="(n, i) in notes" :key="i" class="row">{{ n }}</div>
  <div v-if="visible" ref="box" id="box" />
</template>
```

```text
in setup: Element ref is empty; make sure it is mounted before calling render().
in onMounted: ok
after mount: inside: canvas (hidden), img; picture: 256x256; a reader finds: https://shop.example/items/42
hidden, on click: Element ref is empty; make sure it is mounted before calling render().
hidden, download: Element ref is empty; make sure it is mounted before calling download().
shown again, the new box holds 0 elements
drawn again: inside: canvas (hidden), img; picture: 256x256; a reader finds: https://shop.example/items/42
download name: QR.png
```

The functions accept a ref and also a plain element (they use `unref`). The timing rules follow from the fact that a ref is empty until the element exists:

- **In `setup` the ref is still empty,** and `render` throws "Element ref is empty; make sure it is mounted before calling render()." Draw in `onMounted` or later.
- **In `onMounted` the call works,** and the code reads back correctly.
- **When the container is hidden with `v-if`,** `render` and `download` give the same kind of message, naming the function that was called. The messages differ from those of the id form because there is no id to name.
- **When the container is shown again,** Vue makes a new element and the ref points to it. The new container is empty, so the code has to be drawn again.
- **The file name is still `QR.png`,** as with the id form.

## 6. How much fits

A QR code has a limited capacity. More text needs a denser code, and beyond a certain point no denser code exists. Knowing the limit lets the page warn the visitor instead of failing. The page below tries several texts, one after another:

```vue
<script setup>
const qr = useQR()

const results = ref([])
const wait = (ms) => new Promise(resolve => setTimeout(resolve, ms))

const cases = [
  ['a short address', 'https://shop.example/items/42'],
  ['100 letters', 'a'.repeat(100)],
  ['119 letters', 'a'.repeat(119)],
  ['120 letters', 'a'.repeat(120)],
  ['1000 letters', 'a'.repeat(1000)],
  ['1273 letters', 'a'.repeat(1273)],
  ['1274 letters', 'a'.repeat(1274)],
  ['Persian, 40 letters', 'سلام'.repeat(10)],
  ['Persian, 600 letters', 'س'.repeat(600)],
  ['Persian, 700 letters', 'س'.repeat(700)],
  ['empty text', ''],
  ['a number', 12345],
  ['null', null],
  ['undefined', undefined]
]

onMounted(async () => {
  for (const [label, text] of cases) {
    try {
      qr.render('box', text)
      await wait(30)
      const img = document.querySelector('#box img')
      results.value.push(`${label}: ${img && img.src ? 'drawn' : 'nothing drawn'}`)
    } catch (e) {
      results.value.push(`${label}: ${e.constructor.name}: ${e.message}`)
    }
  }
})
</script>

<template>
  <div v-for="(r, i) in results" :key="i" class="row">{{ r }}</div>
  <div id="box" />
</template>
```

```text
a short address: drawn
100 letters: drawn
119 letters: drawn
120 letters: drawn
1000 letters: drawn
1273 letters: TypeError: Cannot read properties of undefined (reading '3')
1274 letters: TypeError: Cannot read properties of undefined (reading '3')
Persian, 40 letters: drawn
Persian, 600 letters: drawn
Persian, 700 letters: TypeError: Cannot read properties of undefined (reading '3')
empty text: nothing drawn
a number: drawn
null: nothing drawn
undefined: nothing drawn
```

Ordinary addresses and short texts are far below the limit. With the qrcodejs library and the highest level `H`, about 1270 letters of plain ASCII text fit. Persian letters take two bytes each, so about 635 of them fit. Beyond that the library fails with a `TypeError` ("Cannot read properties of undefined (reading '3')"), and this message does not say that the text is too long. The rule is to **check the length yourself before `render`, and keep a margin.**

Values that are not texts need care too. An empty text, `null` and `undefined` draw nothing and raise no error. A number draws a code, but one that reads back as an empty text, so convert it to a string yourself.

Long codes are also hard to scan, even when they fit. Instead of putting the details themselves in the code, put a short address that leads to them.

## 7. Everything on one page

Let us build a "share this item" card. The visitor enters an item number and a campaign name (which may be in Persian); the page builds the address, draws the code and enables "Save as image" only when the picture is really in the container. Invalid input is rejected before drawing, and the old code is removed so that a stale code is never saved:

```vue
<script setup>
const { render, download } = useQRRefBased()

const box = ref()
const itemId = ref('42')
const campaign = ref('پاییز')
const link = ref('')
const message = ref('')
const ready = ref(false)

const wait = (ms) => new Promise(resolve => setTimeout(resolve, ms))

async function whenDrawn(el, tries = 20) {
  for (let i = 0; i < tries; i++) {
    const img = el?.querySelector('img')
    if (img && img.src) return true
    await wait(25)
  }
  return false
}

async function make() {
  ready.value = false
  message.value = ''
  link.value = ''
  if (box.value) box.value.innerHTML = ''

  const id = itemId.value.trim()
  if (!/^\d+$/.test(id)) {
    message.value = 'Enter the item number (digits only).'
    return
  }

  const query = new URLSearchParams({ campaign: campaign.value.trim() })
  const url = `https://shop.example/items/${id}?${query}`

  if (url.length > 1000) {
    message.value = 'The address is too long for a QR code.'
    return
  }

  try {
    render(box, url)
    link.value = url
    ready.value = await whenDrawn(box.value)
  } catch (e) {
    message.value = e.message
  }
}

function save() {
  try {
    download(box)
  } catch (e) {
    message.value = e.message
  }
}
</script>

<template>
  <input id="item" v-model="itemId">
  <input id="campaign" v-model="campaign">
  <button id="make" @click="make">Make the code</button>
  <button id="save" :disabled="!ready" @click="save">Save as image</button>
  <p id="link">{{ link }}</p>
  <p id="message">{{ message }}</p>
  <div ref="box" />
</template>
```

```text
start | link='' | message='' | save enabled=False
after Make | link='https://shop.example/items/42?campaign=%D9%BE%D8%A7%DB%8C%DB%8C%D8%B2' | message='' | save enabled=True
saved 'QR.png'; content: https://shop.example/items/42?campaign=%D9%BE%D8%A7%DB%8C%DB%8C%D8%B2
item 7x | link='' | message='Enter the item number (digits only).' | save enabled=False
picture left on the page: 0
very long campaign | link='' | message='The address is too long for a QR code.' | save enabled=False
summer sale | link='https://shop.example/items/8?campaign=summer+sale' | message='' | save enabled=True
```

The page applies the points of the earlier chapters:

- **The address is built with `URLSearchParams`,** so the Persian campaign name becomes percent-encoded ASCII. The code then holds only plain characters, reads cleanly and takes less room:

```js
const query = new URLSearchParams({ campaign: 'پاییز' })

`https://shop.example/items/42?${query}`
// 'https://shop.example/items/42?campaign=%D9%BE%D8%A7%DB%8C%DB%8C%D8%B2'
```

- **The button waits for the picture.** "Save as image" stays disabled until the picture exists (chapter 4), and then saves a file whose code reads back as exactly the address.
- **Bad input is stopped early.** A bad item number and an address that is too long (chapter 6) both produce a message, leave no code on the page and keep the button disabled.
- **A new valid input** replaces the message and draws the new code.

---

## Appendix A. Quick reference

### A.1 The functions

| Function | Form | Does |
|----------|------|------|
| `render(tagId, text)` | `useQR` | Finds the element by id, empties it, draws the code |
| `download(tagId)` | `useQR` | Saves the picture in the element as `QR.png` |
| `render(tagRef, text)` | `useQRRefBased` | The same, for a `ref` (or an element) |
| `download(tagRef)` | `useQRRefBased` | The same, for a `ref` (or an element) |

### A.2 Fixed settings

| Setting | Value |
|---------|-------|
| Size | 256 by 256 pixels |
| Dark color | `#000000` |
| Light color | `#ffffff` |
| Error correction | `H` (highest) |
| Download file name | `QR.png` |
| Download format | PNG |

### A.3 What is left in the container

| Element | Description |
|---------|-------------|
| `canvas` | Hidden; the drawing surface |
| `img` | The visible picture; `download` uses its source |
| `title` attribute (of the container) | Set by the library to the text |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Id form: no element has that id | `Error: Element with id="..." not found.` |
| Ref form: the ref is empty | `Error: Element ref is empty; make sure it is mounted before calling render().` (or `download()`) |
| `download` and no picture in the container | `Error: QR code not found; render it first.` |
| `download` in the same moment as `render` | The same error; the picture is not there yet |
| Text too long for the code | A `TypeError` from the library; the container stays empty |
| Empty text, `null` or `undefined` | Nothing is drawn; no error |
| A number instead of a text | A code is drawn that reads back as empty text |
| The QR library is not loaded | `render` fails |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Drawing before the element exists.** In `setup` the element is not there. Use `onMounted`.
- **Saving at once.** The picture appears a moment after `render`; wait for it.
- **Duplicate ids** with the id form. Two components with the same id draw into the first one only. The ref form avoids this.
- **A very long text.** The library fails with an unhelpful error. Check the length first.
- **Non-ASCII text in the code.** It adds an invisible mark and uses more room. Percent-encode addresses.
- **Passing a number.** Convert it to a string.
- **A stale code after a failure.** `render` empties the container first; if the page validates before calling it, it must clear the old code itself.
- **Expecting another file name or size.** Both are fixed.
- **The library missing.** Without the `QRCode` object nothing can be drawn.

### C.2 Behavior worth knowing

- `render` sets the container's contents again every time: the old code is removed first.
- The picture is a PNG data URL, built from the canvas; the file is made in the browser and nothing is uploaded.
- `download` prefers the `img` and falls back to the canvas only if there is no image element.
- The container gets a `title` attribute with the text, set by the library.
- Several containers on one page can show different codes.
- The ref form accepts a ref or an element, as it calls `unref`.
- Text with characters outside ASCII starts with a byte order mark when read back.

### C.3 Limits

| What | Limit |
|------|-------|
| Size and colors | Fixed |
| File name | `QR.png` |
| Text, plain ASCII (with the qrcodejs library) | About 1270 letters |
| Text, Persian (with the qrcodejs library) | About 635 letters |
| Several codes in one container | No; each `render` replaces the previous one |

### C.4 Tuning

- Draw in `onMounted`, or after the element is shown.
- Keep the text short; put a short address in the code.
- Percent-encode the query parts of an address, with `URLSearchParams`.
- Check the length before drawing and show a message.
- Enable the save button only when the picture exists.
- Prefer the ref form in components that may be used more than once.

## Appendix D. Message reference

### D.1 Error messages

| Message | Cause |
|---------|-------|
| `Element with id="..." not found.` | Id form: no element has that id |
| `Element ref is empty; make sure it is mounted before calling render().` | Ref form: the ref has no element when `render` is called |
| `Element ref is empty; make sure it is mounted before calling download().` | Ref form: the ref has no element when `download` is called |
| `QR code not found; render it first.` | `download` found no picture in the container |
| `Cannot read properties of undefined (reading '3')` (`TypeError`) | The qrcodejs library received a text that is too long |
