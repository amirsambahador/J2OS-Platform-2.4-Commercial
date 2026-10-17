# useFullscreen

*J2OS Platform 2.4*

Some content is made to be looked at closely. A chart is easy to read on a laptop, but on a meeting-room screen it should fill everything. A photo gallery, a map or a video deserves the whole display when the visitor asks for it. Every modern browser can do this for any element of a page. It is called *fullscreen mode*, and it hides the browser's own bars and everything else on the page until the visitor leaves it.

The browser's interface for it is small but has a few traps: it only works as a reaction to a click, it does not tell you whether it worked, and it knows nothing about Vue. **useFullscreen** is a composable of the Nuxt front end of J2OS that wraps it in four functions. It has three properties:

- **Four functions.** `init`, `open`, `close` and `toggle`; that is the whole interface.
- **One element, chosen by its id.** `init` finds the element once, and the other functions work on it.
- **Quiet failures.** If the browser refuses, the composable writes a message to the console and does not throw.

In a few lines:

```vue
<script setup>
const fullscreen = useFullscreen()

onMounted(() => fullscreen.init('chart'))      // the id of the element to enlarge
</script>

<template>
  <div id="chart" style="background: white">
    <button @click="fullscreen.toggle()">Full screen</button>
    ...
  </div>
</template>
```

This guide starts with how the composable works (chapter 1) and the first page that uses it (2). Chapter 3 shows how the page can follow the real state, which is how a button changes its label. Chapter 4 deals with the most common trouble, an element that Vue has rebuilt. Chapter 5 shows what the browser does to the element, and chapter 6 explains the refusals. Chapter 7 builds a small report page with tabs. The appendices work as a reference.

## Contents

1. [How useFullscreen works](#1-how-usefullscreen-works)
2. [The first fullscreen box](#2-the-first-fullscreen-box)
3. [Following the real state](#3-following-the-real-state)
4. [When the element is rebuilt](#4-when-the-element-is-rebuilt)
5. [What the browser does to the element](#5-what-the-browser-does-to-the-element)
6. [When the browser says no](#6-when-the-browser-says-no)
7. [Putting it together](#7-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are complete pages of a Nuxt project: each is one `.vue` file. In Nuxt `ref`, `watch`, `onMounted` and `useFullscreen` need no `import`, so none is written. Fullscreen mode is something to see, so most pages also write a short log of what happened, and the log is printed under the page.

---

## 1. How useFullscreen works

First, a few terms, because they come up throughout the guide:

- **Fullscreen mode.** The browser stretches one element over the whole screen and hides everything else. The rest of the page is still there and still running; it is only not shown.
- **Fullscreen element.** The element that was stretched. The browser exposes it as `document.fullscreenElement`, which is `null` when nothing is stretched.
- **User gesture.** A click, a touch or a key press by the visitor. The browser allows fullscreen mode only as a reaction to one, so that a page cannot take over the screen on its own.

The functions of this composable:

| Function | Role |
|----------|------|
| `init(id)` | Finds the element with this id and remembers it |
| `open()` | Asks the browser to show that element in fullscreen mode |
| `close()` | Leaves fullscreen mode |
| `toggle()` | Closes if something is in fullscreen mode, otherwise opens |

The way to think about it is simple. The composable holds **one element**. `init` tells it which. Everything else is a request to the browser about that element. Four consequences follow, and the rest of the guide returns to each:

- **`init` comes first,** and the element must already exist. The place for it is `onMounted`.
- **Only the chosen element and its content are shown** while fullscreen mode lasts. A button outside it disappears with the rest of the page, so the controls belong inside the element.
- **The composable keeps no state.** It does not know whether fullscreen mode is on. The browser knows, and tells the page through an event (chapter 3).
- **Problems are not thrown.** If the browser says no, the reason goes to the console (chapter 6).

## 2. The first fullscreen box

The smallest useful page has three parts: an element with an `id`, one call to `init` once the page is on screen, and a button. A photo viewer, for example:

```vue
<script setup>
const fullscreen = useFullscreen()

onMounted(() => fullscreen.init('photo'))
</script>

<template>
  <div id="photo" style="background: #222; color: white; padding: 16px">
    <img src="/gallery/lake.jpg" alt="Lake" style="max-width: 100%">
    <button @click="fullscreen.toggle()">Full screen</button>
  </div>
</template>
```

The `id` is what `init` looks for, and the button sits inside the element, so it is still visible when the element fills the screen. `toggle` is the right choice for one button that does both jobs: it leaves fullscreen mode if the browser is in it and enters otherwise.

Two functions exist for the cases where the page needs to wait for the result, or has separate buttons. The page below has all three, and writes what happens into a log:

```vue
<script setup>
const fullscreen = useFullscreen()
const rows = ref([])
const state = () => document.fullscreenElement ? document.fullscreenElement.id : 'none'

onMounted(() => {
  fullscreen.init('box')
  document.addEventListener('fullscreenchange', () => rows.value.push(`fullscreenchange -> now ${state()}`))
})

async function run(label, action) {
  const result = action()
  rows.value.push(`${label}: returns ${result instanceof Promise ? 'a promise' : JSON.stringify(result)}`)
  await result
  rows.value.push(`${label}: after it settles, fullscreen is ${state()}`)
}
</script>

<template>
  <div id="box" style="background:#2196F3;color:white;padding:8px">
    Box
    <button id="open" @click="run('open', fullscreen.open)">Open</button>
    <button id="close" @click="run('close', fullscreen.close)">Close</button>
    <button id="toggle" @click="run('toggle', fullscreen.toggle)">Toggle</button>
    <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
  </div>
</template>
```

```text
open: returns a promise
open: after it settles, fullscreen is box
fullscreenchange -> now box
toggle: returns undefined
toggle: after it settles, fullscreen is box
fullscreenchange -> now none
toggle: returns undefined
toggle: after it settles, fullscreen is none
fullscreenchange -> now box
close: returns a promise
close: after it settles, fullscreen is none
fullscreenchange -> now none
close: returns a promise
close: after it settles, fullscreen is none
```

Reading the output:

- **`open` and `close` return promises,** and `await` waits until the browser has finished. After `open` the box was the fullscreen element.
- **`toggle` returns nothing,** so it cannot be awaited. The line after the first toggle still says `box` although the exit happened a moment later. Use `toggle` for a simple button and do not read the state on the next line.
- **`fullscreenchange`** is the browser's own event, fired on `document` once for every entry and every exit. It is the dependable way to learn the state, and the next chapter builds on it.
- **`close` when nothing is enlarged** is harmless, and so is calling it twice.

## 3. Following the real state

Fullscreen mode can end without any of your buttons being pressed: the visitor presses the Esc key, or the browser leaves for its own reasons. A button that says "Exit full screen" must notice this, or it will lie. The fix is to listen to `fullscreenchange` and look at `document.fullscreenElement`:

```js
const fullscreen = useFullscreen()
const isFull = ref(false)

function sync() {
  isFull.value = document.fullscreenElement?.id === 'chart'
}

onMounted(() => {
  fullscreen.init('chart')
  document.addEventListener('fullscreenchange', sync)   // also fires when the visitor presses Esc
})
onUnmounted(() => document.removeEventListener('fullscreenchange', sync))
```

The page keeps a `ref` called `isFull` and updates it from the event, never from its own buttons. Then the label is right whichever way the state changed. Remove the listener when the component goes away, because `document` outlives the component.

Only compare with *your* element's id. Another part of the page may enlarge something else, and `document.fullscreenElement` would then be that element.

## 4. When the element is rebuilt

`init` receives an id, but what it remembers is the **element itself**, the object that the browser found at that moment. Vue often replaces elements: a box under `v-if`, a panel with a changing `:key`, a block inside a tab. The new element has the same id, but the composable still holds the old one.

The page below hides and shows its box, and then tries to open it. Then it calls `init` again, and tries once more:

```vue
<script setup>
const fullscreen = useFullscreen()
const rows = ref([])
const log = text => rows.value.push(text)
const state = () => document.fullscreenElement ? document.fullscreenElement.id : 'none'
const show = ref(true)

const original = console.error
console.error = (...args) => log('console.error: ' + args.map(String).join(' '))
onUnmounted(() => { console.error = original })

onMounted(() => {
  fullscreen.init('box')
  document.addEventListener('fullscreenchange', () => log(`fullscreenchange -> now ${state()}`))
})

async function openIt() { await fullscreen.open(); log(`open: fullscreen is ${state()}`) }
async function rebuild() { show.value = false; await nextTick(); show.value = true; await nextTick(); log('the box was rebuilt') }
function again() { fullscreen.init('box'); log('init called again') }
function removeInside() { show.value = false }
</script>

<template>
  <div v-if="show" id="box">
    Box
    <button id="open" @click="openIt">Open</button>
    <button id="remove" @click="removeInside">Remove the box</button>
  </div>
  <button id="rebuild" @click="rebuild">Rebuild</button>
  <button id="again" @click="again">Init again</button>
  <button id="open2" @click="openIt">Open (outside)</button>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
</template>
```

```text
the box was rebuilt
console.error: Failed to enter fullscreen mode: TypeError: Element is not connected
open: fullscreen is none
init called again
open: fullscreen is box
fullscreenchange -> now box
fullscreenchange -> now none
open: fullscreen is box
fullscreenchange -> now box
fullscreenchange -> now none
```

- **After the rebuild, `open` did nothing.** The console received `Failed to enter fullscreen mode: TypeError: Element is not connected`. The old element is no longer on the page, so the browser cannot enlarge it.
- **After `init` was called again, `open` worked.**
- **When an enlarged element is removed from the page,** the browser leaves fullscreen mode by itself and fires `fullscreenchange`, as the last lines show. Nothing is left behind.

The rule is to call `init` whenever the element is created, not only once. For an element under `v-if` or a changing key, a `watch` does it:

```js
const fullscreen = useFullscreen()
const show = ref(true)

// The box is created again each time show turns true, so hand the new element over.
watch(show, async (visible) => {
  if (!visible) return
  await nextTick()
  fullscreen.init('box')
}, { flush: 'post' })

onMounted(() => fullscreen.init('box'))
```

`await nextTick()` matters: the watcher runs before Vue has put the new element on the page, and `init` would find the old one or none.

`init` also has two behaviours worth knowing. It **throws** when the id does not exist, and before it looks it **forgets the previous element**. The page below shows both, and what happens when `init` is called twice:

```vue
<script setup>
const fullscreen = useFullscreen()
const rows = ref([])
const log = text => rows.value.push(text)
const state = () => document.fullscreenElement ? document.fullscreenElement.id : 'none'

const original = console.error
console.error = (...args) => log('console.error: ' + args.map(String).join(' '))
onUnmounted(() => { console.error = original })

async function afterFailedInit() {
  fullscreen.init('box')
  try { fullscreen.init('nope') } catch (e) { log(`second init: ${e.message}`) }
  await fullscreen.open()
  log(`open after the failed init: fullscreen is ${state()}`)
}

async function twice() {
  fullscreen.init('box')
  fullscreen.init('box2')
  await fullscreen.open()
  log(`init twice, open: fullscreen is ${state()}`)
}

async function detached() {
  const el = document.createElement('div')
  el.id = 'temp'
  document.body.appendChild(el)
  fullscreen.init('temp')
  el.remove()
  await fullscreen.open()
  log(`open an element that left the page: fullscreen is ${state()}`)
}
</script>

<template>
  <div id="box">Box</div>
  <div id="box2">Box 2</div>
  <button id="failed" @click="afterFailedInit">Failed init</button>
  <button id="twice" @click="twice">Twice</button>
  <button id="detached" @click="detached">Detached</button>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
</template>
```

```text
second init: Element with id="nope" not found.
open after the failed init: fullscreen is none
init twice, open: fullscreen is box2
console.error: Failed to enter fullscreen mode: TypeError: Element is not connected
open an element that left the page: fullscreen is none
```

- **An unknown id** makes `init` throw `Element with id="nope" not found.` The usual cause is calling `init` while the component is still being set up, before its template exists (chapter 6). Call it in `onMounted`.
- **After that failure `open()` does nothing,** because the earlier element was already forgotten. After a failed `init`, call `init` again with a good id.
- **Two calls to `init`** leave the last one in charge; `box2` opened.
- **An element that has already left the page** is refused with the same `Element is not connected` as before.

## 5. What the browser does to the element

When an element enters fullscreen mode, the browser restyles it. The page below measures a box of 300 by 100 pixels before, during and after:

```vue
<script setup>
const fullscreen = useFullscreen()
const rows = ref([])
const look = () => {
  const box = document.getElementById('box')
  const s = getComputedStyle(box)
  return `position ${s.position}, size ${Math.round(box.getBoundingClientRect().width)}x${Math.round(box.getBoundingClientRect().height)}, margin ${s.marginLeft}, background ${s.backgroundColor}, matches :fullscreen ${box.matches(':fullscreen')}`
}

onMounted(() => {
  fullscreen.init('box')
  rows.value.push(`window ${innerWidth}x${innerHeight}`)
  rows.value.push(`before: ${look()}`)
  document.addEventListener('fullscreenchange', () => rows.value.push(`${document.fullscreenElement ? 'inside' : 'after'}: ${look()}`))
})
</script>

<template>
  <div id="box" style="width:300px;height:100px;margin:auto">
    <button id="open" @click="fullscreen.open()">Open</button>
    <button id="close" @click="fullscreen.close()">Close</button>
    <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
  </div>
</template>
```

```text
window 1280x720
before: position static, size 300x100, margin 482px, background rgba(0, 0, 0, 0), matches :fullscreen false
inside: position fixed, size 1280x720, margin 0px, background rgba(0, 0, 0, 0), matches :fullscreen true
after: position static, size 300x100, margin 482px, background rgba(0, 0, 0, 0), matches :fullscreen false
```

- **The size you set is ignored.** The box filled the whole window, and went back afterwards.
- **The position becomes `fixed` and the margin becomes zero,** so a centring margin is gone while it lasts.
- **The `:fullscreen` selector matches the element,** so a style rule can change its look only while it is enlarged.

Behind the element the browser paints a backdrop, and it is black. An element with no background of its own therefore shows black behind its content. A light page, such as a chart with dark lines, needs a background of its own, and it should be allowed to scroll, because the window may be smaller than the content. Both are one rule:

```vue
<style scoped>
#panel:fullscreen {
  background: white;        /* do not rely on what is behind it */
  overflow: auto;           /* long content can still be scrolled */
  padding: 24px;
}
</style>
```

The page below confirms that scrolling works, that the backdrop is black, and that the rest of the page keeps running underneath:

```vue
<script setup>
const fullscreen = useFullscreen()
const rows = ref([])
const log = text => rows.value.push(text)
const ticks = ref(0)

onMounted(() => {
  setInterval(() => ticks.value++, 100)
  fullscreen.init('tall')
})

async function look() {
  await fullscreen.open()
  const tall = document.getElementById('tall')
  const before = ticks.value
  tall.scrollTop = 150
  await new Promise(resolve => setTimeout(resolve, 600))
  log(`scrolling inside the enlarged element: scrollTop ${tall.scrollTop}, content ${tall.scrollHeight}px in ${tall.clientHeight}px`)
  log(`colour behind it: ${getComputedStyle(tall, '::backdrop').backgroundColor}`)
  log(`the page kept running while enlarged: ${ticks.value > before + 3}`)
}
</script>

<template>
  <div id="tall" style="height:100px;overflow:auto">
    <div style="height:2000px">Tall</div>
  </div>
  <button id="look" @click="look">Look</button>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
</template>
```

```text
scrolling inside the enlarged element: scrollTop 150, content 2000px in 720px
colour behind it: rgb(0, 0, 0)
the page kept running while enlarged: true
```

The rest of the page is not paused. A timer or a data refresh goes on, and the reactive state keeps changing, so the page is up to date when the visitor leaves.

**More than one enlarged element.** Browsers keep a stack: opening a second element while the first is enlarged shows the second, and leaving goes back to the first. The page below has two composables on two boxes; A opens, then B is opened from inside A, then A's composable is asked to close:

```vue
<script setup>
const first = useFullscreen()
const second = useFullscreen()
const rows = ref([])
const log = text => rows.value.push(text)
const state = () => document.fullscreenElement ? document.fullscreenElement.id : 'none'

onMounted(() => {
  first.init('a')
  second.init('b')
})

async function openA() { await first.open(); log(`A opens: fullscreen is ${state()}`) }
async function openB() { await second.open(); log(`B opens: fullscreen is ${state()}`) }
async function closeA() { await first.close(); log(`A closes: fullscreen is ${state()}`) }
async function openAagain() { await first.open(); log(`A opens again: fullscreen is ${state()}`) }
</script>

<template>
  <div id="a">
    A
    <button id="open-a" @click="openA">Open A</button>
    <button id="open-b-from-a" @click="openB">Open B</button>
    <button id="close-a" @click="closeA">Close with A</button>
    <button id="again-a" @click="openAagain">Open A again</button>
    <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
  </div>
  <div id="b">B</div>
</template>
```

```text
A opens: fullscreen is a
B opens: fullscreen is b
A closes: fullscreen is a
A opens: fullscreen is a
A closes: fullscreen is none
```

`close` is not tied to the element that its composable holds. It undoes the latest opening, so the first `close` returned to A and the second left fullscreen mode. The practical rule is to keep to one enlarged element at a time.

## 6. When the browser says no

The browser refuses fullscreen requests in a few situations, and the composable turns each into a console message. Nothing is thrown, so the page keeps going. The page below runs into the common ones:

```vue
<script setup>
const fullscreen = useFullscreen()
const rows = ref([])
const log = text => rows.value.push(text)
const state = () => document.fullscreenElement ? document.fullscreenElement.id : 'none'

const original = console.error
console.error = (...args) => log('console.error: ' + args.map(String).join(' '))
onUnmounted(() => { console.error = original })

try {
  fullscreen.init('box')
} catch (e) {
  log(`init while the page is still being built: ${e.message}`)
}

onMounted(async () => {
  await fullscreen.open()
  log(`open before init: fullscreen is ${state()}`)
  await fullscreen.close()
  log('close while not in fullscreen: nothing happened')
  log(`toggle before init returns ${JSON.stringify(fullscreen.toggle())}`)
  fullscreen.init('box')
  await fullscreen.open()
  log(`open without a click: fullscreen is ${state()}`)
})
</script>

<template>
  <div id="box">Box</div>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
</template>
```

```text
init while the page is still being built: Element with id="box" not found.
open before init: fullscreen is none
close while not in fullscreen: nothing happened
toggle before init returns undefined
console.error: Failed to enter fullscreen mode: TypeError: Permissions check failed
open without a click: fullscreen is none
```

- **`init` during set-up** throws `Element with id="box" not found.` because the template does not exist yet. Use `onMounted`.
- **`open`, `close` and `toggle` before `init`** do nothing at all, silently.
- **`open` without a click** is refused. The console receives `Failed to enter fullscreen mode: TypeError: Permissions check failed`. Call `open` from a click, touch or key handler, not from `onMounted` and not after a long wait.

The browser also announces a refusal on `document` with a `fullscreenerror` event. That event is the way for the page to react, for example by showing a message. The page below produces refusals on purpose and tries a few more cases:

```vue
<script setup>
const fullscreen = useFullscreen()
const rows = ref([])
const log = text => rows.value.push(text)
const state = () => document.fullscreenElement ? document.fullscreenElement.textContent : 'none'

const original = console.error
console.error = (...args) => log('console.error: ' + args.map(String).join(' '))
onUnmounted(() => { console.error = original })

onMounted(async () => {
  document.addEventListener('fullscreenerror', () => log('fullscreenerror event fired'))
  fullscreen.init('box')
  await fullscreen.open()
  await new Promise(resolve => setTimeout(resolve, 50))
  log('on load: open() was called without a click')
})

async function openTwice() {
  fullscreen.init('box')
  await fullscreen.open()
  await fullscreen.open()
  await new Promise(resolve => setTimeout(resolve, 100))
  log(`open called twice: fullscreen is "${state()}"`)
}

async function hiddenBox() {
  fullscreen.init('ghost')
  await fullscreen.open()
  log(`element with display none: fullscreen is "${state()}"`)
}

async function duplicate() {
  for (const text of ['first', 'second']) {
    const el = document.createElement('div')
    el.id = 'dup'
    el.textContent = text
    document.body.appendChild(el)
  }
  fullscreen.init('dup')
  await fullscreen.open()
  log(`two elements with one id: the enlarged one is "${state()}"`)
}
</script>

<template>
  <div id="box">Box</div>
  <div id="ghost" style="display:none">Ghost</div>
  <button id="twice" @click="openTwice">Twice</button>
  <button id="hidden" @click="hiddenBox">Hidden</button>
  <button id="pick" @click="duplicate">Duplicate</button>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
</template>
```

```text
console.error: Failed to enter fullscreen mode: TypeError: Permissions check failed
fullscreenerror event fired
on load: open() was called without a click
console.error: Failed to enter fullscreen mode: TypeError: Permissions check failed
fullscreenerror event fired
open called twice: fullscreen is "Box"
element with display none: fullscreen is "Ghost"
two elements with one id: the enlarged one is "first"
```

- **A refusal fires `fullscreenerror`** after the console message.
- **One click allows one request.** A second `open()` in the same handler failed in the same way. The element stayed enlarged, so nothing was lost; the lesson is only to call `open` once per click.
- **An element with `display: none`** became visible and enlarged: fullscreen mode overrides the hiding.
- **With two elements of the same id,** `init` took the first one on the page. Ids must be unique.

`open` never throws, so when the result matters, listen for the event or look at the result:

```js
const fullscreen = useFullscreen()
const refused = ref(false)

// open() never throws: a refusal is only written to the console,
// and the browser fires "fullscreenerror" on the document.
onMounted(() => {
  document.addEventListener('fullscreenerror', () => { refused.value = true })
})

async function enter() {
  refused.value = false
  await fullscreen.open()
  return document.fullscreenElement !== null    // or look at refused
}
```

Other refusals follow the same path: a browser or device that does not allow fullscreen mode for an element, or a page in an embedded frame that is not permitted to. All of them end with a console message and no change.

## 7. Putting it together

A report page with two tabs. Each tab has a panel with its numbers and a button. The button enlarges the panel and its label follows the real state. The panel is a new element for each tab, so the page hands it over again after every switch. A message is ready for the case of a refusal, and the panel gets its own background and scrolling for fullscreen mode:

```vue
<script setup>
const fullscreen = useFullscreen()
const tab = ref('sales')
const isFull = ref(false)
const refused = ref(false)
const numbers = {
  sales: [['Mon', 12], ['Tue', 18], ['Wed', 9], ['Thu', 22]],
  visits: [['Mon', 140], ['Tue', 205], ['Wed', 98], ['Thu', 260]]
}

function sync() {
  isFull.value = document.fullscreenElement?.id === 'panel'
}

onMounted(() => {
  fullscreen.init('panel')
  document.addEventListener('fullscreenchange', sync)
  document.addEventListener('fullscreenerror', () => { refused.value = true })
})
onUnmounted(() => document.removeEventListener('fullscreenchange', sync))

// The panel is a new element for every tab, so hand it over again.
watch(tab, async () => {
  await nextTick()
  fullscreen.init('panel')
})
</script>

<template>
  <button id="sales" @click="tab = 'sales'">Sales</button>
  <button id="visits" @click="tab = 'visits'">Visits</button>

  <section id="panel" :key="tab">
    <header>
      <b id="title">{{ tab }}</b>
      <button id="toggle" @click="fullscreen.toggle()">{{ isFull ? 'Exit full screen' : 'Full screen' }}</button>
    </header>
    <div v-for="[day, n] in numbers[tab]" :key="day" class="row">{{ day }} {{ n }}</div>
    <p id="state">{{ isFull ? 'full' : 'normal' }}</p>
    <p v-if="refused" id="refused">Your browser did not allow full screen.</p>
  </section>
</template>

<style scoped>
#panel:fullscreen {
  background: white;
  overflow: auto;
  padding: 24px;
}
</style>
```

```text
start:                         sales tab, panel normal, button "Full screen"
after the button is clicked:   sales tab, panel full, button "Exit full screen"
after the browser leaves:      sales tab, panel normal, button "Full screen"
after switching to Visits:     visits tab, panel normal, button "Full screen"
button clicked again:          visits tab, panel full, button "Exit full screen"
after the browser leaves:      visits tab, panel normal, button "Full screen"
```

What each part does:

- **`onMounted`** calls `init` once and attaches the two listeners: `fullscreenchange` keeps `isFull` true, and `fullscreenerror` shows the message.
- **The `watch` on the tab** calls `init` again after Vue has replaced the panel, which is why the button still works on the second tab.
- **The button is inside the panel,** so it is visible while the panel is enlarged, and it uses `toggle` because one button does both jobs.
- **The label comes from `isFull`,** so it returned to "Full screen" when the browser left fullscreen mode, without any click on the button.
- **The style rule** gives the enlarged panel its own white background, scrolling and some space.

---

## Appendix A. Quick reference

### A.1 Functions

| Call | Returns | Notes |
|------|---------|-------|
| `init(id)` | Nothing | Throws if the id is not found; forgets the old element first |
| `open()` | Promise | Needs a click; failures go to the console |
| `close()` | Promise | Leaves one level of fullscreen mode; harmless if there is none |
| `toggle()` | Nothing | Cannot be awaited |

### A.2 Browser pieces worth using

| Piece | Meaning |
|-------|---------|
| `document.fullscreenElement` | The enlarged element, or `null` |
| `fullscreenchange` event on `document` | Fires after every entry and exit |
| `fullscreenerror` event on `document` | Fires when a request is refused |
| `:fullscreen` selector | Matches the enlarged element |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| `init` with an unknown id | Throws `Element with id="..." not found.`; the old element is forgotten |
| `open`, `close` or `toggle` before `init` | Nothing happens, no message |
| `open` without a user gesture | Console: `Failed to enter fullscreen mode:` with `Permissions check failed`, and a `fullscreenerror` event; nothing happens |
| `open` on an element that left the page | Console: `Failed to enter fullscreen mode:` with `Element is not connected`; nothing happens |
| Element removed while enlarged | The browser leaves fullscreen mode; `fullscreenchange` fires |
| `close` when nothing is enlarged | Nothing happens |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Calling `init` while the component is being set up.** The element is not there yet; use `onMounted`.
- **Calling `init` once for an element that is rebuilt.** The old element is stale (chapter 4).
- **Trying again after a failed `init` without calling `init` again.** The old element is gone.
- **Placing the buttons outside the element.** They are hidden while it is enlarged.
- **Calling `open` outside a click,** or twice from one click. It fails quietly.
- **Awaiting `toggle`.** It returns nothing.
- **Relying on an exception from `open`.** Use `fullscreenerror` or check `document.fullscreenElement`.
- **Changing a label from your own buttons.** The visitor can leave with Esc; follow `fullscreenchange`.
- **Giving the element no background.** The backdrop is black.
- **Expecting `close` to always leave completely.** With nested openings it goes back one step.
- **Using the same id twice.** The first one on the page is the one found.
- **Setting a fixed size in pixels** and expecting it to hold. It is ignored while enlarged.

### C.2 Behaviour worth knowing

- `init` forgets the previous element before it looks for the new one.
- The composable keeps no state of its own besides the element: no flag, no event, no cleanup.
- Several instances do not know about each other; the browser keeps the real state.
- Only the enlarged element and its content are shown; the rest of the page stays alive underneath.
- The browser changes the element's size, position and margin while it is enlarged.
- When the element leaves the page the browser exits fullscreen mode by itself.
- A refused request fires `fullscreenerror` on the document.
- The enlarged element can scroll.

### C.3 Limits

| What | Limit |
|------|-------|
| Elements per instance | One |
| Start | Only from a user gesture, one request per gesture |
| Element | Must be on the page when `open` is called |
| Information about the state | None from the composable; use `document.fullscreenElement` and `fullscreenchange` |
| Support | Depends on the browser and the device |

### C.4 Tuning

- Call `init` in `onMounted`, and again whenever the element is rebuilt.
- Put the controls inside the element.
- Give the element its own background and let it scroll.
- Follow `fullscreenchange` for the label, and remove the listener when the component goes away.
- Show a message on `fullscreenerror` where fullscreen mode is important.
- Keep one enlarged element at a time.

## Appendix D. Message reference

### D.1 Errors

| Message | Cause |
|---------|-------|
| `Element with id="..." not found.` | `init` found no element with that id; thrown to the caller |

### D.2 Console messages

| Message | Cause |
|---------|-------|
| `Failed to enter fullscreen mode:` followed by the error | The browser refused; seen with `Permissions check failed` (no user gesture) and `Element is not connected` (element left the page) |
| `Failed to exit fullscreen mode:` followed by the error | The browser refused to leave fullscreen mode |
