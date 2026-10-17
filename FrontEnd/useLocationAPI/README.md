# useLocationAPI

*J2OS Platform 2.4*

A delivery form should fill in the street by itself. A "stores near me" page should say "You seem to be in Paris". A support request should carry the place it was sent from. All three need the same two steps: ask the device where it is (coordinates), then ask a map service what is at those coordinates (a name). The two steps use different tools with different failure modes.

**useLocationAPI** is a composable of the Nuxt front end of J2OS that does both steps behind five functions. It has three properties:

- **Position from the device.** `setupLocationListener` asks the browser for the current coordinates and hands them to your callback.
- **Names from coordinates.** `getAddress`, `getCountry`, `getCity` and `getPostCode` turn coordinates into text through OpenStreetMap's Nominatim service.
- **Careful with requests.** Several questions about the same point send only one request, and a request that fails is never stored.

In a few lines:

```vue
<script setup>
const { setupLocationListener, getCity } = useLocationAPI()

const city = ref('')

function locate() {
  setupLocationListener(
    async (lat, lon) => { city.value = await getCity(lat, lon) },      // "Paris"
    (error) => { city.value = error.message }
  )
}
</script>

<template>
  <button @click="locate">Where am I?</button>
  <p>{{ city }}</p>
</template>
```

This guide starts with how the composable works (chapter 1). Chapter 2 gets the position and every way that can fail. Chapter 3 turns coordinates into names, and chapter 4 explains the cache. Chapter 5 covers answers that come back without a name. Chapter 6 builds a "find me" card. The appendices work as a reference.

## Contents

1. [How useLocationAPI works](#1-how-uselocationapi-works)
2. [Getting the position](#2-getting-the-position)
3. [From coordinates to names](#3-from-coordinates-to-names)
4. [One request for many questions](#4-one-request-for-many-questions)
5. [When there is no name](#5-when-there-is-no-name)
6. [A "find me" card](#6-a-find-me-card)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are complete pages of a Nuxt project: each is one `.vue` file. In Nuxt `ref`, `onMounted` and `useLocationAPI` need no `import`, so none is written.

---

## 1. How useLocationAPI works

First, a few terms, because they come up throughout the guide:

- **Geolocation.** The browser's way of asking the device for its position (GPS, Wi-Fi, mobile network). The visitor must allow it, and the page must be served from a secure address (HTTPS, or `localhost`).
- **Reverse geocoding.** Turning coordinates into a place name: you give the latitude and longitude and get back an address, a city or a country.
- **Nominatim.** The free reverse-geocoding service of OpenStreetMap, which the composable calls from the visitor's browser.

The functions of this composable:

| Function | Source | Gives |
|----------|--------|-------|
| `setupLocationListener(onSuccess, onError)` | The browser | Calls `onSuccess(latitude, longitude)` |
| `getAddress(latitude, longitude)` | Nominatim | The full address as one text |
| `getCountry(latitude, longitude)` | Nominatim | The country name |
| `getCity(latitude, longitude)` | Nominatim | The city name |
| `getPostCode(latitude, longitude)` | Nominatim | The postal code |

Keep these six points in mind:

- **`setupLocationListener` reads the position once.** Despite its name it does not follow the visitor around; call it again for a new reading.
- **Its settings are fixed:** high accuracy, a 10-second limit, and never an older cached position.
- **The other four functions return promises** and send a request to the map service unless the answer is already in the cache.
- **The coordinates are sent to an outside service.** The visitor sees a permission prompt and the position goes to a map service; it is fair to tell them so.
- **Every call of `useLocationAPI()` creates its own cache.** Answers are stored under the coordinates written as text and are never removed.
- **Failures are not hidden.** A failed request rejects the promise, and an answer that lacks the part you need makes some functions throw (chapter 5).

## 2. Getting the position

A "find me" feature starts with one question: where is the visitor? The browser can answer, but only after the visitor agrees, and only if the device has a way to know. `setupLocationListener` asks and reports the result through two callbacks, one for success and an optional one for failure.

The page below tries it in three situations: the visitor allows it, the visitor refuses, and the browser has no geolocation at all. It also records what the composable asked of the browser:

```vue
<script setup>
const loc = useLocationAPI()
const rows = ref([])
const log = text => rows.value.push(text)

// Page-level instrumentation: what the composable asks of the browser, and what it writes to the console.
const real = navigator.geolocation?.getCurrentPosition?.bind(navigator.geolocation)
if (real) {
  navigator.geolocation.getCurrentPosition = (ok, fail, options) => {
    log(`browser asked with ${JSON.stringify(options)}`)
    return real(ok, fail, options)
  }
}
const originalError = console.error
console.error = (...args) => { log(`console.error: ${args.join(' ')}`); originalError(...args) }

function withHandlers(label) {
  loc.setupLocationListener(
    (lat, lon) => log(`${label}: success ${lat}, ${lon}`),
    err => log(`${label}: onError got ${err.constructor.name}: ${err.message}`)
  )
}

function withoutError(label) {
  try {
    loc.setupLocationListener((lat, lon) => log(`${label}: success ${lat}, ${lon}`))
  } catch (e) {
    log(`${label}: thrown ${e.constructor.name}: ${e.message}`)
  }
}
</script>

<template>
  <button id="with" @click="withHandlers('with onError')">With onError</button>
  <button id="without" @click="withoutError('without onError')">Without onError</button>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
</template>
```

```text
-- position available
browser asked with {"enableHighAccuracy":true,"timeout":10000,"maximumAge":0}
with onError: success 48.8584, 2.2945
browser asked with {"enableHighAccuracy":true,"timeout":10000,"maximumAge":0}
without onError: success 48.8584, 2.2945
-- the visitor refuses
browser asked with {"enableHighAccuracy":true,"timeout":10000,"maximumAge":0}
with onError: onError got Error: Failed to get data from the geolocation sensor: User denied Geolocation
browser asked with {"enableHighAccuracy":true,"timeout":10000,"maximumAge":0}
console.error: Failed to get data from the geolocation sensor: User denied Geolocation
-- a browser without geolocation
with onError: onError got Error: Browser does not support geolocation
without onError: thrown Error: Browser does not support geolocation
```

On success the callback receives two numbers, latitude and longitude, and nothing else: accuracy, altitude and time are dropped. The composable asks the browser for the most accurate position the device can give (`enableHighAccuracy: true`), within ten seconds (`timeout: 10000`), and never accepts a position stored earlier (`maximumAge: 0`). These settings are fixed, so a page cannot trade accuracy for speed.

Failures arrive in two ways, and which one depends on whether you passed an `onError`:

- **With an `onError`,** it receives an `Error` whose message starts with `Failed to get data from the geolocation sensor:` and ends with the browser's own reason (`User denied Geolocation` here).
- **Without an `onError`,** the same message goes to the console with `console.error` and nothing is thrown, so the page never learns that the call failed.
- **When the browser has no geolocation at all,** the message is `Browser does not support geolocation`. With an `onError` it is passed to it; without one the call **throws** that error at once.

So always pass an `onError`, and show the visitor a way to continue. Also mind the timing: the first call is what makes the browser show its permission prompt, so make it in answer to a click, not on page load. A prompt that appears suddenly, for no visible reason, is usually refused. The browser remembers the answer, and a later call does not ask again.

Since the function reports through callbacks, wrapping it in a promise makes it easier to use with `await`:

```js
// The position as a promise, so it can be awaited.
const { setupLocationListener } = useLocationAPI()

const currentPosition = () => new Promise((resolve, reject) => {
  setupLocationListener((lat, lon) => resolve({ lat, lon }), reject)
})

const { lat, lon } = await currentPosition()
```

## 3. From coordinates to names

Two numbers mean little to a visitor. A card that says "Paris, France" is friendlier than one that says `48.8584, 2.2945`. Turning coordinates into names is called reverse geocoding, and the composable does it with four functions that call the free Nominatim service of OpenStreetMap from the visitor's browser.

The page asks for the position and then for four details about it:

```vue
<script setup>
const loc = useLocationAPI()
const rows = ref([])

function find() {
  loc.setupLocationListener(async (lat, lon) => {
    rows.value.push(`position: ${lat}, ${lon}`)
    rows.value.push(`country: ${await loc.getCountry(lat, lon)}`)
    rows.value.push(`city: ${await loc.getCity(lat, lon)}`)
    rows.value.push(`post code: ${await loc.getPostCode(lat, lon)}`)
    rows.value.push(`address: ${await loc.getAddress(lat, lon)}`)
  }, (err) => {
    rows.value.push(`error: ${err.message}`)
  })
}
</script>

<template>
  <button id="find" @click="find">Where am I</button>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
</template>
```

```text
position: 48.8584, 2.2945
country: France
city: Paris
post code: 75007
address: Eiffel Tower, Avenue Anatole France, Paris, France
requests: 1
  nominatim.openstreetmap.org/reverse lat=48.8584 lon=2.2945 format=jsonv2 accept=application/json
```

`getCountry`, `getCity` and `getPostCode` return the matching part of the service's `address`, and `getAddress` returns the full `display_name`, one long text with every level of the address. All four questions were served by one request, because they were about the same point (the next chapter explains how). The request is a plain `GET` to the service's `reverse` endpoint with `format=jsonv2` and the two coordinates.

The service decides the language and the words of the answer; the composable does not ask for a language. All the page controls is the input, and there are two rules for it:

- **Pass numbers or numeric text, never text typed by a visitor.** The latitude and longitude are put into the address as they are, so if you pass the text `1&format=xml&x` as a latitude, a second `format` parameter is added to the request.
- **Wait for the position before asking.** Values that are not known yet go out as the word `undefined` (`lat=undefined`), which wastes a request and gives a useless answer.

## 4. One request for many questions

A card that shows the city, the country and the postal code needs three answers, and it is natural to ask three questions. It would be wasteful to send three requests for one place, and the map service limits how often one visitor may ask. The composable therefore shares requests: questions about the same point share one request, and answers are remembered.

The page below asks several times, in several ways:

```vue
<script setup>
const loc = useLocationAPI()
const other = useLocationAPI()
const rows = ref([])

onMounted(async () => {
  const log = text => rows.value.push(text)
  const answers = await Promise.all([
    loc.getCountry(48.8584, 2.2945), loc.getCity(48.8584, 2.2945), loc.getPostCode(48.8584, 2.2945), loc.getAddress(48.8584, 2.2945)
  ])
  log('same point, four questions: ' + JSON.stringify(answers.map(a => String(a).slice(0, 14))))
  log('asked again: ' + (await loc.getCity(48.8584, 2.2945)))
  log('same point as text: ' + (await loc.getCity('48.8584', '2.2945')))
  log('same point, a trailing zero in the text: ' + (await loc.getCity('48.85840', '2.2945')))
  log('a point 11 metres away: ' + (await loc.getCity(48.8585, 2.2945)))
  log('a second useLocationAPI(): ' + (await other.getCity(48.8584, 2.2945)))
})
</script>

<template>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
  <p>Cache</p>
</template>
```

```text
same point, four questions: ["France","Paris","75007","Eiffel Tower, "]
asked again: Paris
same point as text: Paris
same point, a trailing zero in the text: Paris
a point 11 metres away: Paris
a second useLocationAPI(): Paris
requests sent: 4
  nominatim.openstreetmap.org/reverse lat=48.8584 lon=2.2945 format=jsonv2
  nominatim.openstreetmap.org/reverse lat=48.85840 lon=2.2945 format=jsonv2
  nominatim.openstreetmap.org/reverse lat=48.8585 lon=2.2945 format=jsonv2
  nominatim.openstreetmap.org/reverse lat=48.8584 lon=2.2945 format=jsonv2
```

In the output above, four questions asked at once about one point sent only one request, because the cache keeps a request that is still on its way and all the questions use it. Asking again, or asking with the same numbers written as text, sent no new request.

The cache has one weak point: its **key** is made by joining the two coordinates as text. Two things follow from this:

- **A trailing zero makes a different key.** `'48.85840'` sent a new request, although it is the same place as `48.8584`.
- **Coordinates are not rounded.** A point 11 metres away sent a new request, so a GPS position that wobbles by a few metres sends a new request every time.

The remedy is to round before asking, so that the same place always gives the same key:

```js
// A GPS fix wobbles by a few metres; four decimals is about 11 metres.
// Round before asking, so the same place gives the same cache key.
const round = (value) => Math.round(value * 1e4) / 1e4

const city = await getCity(round(lat), round(lon))
```

Each call of `useLocationAPI()` has its own cache. In the output above the second call knew nothing about the first one's answers and asked again. To share answers across the whole app, create the composable only once:

```js
// utils/location.js : one shared instance, one shared cache
export const location = useLocationAPI()

// anywhere else
import { location } from '~/utils/location'
const address = await location.getAddress(lat, lon)
```

No answer expires and none can be cleared, so a page that keeps asking about many different points keeps all the answers in memory. A failed request is the exception: it is removed from the cache at once, and the next call tries again (chapter 5).

## 5. When there is no name

The map service is a third party. It can be unavailable, it can refuse, and it can answer without the part you asked for. Treat every result as something that may be missing. The page below meets each case:

```vue
<script setup>
const loc = useLocationAPI()
const rows = ref([])

const attempt = async (label, read) => {
  try {
    rows.value.push(`${label}: ${JSON.stringify(await read())}`)
  } catch (e) {
    rows.value.push(`${label}: ${e.constructor.name}: ${e.message}`)
  }
}

onMounted(async () => {
  await attempt('city of a village', () => loc.getCity(45.1, 7.1))
  await attempt('country of a village', () => loc.getCountry(45.1, 7.1))
  await attempt('post code of a village', () => loc.getPostCode(45.1, 7.1))
  await attempt('country in the middle of the ocean', () => loc.getCountry(0, -30))
  await attempt('address in the middle of the ocean', () => loc.getAddress(0, -30))
  await attempt('service refuses (429)', () => loc.getCity(1, 1))
  await attempt('service unreachable', () => loc.getCity(2, 2))
  await attempt('retry after the refusal', () => loc.getCity(1, 1))
})
</script>

<template>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
  <p>Gaps</p>
</template>
```

```text
city of a village: undefined
country of a village: "Italy"
post code of a village: "10070"
country in the middle of the ocean: TypeError: Cannot read properties of undefined (reading 'country')
address in the middle of the ocean: undefined
service refuses (429): Error: Failed to fetch data
service unreachable: TypeError: Failed to fetch
retry after the refusal: "Paris"
```

The failures divide into two families. In the first, the **request fails** and the promise rejects: when the service refuses (a rate limit, for example) with `Failed to fetch data`, and when it cannot be reached with the browser's own network error (`Failed to fetch` in Chromium). A failed request is not remembered, which is why the retry for the same point in the page above succeeded. The composable never retries by itself, so the page decides when to ask again.

In the second family the request succeeds but the **answer has a gap**:

- **A place that is not a city.** `getCity` reads only the service's `city` entry, so for a village it gives `undefined` while `getCountry` and `getPostCode` work.
- **A point with no address** (the middle of the ocean). The service answers normally, with an error note instead of an address. `getAddress` gives `undefined`, but `getCountry`, `getCity` and `getPostCode` throw a `TypeError`, because the answer has no `address` part to read from.

So call these functions inside `try` and `catch`, and allow every result to be `undefined`:

```js
// A reading that never throws and never shows "undefined".
const safe = async (read, fallback = '') => {
  try {
    return (await read()) ?? fallback
  } catch {
    return fallback
  }
}

const country = await safe(() => getCountry(lat, lon), 'unknown country')
```

The service also has usage rules, such as a low limit on how often one visitor may ask. A page that asks on every GPS update, or a site with many visitors, should put its own server between the page and the service.

## 6. A "find me" card

Everything so far makes one small feature. A button starts the search and is disabled while it runs. If the position is found, the card shows the city, the coordinates and the address. If only the address fails, the position is still shown with a message. If the position fails, a message offers another way:

```vue
<script setup>
const loc = useLocationAPI()

const state = ref('idle')
const place = ref(null)
const message = ref('')

function find() {
  state.value = 'locating'
  message.value = ''
  loc.setupLocationListener(
    async (lat, lon) => {
      try {
        const [address, city] = await Promise.all([loc.getAddress(lat, lon), loc.getCity(lat, lon)])
        place.value = { lat, lon, address, city: city ?? 'an unnamed place' }
        state.value = 'ready'
      } catch {
        place.value = { lat, lon, address: null, city: null }
        message.value = 'The address is not available, but your position is known.'
        state.value = 'partial'
      }
    },
    () => {
      message.value = 'Your position is not available. You can type an address instead.'
      state.value = 'failed'
    }
  )
}
</script>

<template>
  <button id="find" :disabled="state === 'locating'" @click="find">Find me</button>
  <p id="state">{{ state }}</p>
  <p v-if="message" id="message">{{ message }}</p>
  <div v-if="place" id="place">
    <strong id="city">{{ place.city ?? "unknown" }}</strong>
    <span id="coords">{{ place.lat.toFixed(4) }}, {{ place.lon.toFixed(4) }}</span>
    <span id="address">{{ place.address ?? "unknown" }}</span>
  </div>
</template>
```

```text
-- position and address: state = ready
Paris
48.8584, 2.2945
Eiffel Tower, Avenue Anatole France, Paris, France
-- address service down: state = partial
The address is not available, but your position is known.
unknown
48.8584, 2.2945
unknown
-- position refused: state = failed
Your position is not available. You can type an address instead.
```

The output shows the three outcomes. When both steps worked, the card has the city, the coordinates and the address, and the state is `ready`. When the address service was down, the state is `partial`: the coordinates are shown and the missing parts say `unknown`, so the visitor still gets something. When the position was refused, the state is `failed` and the message points to typing an address instead.

The design choices are worth copying: ask for the two details together with `Promise.all` so that they share one request, disable the button while the position is being found so that one click means one reading, treat the address as an extra that may fail without spoiling the position, and always keep a manual way for the visitor who refuses.

---

## Appendix A. Quick reference

### A.1 Functions

| Call | Returns | Notes |
|------|---------|-------|
| `setupLocationListener(onSuccess, onError)` | Nothing | One reading; `onSuccess(latitude, longitude)`; `onError` optional |
| `getAddress(lat, lon)` | Promise of text | The full `display_name`; `undefined` if the service has none |
| `getCountry(lat, lon)` | Promise of text | Throws `TypeError` if the service gave no `address` |
| `getCity(lat, lon)` | Promise of text | `undefined` for places that are not cities; throws like `getCountry` |
| `getPostCode(lat, lon)` | Promise of text | Same |

### A.2 What is asked of the browser

| Setting | Value |
|---------|-------|
| `enableHighAccuracy` | `true` |
| `timeout` | 10000 ms |
| `maximumAge` | 0 (always a fresh reading) |

### A.3 What is sent to the service

| Part | Value |
|------|-------|
| Address | `https://nominatim.openstreetmap.org/reverse` |
| Query | `format=jsonv2`, `lat`, `lon` |
| Header | `Accept: application/json` |
| Cache key | `latitude,longitude` as text |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Visitor refuses, or the device cannot find the position | `onError(Error)` with the message `Failed to get data from the geolocation sensor: ` and the browser's reason; with no `onError`, a `console.error` |
| Browser without geolocation | `onError(Error('Browser does not support geolocation'))`; with no `onError` it throws |
| Service answers with an error status | `Error: Failed to fetch data`; not cached |
| Service unreachable | The browser's own error; not cached |
| Answer is not JSON | `SyntaxError`; not cached |
| Answer without `address` | `getAddress` gives `undefined`; the other three throw `TypeError` |
| Answer without the part asked for | `undefined` |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Expecting continuous tracking.** The position is read once per call.
- **Calling it on page load.** The permission prompt belongs to a click.
- **Leaving out `onError`.** The failure only goes to the console, and the visitor sees nothing.
- **Calling it where geolocation may be missing without `onError`.** It throws.
- **Awaiting the position as if it were a promise.** It uses callbacks (chapter 2 has a wrapper).
- **Assuming `getCity` always has a value.** Villages and towns often have none.
- **Calling `getCountry` on any point without `try`.** A point with no address throws.
- **Passing unrounded GPS values.** Every wobble is a new request.
- **Passing text typed by visitors as coordinates.** It is put into the request address.
- **Asking before the position is known.** The request goes out with `lat=undefined`.
- **Creating the composable in many components** and expecting shared answers.
- **Calling the service at high volume.** It has usage rules.
- **Using it on a plain HTTP page.** Browsers do not give out the position there.

### C.2 Behaviour worth knowing

- The permission is asked by the browser, once per site; the composable cannot ask in any other way.
- The callback receives the latitude and longitude only.
- The error message of a refusal is the browser's own text, so it varies between browsers and languages.
- A request that is still on its way is kept in the cache too, so questions asked at the same time use the same request.
- A failed request is removed from the cache; nothing else ever is.
- The coordinates are sent from the visitor's browser to the map service.

### C.3 Limits

| What | Limit |
|------|-------|
| Position reading | One per call, 10 seconds at most |
| Position data | Latitude and longitude only |
| Names | What Nominatim returns; language not chosen |
| Cache | One per `useLocationAPI()`, keyed by the exact text, never cleared |
| Request volume | Set by the map service |
| Browsers | Those with geolocation, on secure pages |

### C.4 Tuning

- Ask for the position in response to a click, and explain why first.
- Always pass an `onError`, and offer a manual way (a typed address).
- Round coordinates to four decimals before asking.
- Create `useLocationAPI()` once and share it.
- Wrap name lookups in a helper that never throws and never shows `undefined`.
- Ask only for the details you show; they share one request anyway.
- Put your own server between the page and the map service if volume matters.
- Tell visitors that their position is used, and that it goes to a map service.

## Appendix D. Message reference

### D.1 Errors

| Message | Cause |
|---------|-------|
| `Browser does not support geolocation` | The browser has no geolocation |
| `Failed to get data from the geolocation sensor: ...` | The visitor refused, or the position could not be found; the end is the browser's reason |
| `Failed to fetch data` | Nominatim answered with an error status |
