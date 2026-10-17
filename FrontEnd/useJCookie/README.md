# useJCookie

*J2OS Platform 2.4*

A visitor chooses Persian once and expects the next visit to start in Persian. A cookie banner must remember that the visitor already said yes. A short-lived choice, such as an open shopping cart, should disappear on its own after a while. Each of these needs a small note that travels with the browser, and the browser has one: the cookie.

**useJCookie** is a composable of the Nuxt front end of J2OS that reads, writes and deletes cookies through three functions. It has three properties:

- **Three functions.** `get`, `set` and `remove`; that is the whole interface.
- **A lifetime in minutes.** `set` takes the lifetime as a number of minutes, and the default is about 400 days.
- **Clear errors for bad input.** A blank name or an invalid lifetime throws an `Error` with a plain message.

In a few lines:

```vue
<script setup>
const cookie = useJCookie()

const lang = ref(cookie.get('lang') ?? 'en')       // null if never saved

function choose(value) {
  lang.value = value
  cookie.set('lang', value, 60 * 24 * 30)          // lives for 30 days
}
</script>

<template>
  <button @click="choose('fa')">فارسی</button>
  <button @click="choose('en')">English</button>
  <p>{{ lang }}</p>
</template>
```

This guide starts with how the composable works (chapter 1). Chapter 2 saves, reads and deletes a cookie, and chapter 3 looks at what comes back for different kinds of values. Chapter 4 is about the lifetime. Chapter 5 covers the size limit and the errors. Chapter 6 builds a consent banner. The appendices work as a reference.

## Contents

1. [How useJCookie works](#1-how-usejcookie-works)
2. [Saving, reading and deleting](#2-saving-reading-and-deleting)
3. [What comes back](#3-what-comes-back)
4. [The lifetime](#4-the-lifetime)
5. [Size, characters and errors](#5-size-characters-and-errors)
6. [Putting it together](#6-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are complete pages of a Nuxt project: each is one `.vue` file. In Nuxt `ref`, `onMounted`, `nextTick` and `useJCookie` need no `import`, so none is written.

---

## 1. How useJCookie works

A few terms first, since they return throughout the guide:

- **Cookie.** A small piece of text with a name that the browser keeps for a site and sends back to it with every request.
- **Lifetime.** How long the browser keeps the cookie. After that it is deleted without anyone doing anything.
- **Path.** The part of the site a cookie belongs to. Every cookie of this composable belongs to the whole site (`/`).

The functions of this composable:

| Function | Role |
|----------|------|
| `get(name)` | Returns the value of a cookie, or `null` when there is none |
| `set(name, value, minutes)` | Saves a cookie for the given number of minutes |
| `remove(name)` | Deletes a cookie |

The composable is a thin layer over `useCookie`, the cookie tool that Nuxt itself provides. It adds a check of the name and of the lifetime, a lifetime counted in minutes, and a sensible default of 400 days. A simple picture: a cookie is a *named note that survives the page*. You write it with `set`, read it with `get`, and tear it up with `remove`.

Four points to keep in mind, because the rest of the guide returns to each:

- **A write lands a moment later,** not on the same line. A page that needs the value at once must keep its own copy (chapter 2).
- **Values are converted when read.** The text `"42"` returns as the number `42`, and objects return as objects (chapter 3).
- **The lifetime is in minutes,** and a cookie without one lives about 400 days (chapter 4).
- **Cookies are small and readable.** The browser drops a cookie that is too large, and any script of the page can read it (chapter 5).

## 2. Saving, reading and deleting

The most common use of a cookie is a choice that should survive between visits: a language, a theme, a banner that has been dismissed. The three calls cover it. Read the cookie when the page starts, write it when the visitor chooses, and remove it when the choice is withdrawn:

```vue
<script setup>
const cookie = useJCookie()
const rows = ref([])
const log = (label, value) => rows.value.push(`${label}: ${JSON.stringify(value)}`)

onMounted(async () => {
  log('missing cookie', cookie.get('theme'))
  log('set returns', cookie.set('theme', 'dark'))
  log('read at once, through get', cookie.get('theme'))
  log('read at once, document.cookie', document.cookie)
  await nextTick()
  log('after a tick, through get', cookie.get('theme'))
  log('after a tick, document.cookie', document.cookie)
  cookie.set('theme', 'light')
  await nextTick()
  log('overwritten', cookie.get('theme'))
  log('remove returns', cookie.remove('theme'))
  await nextTick()
  log('after remove', cookie.get('theme'))
  log('after remove, document.cookie', document.cookie)
  log('remove a cookie that is not there', cookie.remove('nope'))
})
</script>

<template>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
  <p>Basic</p>
</template>
```

```text
missing cookie: null
set returns: undefined
read at once, through get: null
read at once, document.cookie: ""
after a tick, through get: "dark"
after a tick, document.cookie: "theme=dark"
overwritten: "light"
remove returns: undefined
after remove: null
after remove, document.cookie: ""
remove a cookie that is not there: undefined
```

Most of the output is what you would expect: a cookie that was never saved reads as `null` (never as an error or an empty text), saving under the same name again replaces the value, and removing a cookie that does not exist is harmless.

One line deserves attention. **Right after `set`, `get` still returns `null`,** and `document.cookie` is still empty. A tick later (`await nextTick()`) both show the new value. The write is real, but it lands a moment after the call returns. The same holds after `remove`.

In practice this almost never matters, because a page seldom saves a value and needs it back on the next line. When it does, the page should keep the value in its own variable and use the cookie only for the next visit:

```js
const cookie = useJCookie()
const theme = ref(cookie.get('theme') ?? 'light')

function choose(value) {
  cookie.set('theme', value)
  theme.value = value            // the page uses its own copy, not cookie.get()
}
```

The variable `theme` is what the page shows right now; the cookie is what the next visit will find. Keeping the two apart also makes the page simpler to read.

## 3. What comes back

A cookie is stored as text, but `get` does not always return text. It converts what it reads, which is convenient for objects and sometimes surprising for plain strings. The page below saves a series of values under one name and shows what each returns:

```vue
<script setup>
const cookie = useJCookie()
const rows = ref([])

onMounted(async () => {
  const samples = [
    ['text', 'hello'], ['number 42', 42], ['text "42"', '42'], ['text "007"', '007'], ['text "1e3"', '1e3'],
    ['boolean true', true], ['text "true"', 'true'], ['text "null"', 'null'],
    ['object', { id: 7, tags: ['a', 'b'] }], ['array', [1, 2]], ['empty text', ''], ['text with spaces', 'two words']
  ]
  for (const [label, value] of samples) {
    cookie.set('k', value)
    await nextTick()
    const back = cookie.get('k')
    rows.value.push(`${label} -> ${typeof back}: ${JSON.stringify(back)}`)
    cookie.remove('k')
    await nextTick()
  }
})
</script>

<template>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
  <p>Types</p>
</template>
```

```text
text -> string: "hello"
number 42 -> number: 42
text "42" -> number: 42
text "007" -> string: "007"
text "1e3" -> string: "1e3"
boolean true -> boolean: true
text "true" -> boolean: true
text "null" -> object: null
object -> object: {"id":7,"tags":["a","b"]}
array -> object: [1,2]
empty text -> string: ""
text with spaces -> string: "two words"
```

The rules, in the order they matter:

- **Objects and arrays are saved as JSON and come back whole.** A profile such as `{ id: 7, tags: [...] }` can be stored and read back as it is; there is no need to call `JSON.stringify` yourself.
- **Text that is plainly a number or a boolean is converted.** `"42"` (and the number `42`) returns as the number `42`, and `"true"` as the boolean `true`. Text that only resembles a number, such as `"007"` or `"1e3"`, stays text.
- **The text `"null"` returns as `null`,** which is exactly what a missing cookie looks like.
- **An empty text stays an empty text,** which is different from `null`. Use `=== null` when you mean "was never saved".

So the type you read is not always the type you saved. The simple defence is to treat every value as *possibly converted* when the type matters. An identifier such as `"12"` that must stay text can be turned back with `String(...)`, or given a fixed prefix when it is written (`"id-12"`) so that it never looks like a number.

## 4. The lifetime

Not every choice should last forever. A language preference may live for a year, an open shopping cart for a couple of hours, a "welcome" banner for a week. `set` takes the lifetime as its third argument, in **minutes**. Writing the numbers as minutes is error-prone, so two small helpers make them readable:

```js
const cookie = useJCookie()

// The lifetime is counted in minutes. Small helpers keep the numbers readable.
const hours = n => n * 60
const days = n => n * 60 * 24

cookie.set('cart', 'c-17', hours(2))     // two hours
cookie.set('lang', 'fa', days(30))       // thirty days
cookie.set('seen-tour', 'yes')           // no lifetime: about 400 days
```

A cookie saved without a lifetime lives about 400 days. That is also the longest the browser allows, and a longer lifetime is simply cut to it.

The page below sets four cookies with different lifetimes and then asks the browser how long each will live and which attributes it carries:

```vue
<script setup>
const cookie = useJCookie()

onMounted(() => {
  cookie.set('default-life', 'a')
  cookie.set('one-hour', 'b', 60)
  cookie.set('half-minute', 'c', 0.5)
  cookie.set('forever', 'd', 10_000_000)
})
</script>

<template>
  <p>Life</p>
</template>
```

```text
default-life  value 'a', path /, lasts about 400 days, secure False, httpOnly False, sameSite Lax
forever       value 'd', path /, lasts about 400 days, secure False, httpOnly False, sameSite Lax
half-minute   value 'c', path /, lasts about 29 seconds, secure False, httpOnly False, sameSite Lax
one-hour      value 'b', path /, lasts about 60 minutes, secure False, httpOnly False, sameSite Lax
```

The output shows three things. The default is the 400 days; a lifetime of 60 minutes lasts about an hour, and `10000000` minutes is cut back to 400 days. Half a minute (`0.5`) lasts about half a minute, because the lifetime is rounded down to whole seconds; the 29 seconds shown is those 30 seconds with a moment already gone.

The last columns matter for security. **The composable sets nothing beyond the lifetime and the path.** The cookies are not `Secure`, not `HttpOnly`, and the `SameSite` value is only the browser's own default (`Lax` here). Page code can read every one of them. This is perfectly fine for a language or a theme, and wrong for anything secret: a login or a token should be set by the server as an `HttpOnly` cookie.

Two special values need care. A lifetime of `0` does not make a cookie that lasts zero minutes; it means *already expired*, so no cookie is stored (the next chapter shows it). And a lifetime under one second does the same. To delete a cookie, call `remove`.

## 5. Size, characters and errors

Cookies are small, and the browser is strict about them. Before relying on one, it helps to know three things: what characters are safe, how large a cookie may be, and what happens when the input is wrong.

**Characters.** The composable encodes the value, so any text is safe, including semicolons, equal signs and Persian. The page below saves four values:

```vue
<script setup>
const cookie = useJCookie()
const rows = ref([])

onMounted(async () => {
  for (const value of ['a; b=c', 'سلام دنیا', 'x'.repeat(5000), 'y'.repeat(3000)]) {
    cookie.set('note', value)
    await nextTick()
    const stored = document.cookie.includes('note=')
    const back = cookie.get('note')
    rows.value.push(`${value.length > 20 ? value.length + ' characters' : JSON.stringify(value)} -> stored ${stored}, read back ${back === null ? 'null' : back === value ? 'the same' : JSON.stringify(back)}`)
    if (value.length < 20) rows.value.push(`   raw: ${document.cookie}`)
    cookie.remove('note')
    await nextTick()
  }
})
</script>

<template>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
  <p>Encode</p>
</template>
```

```text
"a; b=c" -> stored true, read back the same
raw: note=a%3B%20b%3Dc
"سلام دنیا" -> stored true, read back the same
raw: note=%D8%B3%D9%84%D8%A7%D9%85%20%D8%AF%D9%86%DB%8C%D8%A7
5000 characters -> stored false, read back null
3000 characters -> stored true, read back the same
```

The special characters and the Persian text are stored as percent-escapes (`a%3B%20b%3Dc`) and read back exactly. A value of 3000 characters is stored too. A value of 5000 characters is **not stored, and nothing complains:** `get` gives `null`. The browser refuses a cookie that is too large and does it silently. The exact limit comes a little further on.

**Wrong input.** Three things are checked by the composable and one by the cookie library underneath. The page below gives the functions bad input and writes down what happens:

```vue
<script setup>
const cookie = useJCookie()
const rows = ref([])
const caught = []
window.addEventListener('error', (event) => { caught.push(event.message) })

const attempt = (label, action) => {
  try {
    const result = action()
    rows.value.push(`${label}: ok${result === undefined ? '' : ' -> ' + JSON.stringify(result)}`)
  } catch (e) {
    rows.value.push(`${label}: ${e.constructor.name}: ${e.message}`)
  }
}

onMounted(async () => {
  attempt('get with an empty name', () => cookie.get(''))
  attempt('get with a blank name', () => cookie.get('   '))
  attempt('get without a name', () => cookie.get())
  attempt('set with a blank name', () => cookie.set(' ', 'x'))
  attempt('remove with a blank name', () => cookie.remove(''))
  attempt('negative lifetime', () => cookie.set('a', 'x', -1))
  attempt('lifetime NaN', () => cookie.set('a', 'x', Number.NaN))
  attempt('lifetime Infinity', () => cookie.set('a', 'x', Infinity))
  attempt('lifetime as text "10"', () => cookie.set('a', 'x', '10'))
  attempt('lifetime 0', () => cookie.set('zero', 'x', 0))
  await nextTick()
  rows.value.push(`lifetime 0 left a cookie: ${document.cookie.includes('zero=')}`)
  attempt('a name with a space', () => cookie.set('bad name', 'x'))
  await nextTick()
  await new Promise(resolve => setTimeout(resolve, 50))
  rows.value.push(`after waiting, reported to the page: ${JSON.stringify(caught)}`)
  rows.value.push(`cookie with that name exists: ${document.cookie.includes('bad')}`)
})
</script>

<template>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
  <p>Errors</p>
</template>
```

```text
get with an empty name: Error: Cookie name must not be null or blank
get with a blank name: Error: Cookie name must not be null or blank
get without a name: Error: Cookie name must not be null or blank
set with a blank name: Error: Cookie name must not be null or blank
remove with a blank name: Error: Cookie name must not be null or blank
negative lifetime: Error: Cookie lifetime must be a finite non-negative number
lifetime NaN: Error: Cookie lifetime must be a finite non-negative number
lifetime Infinity: Error: Cookie lifetime must be a finite non-negative number
lifetime as text "10": Error: Cookie lifetime must be a finite non-negative number
lifetime 0: ok
lifetime 0 left a cookie: false
a name with a space: TypeError: argument name is invalid: bad name
after waiting, reported to the page: []
cookie with that name exists: false
```

A blank, empty or missing name makes `get`, `set` and `remove` throw `Cookie name must not be null or blank`. A lifetime that is negative, `NaN`, infinite or text (even the text `"10"`) makes `set` throw `Cookie lifetime must be a finite non-negative number`, so pass a real number. A lifetime of `0` is accepted but stores nothing, and Nuxt prints a warning in the console. A name with a space is refused by the cookie library with a `TypeError`. Names should be made of letters, digits, `-` and `_`.

All of these are thrown at the moment of the call, so a `try` and `catch` around the call is enough. Because names come from your own code, keep them together, so that a typo cannot create a second cookie:

```js
const cookie = useJCookie()

// Names come from your own code, so keep them in one place.
const KEYS = { lang: 'lang', consent: 'consent', theme: 'theme' }

cookie.set(KEYS.lang, 'fa')
```

**The corners.** The page below looks at the exact size limit and at a few unusual cases:

```vue
<script setup>
const cookie = useJCookie()
const rows = ref([])
const log = (l, v) => rows.value.push(`${l}: ${typeof v === 'string' ? v : JSON.stringify(v)}`)
const tick = () => nextTick().then(() => new Promise(r => setTimeout(r, 20)))
const attempt = (l, f) => { try { const r = f(); log(l, r === undefined ? 'ok' : r) } catch (e) { log(l, e.constructor.name + ': ' + e.message) } }

onMounted(async () => {
  for (const [label, ch] of [['latin', 'x'], ['persian', 'س']]) {
    let lo = 1, hi = 6000
    while (lo < hi) {
      const mid = Math.ceil((lo + hi) / 2)
      cookie.remove('big'); await tick()
      cookie.set('big', ch.repeat(mid)); await tick()
      if (cookie.get('big') === ch.repeat(mid)) lo = mid; else hi = mid - 1
    }
    cookie.remove('big'); await tick(); cookie.set('big', ch.repeat(lo)); await tick()
    log(`largest ${label} value kept (characters)`, lo); log('  raw cookie length', document.cookie.length)
    cookie.remove('big'); await tick()
  }
  cookie.set('keep', 'y'.repeat(3000)); await tick()
  cookie.set('keep', 'z'.repeat(5000)); await tick()
  const k = cookie.get('keep'); log('overwrite with a too large value -> get', k === null ? null : k.length + ' x ' + k[0])
  log('  cookie really holds', (document.cookie.match(/keep=(.)/) || [])[1] ?? 'nothing')
  cookie.remove('keep'); await tick()
  cookie.set('u', undefined); await tick(); log('set undefined -> get', cookie.get('u')); log('  raw', document.cookie)
  cookie.set('n', null); await tick(); log('set null -> get', cookie.get('n')); log('  raw', document.cookie)
  cookie.set('two', 'first'); cookie.set('two', 'second'); await tick(); log('two sets in one tick', cookie.get('two'))
  cookie.remove('two'); await tick()
  cookie.set('short', 'x', 0.01); await tick(); log('lifetime 0.01 minutes stored', document.cookie.includes('short='))
  document.cookie = 'j=%7B%22a%22%3A1%7D'; document.cookie = 'plain={"a":1}'; document.cookie = 'raw=%E0%A4%A'
  await tick()
  log('foreign encoded json', cookie.get('j')); log('foreign plain json', cookie.get('plain')); attempt('foreign broken escape', () => cookie.get('raw'))
  cookie.set('first', 'a'); const other = useJCookie(); log('other instance reads at once', other.get('first')); await tick(); log('other instance after tick', other.get('first'))
  attempt('name with equals', () => cookie.set('a=b', 'x'))
  attempt('name with semicolon', () => cookie.set('a;b', 'x'))
  attempt('name non-ascii', () => cookie.set('نام', 'x'))
  attempt('value as Date', () => { cookie.set('d', new Date(0)); return 'set' }); await tick(); log('  back', [cookie.get('d')])
  attempt('value function', () => { cookie.set('f', () => 1); return 'set' }); await tick(); log('  back', [cookie.get('f')])
})
</script>

<template>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
  <p>Edge</p>
</template>
```

```text
largest latin value kept (characters): 4093
raw cookie length: 4097
largest persian value kept (characters): 682
raw cookie length: 4096
overwrite with a too large value -> get: 3000 x y
cookie really holds: y
set undefined -> get: null
raw:
set null -> get: null
raw:
two sets in one tick: second
lifetime 0.01 minutes stored: false
foreign encoded json: {"a":1}
foreign plain json: {"a":1}
foreign broken escape: URIError: URI malformed
other instance reads at once: null
other instance after tick: a
name with equals: TypeError: argument name is invalid: a=b
name with semicolon: TypeError: argument name is invalid: a;b
name non-ascii: TypeError: argument name is invalid: نام
value as Date: set
back: ["1970-01-01T00:00:00.000Z"]
value function: set
back: [""]
```

- **The size limit** counts the name and the value together, after encoding, and allows 4096 bytes. Under the name `big` the largest Latin text that is kept is 4093 characters. For Persian it is 682 characters, because every letter becomes six characters (`%D8%B3`). A shorter name leaves more room.
- **A refused overwrite keeps the old cookie,** and `get` keeps returning the old value as if the write had worked: after saving 5000 `z` over 3000 `y`, the cookie still holds the `y`. For that reason "the result is not `null`" is not proof that a save worked.
- **`null` and `undefined`** as the value delete the cookie, like `remove`.
- **Two `set` calls in one tick:** the last one wins. A second `useJCookie()` instance sees a write only after the tick, like the first.
- **Cookies written by other code** are read too. Percent-encoded and plain JSON both come back as objects. A cookie with broken percent-escapes makes `get` throw `URIError: URI malformed`.
- **Names with `=`, `;` or non-Latin letters** are refused with the `TypeError` from before.
- **A date** comes back as ISO text and **a function** as an empty text; convert such values yourself.

When a cookie must not be lost, compare what was kept with what was meant to be saved:

```js
const cookie = useJCookie()

// Returns true only if the browser really kept this text value.
async function setChecked(name, value, minutes) {
  try {
    cookie.set(name, value, minutes)
  } catch (error) {
    return false                 // bad name, bad lifetime, bad characters
  }
  await nextTick()               // the write happens a moment later
  return cookie.get(name) === value   // not just "not null": an old value may remain
}
```

## 6. Putting it together

A consent banner is the usual place where cookies meet a rule: nothing that is not strictly needed may be saved before the visitor agrees. The page below shows the banner until the visitor answers, keeps the answer for a year, and saves the theme (for 30 days) only if the answer was yes. *Forget me* removes both cookies and brings the banner back:

```vue
<script setup>
const cookie = useJCookie()

const consent = ref(cookie.get('consent'))
const theme = ref(cookie.get('theme') ?? 'light')

function accept(value) {
  cookie.set('consent', value, 60 * 24 * 365)
  consent.value = value
}

function choose(value) {
  theme.value = value
  if (consent.value === 'yes') cookie.set('theme', value, 60 * 24 * 30)
}

function forget() {
  cookie.remove('consent')
  cookie.remove('theme')
  consent.value = null
  theme.value = 'light'
}
</script>

<template>
  <div v-if="consent === null" id="banner">
    <button id="yes" @click="accept('yes')">Accept</button>
    <button id="no" @click="accept('no')">Decline</button>
  </div>
  <button id="dark" @click="choose('dark')">Dark</button>
  <button id="forget" @click="forget">Forget me</button>
  <p id="theme">{{ theme }}</p>
  <p id="consent">{{ consent }}</p>
</template>
```

```text
first visit: banner shown True, theme light
dark chosen before consent: cookies []
consent given, dark chosen: cookies ['consent', 'theme']
next visit: banner shown False, theme dark, consent yes
after Forget me: cookies [], banner shown True
```

How it works:

- **The banner depends on one cookie.** `consent` is `null` on a first visit, so the banner shows; after the visitor answers, the page keeps the answer in its own variable and the cookie for the next visit (chapter 2).
- **The theme is written only after a yes.** Choosing dark before answering changes the page but writes nothing; after accepting, two cookies exist, `consent` and `theme`.
- **The next visit starts correctly.** The banner stays hidden and the page opens already dark, because both cookies are read at start.
- **Withdrawing is `remove`.** *Forget me* removes both cookies and the banner returns.
- **Lifetimes follow the purpose:** a year for the consent, thirty days for a preference, both written as minutes.

---

## Appendix A. Quick reference

### A.1 Functions

| Call | Returns | Notes |
|------|---------|-------|
| `get(name)` | The value or `null` | Throws on a blank name; the value is converted (numbers, booleans, objects) |
| `set(name, value, minutes)` | Nothing | Default `576000` minutes; the write lands a moment later; `null` or `undefined` deletes |
| `remove(name)` | Nothing | A missing cookie is fine |

### A.2 How values come back

| Saved | Read as |
|-------|---------|
| Ordinary text | The same text |
| `"42"` or the number `42` | The number `42` |
| `"true"` or `true` | The boolean `true` |
| `"null"` | `null` |
| `"007"`, `"1e3"` | The same text |
| Object, array | The same object or array |
| Date, function | ISO text, empty text |
| Empty text | Empty text |
| Never saved, removed, or refused by the browser | `null` |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Blank, empty or missing name | `Error: Cookie name must not be null or blank` |
| Lifetime negative, `NaN`, infinite or not a number | `Error: Cookie lifetime must be a finite non-negative number` |
| Lifetime `0` | No error; no cookie is stored (already expired) |
| Name with invalid characters | `TypeError` from the cookie library; nothing is stored |
| Value too large | No error; the browser drops it; `get` gives `null`, or the old value if there was one |
| Cookie written by others with broken escapes | `get` throws `URIError: URI malformed` |
| Cookie does not exist | `get` gives `null`; `remove` does nothing |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Reading straight after `set`.** The old value is still there; keep your own copy.
- **Giving the lifetime in seconds or days.** It is minutes.
- **Passing the lifetime as text.** Even `"10"` throws.
- **Using `0` to delete.** It stores nothing; call `remove`.
- **Expecting the type you saved.** `"42"` returns as a number and `"true"` as a boolean.
- **Treating `null` as "missing".** The text `"null"` returns `null` too.
- **Saving large values.** Over 4096 bytes (name and value together, after encoding) the browser silently drops the cookie.
- **Trusting `get` after a refused overwrite.** It still returns the old value.
- **Putting secrets in a cookie** that page code can read: there is no `HttpOnly`.
- **Counting on `Secure` or `SameSite`.** The composable sets neither.
- **Names with spaces or special characters.**
- **Saving tracking cookies before consent.**

### C.2 Behaviour worth knowing

- Every cookie is written with the path `/`, so the whole site sees it.
- The write happens a moment after `set`, not on the same line.
- Removing is done by clearing the value; the browser then drops the cookie.
- The default lifetime is 400 days, which is also the longest the browser allows.
- The lifetime is rounded down to whole seconds; under one second stores nothing.
- Values are encoded, so any characters are safe to store (within the size limit).
- The composable has no encryption, signing or notification features of its own.

### C.3 Limits

| What | Limit |
|------|-------|
| Lifetime | Minutes; default `576000`; the browser caps it at 400 days |
| Size of one cookie | 4096 bytes for name and value together, after encoding |
| Name | No spaces or special characters |
| Scope | The whole site (`/`) |
| Security | Readable by page code; no `Secure`, `HttpOnly` or `SameSite` set by the composable |

### C.4 Tuning

- Keep the names in one place.
- Use small helpers such as `hours` and `days` for lifetimes.
- Give each cookie the shortest lifetime that is useful.
- Keep values small; store an identifier, not a record.
- Keep your own copy of a value you have just saved.
- Check that important cookies were kept, by comparing with what you saved (chapter 5).
- Ask for consent before saving anything that is not strictly needed.

## Appendix D. Message reference

### D.1 Errors

| Message | Cause |
|---------|-------|
| `Cookie name must not be null or blank` | The name given to `get`, `set` or `remove` is empty, blank or missing |
| `Cookie lifetime must be a finite non-negative number` | The lifetime given to `set` is negative, `NaN`, infinite or not a number |
| `argument name is invalid: ...` | The name has characters a cookie name cannot have (a `TypeError` from the cookie library) |
| `URI malformed` | `get` met a cookie, written by other code, with broken percent-escapes (a `URIError`) |
