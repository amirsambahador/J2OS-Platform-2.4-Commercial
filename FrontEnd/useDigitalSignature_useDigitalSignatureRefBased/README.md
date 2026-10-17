# useDigitalSignature

*J2OS Platform 2.4*

A courier hands over a tablet and the customer signs for the parcel with a finger. A rental contract is signed on the counter screen. A consent form collects a signature before a procedure. In each case the page needs a box in which the visitor can draw, a button to start over, and a way to keep the result as an image. Browsers have everything needed for this (the `<canvas>` element and mouse and touch events), but turning those into a pen that behaves is the same careful code on every page.

**useDigitalSignature** is a composable of the Nuxt front end of J2OS that provides it. **useDigitalSignatureRefBased** is its companion that finds the canvas through a ref instead of an id. This guide explains both. Each has three features:

- **A pen for a canvas.** `init` attaches mouse and touch listeners to a canvas, so the visitor can draw with a mouse, a stylus or a finger.
- **Four functions.** `init`, `clear`, `save` and `destroy`; that is the whole interface.
- **Plain output.** The signature is just the canvas. Saving gives a PNG file, and anything else (checking, uploading) uses the canvas directly.

In a few lines:

```vue
<script setup>
const signature = useDigitalSignature()

onMounted(() => signature.init('sign-here'))        // the id of the canvas
onBeforeUnmount(() => signature.destroy())
</script>

<template>
  <canvas id="sign-here" width="400" height="160" />
  <button @click="signature.clear()">Clear</button>
  <button @click="signature.save('signature.png')">Save</button>
</template>
```

This guide goes step by step: chapter 1 explains how the composable works and chapter 2 builds the first signature pad. Chapter 3 explains what the pen does and does not draw, and chapter 4 the one trap that appears on real screens: the size of the canvas. Chapter 5 is about detecting that the visitor has signed, and saving and sending the image. Chapter 6 covers pads inside dialogs that open and close. Chapter 7 shows the ref-based version, and chapter 8 builds a signing form. The appendices are a quick reference.

## Contents

1. [How useDigitalSignature works](#1-how-usedigitalsignature-works)
2. [The first signature pad](#2-the-first-signature-pad)
3. [What the pen does](#3-what-the-pen-does)
4. [Size and sharpness](#4-size-and-sharpness)
5. [Knowing it is signed, saving and sending](#5-knowing-it-is-signed-saving-and-sending)
6. [A pad that comes and goes](#6-a-pad-that-comes-and-goes)
7. [The ref-based version](#7-the-ref-based-version)
8. [A signing form](#8-a-signing-form)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are complete pages of a Nuxt project: each is one `.vue` file. In Nuxt `ref`, `watch`, `nextTick`, `onMounted`, `onBeforeUnmount` and the composables need no `import`, which is why none is written. The output under a page is what the visitor sees while using that page.

---

## 1. How useDigitalSignature works

First, a few terms, because they come back throughout the guide:

- **Canvas.** An HTML element that holds a picture made of pixels, which code can draw on. It has two sizes that are easy to confuse: the size of its pixel picture (the `width` and `height` attributes) and the size of its box on the page (set by CSS).
- **Pen.** The behaviour that `init` adds to the canvas: pressing starts a line, moving extends it, releasing or leaving ends it.
- **Context.** The object the canvas gives to code for drawing (`getContext('2d')`). The composable draws with it, and so can the page.

The functions of this composable:

| Function | Role |
|----------|------|
| `init(canvasId)` | Finds the canvas by its id and attaches the pen to it |
| `clear()` | Wipes the whole canvas |
| `save(fileName)` | Downloads the canvas as a PNG file; the default name is `signature.png` |
| `destroy()` | Takes the pen off the canvas |

The main idea is simple. The composable holds **one canvas**. `init` says which one, and the other three functions work on that canvas. It keeps no list of strokes and has no "undo": the picture on the canvas is the only record. 5 consequences follow, and the rest of the guide returns to each:

- **The canvas must exist when `init` is called,** so the place for it is `onMounted`.
- **The pen reads the pointer in page pixels and draws at the same numbers in bitmap pixels,** so the bitmap and its box must have the same size, or be scaled deliberately (chapter 4).
- **The composable does not tell the page that the visitor has signed.** The page has to look at the canvas itself (chapter 5).
- **Cleanup is not automatic.** The page calls `destroy` when the canvas goes away (chapter 6).
- **Calls on a pad that was never set up fail with an error,** a `TypeError` (chapter 6).

## 2. The first signature pad

The project's own example is the best starting point: a canvas with an `id`, one call to `init` once the page is on screen, a `destroy` when it leaves, and two buttons. The page below is a delivery note. It also keeps a status line, so the visitor knows whether the pad holds a signature:

```vue
<script setup>
const signature = useDigitalSignature()
const status = ref('Please sign in the box.')
let blank = ''

const pad = () => document.getElementById('signature')

onMounted(() => {
  signature.init('signature')
  blank = pad().toDataURL()          // what the empty pad looks like
})

onBeforeUnmount(() => signature.destroy())

function check() {
  status.value = pad().toDataURL() === blank ? 'Please sign in the box.' : 'Signature captured.'
}

function clear() {
  signature.clear()
  check()
}

function save() {
  signature.save('delivery-note.png')
  status.value = 'Saved as delivery-note.png.'
}
</script>

<template>
  <canvas
    id="signature"
    width="400"
    height="160"
    style="border: 1px solid #999; background: #fff; display: block"
    @mouseup="check"
    @touchend="check"
  />
  <button id="clear" @click="clear">Clear</button>
  <button id="save" @click="save">Save</button>
  <p id="status">{{ status }}</p>
</template>
```

```text
Page opens:                  Please sign in the box.
Visitor signs:               Signature captured.
Visitor presses Clear:       Please sign in the box.
Visitor signs, presses Save: Saved as delivery-note.png.  (file offered: delivery-note.png)
```

The `id` is what `init` looks for, and `width` and `height` give the pixel picture a size. `clear` wipes the canvas, and `save` downloads a PNG with the given name, here `delivery-note.png`; without a name the file is `signature.png`.

The composable does not notify the page of anything, so the status line is the page's own work. After each stroke (the `mouseup` and `touchend` of the canvas), the page compares the canvas with a picture of the empty pad taken after `init`. This is the pattern for any "please sign first" rule, and chapter 5 returns to it.

## 3. What the pen does

Before building on the pen, it helps to know exactly what it draws. A line is made of three steps: pressing starts it, moving extends it, and releasing or leaving the canvas ends it. What that means for the visitor:

| The visitor | What appears on the pad |
|-------------|-------------------------|
| Moves the mouse over the pad without pressing | Nothing |
| Presses, moves and releases | A line that follows the pointer |
| Presses and releases without moving | Nothing; a single dot is not drawn |
| Presses, leaves the pad and comes back with the button still held | The line stops at the edge and does not continue |
| Uses the right mouse button instead of the left | A line, exactly as with the left button |

Two of these rows matter for real signatures. A signature often ends with a dot or contains the dot of an "i", and **dots do not show**: a press without movement draws nothing. If dots matter, a short drag is needed, so say so on the page. And a stroke that leaves the box is cut off, which is why the box should be generous: wider than the longest signature, with room at the edges.

Touch screens work the same way. The pad listens to touch start, move and end, and follows the first finger. A finger dragged across the pad draws a line, and the canvas cancels the browser's own handling of the movement, so the page does not scroll under the finger. A tap with no movement leaves no mark. To also stop the browser from claiming the gesture for itself (zooming, pull-to-refresh), tell it that this element takes no touch gestures:

```html
<canvas id="sign-here" width="400" height="160" style="touch-action: none" />
```

## 4. Size and sharpness

Everything above works as long as the pad is shown at its natural size. Real pages shrink it: a canvas that is 400 pixels wide is made `width: 100%` on a phone, or smaller to fit a column. Then the pen goes wrong, in a way that is easy to miss until someone tries it on a phone.

The reason is that the composable reads the pointer's position in **page pixels** (relative to the box) and draws at those same numbers in **bitmap pixels**. When the two sizes differ, the line lands in the wrong place. The page below has three canvases with a 300 pixel wide bitmap. The first is shown at its natural size, the second is shrunk to half its width by CSS, and the third is also shrunk but has its pen scaled (explained below). In each, the pointer moves from 30 to 70:

```vue
<script setup>
const natural = useDigitalSignature()
const squeezed = useDigitalSignature()
const sharp = useDigitalSignature()

onMounted(() => {
  natural.init('natural')
  squeezed.init('squeezed')
  sharp.init('sharp')
  // the bitmap of this one is twice the size of its box, so the pen is scaled by two
  document.getElementById('sharp').getContext('2d').scale(2, 2)

  window.__box = (id) => {
    const canvas = document.getElementById(id)
    const { data, width, height } = canvas.getContext('2d').getImageData(0, 0, canvas.width, canvas.height)
    let left = width, right = -1
    for (let y = 0; y < height; y++) for (let x = 0; x < width; x++) {
      if (data[(y * width + x) * 4 + 3] > 0) { left = Math.min(left, x); right = Math.max(right, x) }
    }
    return { left, right, cssWidth: canvas.clientWidth, width }
  }
})
onBeforeUnmount(() => { natural.destroy(); squeezed.destroy(); sharp.destroy() })
</script>

<template>
  <canvas id="natural" width="300" height="100" style="border: 1px solid #999; display: block; margin: 20px" />
  <canvas id="squeezed" width="300" height="100" style="width: 150px; height: 50px; border: 1px solid #999; display: block; margin: 20px" />
  <canvas id="sharp" width="300" height="100" style="width: 150px; height: 50px; border: 1px solid #999; display: block; margin: 20px" />
  <p>Sizes</p>
</template>
```

```text
Bitmap 300 wide, shown 300 wide:                   pointer from 30 to 70, line from 29 to 70
Bitmap 300 wide, shown 150 wide:                   pointer from 30 to 70, line from 14 to 35
Bitmap 300 wide, shown 150 wide, pen scaled by 2:  pointer from 30 to 70, line from 29 to 70
```

In the shrunk canvas, the line lands at 14 to 35: it is drawn at half scale and to the left of where the finger is. The visitor sees the line run away from the pen, and a signature drawn this way is unusable. The third canvas shows the cure, which is to **make the bitmap match the box** and, when more sharpness is wanted, to scale the pen:

```js
const SCALE = 2                       // 2 is enough for most phones; window.devicePixelRatio is the exact value

onMounted(() => {
  const canvas = document.getElementById('sign-here')

  // 1. Size the bitmap to the box on the page, times SCALE. Do it first:
  //    changing width or height wipes the canvas and resets the pen.
  canvas.width = canvas.clientWidth * SCALE
  canvas.height = canvas.clientHeight * SCALE

  // 2. Let the composable attach the pen.
  signature.init('sign-here')

  // 3. Tell the pen that one page pixel is SCALE bitmap pixels.
  canvas.getContext('2d').scale(SCALE, SCALE)
})
```

3 points make this work:

- **Set `width` and `height` before `init`.** Assigning either one wipes the canvas and resets the pen (colour, thickness and round ends go back to the defaults), so doing it after `init` loses those settings.
- **Use a scale for sharpness.** On a screen that has more physical pixels than page pixels, a bitmap the size of the box looks blurry. A bitmap twice as large, with the pen scaled by two, draws the same signature with finer lines. The scale is applied through the context after `init`; the composable's calculations then stay in page pixels, and the browser does the multiplication.
- **Give the canvas no padding and no thick border.** The pointer's position is measured from the outer edge of the box, but the bitmap starts inside the border and padding. A canvas with a 10 pixel border and 20 pixels of padding draws its line 30 pixels to the right of, and below, the pointer. Put the frame on a wrapper element instead.

The pen's look can be changed in the same way. The composable draws with a black line 2 pixels wide with round ends. It does not offer settings, but the canvas hands out the same context every time, so the page can change them after `init`:

```js
signature.init('sign-here')

// The composable draws with a black line, 2 pixels wide, with round ends.
// init() returns nothing, but the canvas hands out the same context every time:
const context = document.getElementById('sign-here').getContext('2d')
context.strokeStyle = '#1d3a8a'
context.lineWidth = 3
```

`clear` does not reset these settings, but assigning a new `width` or `height` does, as above.

## 5. Knowing it is signed, saving and sending

A signature pad is only half of the job. The page must know whether the visitor has signed, and the image must go somewhere. The composable has no `isEmpty` and no way to hand the image over, because the canvas already is the data. Both jobs use the canvas directly.

**Is the pad empty?** Compare the canvas with a picture of the empty pad taken right after `init`. The two pictures (as text, from `toDataURL`) are equal only if nothing was drawn. The delivery note of chapter 2 does this, and so does the form of chapter 8:

```js
let emptyPad = ''

onMounted(() => {
  signature.init('sign-here')
  emptyPad = document.getElementById('sign-here').toDataURL()       // the picture of an empty pad
})

const isEmpty = () => document.getElementById('sign-here').toDataURL() === emptyPad
```

**Saving to a file.** `save` downloads the canvas as a PNG file with the given name. 3 details to know:

- **The file is the bitmap, at its own size.** A 300 by 120 canvas gives a 300 by 120 image, which is why chapter 4's sharper canvas gives a larger file.
- **The background is transparent.** A white background set by CSS is not part of the image, so the file shows a black signature on a transparent background. On a white page this looks right, but in a dark viewer the signature disappears. If the file needs a white background, draw the signature onto a white canvas before saving.
- **An empty pad is saved too.** `save` does not check anything; run the test above first.

**Sending to a server.** `save` only downloads. To upload, get a `Blob` from the canvas and send it as a file in a form, which is what the server expects for an image:

```js
async function send() {
  const canvas = document.getElementById('sign-here')
  const blob = await new Promise(resolve => canvas.toBlob(resolve, 'image/png'))

  const form = new FormData()
  form.append('signature', blob, 'signature.png')

  const response = await fetch('/api/signature', { method: 'POST', body: form })
  if (!response.ok) throw new Error(`Server answered ${response.status}`)
}
```

The request carries a PNG in the form field whose name you choose. The server side is the same as for any file upload.

## 6. A pad that comes and goes

Many pads live in dialogs, tabs or wizard steps. The canvas does not exist until the dialog opens, and is thrown away when it closes. The pen must follow: attached when the canvas appears, released when it goes. The page below opens a dialog with a pad inside, closes it and opens it again:

```vue
<script setup>
const signature = useDigitalSignature()
const open = ref(false)
const status = ref('The signing dialog is closed.')
let blank = ''

const pad = () => document.getElementById('dialog-pad')

// the canvas exists only while the dialog is open, so the pen is attached
// when it opens and released when it closes
watch(open, async (isOpen) => {
  if (isOpen) {
    await nextTick()
    signature.init('dialog-pad')
    blank = pad().toDataURL()
    status.value = 'Pad ready.'
  } else {
    signature.destroy()
    status.value = 'The signing dialog is closed.'
  }
})

onBeforeUnmount(() => signature.destroy())

function check() {
  status.value = pad().toDataURL() === blank ? 'Pad ready.' : 'Signature captured.'
}
</script>

<template>
  <button id="toggle" @click="open = !open">{{ open ? 'Close' : 'Sign' }}</button>
  <div v-if="open" style="border: 1px solid #999; padding: 8px; width: 320px">
    <canvas id="dialog-pad" width="300" height="120" style="border: 1px solid #ccc; display: block" @mouseup="check" />
  </div>
  <p id="status">{{ status }}</p>
</template>
```

```text
Page opens:                The signing dialog is closed.
Visitor opens the dialog:  Pad ready.
Visitor signs:             Signature captured.
Visitor closes the dialog: The signing dialog is closed.
Visitor opens it again:    Pad ready.
Visitor signs again:       Signature captured.
```

When the dialog opens again the signature is gone and the pad works again: a new canvas has been created and `init` has attached the pen to it. The watcher does this:

```js
// the canvas is rebuilt whenever the dialog opens, so the pen is attached again
watch(open, async (isOpen) => {
  if (isOpen) {
    await nextTick()                 // wait until the new canvas is in the page
    signature.init('dialog-pad')
  } else {
    signature.destroy()
  }
})
```

`await nextTick()` matters. The canvas is created after the flag changes, so `init` called straight away would not find it and would throw. Notice also that `destroy` is called on close and again in `onBeforeUnmount`: calling it twice is harmless.

The composable does not protect against the wrong order of calls. This is what each mistake does, so that the messages are recognisable when they appear:

| Call | Result |
|------|--------|
| `clear()` or `save()` before `init` | `TypeError` ("Cannot read properties of undefined") |
| `destroy()` before `init`, or twice | Nothing happens |
| `init` with an id that does not exist | `Error: Canvas with id="..." not found.` |
| `clear()` after such a failed `init` | `TypeError` ("Cannot read properties of null"); the pad is lost |
| `clear()` after `destroy()` | `TypeError` ("Cannot read properties of null") |
| Drawing after `destroy()` | Nothing is drawn |
| `init` twice on the same canvas | Harmless; the pen is still attached once |
| `init` on another canvas without `destroy` | The pen moves to the new canvas, and the first one stops drawing |

The last row is the one to remember: one instance serves one canvas at a time. If a page has two pads, call `useDigitalSignature()` twice.

## 7. The ref-based version

The plain functions find the canvas by `id`, which has two drawbacks: ids must be unique on the page, and the lookup happens at one moment. `useDigitalSignatureRefBased()` takes a Vue ref (or the element itself) instead, with the same four functions. The page of chapter 2, written with it:

```vue
<script setup>
const signature = useDigitalSignatureRefBased()
const padRef = ref()                                  // points at the canvas once it exists

onMounted(() => signature.init(padRef))               // a ref, or the element itself
onBeforeUnmount(() => signature.destroy())
</script>

<template>
  <canvas ref="padRef" width="400" height="160" />
</template>
```

```vue
<script setup>
const signature = useDigitalSignatureRefBased()
const status = ref('Please sign in the box.')
let blank = ''

const padRef = ref()
const pad = () => padRef.value

onMounted(() => {
  signature.init(padRef)
  blank = pad().toDataURL()          // what the empty pad looks like
})

onBeforeUnmount(() => signature.destroy())

function check() {
  status.value = pad().toDataURL() === blank ? 'Please sign in the box.' : 'Signature captured.'
}

function clear() {
  signature.clear()
  check()
}

function save() {
  signature.save('delivery-note.png')
  status.value = 'Saved as delivery-note.png.'
}
</script>

<template>
  <canvas
    ref="padRef"
    width="400"
    height="160"
    style="border: 1px solid #999; background: #fff; display: block"
    @mouseup="check"
    @touchend="check"
  />
  <button id="clear" @click="clear">Clear</button>
  <button id="save" @click="save">Save</button>
  <p id="status">{{ status }}</p>
</template>
```

```text
Page opens:                  Please sign in the box.
Visitor signs:               Signature captured.
Visitor presses Clear:       Please sign in the box.
Visitor signs, presses Save: Saved as delivery-note.png.  (file offered: delivery-note.png)
```

The output is the same as in chapter 2, because only the way the canvas is found has changed. Three differences remain:

- **No id is needed.** A component can have several pads, or be used several times on a page, without id clashes.
- **An empty ref is an error.** If `init` is called before the canvas exists, it throws `Canvas ref is empty; make sure it is mounted before calling init().` Calling it in `onMounted` avoids this.
- **There is still no automatic cleanup.** Unlike the ref-based key listener, this variant does not remove its handlers when the component is unmounted; `destroy` is still the page's job.

A canvas that is re-created needs the same care as in chapter 6. When a `v-if` throws the canvas away and makes a new one, the ref points at the new element, but the pen is still on the old one: drawing on the new canvas shows nothing until `destroy()` and `init(ref)` are called again.

## 8. A signing form

Everything so far goes into one realistic form. A visitor types a name, signs in a box that fits the page and sends. The page refuses an empty name or an empty pad, sends the signature as a PNG file, and when the server fails it says so and keeps the signature, so the visitor does not have to sign again:

```vue
<script setup>
const signature = useDigitalSignature()
const name = ref('')
const message = ref('')
const rows = ref([])
const log = text => rows.value.push(text)
let blank = ''
const SCALE = 2

onMounted(() => {
  const canvas = document.getElementById('sign')
  // size first: changing width or height resets the pen
  canvas.width = canvas.clientWidth * SCALE
  canvas.height = canvas.clientHeight * SCALE
  signature.init('sign')
  canvas.getContext('2d').scale(SCALE, SCALE)
  canvas.getContext('2d').lineWidth = 2.5
  blank = canvas.toDataURL()
})
onBeforeUnmount(() => signature.destroy())

async function submit() {
  const canvas = document.getElementById('sign')
  if (!name.value.trim()) { message.value = 'Please type your name.'; log(message.value); return }
  if (canvas.toDataURL() === blank) { message.value = 'Please sign in the box.'; log(message.value); return }
  const blob = await new Promise(resolve => canvas.toBlob(resolve, 'image/png'))
  const form = new FormData()
  form.append('name', name.value.trim())
  form.append('signature', blob, 'signature.png')
  try {
    const response = await fetch('/api/contract', { method: 'POST', body: form })
    if (!response.ok) throw new Error(`Server answered ${response.status}`)
    message.value = 'Thank you, your signature was received.'
  } catch (error) {
    message.value = `Could not send: ${error.message}. Your signature is still here, try again.`
  }
  log(message.value)
}

function clear() {
  signature.clear()
  message.value = ''
}
</script>

<template>
  <input id="name" v-model="name" placeholder="Full name">
  <canvas id="sign" style="width: 300px; height: 120px; border: 1px solid #999; display: block; margin: 10px 0; box-sizing: content-box; touch-action: none" />
  <button id="clear" @click="clear">Clear</button>
  <button id="submit" @click="submit">Sign</button>
  <p id="message">{{ message }}</p>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
</template>
```

```text
Please type your name.
Please sign in the box.
Could not send: Server answered 500. Your signature is still here, try again.
Thank you, your signature was received.
The server received 2 requests; the last carried the name Maria Lopez and a PNG image of 600 by 240 pixels
```

Follow the messages from the top. The first press of Sign has no name, and the second has a name but no signature; each gets its own sentence. Then the signature is drawn and sent, the server fails once, and the page says that the signature is still there. The second send succeeds. The server receives two requests, and the last one carries the name and a PNG image of 600 by 240 pixels, which is the 300 by 120 box at scale two (chapter 4).

For your own forms the pattern is the same: size the canvas to its box before `init`, scale the pen if sharpness matters, keep an empty-pad picture for the "please sign" rule, send the canvas as a file, and call `destroy` when the page goes. Give the box room, and remember that a dot needs a short drag.

---

## Appendix A. Quick reference

### A.1 Functions

| Call | Returns | Notes |
|------|---------|-------|
| `init(canvasId)` | Nothing | Throws if the id is not found; sets the pen (black, 2 pixels, round ends) |
| `init(canvasRef)` | Nothing | Ref-based version; accepts a ref or an element; throws if it is empty |
| `clear()` | Nothing | Wipes the whole bitmap; keeps the pen settings |
| `save(fileName)` | Nothing | Downloads a PNG of the bitmap; the default name is `signature.png` |
| `destroy()` | Nothing | Removes the handlers and forgets the canvas; harmless if repeated |

### A.2 Browser pieces worth using

| Piece | Meaning |
|-------|---------|
| `canvas.toDataURL()` | The picture as text; equal for equal pictures |
| `canvas.toBlob(callback, 'image/png')` | The picture as a file object for uploading |
| `canvas.getContext('2d')` | The drawing context; settings such as `strokeStyle` and `lineWidth` can be changed |
| `touch-action: none` | CSS that keeps the browser from claiming touch gestures on the canvas |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| `init` with an unknown id | Throws `Canvas with id="..." not found.`; the previous canvas is forgotten |
| `init` (ref-based) with an empty ref | Throws `Canvas ref is empty; make sure it is mounted before calling init().` |
| `clear` or `save` before `init` | `TypeError`; the pad is not set up |
| `clear` after `destroy` | `TypeError`; the canvas is forgotten |
| Drawing after `destroy` | Nothing is drawn |
| Pointer leaves the canvas while drawing | The line ends; it does not continue on return |
| Press without movement | Nothing is drawn |
| Canvas rebuilt by the page | The pen stays on the old canvas until `destroy` and `init` are called again |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Calling `init` while the component is being set up.** The canvas is not there yet; use `onMounted`.
- **Calling `init` before the canvas of a dialog exists.** Wait with `nextTick`.
- **Showing the canvas at a different size from its bitmap.** The line lands away from the pointer (chapter 4).
- **Changing `width` or `height` after `init`.** It wipes the canvas and resets the pen.
- **Padding or a thick border on the canvas.** The line lands away from the pointer by their width.
- **Expecting a dot to appear.** A press without movement draws nothing.
- **Giving the visitor a box that is too small.** A stroke that leaves the box is cut off.
- **Saving an empty pad.** `save` does not check.
- **Relying on a white CSS background.** The saved PNG is transparent.
- **Forgetting `destroy`.** The handlers stay on a canvas that may be reused.
- **Using one instance for two canvases.** The pen moves to the last one.
- **Rebuilding the canvas with `v-if` without calling `init` again.**

### C.2 Behaviour worth knowing

- The composable keeps one canvas and the pen's state; it keeps no strokes. The bitmap is the only record.
- Handlers are attached for mouse (down, move, up, leave) and touch (start, move, end), and touch uses the first finger.
- The canvas cancels the default action of mouse and touch events it draws on, so the page does not scroll or select text while signing.
- Any mouse button draws.
- The pen is a black line, 2 pixels wide, with round ends. It can be changed through the context after `init`.
- Positions are measured from the canvas's box in page pixels and used as bitmap pixels.
- Several instances do not know about each other.
- Both versions are client-side only and need a real canvas; `save` needs a browser that allows downloads.

### C.3 Limits

| What | Limit |
|------|-------|
| Canvases per instance | One |
| Pen settings | Black, 2 pixels, round ends; others through the context |
| Undo, redo, stroke list | None |
| Dots | Not drawn |
| Output format | PNG, through `save`; anything else through the canvas |
| Information about the state | None; look at the canvas |

### C.4 Tuning

- Call `init` in `onMounted` (or after `nextTick` for a dialog), and `destroy` when the canvas goes.
- Make the bitmap match the box, times a scale for sharpness, before `init`.
- Keep a picture of the empty pad for the "please sign" rule.
- Set `touch-action: none` on the canvas.
- Give the box room: wider than the longest signature.
- Upload with `toBlob`; draw on a white copy if a white background is needed.
- Use the ref-based version when the canvas comes and goes or the component is reused.

## Appendix D. Message reference

### D.1 Errors

| Message | Cause |
|---------|-------|
| `Canvas with id="..." not found.` | `init` found no canvas with that id; thrown to the caller |
| `Canvas ref is empty; make sure it is mounted before calling init().` | Ref-based `init` was called before the canvas existed |

### D.2 Browser errors seen when the order of calls is wrong

| Message | Cause |
|---------|-------|
| `TypeError: Cannot read properties of undefined (reading ...)` | `clear` or `save` before any `init` |
| `TypeError: Cannot read properties of null (reading ...)` | `clear` after `destroy`, or after an `init` that failed |
