# useFileLoader

*J2OS Platform 2.4*

Some pages need a big library that most pages do not. A map needs its mapping library, a text editor needs its editor, a chart needs its chart engine. Loading them for every page would slow down the whole application, so they are loaded only when a page that needs them is opened, and are removed again when the visitor leaves. This is what the `<script>` and `<link>` tags are for, but writing those tags by hand inside a Vue component brings a list of questions. When is the file ready? What if two components ask for the same file? What if the server does not answer?

**useFileLoader** is a composable of the Nuxt front end of J2OS that answers these questions. It adds script and style sheet files to the page while the application is running, and takes them away again. It has three main features:

- **Promises.** Every import returns a promise that settles when the file has loaded or failed, so the page can wait with `await`.
- **Shared files.** If two components ask for the same address, the file is loaded once, and it is removed only when the last of them lets go.
- **Clean failures.** A file that cannot be loaded rejects the promise and leaves nothing behind on the page.

Here is the shape of it:

```vue
<script setup>
const loader = useFileLoader()

onMounted(async () => {
  await loader.importJavaScriptSourceFile('/vendor/slider.js')   // wait until it has run
  window.Slider.start('#gallery')
})

onUnmounted(() => loader.removeJavaScriptSourceFile('/vendor/slider.js'))
</script>

<template>
  <div id="gallery" />
</template>
```

Start with how the composable works (chapter 1) and the first page that loads a style sheet and a script (2). Chapter 3 explains what happens when two components share a file, and chapter 4 what to do when one file needs another. Chapter 5 is about failures and trying again. Chapter 6 shows what removing a file does and does not undo. Chapter 7 builds a small dashboard with chart panels from everything before it. The appendices are meant for quick lookups.

## Contents

1. [How useFileLoader works](#1-how-usefileloader-works)
2. [The first style sheet and script](#2-the-first-style-sheet-and-script)
3. [Two components, one file](#3-two-components-one-file)
4. [Files that need each other](#4-files-that-need-each-other)
5. [When a file cannot be loaded](#5-when-a-file-cannot-be-loaded)
6. [Loading again, and what removing does not undo](#6-loading-again-and-what-removing-does-not-undo)
7. [A dashboard of chart panels](#7-a-dashboard-of-chart-panels)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Each example is a page you can drop into a Nuxt project, and each is a single `.vue` file. In Nuxt `ref`, `onMounted`, `onUnmounted` and `useFileLoader` need no `import`, so none is written. The files that the pages load (`/ext/...`) are small stand-ins for a real library, and most pages write a short log of what happened, printed under the page.

---

## 1. How useFileLoader works

Before anything else, the terms you need:

- **Dynamic file.** A script or style sheet added to the page by the loader, while the application is already running. The loader marks its tags with the attribute `data-dynamic="true"`, which is how they are told apart from the files of the page itself.
- **Address.** The text passed to a function, such as `/vendor/slider.js`. The loader uses it as a key to remember files, so it is compared as plain text.
- **Counter.** For every address the loader counts how many times it was asked for the file, and how many times that was undone. This is what lets two components share a file.

What this composable gives you:

| Function | Role |
|----------|------|
| `importStyleSheetFile(href)` | Adds a `<link rel="stylesheet">` and returns a promise |
| `importJavaScriptSourceFile(src)` | Adds a `<script>` and returns a promise |
| `removeStyleSheetFile(href)` | Gives the style sheet back; removes the tag when nobody uses it any more |
| `removeJavaScriptSourceFile(src)` | The same for a script |
| `removeAllDynamicFiles()` | Removes every dynamic tag at once, whoever asked for it |

The way to think about it: every `import` is a **request** and every `remove` is a **release**. A file stays on the page while the requests outnumber the releases. Four consequences follow, and the rest of the guide returns to each:

- **The memory is shared by the whole application,** not by one component. The loader is a function, but the maps behind it live outside it, so every component sees the same state.
- **A promise is the only signal.** The tag is added at once, but the file is usable only after the promise resolves.
- **Removing a tag does not undo what the file did.** A script that has run has already set its global variables (chapter 6).
- **Addresses are compared as text.** `/vendor/a.js` and `./vendor/a.js` are two different files for the loader.

## 2. The first style sheet and script

A page that needs a library has a fixed routine: ask for the files when the page appears, wait, use them, and give them back when the page goes away. In code, `onMounted` asks and `onUnmounted` releases:

```js
onMounted(async () => {
  await Promise.all([
    loader.importStyleSheetFile('/vendor/slider.css'),
    loader.importJavaScriptSourceFile('/vendor/slider.js')
  ])
  // from here on both files are in place
})

onUnmounted(() => {
  loader.removeStyleSheetFile('/vendor/slider.css')
  loader.removeJavaScriptSourceFile('/vendor/slider.js')
})
```

`Promise.all` is a good fit when the files do not depend on each other, as a style sheet and a script usually do not. The page below does the same with stand-in files, and writes what the page looks like at each step: how many tags exist, what color the text has and whether the function from the script exists.

```vue
<script setup>
const loader = useFileLoader()
const rows = ref([])
const log = text => rows.value.push(text)
const tags = () => `${document.querySelectorAll('link[data-dynamic]').length} link, ${document.querySelectorAll('script[data-dynamic]').length} script`
const color = () => getComputedStyle(document.getElementById('sample')).color

onMounted(async () => {
  log(`before: ${tags()}, text colour ${color()}, function ${typeof window.externalFunction}`)

  const css = loader.importStyleSheetFile('/ext/box.css')
  const js = loader.importJavaScriptSourceFile('/ext/box.js')
  log(`both calls return a promise: ${css instanceof Promise && js instanceof Promise}`)
  log(`straight after the calls: ${tags()}, function ${typeof window.externalFunction}`)

  await Promise.all([css, js])
  log(`after the promises settle: text colour ${color()}, function ${typeof window.externalFunction}`)
  log(`the function answers: ${window.externalFunction()}`)

  loader.removeStyleSheetFile('/ext/box.css')
  loader.removeJavaScriptSourceFile('/ext/box.js')
  log(`after removing: ${tags()}, text colour ${color()}`)
  log(`the function after removing: ${typeof window.externalFunction}`)
})
</script>

<template>
  <p id="sample" class="ext-box">Sample</p>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
</template>
```

```text
before: 0 link, 0 script, text colour rgb(0, 0, 0), function undefined
both calls return a promise: true
straight after the calls: 1 link, 1 script, function undefined
after the promises settle: text colour rgb(255, 0, 0), function function
the function answers: hello from box.js
after removing: 0 link, 0 script, text colour rgb(0, 0, 0)
the function after removing: function
```

Reading the output:

- **The tags appear at once.** Straight after the calls there is already one `link` and one `script` in the page, but the function does not exist yet, because the script has not run.
- **The promise is the signal.** After `await` the text has the color from the style sheet and the function answers. Anything that uses the library belongs after the `await`.
- **Removing takes the tags away,** and the style sheet stops working, so the text is black again.
- **The function is still there after removing the script.** The script had already run, and removing its tag cannot take back what it did. Chapter 6 comes back to this.

A habit that saves trouble: write the addresses once, as constants, and use the constants for both the import and the release.

```js
// one place for the addresses, so that loading and releasing always match
const SLIDER_CSS = '/vendor/slider.css'
const SLIDER_JS = '/vendor/slider.js'
```

## 3. Two components, one file

A dashboard rarely has one chart. Each chart panel is its own component, and each asks for the same chart library. If every panel added its own tag, the library would be downloaded and run once per panel, and closing one panel would remove the library from under the others.

The loader prevents both. For every address it keeps **one promise** and **one counter**. The first request creates the tag; the later ones get the same promise back and add one to the counter. A release subtracts one, and only when the counter reaches zero is the tag removed.

```vue
<script setup>
const first = useFileLoader()
const second = useFileLoader()
const rows = ref([])
const log = text => rows.value.push(text)
const links = () => document.querySelectorAll('link[data-dynamic]').length

onMounted(async () => {
  const a = first.importStyleSheetFile('/ext/box.css')
  const b = second.importStyleSheetFile('/ext/box.css')
  log(`two users ask for the same file: same promise ${a === b}`)
  await a
  log(`link tags in the page: ${links()}`)

  first.removeStyleSheetFile('/ext/box.css')
  log(`after the first user lets go: ${links()}`)
  second.removeStyleSheetFile('/ext/box.css')
  log(`after the second user lets go: ${links()}`)
  second.removeStyleSheetFile('/ext/box.css')
  log(`one release too many: ${links()} (no error)`)

  await first.importStyleSheetFile('/ext/box.css')
  log(`asked for again: ${links()}`)
  second.removeStyleSheetFile('./ext/box.css')
  log(`released with another spelling of the address: ${links()}`)
  first.removeAllDynamicFiles()
  log(`removeAllDynamicFiles: ${links()}`)
})
</script>

<template>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
</template>
```

```text
two users ask for the same file: same promise true
link tags in the page: 1
after the first user lets go: 1
after the second user lets go: 0
one release too many: 0 (no error)
asked for again: 1
released with another spelling of the address: 1
removeAllDynamicFiles: 0
```

The page shows the whole life of one style sheet. Two users ask, and they get the identical promise (`same promise true`), with one tag on the page. The first user lets go and the tag stays, because the second still needs it. When the second lets go, the tag is removed. A third release does nothing and does not fail.

The last two lines are warnings:

- **Another spelling of the address is another file.** Releasing `./ext/box.css` did not touch the tag loaded as `/ext/box.css`. Always use the same constant.
- **`removeAllDynamicFiles` ignores the counters.** It removes everything, including files that other components still use. It belongs to a place where the whole application is torn down, not to a single component.

The counter counts **requests**, not components. If one component asks for the same file three times, it must release it three times:

```vue
<script setup>
const loader = useFileLoader()
const rows = ref([])
const log = text => rows.value.push(text)
const links = () => document.querySelectorAll('link[data-dynamic]').length

onMounted(async () => {
  await loader.importStyleSheetFile('/ext/box.css')
  await loader.importStyleSheetFile('/ext/box.css')
  await loader.importStyleSheetFile('/ext/box.css')
  log(`asked for three times: ${links()} link tag`)
  loader.removeStyleSheetFile('/ext/box.css')
  log(`released once: ${links()}`)
  loader.removeStyleSheetFile('/ext/box.css')
  log(`released twice: ${links()}`)
  loader.removeStyleSheetFile('/ext/box.css')
  log(`released three times: ${links()}`)
})
</script>

<template>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
  <p>Count</p>
</template>
```

```text
asked for three times: 1 link tag
released once: 1
released twice: 1
released three times: 0
```

This matters most with retry buttons (chapter 5): a retry that asks again for a file that is already loaded raises the counter without anybody noticing, and the file is never removed. The rule is simple: **every import is paired with one release**, and a retry repeats only what failed.

## 4. Files that need each other

Many libraries come in two parts: the library itself and a plugin or a language file that needs the library to exist. Both are scripts, and the first idea is to start them together with `Promise.all`. The loader adds both tags at once and the browser runs each script **as soon as it arrives**, in whatever order that happens. If the plugin arrives first, it looks for the library and does not find it.

The page below asks for a slow library and a quick plugin together, and then for a second pair one after the other:

```vue
<script setup>
const loader = useFileLoader()
const rows = ref([])
const log = text => rows.value.push(text)
window.__seen = []

onMounted(async () => {
  // libA.js is slow, pluginA.js is quick and needs it
  await Promise.all([
    loader.importJavaScriptSourceFile('/ext/libA.js'),
    loader.importJavaScriptSourceFile('/ext/pluginA.js')
  ])
  log(`together: ${window.__seen.join(', ')}`)

  await loader.importJavaScriptSourceFile('/ext/libB.js')
  await loader.importJavaScriptSourceFile('/ext/pluginB.js')
  log(`one after the other: ${window.__seen.join(', ')}`)
})
</script>

<template>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
  <p>Order</p>
</template>
```

```text
together: pluginA (libA missing), libA
one after the other: pluginA (libA missing), libA, libB, pluginB (found libB)
```

In the first pair the plugin ran before the library and reported that the library was missing. In the second pair each file waited for the one before it, and everything was in place. When one file needs another, the page asks for them in sequence:

```js
// wrong: the two files load side by side and the plugin may run first
await Promise.all([
  loader.importJavaScriptSourceFile('/vendor/lib.js'),
  loader.importJavaScriptSourceFile('/vendor/lib-plugin.js')
])

// right: the plugin is requested only after the library has run
await loader.importJavaScriptSourceFile('/vendor/lib.js')
await loader.importJavaScriptSourceFile('/vendor/lib-plugin.js')
```

Files that do not depend on each other, such as a style sheet and a script, are still best loaded together, because it is faster.

## 5. When a file cannot be loaded

A file can fail to load: the server is down, the address is wrong, the visitor has lost the connection. The promise then **rejects** with an `Error` whose message names the file, and the tag is removed from the page and from the loader's memory. Nothing is left behind.

```vue
<script setup>
const loader = useFileLoader()
const rows = ref([])
const log = text => rows.value.push(text)
const count = selector => document.querySelectorAll(selector).length

const attempt = (label, promise) => promise.then(
  () => log(`${label}: loaded`),
  error => log(`${label}: ${error.constructor.name}: ${error.message}`)
)

onMounted(async () => {
  await attempt('missing style sheet', loader.importStyleSheetFile('/ext/missing.css'))
  await attempt('missing script', loader.importJavaScriptSourceFile('/ext/missing.js'))
  log(`tags left behind: ${count('[data-dynamic]')}`)

  const same = [loader.importJavaScriptSourceFile('/ext/missing.js'), loader.importJavaScriptSourceFile('/ext/missing.js')]
  const results = await Promise.allSettled(same)
  log(`the same missing script asked twice: ${results.map(r => r.status).join(', ')}`)

  await attempt('script that throws when it runs', loader.importJavaScriptSourceFile('/ext/broken.js'))
  log(`and its tag is still there: ${count('script[data-dynamic]')}`)
})
</script>

<template>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
  <p>Errors</p>
</template>
```

```text
missing style sheet: Error: Failed to load CSS: /ext/missing.css
missing script: Error: Failed to load JS: /ext/missing.js
tags left behind: 0
the same missing script asked twice: rejected, rejected
script that throws when it runs: loaded
and its tag is still there: 1
```

Three things to take from the output:

- **The message tells the file and the kind.** `Failed to load CSS:` or `Failed to load JS:` followed by the address. It is the only information the loader gives, with no status code.
- **Everyone waiting gets the failure.** Asking twice for the same missing script gives two rejections.
- **A script that loads but throws when it runs is a success** as far as the loader is concerned. The promise resolves and the error goes to the console of the browser. A script that sets up a library should be checked by looking for what it was supposed to create.

An unhandled rejection is silent in the page, so every `await` of an import needs a `try/catch`, or a `.catch`. The failure also leaves the loader clean, which makes **trying again** safe. The page below fails twice and then succeeds, with a button that only calls the same function again:

```vue
<script setup>
const loader = useFileLoader()
const rows = ref([])
const log = text => rows.value.push(text)
const links = () => document.querySelectorAll('script[data-dynamic]').length

async function load() {
  try {
    await loader.importJavaScriptSourceFile('/ext/late.js')
    log(`loaded, tags ${links()}, value ${window.__late}`)
  } catch (error) {
    log(`failed: ${error.message}, tags ${links()}`)
  }
}
</script>

<template>
  <button id="load" @click="load">Load</button>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
</template>
```

```text
failed: Failed to load JS: /ext/late.js, tags 0
failed: Failed to load JS: /ext/late.js, tags 0
loaded, tags 1, value ready
loaded, tags 1, value ready
```

The second attempt really asked the server again (the tag was gone and the memory forgotten), and when the server recovered it worked. After success the next click returns at once, because the file is already loaded.

One more rule: if a page asks for several files and only one failed, a retry should ask only for that one. The style sheet that succeeded still holds a counter, and asking again would raise it (chapter 3). Chapter 7 shows how.

## 6. Loading again, and what removing does not undo

A removed file can be requested again, and then it is a new tag and the script runs a second time. For a library that sets up its own state this is usually harmless, but for a script that counts, registers handlers or adds elements, a second run can do harm.

```vue
<script setup>
const loader = useFileLoader()
const rows = ref([])
const log = text => rows.value.push(text)

onMounted(async () => {
  await loader.importJavaScriptSourceFile('/ext/counter.js')
  log(`first import: the file has run ${window.__runs} time(s)`)

  loader.removeJavaScriptSourceFile('/ext/counter.js')
  log(`after removing: tag gone, its function is ${typeof window.counterFunction}, runs ${window.__runs}`)

  await loader.importJavaScriptSourceFile('/ext/counter.js')
  log(`imported again: the file has run ${window.__runs} time(s)`)
  loader.removeJavaScriptSourceFile('/ext/counter.js')

  const early = loader.importJavaScriptSourceFile('/ext/counter.js')
  loader.removeJavaScriptSourceFile('/ext/counter.js')
  await early
  log(`removed before it finished loading: the promise still settled, runs ${window.__runs}`)

  const other = document.createElement('script')
  other.src = '/ext/counter.js'
  document.head.appendChild(other)
  await new Promise(resolve => setTimeout(resolve, 150))
  const before = window.__runs
  await loader.importJavaScriptSourceFile('/ext/counter.js')
  log(`file already on the page by other means: script tags ${document.querySelectorAll('script[src="/ext/counter.js"]').length}, runs added ${window.__runs - before}`)
})
</script>

<template>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
  <p>Reload</p>
</template>
```

```text
first import: the file has run 1 time(s)
after removing: tag gone, its function is function, runs 1
imported again: the file has run 2 time(s)
removed before it finished loading: the promise still settled, runs 3
file already on the page by other means: script tags 2, runs added 1
```

Four things stand out:

- **Removing does not undo.** After the script's tag was removed, the function it had defined was still there. Removing a script frees the tag and the loader's memory; the library stays in memory until the page is reloaded.
- **Importing again runs the script again.** The counter in the stand-in file went from one to two.
- **Removing before the file has arrived is allowed,** but it does not stop the download: the promise still settles and the script still runs.
- **The loader does not look at the page.** A file that was added by other means (a tag in the HTML, or another library) is not recognized; the loader added a second tag and the script ran a second time.

Two practical rules follow. First, a library's global variable only shows that the library is present, not that this loader put it there. The check below skips the import when the library is already present, whoever put it there:

```js
const already = typeof window.Slider !== 'undefined'
if (!already) {
  await loader.importJavaScriptSourceFile('/vendor/slider.js')
}
```

Second, for a library that must not run twice, do not release it on unmount; load it once and keep it. Releasing is for style sheets and for libraries that are cheap to reload.

## 7. A dashboard of chart panels

Now everything is put to work in one realistic page: a dashboard with two chart panels. Each panel needs a style sheet and a script from a chart library (stand-ins here), shows a message while it loads, shows an error with a **Try again** button when it fails, and gives the files back when it is closed.

The panel is the component that uses the loader:

```vue
<script setup>
defineProps({ title: { type: String, required: true } })

const loader = useFileLoader()
const status = ref('loading')
const message = ref('')
const target = ref(null)
const files = ['/ext/minichart.css', '/ext/minichart.js']
let styleRequest = null

// the style sheet is requested once; a retry only repeats what failed
function requestStyle() {
  styleRequest ??= loader.importStyleSheetFile(files[0]).catch(error => {
    styleRequest = null
    throw error
  })
  return styleRequest
}

async function start() {
  status.value = 'loading'
  try {
    await Promise.all([
      requestStyle(),
      loader.importJavaScriptSourceFile(files[1])
    ])
    window.MiniChart.draw(target.value, [3, 5, 2])
    status.value = 'ready'
  } catch (error) {
    message.value = error.message
    status.value = 'failed'
  }
}

onMounted(start)
onUnmounted(() => {
  loader.removeStyleSheetFile(files[0])
  loader.removeJavaScriptSourceFile(files[1])
})
</script>

<template>
  <section class="panel">
    <b>{{ title }}</b>
    <p v-if="status === 'loading'" class="status">Loading chart...</p>
    <p v-else-if="status === 'failed'" class="status">
      {{ message }}
      <button class="retry" @click="start">Try again</button>
    </p>
    <div v-show="status === 'ready'" ref="target" class="mini-chart" />
  </section>
</template>
```

A few lines deserve a look. `requestStyle` remembers the promise of the style sheet, so a retry does not ask for it twice, and forgets it if it failed, so a retry can ask again (chapters 3 and 5). The script is requested inside `start`, because a failed script is forgotten by the loader and may safely be requested again. The release in `onUnmounted` pairs with the requests: one for each file.

The dashboard shows or hides the panels with two buttons:

```vue
<script setup>
import ChartPanel from './ChartPanel.vue'

const showSales = ref(true)
const showVisits = ref(true)
</script>

<template>
  <button id="sales" @click="showSales = !showSales">Sales</button>
  <button id="visits" @click="showVisits = !showVisits">Visits</button>
  <ChartPanel v-if="showSales" title="Sales" class="p-sales" />
  <ChartPanel v-if="showVisits" title="Visits" class="p-visits" />
  <p>Dashboard</p>
</template>
```

The demo run starts with the chart server down, brings it back, retries the panels one by one and then closes them:

```text
server down: Sales: Failed to load JS: /ext/minichart.js Try again | Visits: Failed to load JS: /ext/minichart.js Try again
tags (link, script): [1, 0]
server back, Sales retried: Sales: chart bars 3-5-2 | Visits: Failed to load JS: /ext/minichart.js Try again
Visits retried: Sales: chart bars 3-5-2 | Visits: chart bars 3-5-2
tags (link, script): [1, 1]
Sales panel closed: tags [1, 1], panels 1
Visits panel closed: tags [0, 0], panels 0
Sales panel opened again: Sales: chart bars 3-5-2, tags [1, 1]
```

The output can be read from top to bottom. With the server down both panels show the failure, and the style sheet, which loaded, is on the page once. After the server returns, each panel recovers with its own button. Closing the first panel leaves both files in place for the second, and closing the second removes them. When a panel is opened again, the files come back.

---

## Appendix A. Quick reference

### A.1 Functions

| Call | Returns | Notes |
|------|---------|-------|
| `importStyleSheetFile(href)` | Promise | Rejects with `Failed to load CSS: <href>` |
| `importJavaScriptSourceFile(src)` | Promise | Rejects with `Failed to load JS: <src>`; the script is added with `async` |
| `removeStyleSheetFile(href)` | Nothing | Subtracts one from the counter; removes the tag at zero |
| `removeJavaScriptSourceFile(src)` | Nothing | The same for scripts |
| `removeAllDynamicFiles()` | Nothing | Removes every tag with `data-dynamic="true"` and logs each one |

### A.2 What the loader adds to the page

| Piece | Meaning |
|-------|---------|
| `<link rel="stylesheet" data-dynamic="true">` | A style sheet added by the loader |
| `<script async data-dynamic="true">` | A script added by the loader |
| `[data-dynamic="true"]` | Selector for finding the tags of the loader |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| File not found or server down | The promise rejects; the tag and the memory of the file are removed |
| The same failing file requested twice | Both promises reject |
| Request after a failure | A new attempt; the server is asked again |
| Script that throws when it runs | The promise resolves; the error goes to the browser console |
| Release without a request | Nothing happens |
| Release with another spelling of the address | Nothing happens; the file stays |
| Release before the file arrived | The tag is removed when the counter allows, but a download already started still finishes and runs |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Using the library before the promise resolves.** The tag exists, the code has not run.
- **Loading dependent files with `Promise.all`.** They run in the order they arrive.
- **Not catching a rejection.** The failure goes unseen and the page keeps waiting for a library that is not there.
- **Different spellings of one address.** They are different files for the loader and for the counters.
- **A retry that asks again for files that loaded.** The counter grows and the file is never removed.
- **Forgetting to release on unmount.** The files stay for the rest of the visit.
- **Using `removeAllDynamicFiles` in a component.** It removes files other components still need.
- **Expecting a removal to undo the script.** Globals, handlers and added elements remain.
- **Importing a file the page already has.** The loader does not look at the existing tags and runs it again.

### C.2 Behavior worth knowing

- The memory is shared by the whole application, so components cannot see each other's calls but do share files.
- The first request creates the tag; later requests for the same address receive the same promise.
- A failed file is forgotten completely, including its counter.
- The tags are added to `<head>`.
- A script tag is added with `async`; the order of arrival decides the order of running.
- A script that fails to run does not make the promise reject.
- Importing again after a removal downloads and runs the script again (the browser cache may answer).
- The loader works with the browser's `document` when a function is called, so the calls belong in code that runs on the client, such as `onMounted`.

### C.3 Limits

| What | Limit |
|------|-------|
| File types | Scripts (`src`) and style sheets (`href`) |
| Key | The address as plain text |
| Information about a failure | The message only; no status code |
| Loading order | None guaranteed between files requested together |
| Server-side rendering | Creating the loader is harmless, but every function fails there with `document is not defined`; call them on the client |

### C.4 Tuning

- Keep the addresses in constants and use them for both import and release.
- Wait with `await` and wrap it in `try/catch`.
- Load independent files together, dependent files one after the other.
- Pair every import with one release, in `onUnmounted`.
- On retry, ask again only for the files that failed.
- Check for the library's global before loading when it may already be present.

## Appendix D. Message reference

### D.1 Errors

| Message | Cause |
|---------|-------|
| `Failed to load CSS: <href>` | The style sheet could not be loaded |
| `Failed to load JS: <src>` | The script could not be loaded |

### D.2 Console messages

| Message | Cause |
|---------|-------|
| `Removing dynamic file: <address>` | `removeAllDynamicFiles` logs one line for every tag it removes |
