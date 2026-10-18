# useNetworkAPI

*J2OS Platform 2.4*

A shop wants to greet visitors with "Hello from Amsterdam", hide the video player for visitors on a slow mobile link, and show a calm "you are offline" message instead of an empty page when the connection drops. Each of these needs a small piece of information about the network, and each is awkward to get on its own: some of it lives in the browser, some has to be asked from an outside service.

**useNetworkAPI** is a composable of the Nuxt front end of J2OS that gathers these pieces behind simple functions. It has three main features:

- **Browser facts, instantly.** Whether the browser thinks it is online, and what it knows about the connection (speed, delay, type), without any request.
- **Location facts, by IP.** The visitor's public address, and from it the country, city, provider, flag and coordinates.
- **Careful with requests.** Answers about the same address share one request, and a failed request is never remembered.

The basic use, briefly:

```vue
<script setup>
const { isInternetConnected, getIP, getCountry, getCity } = useNetworkAPI()

const place = ref('')

onMounted(async () => {
  if (!isInternetConnected()) return
  const ip = await getIP()
  place.value = `${await getCity(ip)}, ${await getCountry(ip)}`   // "Amsterdam, Netherlands"
})
</script>

<template>
  <p>{{ place || 'Unknown location' }}</p>
</template>
```

We begin with how the composable works (chapter 1). Chapter 2 covers the online status and the connection details of the browser, and chapter 3 the address and what is known about it. Chapter 4 explains the cache and retries, and chapter 5 what happens when the outside service gives no answer. Chapter 6 builds a visitor card. The appendices are meant for quick lookups.

## Contents

1. [How useNetworkAPI works](#1-how-usenetworkapi-works)
2. [Online status and the connection](#2-online-status-and-the-connection)
3. [The address and what is known about it](#3-the-address-and-what-is-known-about-it)
4. [One request for many questions](#4-one-request-for-many-questions)
5. [When there is no answer](#5-when-there-is-no-answer)
6. [A real use](#6-a-real-use)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Each example is a page you can drop into a Nuxt project, and each is a single `.vue` file. In Nuxt `ref`, `onMounted` and `useNetworkAPI` need no `import`, so none is written.

---

## 1. How useNetworkAPI works

First, the terms you will keep meeting:

- **Public IP.** The address the visitor's connection shows to the internet. Behind a VPN or a company proxy it is the address of that gateway, not of the visitor's device.
- **Lookup.** Asking an outside service what it knows about an address: where it is, who runs it.
- **Connection information.** What the browser itself estimates about the link: downlink speed, round-trip time, type and effective generation.

The functions, one by one:

| Function | Source | Gives |
|----------|--------|-------|
| `isInternetConnected()` | Browser | `true` or `false` |
| `getPing()`, `getDownloadSpeed()`, `getNetworkType()`, `getNetworkGeneration()` | Browser | A value, or `null` |
| `getIP()` | `api.ipify.org` | The visitor's public address |
| `getContinent`, `getCountry`, `getCity`, `getLatitude`, `getLongitude`, `getEmoji` `(ip)` | `ipwho.is` | Location details |
| `getISP`, `getOrganization`, `getDomain` `(ip)` | `ipwho.is` | Who runs the connection |

Six things to hold on to:

- **The first group is synchronous,** the rest return promises. Only the promise functions make requests.
- **`getIP` is not cached.** Every call is a new request.
- **The lookup functions share one answer per address.** Asking for the country, the city and the provider of the same address sends one request, not three.
- **The cache belongs to one `useNetworkAPI()` call.** A second call starts with an empty cache.
- **Two outside services are involved.** The visitor's address is sent to them, from the visitor's browser, on every uncached call.
- **Failures are not softened.** A failed request rejects the promise; an answer without the expected part makes some functions throw (chapter 5).

## 2. Online status and the connection

Two questions come up in almost every web application: "is the visitor online?" and "how good is the link?" The first decides whether to send a request at all; the second decides whether to load a heavy image or a video. Both are answered by the browser alone, without a request, through the functions that read the browser's own information. The page below reads them in three situations: a desktop browser that does not report the connection, a browser that does, and a browser after the network was switched off:

```vue
<script setup>
const net = useNetworkAPI()
const online = ref(net.isInternetConnected())
const rows = ref([])

const read = () => {
  rows.value = [
    `online: ${net.isInternetConnected()}`,
    `ping: ${JSON.stringify(net.getPing())}`,
    `downloadSpeed: ${JSON.stringify(net.getDownloadSpeed())}`,
    `networkType: ${JSON.stringify(net.getNetworkType())}`,
    `networkGeneration: ${JSON.stringify(net.getNetworkGeneration())}`
  ]
}

onMounted(read)
</script>

<template>
  <button id="read" @click="read">Read</button>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
  <p>State</p>
</template>
```

```text
-- a desktop browser without connection info
online: true
ping: 0
downloadSpeed: 10
networkType: null
networkGeneration: "4g"
-- a browser that reports the connection
online: true
ping: 50
downloadSpeed: 8.4
networkType: "wifi"
networkGeneration: "4g"
-- the network switched off
online: false
ping: 0
downloadSpeed: 0
networkType: null
networkGeneration: "4g"
```

`isInternetConnected()` returns `window.navigator.onLine`, and it turned to `false` as soon as the network was switched off. The four connection functions read `navigator.connection` (or the old vendor-prefixed names) and return `null` when the browser has no such object or the field is missing. What they return depends on the browser:

- **In a desktop Chromium** the object exists but has no `type`, so `getNetworkType()` is `null`, while the other three give estimates (such as `0` for `getPing()` and `10` for `getDownloadSpeed()`). These values are meaningful mainly on mobile devices.
- **When the browser reports the connection,** the values come through unchanged: delay in milliseconds, speed in megabits per second, and a type such as `wifi`.
- **The values are read at the moment of the call.** Nothing listens for changes, so call the function again when the current state is needed (the page above does it on a button).

`getNetworkGeneration()` returns the browser's `effectiveType` (`slow-2g`, `2g`, `3g`, `4g`). It says how the link *behaves*, not what it *is*, which makes it a good basis for a "is this link slow" decision:

```js
const { getNetworkGeneration, getDownloadSpeed } = useNetworkAPI()

// Both can be null (unsupported) and are rough estimates.
const slow = ['slow-2g', '2g'].includes(getNetworkGeneration()) || (getDownloadSpeed() ?? 10) < 1
```

One caution about `isInternetConnected()`: `true` means only that the browser has some network, not that the internet or your server can be reached. Treat it as a quick way to skip a request that cannot work, and still handle failures of the requests themselves.

## 3. The address and what is known about it

The other group of functions talks to outside services. `getIP()` asks the first service for the visitor's public address, and each lookup function asks the second service about an address. A typical use is to show where a visitor comes from, to pick a default language or currency, or to record the provider in a support ticket:

```vue
<script setup>
const net = useNetworkAPI()
const rows = ref([])

onMounted(async () => {
  const add = (label, value) => rows.value.push(`${label}: ${JSON.stringify(value)}`)
  add('online', net.isInternetConnected())
  const ip = await net.getIP()
  add('ip', ip)
  add('continent', await net.getContinent(ip))
  add('country', await net.getCountry(ip))
  add('city', await net.getCity(ip))
  add('isp', await net.getISP(ip))
  add('organization', await net.getOrganization(ip))
  add('domain', await net.getDomain(ip))
  add('emoji', await net.getEmoji(ip))
  add('latitude', await net.getLatitude(ip))
  add('longitude', await net.getLongitude(ip))
  add('ping', net.getPing())
  add('downloadSpeed', net.getDownloadSpeed())
  add('networkType', net.getNetworkType())
  add('networkGeneration', net.getNetworkGeneration())
})
</script>

<template>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
  <p>Network</p>
</template>
```

```text
online: true
ip: "198.51.100.7"
continent: "Europe"
country: "Netherlands"
city: "Amsterdam"
isp: "Example Net"
organization: "Example Hosting"
domain: "example.net"
emoji: "🇳🇱"
latitude: 52.3676
longitude: 4.9041
ping: 50
downloadSpeed: 8.4
networkType: "wifi"
networkGeneration: "4g"
```

`getIP()` returns the address as text. The lookup functions fall into two groups: those about place (`getContinent`, `getCountry` and `getCity` return names, `getLatitude` and `getLongitude` return numbers, `getEmoji` returns the flag as an emoji) and those about the owner of the connection (`getISP`, `getOrganization` and `getDomain` return the provider's name, the organization that owns the address range and its domain).

Any address works for a lookup, not only the visitor's. The functions take an address as text and know nothing about where it came from. That also means the address goes into the request address as it is: text such as `a b/c?x=1` becomes a different path and query on the service. So do not pass text that visitors typed without checking it first:

```js
// The address is put straight into the request address. Allow only
// what looks like an address before asking about it.
const looksLikeIp = (value) => /^[0-9a-fA-F:.]{3,45}$/.test(value)

if (looksLikeIp(ip)) {
  city.value = await net.getCity(ip)
}
```

The accuracy of a lookup is the service's, not the composable's. City-level answers are often approximate, and mobile and VPN addresses can point to a different place from the one the visitor is in. Use the result as a hint, never as proof.

## 4. One request for many questions

A page often wants several details of the same visitor: the country, the city and the provider together. Naively that would be three requests for one fact. The composable avoids it: **questions about the same address share one request.** The page below asks five questions at once, then again later, then about another address, then through a second `useNetworkAPI()`. The last line lists the requests that left the page:

```vue
<script setup>
const net = useNetworkAPI()
const other = useNetworkAPI()
const rows = ref([])

onMounted(async () => {
  const log = text => rows.value.push(text)
  const ip = '198.51.100.7'
  // Five questions about the same address at the same moment.
  const answers = await Promise.all([
    net.getCountry(ip), net.getCity(ip), net.getISP(ip), net.getLatitude(ip), net.getEmoji(ip)
  ])
  log('same address, five questions: ' + JSON.stringify(answers))
  log('asked again later: ' + await net.getCity(ip))
  log('another address: ' + await net.getCity('203.0.113.9'))
  log('a second useNetworkAPI(): ' + await other.getCity(ip))
})
</script>

<template>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
  <p>Cache</p>
</template>
```

```text
same address, five questions: ["Netherlands","Amsterdam","Example Net",52.3676,"🇳🇱"]
asked again later: Amsterdam
another address: Toronto
a second useNetworkAPI(): Amsterdam
requests sent: whois 198.51.100.7, whois 203.0.113.9, whois 198.51.100.7
```

Five questions at once, about the same address, produced a single request. The cache keeps the pending request, so questions asked while it is still on its way share it. Asking again later produced no request, because the answer was already there, and another address produced its own request.

The cache belongs to one `useNetworkAPI()` call. A second call has its own cache and asks again. To share answers across the whole app, create the composable once and share it:

```js
// Cache lives inside one useNetworkAPI() call, so keep one and share it.
// utils/network.js
export const network = useNetworkAPI()

// anywhere else
import { network } from '~/utils/network'
const city = await network.getCity(ip)
```

Two properties of the cache matter in a long-lived page. Nothing expires: an answer stays for as long as the instance lives and there is no way to clear it, so a page that looks up many different addresses keeps all of them. And a failed request is *not* kept. It is removed from the cache as soon as it fails, so the next call tries again:

```vue
<script setup>
const net = useNetworkAPI()
const rows = ref([])

onMounted(async () => {
  const log = text => rows.value.push(text)
  const ip = '192.0.2.44'
  try {
    await net.getCountry(ip)
  } catch (e) {
    log(`first try failed: ${e.constructor.name}: ${e.message}`)
  }
  log('second try: ' + await net.getCountry(ip))
  log('third try: ' + await net.getCountry(ip))
})
</script>

<template>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
  <p>Retry</p>
</template>
```

```text
first try failed: Error: Failed to fetch IP information
second try: Netherlands
third try: Netherlands
requests sent: whois 192.0.2.44, whois 192.0.2.44
```

The first try failed with `Failed to fetch IP information`, because the service answered with an error status. The second try sent a new request and succeeded, and the third was answered from the cache. Nobody retries by itself: the page decides when to ask again.

## 5. When there is no answer

The outside services can be down, can refuse, or can answer in a way the composable did not expect. A page that depends on them should expect every one of these. The page below meets each case. The first line is for a visit while the address service was down:

```vue
<script setup>
const net = useNetworkAPI()
const rows = ref([])

const attempt = async (label, read) => {
  try {
    rows.value.push(`${label}: ${JSON.stringify(await read())}`)
  } catch (e) {
    rows.value.push(`${label}: ${e.constructor.name}: ${e.message}`)
  }
}

onMounted(async () => {
  await attempt('getIP when the service is down', () => net.getIP())
  await attempt('getCountry of an address with no data', () => net.getCountry('10.0.0.5'))
  await attempt('getCity of an address with no data', () => net.getCity('10.0.0.5'))
  await attempt('getISP of an address with no data', () => net.getISP('10.0.0.5'))
  await attempt('getEmoji of an address with no data', () => net.getEmoji('10.0.0.5'))
  await attempt('getCountry when the answer is not JSON', () => net.getCountry('192.0.2.1'))
})
</script>

<template>
  <div v-for="r in rows" :key="r" class="row">{{ r }}</div>
  <p>Gaps</p>
</template>
```

```text
getIP when the service is down: Error: Failed to fetch IP address
getCountry of an address with no data: undefined
getCity of an address with no data: undefined
getISP of an address with no data: TypeError: Cannot read properties of undefined (reading 'isp')
getEmoji of an address with no data: TypeError: Cannot read properties of undefined (reading 'emoji')
getCountry when the answer is not JSON: SyntaxError: Unexpected token '<', "<html>not "... is not valid JSON
```

There are two kinds of failure. The first is an outright failure of the request:

- **`getIP()` with the service down** rejects with `Failed to fetch IP address`. If the browser cannot connect at all (a blocked or dropped request), the promise rejects with the browser's own network error instead: a `TypeError` whose message is `Failed to fetch` in Chromium. A failed lookup of this kind is not cached either.
- **An answer that is not JSON** rejects with a `SyntaxError`.

The second is quieter: a lookup answered normally but without the data, such as a successful HTTP answer whose body only says that the lookup failed. It does not reject, and what you get depends on the function. Functions that read a top-level field, such as `getCountry` and `getCity`, return `undefined`. The functions that read inside a group (`getISP`, `getOrganization`, `getDomain`, `getEmoji`) throw a `TypeError`, because the group is missing.

So every promise function should be called inside `try` and `catch`, and every result should be allowed to be `undefined`. A small helper keeps this short:

```js
// One helper that turns every failure into a fallback value.
const safe = async (read, fallback = null) => {
  try {
    return (await read()) ?? fallback
  } catch {
    return fallback
  }
}

const isp = await safe(() => net.getISP(ip), 'unknown')
```

## 6. A real use

A visitor card. It does nothing when the browser is offline, shows a short message when the lookup fails, and otherwise shows a flag, a place and the address with the provider. The four lookups run side by side and share one request:

```vue
<script setup>
const net = useNetworkAPI()

const card = ref(null)
const message = ref('Looking you up…')

onMounted(async () => {
  if (!net.isInternetConnected()) {
    message.value = 'You appear to be offline.'
    return
  }
  try {
    const ip = await net.getIP()
    const [country, city, emoji, isp] = await Promise.all([
      net.getCountry(ip), net.getCity(ip), net.getEmoji(ip), net.getISP(ip)
    ])
    card.value = { ip, country, city, emoji, isp }
  } catch {
    message.value = 'Location is not available right now.'
  }
})
</script>

<template>
  <p id="state">{{ card ? 'ready' : message }}</p>
  <div v-if="card" id="card">
    <span id="flag">{{ card.emoji }}</span>
    <strong id="where">{{ card.city }}, {{ card.country }}</strong>
    <small id="who">{{ card.ip }} · {{ card.isp }}</small>
  </div>
</template>
```

```text
-- online: state = ready
🇳🇱
Amsterdam, Netherlands
198.51.100.7 · Example Net
requests: ipify, whois 198.51.100.7
-- service down: state = Location is not available right now.
requests: ipify
-- offline: state = You appear to be offline.
requests: none
```

The page uses the whole guide:

- **It checks first.** When offline it makes no request at all (chapter 2).
- **It asks economically.** Online, two requests in all, one for the address and one for the lookup, although four details were read (chapter 4). `Promise.all` is safe here because all four calls share the same lookup; if it fails, it fails once.
- **It plans for failure.** With the service down, the address request failed, the lookup was never sent, and the page showed its message instead of an empty card.
- **It never shows `undefined`.** The card appears only when everything is there, and a message covers everything else (chapter 5).

---

## Appendix A. Quick reference

### A.1 Functions

| Call | Returns | Notes |
|------|---------|-------|
| `isInternetConnected()` | `true` or `false` | `navigator.onLine`; not a reachability test |
| `getPing()` | Milliseconds or `null` | Browser estimate of the round trip |
| `getDownloadSpeed()` | Megabits per second or `null` | Browser estimate |
| `getNetworkType()` | Text or `null` | Often `null` on desktop |
| `getNetworkGeneration()` | `slow-2g`, `2g`, `3g`, `4g` or `null` | Effective type |
| `getIP()` | Promise of text | New request on every call |
| `getContinent(ip)`, `getCountry(ip)`, `getCity(ip)` | Promise of text | `undefined` if the service had none |
| `getLatitude(ip)`, `getLongitude(ip)` | Promise of number | Same |
| `getEmoji(ip)` | Promise of text | Throws if the service gave no flag |
| `getISP(ip)`, `getOrganization(ip)`, `getDomain(ip)` | Promise of text | Throws if the service gave no connection data |

### A.2 Services

| Service | Used by | Receives |
|---------|---------|----------|
| `api.ipify.org` | `getIP` | The visitor's request |
| `ipwho.is` | All lookup functions | The address in the path |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Address service answers with an error status | `Error: Failed to fetch IP address` |
| Lookup service answers with an error status | `Error: Failed to fetch IP information`; not cached |
| No network, blocked request | The browser's own error (`TypeError: Failed to fetch` in Chromium); not cached |
| Lookup answers normally but without data | Top-level functions give `undefined`; grouped functions throw `TypeError` |
| Lookup answer is not JSON | `SyntaxError`; not cached |
| Browser does not report the connection | The four connection functions give `null` |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Treating `onLine` as "the server is reachable".** It only says a network exists.
- **Calling the promise functions without `try` and `catch`.** Services fail.
- **Using `getISP`, `getEmoji` and the like without a `try`.** They can throw on a missing group.
- **Showing `undefined`.** Top-level functions return it for unknown addresses.
- **Creating `useNetworkAPI()` in many components and expecting shared answers.** Each call has its own cache.
- **Calling `getIP()` repeatedly.** It is a request each time; keep the result.
- **Passing untrusted text as the address.**
- **Sending visitors' addresses to outside services without telling them.** Check the privacy rules that apply to your site.
- **Trusting the location.** It is an estimate; VPN and mobile addresses are often wrong.
- **Relying on the connection values on desktop.** They are rough, and may be `null`.
- **Expecting the connection functions to update by themselves.** They are read at call time.

### C.2 Behavior worth knowing

- The answer for an address is cached as a pending request, so simultaneous questions share it.
- A failed lookup is removed from the cache; a later call tries again.
- Nothing is cleared or expires, and there is no function to clear the cache.
- `getIP` has no cache.
- The lookup is sent from the visitor's browser, so ad blockers, firewalls and browser privacy settings can stop it.
- Nothing is stored in the browser between page loads.

### C.3 Limits

| What | Limit |
|------|-------|
| Online status | What the browser believes |
| Connection details | Browser estimates, often missing on desktop |
| Location accuracy | The service's |
| Request volume | Set by the outside services |
| Cache | One per `useNetworkAPI()` call, never cleared |
| Changes | None are announced |

### C.4 Tuning

- Create `useNetworkAPI()` once and share it, so addresses are looked up once.
- Keep the visitor's address after the first `getIP()` and pass it to the lookups.
- Read only the details you need; the cost is one request anyway.
- Always have a fallback text for location and provider.
- Check `isInternetConnected()` first, to avoid pointless requests while offline.
- Call the connection functions again when you need the current state, for example when a button is pressed.
- If the lookups are important, put your own server between the page and the service.

## Appendix D. Message reference

### D.1 Errors

| Message | Cause |
|---------|-------|
| `Failed to fetch IP address` | `api.ipify.org` answered with an error status |
| `Failed to fetch IP information` | `ipwho.is` answered with an error status |
