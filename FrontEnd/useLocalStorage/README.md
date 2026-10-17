# useLocalStorage

*J2OS Platform 2.4*

A visitor picks the dark theme and expects it to still be dark tomorrow. A long form should not lose the half-written text when the tab is closed by mistake. A login should be remembered until the visitor signs out. All three need a small place in the browser that outlives the page, and the browser has one: local storage.

**useLocalStorage** is a composable of the Nuxt front end of J2OS that gives access to it through four functions. It has three properties:

- **Four functions.** `persist`, `findByKey`, `removeByKey` and `getKeys`; that is the whole interface.
- **Text in, text out.** Values are saved as text and come back as text, the way the browser stores them.
- **Honest failures.** A missing or full storage throws an `Error` with a clear message instead of quietly doing nothing.

In a few lines:

```vue
<script setup>
const store = useLocalStorage()

const theme = ref(store.findByKey('theme') ?? 'light')   // null if never saved

function choose(value) {
  theme.value = value
  store.persist('theme', value)                         // survives closing the browser
}
</script>

<template>
  <button @click="choose('dark')">Dark</button>
  <button @click="choose('light')">Light</button>
  <p>{{ theme }}</p>
</template>
```

This guide starts with how the composable works (chapter 1). Chapter 2 saves and reads text, and chapter 3 saves objects as JSON. Chapter 4 looks at what is shared (between instances, tabs and other code). Chapter 5 covers a storage that is blocked or full. Chapter 6 builds a draft note that survives a reload. The appendices work as a reference.

## Contents

1. [How useLocalStorage works](#1-how-uselocalstorage-works)
2. [Saving and reading text](#2-saving-and-reading-text)
3. [Objects and JSON](#3-objects-and-json)
4. [What is shared](#4-what-is-shared)
5. [When the storage is blocked or full](#5-when-the-storage-is-blocked-or-full)
6. [Putting it together](#6-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are complete pages of a Nuxt project: each is one `.vue` file. In Nuxt `ref`, `watch`, `onMounted` and `useLocalStorage` need no `import`, so none is written.

---

## 1. How useLocalStorage works

First, a few terms, because they come up throughout the guide:

- **Local storage.** A small key and value store that every site gets in the visitor's browser. It stays after the page is closed, the browser is restarted and the computer is switched off.
- **Key.** The name a value is saved under, such as `theme`.
- **Quota.** The amount of data a site may keep there. It is small, usually around five million characters in total.

The functions of this composable:

| Function | Role |
|----------|------|
| `persist(key, value)` | Saves a value under a key, replacing an older one |
| `findByKey(key)` | Returns the saved text, or `null` when there is none |
| `removeByKey(key)` | Deletes a key |
| `getKeys()` | Lists the keys of the whole storage |

Keep these six points in mind:

- **Everything is text.** The browser turns whatever it is given into text. Objects need `JSON.stringify` and `JSON.parse` (chapter 3).
- **Reading a missing key gives `null`,** not an error and not an empty string.
- **The storage is checked when the composable is created.** If it cannot be used then, every function throws for the life of that instance (chapter 5).
- **The storage is shared.** All instances, all tabs of the site and all code on the page see the same keys.
- **Nothing expires.** A value stays until it is removed, or until the visitor clears the site's data.
- **Nothing is encrypted,** and any script of the page can read it.

## 2. Saving and reading text

Start with the plain case: remember a short text. The composable offers exactly what is needed for that, a call to save, a call to read, and a call to delete. The page below goes through each of them and prints what comes back:

```vue
<script setup>
const store = useLocalStorage()
const rows = ref([])
const log = (label, value) => rows.value.push(`${label}: ${JSON.stringify(value)}`)

onMounted(() => {
  log('missing key', store.findByKey('theme'))
  log('persist returns', store.persist('theme', 'dark'))
  log('theme', store.findByKey('theme'))
  store.persist('theme', 'light')
  log('theme after overwrite', store.findByKey('theme'))
  store.persist('empty', '')
  log('empty text', store.findByKey('empty'))
  store.persist('visits', 42)
  log('number 42 comes back as', store.findByKey('visits'))
  store.persist('flag', true)
  log('true comes back as', store.findByKey('flag'))
  store.persist('nothing', null)
  log('null comes back as', store.findByKey('nothing'))
  store.persist('unknown', undefined)
  log('undefined comes back as', store.findByKey('unknown'))
  store.persist('profile', { name: 'Sara' })
  log('an object comes back as', store.findByKey('profile'))
  log('keys', store.getKeys())
  log('remove returns', store.removeByKey('theme'))
  log('removed key', store.findByKey('theme'))
  log('remove a missing key', store.removeByKey('nope'))
  log('keys now', store.getKeys())
})
</script>

<template>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
  <p>Basic</p>
</template>
```

```text
missing key: null
persist returns: undefined
theme: "dark"
theme after overwrite: "light"
empty text: ""
number 42 comes back as: "42"
true comes back as: "true"
null comes back as: "null"
undefined comes back as: "undefined"
an object comes back as: "[object Object]"
keys: ["empty","nothing","theme","unknown","flag","profile","visits"]
remove returns: undefined
removed key: null
remove a missing key: undefined
keys now: ["empty","nothing","unknown","flag","profile","visits"]
```

The first thing to take from this is how a *missing* key behaves. `findByKey` returns `null` for a key that was never saved. It does not throw, and it does not return an empty text. That makes the usual pattern very short: read, and fall back to a default when the answer is `null`. Saving to an existing key replaces the value, and deleting a key that is not there is harmless, so neither call needs a check before it.

The second thing is that **storage keeps text, and nothing but text.** Anything else is turned into text by the browser on the way in, and it comes back as that text:

- A number or a boolean returns as `"42"` or `"true"`. Convert it when you read, with `Number(...)` or a comparison with `"true"`.
- `null` and `undefined` return as the words `"null"` and `"undefined"`, which are not `null`. This is the surprise that causes most bugs, so do not save these two at all: remove the key instead.
- An object returns as `"[object Object]"`, and the data is lost for good. The next chapter shows how to save objects properly.
- An empty text is a real value (`""`), different from a key that is missing.

The third call, `getKeys()`, lists every key of the site, including ones that other code has saved. The order is the browser's own, not the order of saving and not alphabetical. When the order matters, sort it yourself:

```js
// The order of getKeys() is the browser's own; sort when the order matters.
const keys = store.getKeys().sort()
```

A site seldom has only one part that saves things. A common prefix on your keys keeps them recognisable and makes it easy to find and remove them together:

```js
// All keys of this feature start with one prefix, so they are easy to find and to remove.
const PREFIX = 'shop.'
const store = useLocalStorage()

export const shopStore = {
  save: (key, value) => store.persist(PREFIX + key, value),
  find: (key) => store.findByKey(PREFIX + key),
  remove: (key) => store.removeByKey(PREFIX + key),
  clear: () => store.getKeys().filter(k => k.startsWith(PREFIX)).forEach(k => store.removeByKey(k))
}
```

One name is already taken. The J2OS `useApiClient` reads the key `token` to build the `Authorization` header, so do not use that name for anything else.

## 3. Objects and JSON

Real pages rarely remember a single word. They remember a filter, a cart, a form in progress, and these are objects and arrays. The browser cannot store them directly, but turning them into JSON text and back is easy. Two small helpers keep the idea in one place and add a fallback for a key that is missing or damaged:

```js
const store = useLocalStorage()

// Save any JSON-friendly value, load it back with a fallback.
export const saveJson = (key, value) => store.persist(key, JSON.stringify(value))

export const loadJson = (key, fallback = null) => {
  const text = store.findByKey(key)
  if (text === null) return fallback
  try {
    return JSON.parse(text)
  } catch {
    return fallback
  }
}
```

Every page of the project can then save and read data with one line each. The page below sends a variety of values through the helpers:

```vue
<script setup>
const store = useLocalStorage()
const rows = ref([])

const save = (key, value) => store.persist(key, JSON.stringify(value))
const load = (key, fallback = null) => {
  const text = store.findByKey(key)
  if (text === null) return fallback
  try {
    return JSON.parse(text)
  } catch {
    return fallback
  }
}

onMounted(() => {
  const log = (label, value) => rows.value.push(`${label}: ${JSON.stringify(value)}`)
  save('cart', { items: [{ id: 7, qty: 2 }], coupon: null })
  log('stored text', store.findByKey('cart'))
  log('loaded back', load('cart'))
  log('a key that was never saved', load('wishlist', []))
  store.persist('broken', '{"items": [')
  log('text that is not JSON', load('broken', { items: [] }))
  save('date', new Date(Date.UTC(2026, 0, 15)))
  log('a Date comes back as', load('date'))
  save('set', new Set([1, 2]))
  log('a Set comes back as', load('set'))
  save('nested', { a: { b: [1, 2.5, 'x', true, null] } })
  log('nested values', load('nested'))
})
</script>

<template>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
  <p>Json</p>
</template>
```

```text
stored text: "{\"items\":[{\"id\":7,\"qty\":2}],\"coupon\":null}"
loaded back: {"items":[{"id":7,"qty":2}],"coupon":null}
a key that was never saved: []
text that is not JSON: {"items":[]}
a Date comes back as: "2026-01-15T00:00:00.000Z"
a Set comes back as: {}
nested values: {"a":{"b":[1,2.5,"x",true,null]}}
```

Plain data makes the trip unchanged: objects, arrays, numbers, booleans, `null` and text, at any depth. A key that was never saved gives the fallback, and so does text that is not valid JSON, because the helper catches the error. This second case matters. A value written by an older version of the page, or by other code, may be in a different shape, and a page that falls back instead of crashing will survive it.

JSON has limits, and they show on the last rows. A date comes back as ISO text, so turn it into a `Date` again when you read. A `Set` comes back as an empty object, and a `Map` or a function is lost the same way. Convert these to arrays or plain objects before saving.

A good habit is to check the fields you rely on after reading. If a new version of the page adds a field, old saved data will not have it, and a default for the missing field costs one line.

## 4. What is shared

Local storage belongs to the *site*, not to a page and not to a composable. Everything that runs on the site sees the same keys, and it helps to know exactly how far that goes. The page below saves a counter and reads it back in several ways:

```vue
<script setup>
const first = useLocalStorage()
const second = useLocalStorage()
const rows = ref([])

onMounted(() => {
  const log = (label, value) => rows.value.push(`${label}: ${JSON.stringify(value)}`)
  const before = first.findByKey('counter')
  log('counter found on load', before)
  first.persist('counter', String(Number(before ?? 0) + 1))
  log('read through the other instance', second.findByKey('counter'))
  log('read through window.localStorage', window.localStorage.getItem('counter'))
  window.localStorage.setItem('from-outside', 'yes')
  log('keys, seen by the composable', first.getKeys())
  log('temporary test key left behind', first.getKeys().includes('__storage_test__'))
})
</script>

<template>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
  <p>Shared</p>
</template>
```

```text
-- first visit
counter found on load: null
read through the other instance: "1"
read through window.localStorage: "1"
keys, seen by the composable: ["counter","from-outside"]
temporary test key left behind: false
-- after reloading the page
counter found on load: "1"
-- a second tab of the same site
counter found on load: "2"
```

The output shows the sharing from every side. Two separate instances of `useLocalStorage()` see the same data, and so does `window.localStorage` itself. Reloading the page finds the counter saved by the first visit, and a second tab of the same site sees it as well (and counts on from there). Keys saved by other code appear in `getKeys()` too. The temporary key that the composable uses to test the storage is removed at once, so it leaves nothing behind.

This leads to a few rules of good conduct:

- **Data belongs to the site's address** (scheme, host and port). Another site, another port or another browser profile has its own storage and cannot see yours.
- **Anything on the page can read it,** including a script that got in through a cross-site scripting hole. Do not keep secrets here. For a long-lived login, prefer a cookie that page code cannot read, when your back end supports one.
- **Other tabs are not told about changes.** The composable gives no notification, so read again when you need the current state.

## 5. When the storage is blocked or full

Local storage is a convenience, and it can be unavailable. The visitor may have blocked site data, a private window may refuse it, and a page in an embedded frame may not be allowed to use it. A good page keeps working without it.

The composable looks once, when it is created, and remembers the answer. The page below creates it where the storage is blocked and calls every function:

```vue
<script setup>
const rows = ref([])

onMounted(() => {
  const log = text => rows.value.push(text)
  let store
  try {
    store = useLocalStorage()
    log('creating it: no error')
  } catch (e) {
    log(`creating it: ${e.name}: ${e.message}`)
    return
  }
  for (const [label, call] of [
    ['persist', () => store.persist('a', '1')],
    ['findByKey', () => store.findByKey('a')],
    ['removeByKey', () => store.removeByKey('a')],
    ['getKeys', () => store.getKeys()]
  ]) {
    try {
      call()
      log(`${label}: worked`)
    } catch (e) {
      log(`${label}: ${e.name}: ${e.message}`)
    }
  }
})
</script>

<template>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
  <p>Blocked</p>
</template>
```

```text
creating it: no error
persist: Error: Browser does not support local storage
findByKey: Error: Browser does not support local storage
removeByKey: Error: Browser does not support local storage
getKeys: Error: Browser does not support local storage
```

Creating the composable never throws, even when the storage is blocked. The failure comes later: every function throws `Error: Browser does not support local storage`, from the first call on.

The storage can also be full, because the quota is shared by everything the site keeps. The page below tries to save a value that is too large, and then creates the composable after the storage was filled up:

```vue
<script setup>
const store = useLocalStorage()
const rows = ref([])
const fill = ref(false)

function tryLarge() {
  const log = text => rows.value.push(text)
  // About ten million characters, more than the usual five-million-character limit.
  const big = 'x'.repeat(10_000_000)
  try {
    store.persist('large', big)
    log('large value stored')
  } catch (e) {
    log(`error message starts with: ${e.message.slice(0, 18)}`)
    log(`cause: ${e.cause?.name}`)
    log(`stored anyway: ${store.findByKey('large') !== null}`)
  }
  store.persist('small', 'still works')
  log(`a small value afterwards: ${store.findByKey('small')}`)
}

function fillUp() {
  // Another part of the page fills the storage to the brim.
  let chunk = 1_000_000
  let n = 0
  while (chunk >= 1) {
    try {
      window.localStorage.setItem('f' + n++, 'x'.repeat(chunk))
    } catch {
      chunk = Math.floor(chunk / 2)
    }
  }
  fill.value = true
}

function afterFull() {
  const log = text => rows.value.push(text)
  try {
    const late = useLocalStorage()
    late.findByKey('anything')
    log('created after the storage was full: works')
  } catch (e) {
    log(`created after the storage was full: ${e.message}`)
  }
  try {
    window.localStorage.removeItem('f0')
    const again = useLocalStorage()
    again.findByKey('anything')
    log('created again after freeing space: works')
  } catch (e) {
    log(`created again after freeing space: ${e.message}`)
  }
}
</script>

<template>
  <button id="large" @click="tryLarge">Store a huge value</button>
  <button id="fill" @click="fillUp">Fill the storage</button>
  <button id="late" @click="afterFull">Create after full</button>
  <p>{{ fill ? 'filled' : 'free' }}</p>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
</template>
```

```text
error message starts with: Failed to persist:
cause: QuotaExceededError
stored anyway: false
a small value afterwards: still works
created after the storage was full: Browser does not support local storage
created again after freeing space: works
```

Two different things happen here. A value that does not fit makes `persist` throw an `Error` whose message begins with `Failed to persist:`, with the original browser error in `error.cause` (`QuotaExceededError`). Nothing was stored and the storage worked normally afterwards.

The second case is easy to miss. When the storage is *already full at creation*, the check at creation itself needs a little room, finds none, and the composable believes the storage is unsupported. Every function then throws `Browser does not support local storage`, and an instance created at that moment stays unusable even after space is freed. Only an instance created afterwards works, as the last line shows.

So create the composable once, early, while there is room, and share it:

```js
// Create the composable once, at startup, while the storage still has room.
// utils/storage.js
export const storage = useLocalStorage()
```

Because the storage may fail for any of these reasons, wrap each use so that the page keeps working without it:

```js
// Storage is a convenience: when it is missing or full, the page keeps working.
const safe = (action, fallback = null) => {
  try {
    return action()
  } catch {
    return fallback
  }
}

const theme = safe(() => store.findByKey('theme'), 'light')
safe(() => store.persist('theme', theme))
```

## 6. Putting it together

A draft note is the classic use: whatever the visitor types is saved at once, so closing the tab by mistake loses nothing. The page below saves the text under one key as it is typed and restores it on the next visit with a short message. *Clear* removes the key instead of saving an empty text. Where the storage is blocked, the page still works with the text in memory and warns that it will be lost:

```vue
<script setup>
const store = useLocalStorage()
const KEY = 'draft-note'

const available = ref(true)
const fallback = {}                        // used only when the storage is not available

function read() {
  try {
    return store.findByKey(KEY)
  } catch {
    available.value = false
    return fallback[KEY] ?? null
  }
}

function write(text) {
  try {
    if (text === '') store.removeByKey(KEY)
    else store.persist(KEY, text)
  } catch {
    available.value = false
    fallback[KEY] = text
  }
}

const note = ref(read() ?? '')
const restored = ref(note.value !== '')
watch(note, write)
</script>

<template>
  <textarea id="note" v-model="note" />
  <p v-if="restored" id="restored">Draft restored.</p>
  <p v-if="!available" id="warn">Your draft will be lost when you close this page.</p>
  <button id="clear" @click="note = ''">Clear</button>
</template>
```

```text
first visit: note '', restored message False
stored under the key: 'Call the lab at nine'
after reloading: note 'Call the lab at nine', restored message True
after Clear: key is None
storage blocked: warning shown True, note 'Typed with no storage'
after reloading: note ''
```

The pieces fit the earlier chapters:

- **Reading is a fallback.** The first visit finds `null` and shows an empty note with no message (chapter 2).
- **Saving happens on every change,** under the single key `draft-note`; a reload brings the text back and the page says so.
- **Clearing is `removeByKey`,** not saving an empty text, so the key does not linger.
- **Every use is wrapped** in `try` and `catch`, as in the previous chapter. With the storage blocked the page showed a warning and kept the text for as long as the page lived; after a reload it was gone, which is exactly what the warning promised.

---

## Appendix A. Quick reference

### A.1 Functions

| Call | Returns | Notes |
|------|---------|-------|
| `persist(key, value)` | Nothing | Saves as text; replaces an older value; throws if unavailable or full |
| `findByKey(key)` | Text or `null` | `null` when the key does not exist |
| `removeByKey(key)` | Nothing | A missing key is fine |
| `getKeys()` | Array of text | All keys of the site; browser's order |

### A.2 How values come back

| Saved | Read as |
|-------|---------|
| Text | The same text |
| Number, boolean | The text of it, such as `"42"` or `"true"` |
| `null`, `undefined` | `"null"`, `"undefined"` |
| Object | `"[object Object]"` (use JSON) |
| Never saved | `null` |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Storage blocked or unusable when the composable was created | Every function throws `Browser does not support local storage` |
| Storage full when the composable was created | The same, as if it were unsupported |
| Value does not fit | `persist` throws `Failed to persist: ...`; the browser's error is in `cause` |
| Key does not exist | `findByKey` gives `null`; `removeByKey` does nothing |
| Damaged JSON | Up to your own `JSON.parse`; the composable does not parse |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Saving objects directly.** They become `"[object Object]"`.
- **Expecting numbers or booleans back.** They come back as text.
- **Treating `"null"` as `null`.** Only a missing key gives `null`.
- **Not catching errors.** A blocked or full storage throws.
- **Creating the composable late,** when the storage may already be full.
- **Storing secrets.** Any script of the page can read them.
- **Using the key `token`** for anything but the login token.
- **Using the key `__storage_test__`.** It is the composable's temporary test key, and creating an instance removes any value saved under it.
- **Relying on the order of `getKeys()`.**
- **Expecting other tabs to be told.** The composable does not notify.
- **Storing large data.** The quota is small and shared.
- **Reading old data with the new shape in mind.** Use fallbacks.

### C.2 Behaviour worth knowing

- The storage is tested once, when the composable is created, by writing and removing a temporary key.
- A failed test is remembered for the life of that instance.
- Only text is stored; the browser converts anything else.
- The storage is shared by all instances, tabs and scripts of the same site.
- Data stays until removed or until the visitor clears it.
- The composable has no expiry, prefix, JSON or notification features of its own.

### C.3 Limits

| What | Limit |
|------|-------|
| Total size | The browser's quota, usually around five million characters |
| Value type | Text |
| Lifetime | Until removed or cleared by the visitor |
| Scope | One site address in one browser profile |
| Security | Readable by any script of the page; not encrypted |
| Change notifications | None |

### C.4 Tuning

- Create the composable once, at startup, and share it.
- Wrap every use in a helper that falls back, as in chapter 5.
- Convert to and from JSON in two small helpers.
- Give your keys a common prefix.
- Keep values small; store identifiers rather than whole records.
- Remove what is no longer needed (drafts after sending, for example).
- Keep secrets out, or keep them short-lived.
- Tell the visitor when a draft cannot be saved.

## Appendix D. Message reference

### D.1 Errors

| Message | Cause |
|---------|-------|
| `Browser does not support local storage` | The storage could not be used when the composable was created (blocked, unavailable or full) |
| `Failed to persist: ...` | The browser refused to save; the end of the message is the browser's reason, and `cause` holds the original error |
