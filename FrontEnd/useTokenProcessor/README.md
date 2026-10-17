# useTokenProcessor

*J2OS Platform 2.4*

After signing in, the server hands the browser a token. The page wants to greet the user by name, show the editor menu only to editors, and send the user back to the login page when the session has ended. All of that is already inside the token, and the page can read it without asking the server again.

**useTokenProcessor** is a small composable of the Nuxt front end of J2OS that reads a JWT (JSON Web Token) in the browser. It has three properties:

- **Read only.** It decodes a token and returns its subject, its claims and whether it has expired. It does not create or renew tokens, and it does not check their signature.
- **The type you ask for.** A claim is read as a string, a list of strings or a map. If it has any other type, the answer is `null`.
- **Two families.** Every reader has a strict version that refuses an expired token, and an `...IgnoringExpiration` version that does not care.

In a few lines:

```js
const { getSubject, getClaimAsString, isTokenExpired } = useTokenProcessor()

if (!isTokenExpired(token)) {
  const id = getSubject(token)                // 'user-42'
  const name = getClaimAsString(token, 'name')
}
```

This guide goes step by step: chapter 1 explains how the composable works and chapter 2 shows the first read. Chapter 3 covers lists and maps, chapter 4 expiry, chapter 5 bad tokens and chapter 6 what the composable cannot do. Chapter 7 builds a small session bar. The appendices are for quick reference.

## Contents

1. [How useTokenProcessor works](#1-how-usetokenprocessor-works)
2. [The first read](#2-the-first-read)
3. [Lists and maps](#3-lists-and-maps)
4. [Expiry](#4-expiry)
5. [When the token is bad](#5-when-the-token-is-bad)
6. [What it cannot do](#6-what-it-cannot-do)
7. [Putting it together](#7-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are complete pages of a Nuxt project: each is one `.vue` file. In Nuxt `ref`, `computed` and `useTokenProcessor` need no `import`, so none is written.

---

## 1. How useTokenProcessor works

First, a few terms, because they come up throughout the guide:

- **JWT.** A text of three parts separated by dots: header, payload and signature. The server makes it at sign-in.
- **Payload.** The middle part. It is a JSON object with information about the user.
- **Claim.** One entry of the payload, such as `name` or `roles`.
- **Subject (`sub`).** The claim that says who the token is about, usually the user's id.
- **`exp`.** The moment the token stops being valid, as a number of seconds since the start of 1970.
- **Base64Url.** The format in which each part is written as text. It is an encoding, not encryption: anyone can decode it.

A token has this structure, and its payload, once decoded, looks like the second block:

```text
eyJhbGciOiJIUzI1NiJ9 . eyJzdWIiOiJ1c2VyLTQyIn0 . c2lnbmF0dXJl
       header                  payload              signature
```

```json
{
  "sub": "user-42",
  "name": "Sara",
  "roles": ["editor"],
  "exp": 4102444800
}
```

The functions of this composable:

| Function | Role |
|----------|------|
| `getSubject(token)` and `getSubjectIgnoringExpiration(token)` | Reads the `sub` claim |
| `getClaimAsString(token, name)` and `getClaimAsStringIgnoringExpiration(token, name)` | Reads a claim that is a string |
| `getClaimAsList(token, name)` and `getClaimAsListIgnoringExpiration(token, name)` | Reads a claim that is a list of strings |
| `getClaimAsMap(token, name)` and `getClaimAsMapIgnoringExpiration(token, name)` | Reads a claim that is an object |
| `isTokenExpired(token)` | Says whether the token has expired (`true`) or not (`false`) |

That is the whole interface: nine functions. Five points to keep in mind:

- **Decoding is not trusting.** The signature is never checked, because the browser has no secret to check it with. Anyone can write a token that this composable reads without complaint (chapter 6). Use it for display and convenience, never as proof of access; the server must check every request.
- **Every call decodes again.** Nothing is cached and nothing is stored between calls.
- **Missing is `null`.** An absent claim, or one of another type, is not an error, just `null`.
- **A bad token throws.** It throws an ordinary `Error` with a readable message (chapter 5).
- **Time comes from the visitor's clock.** `isTokenExpired` compares `exp` with the clock of the browser, so a wrong clock gives a wrong answer.

In J2OS the token is usually kept in local storage under the key `token`; the API client reads it from there and sends it as a `Bearer` header.

## 2. The first read

The pages in this guide make their own test tokens with a small helper, `fakeToken`: it converts a payload to Base64Url and puts a made-up signature after it. Real tokens come from the server, but the helper keeps the examples short. Later pages repeat it, so that each stays a complete page.

```vue
<script setup>
const b64 = (o) => btoa(String.fromCharCode(...new TextEncoder().encode(JSON.stringify(o)))).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '')
const fakeToken = (payload) => b64({ alg: 'HS256', typ: 'JWT' }) + '.' + b64(payload) + '.signature'

const { getSubject, getClaimAsString } = useTokenProcessor()

const token = fakeToken({ sub: 'user-42', name: 'سارا احمدی', exp: 4102444800 })

const id = getSubject(token)
const name = getClaimAsString(token, 'name')
const nickname = getClaimAsString(token, 'nickname')
</script>

<template>
  <p id="parts">{{ token.split('.').length }} parts</p>
  <p id="id">{{ id }}</p>
  <p id="name">{{ name }}</p>
  <p id="nickname">{{ nickname }}</p>
</template>
```

```text
parts: 3 parts
id: user-42
name: سارا احمدی
nickname: 
```

`getSubject` returns the id, and `getClaimAsString` returns the name, with its Persian letters intact. The missing `nickname` becomes `null` (shown as an empty line), not an error.

The functions do not depend on each other, so they can be pulled out of the object with destructuring, as the page does.

## 3. Lists and maps

A token rarely carries only a name. Roles are usually a list of strings, and a profile is often a small object. These are the shapes a page needs most, so the composable has a reader for each: `getClaimAsList` for a list of strings and `getClaimAsMap` for an object. The page below reads them, and also asks for a claim in the *wrong* shape to show what happens:

```vue
<script setup>
const b64 = (o) => btoa(String.fromCharCode(...new TextEncoder().encode(JSON.stringify(o)))).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '')
const fakeToken = (payload) => b64({ alg: 'HS256', typ: 'JWT' }) + '.' + b64(payload) + '.signature'

const { getClaimAsList, getClaimAsMap, getClaimAsString } = useTokenProcessor()

const token = fakeToken({
  sub: 'user-42',
  roles: ['editor', 'reviewer'],
  tags: [],
  mixed: ['a', 1],
  profile: { city: 'Shiraz', plan: 'pro' },
  level: 5,
  active: true,
  exp: 4102444800
})

const rows = [
  ['list of roles', () => getClaimAsList(token, 'roles')],
  ['list of tags (empty)', () => getClaimAsList(token, 'tags')],
  ['list with a number inside', () => getClaimAsList(token, 'mixed')],
  ['roles read as a map', () => getClaimAsMap(token, 'roles')],
  ['map of profile', () => getClaimAsMap(token, 'profile')],
  ['profile read as a string', () => getClaimAsString(token, 'profile')],
  ['level (a number) as a string', () => getClaimAsString(token, 'level')],
  ['active (true) as a string', () => getClaimAsString(token, 'active')],
  ['missing claim as a list', () => getClaimAsList(token, 'groups')]
].map(([label, read]) => ({ label, value: JSON.stringify(read()) }))
</script>

<template>
  <div v-for="r in rows" :key="r.label" class="row">{{ r.label }}: {{ r.value }}</div>
</template>
```

```text
list of roles: ["editor","reviewer"]
list of tags (empty): []
list with a number inside: null
roles read as a map: null
map of profile: {"city":"Shiraz","plan":"pro"}
profile read as a string: null
level (a number) as a string: null
active (true) as a string: null
missing claim as a list: null
```

The idea behind all the readers is the same: **ask for the type you expect, and you get either that type or `null`.** Nothing is converted. A list of strings comes back as it is, and an empty list is a real answer (`[]`), not `null`. But a list that contains a number gives `null`, because every item must be a string. A list asked for as a map gives `null`, and a map asked for as a string does too. A number or a boolean read as a string is also `null`: `5` is never turned into `"5"`. And a claim that is absent is `null`, like all the others.

This is convenient, because the page needs only one check for all these cases: "did I get something?" There is no need to guess whether the claim is missing, has another type or is damaged. A default value takes just one operator:

```js
const roles = getClaimAsList(token, 'roles') ?? []
const city = getClaimAsMap(token, 'profile')?.city ?? 'unknown'
const name = getClaimAsString(token, 'name') || 'Guest'
```

Use `??` for lists and maps. For a string, `||` also replaces an empty text, which is usually what a page wants. If exactness really matters (an `admin` check, for example), compare with a value you know, such as `roles.includes('admin')`, rather than relying on a truthy text.

## 4. Expiry

A token is not meant to live forever. The server writes the end of its life into the claim `exp`, and the page asks whether that moment has passed. This is what `isTokenExpired` is for, and it also explains the two families of readers: the **strict** ones, which refuse an expired token, and the ones named `...IgnoringExpiration`, which read it anyway. The page below uses a fresh token, an old one and one without `exp`:

```vue
<script setup>
const b64 = (o) => btoa(String.fromCharCode(...new TextEncoder().encode(JSON.stringify(o)))).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '')
const fakeToken = (payload) => b64({ alg: 'HS256', typ: 'JWT' }) + '.' + b64(payload) + '.signature'

const t = useTokenProcessor()

const fresh = fakeToken({ sub: 'user-1', name: 'Fresh', exp: 4102444800 })
const old = fakeToken({ sub: 'user-2', name: 'Old', exp: 946684800 })
const eternal = fakeToken({ sub: 'user-3', name: 'Eternal' })

const attempt = (read) => {
  try { return JSON.stringify(read()) } catch (e) { return 'throws: ' + e.message }
}

const rows = [
  ['isTokenExpired(fresh)', () => t.isTokenExpired(fresh)],
  ['isTokenExpired(old)', () => t.isTokenExpired(old)],
  ['isTokenExpired(eternal)', () => t.isTokenExpired(eternal)],
  ['getSubject(fresh)', () => t.getSubject(fresh)],
  ['getSubject(old)', () => t.getSubject(old)],
  ['getSubjectIgnoringExpiration(old)', () => t.getSubjectIgnoringExpiration(old)],
  ['getClaimAsString(old, name)', () => t.getClaimAsString(old, 'name')],
  ['getClaimAsStringIgnoringExpiration(old, name)', () => t.getClaimAsStringIgnoringExpiration(old, 'name')],
  ['getSubject(eternal)', () => t.getSubject(eternal)]
].map(([label, read]) => ({ label, value: attempt(read) }))
</script>

<template>
  <div v-for="r in rows" :key="r.label" class="row">{{ r.label }} → {{ r.value }}</div>
</template>
```

```text
isTokenExpired(fresh) → false
isTokenExpired(old) → true
isTokenExpired(eternal) → false
getSubject(fresh) → "user-1"
getSubject(old) → throws: Token has expired
getSubjectIgnoringExpiration(old) → "user-2"
getClaimAsString(old, name) → throws: Token has expired
getClaimAsStringIgnoringExpiration(old, name) → "Old"
getSubject(eternal) → "user-3"
```

The fresh token reads normally. The old one is expired: the strict readers (`getSubject`, `getClaimAsString` and their list and map versions) throw "Token has expired", while the `...IgnoringExpiration` readers return the same values as for a fresh one. The token without `exp` counts as never expired; whether such a token is acceptable is for the server to decide.

It helps to know the rules of this check, because they decide edge cases:

- `exp` is in **seconds**, the page's clock in milliseconds; the composable multiplies by 1000.
- A token is expired when now is **equal to or later** than `exp`. No leeway is allowed for clocks that differ a little.
- A missing or `null` `exp` means "not expired". An `exp` that is not a finite number (a text, a boolean) gives an error.
- Other time claims, such as `nbf` ("not before"), are not looked at.

Choosing the family depends on your goal:

- **Use the strict readers for a decision made right now:** for example, show the page only if the token is still valid.
- **Use the `...IgnoringExpiration` readers when expiry does not matter:** for example, showing who was signed in on a session that has just ended, or reading the subject to ask the server for a new token.
- **Check once, then read.** A token can expire between two strict calls, and the second would throw in the middle of rendering the page. When several values are needed, ask `isTokenExpired` once and read with the `...IgnoringExpiration` versions (chapter 7 does this).

## 5. When the token is bad

Tokens come from storage, from headers and from pasted text, and not everything that arrives is a token. A page that reads them should know which message each mistake produces, so that it can react sensibly. The page below feeds the composable a range of invalid input:

```vue
<script setup>
const b64 = (o) => btoa(String.fromCharCode(...new TextEncoder().encode(JSON.stringify(o)))).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '')
const fakeToken = (payload) => b64({ alg: 'HS256', typ: 'JWT' }) + '.' + b64(payload) + '.signature'

const t = useTokenProcessor()

const part = (o) => b64(o)
const noJson = btoa('not json')

const cases = [
  ['null', () => t.getSubjectIgnoringExpiration(null)],
  ['undefined', () => t.getSubjectIgnoringExpiration(undefined)],
  ['empty text', () => t.getSubjectIgnoringExpiration('')],
  ['spaces only', () => t.getSubjectIgnoringExpiration('   ')],
  ['a number', () => t.getSubjectIgnoringExpiration(12345)],
  ['one part', () => t.getSubjectIgnoringExpiration('abc')],
  ['two parts', () => t.getSubjectIgnoringExpiration('abc.def')],
  ['four parts', () => t.getSubjectIgnoringExpiration('a.b.c.d')],
  ['characters outside Base64Url', () => t.getSubjectIgnoringExpiration('!!!.@@@.###')],
  ['payload is not JSON', () => t.getSubjectIgnoringExpiration(part({ alg: 'HS256' }) + '.' + noJson + '.sig')],
  ['exp is a text', () => t.isTokenExpired(fakeToken({ sub: 'u', exp: '4102444800' }))],
  ['exp is null', () => t.isTokenExpired(fakeToken({ sub: 'u', exp: null }))],
  ['blank claim name', () => t.getClaimAsString(fakeToken({ sub: 'u' }), '')],
  ['expired token, strict read', () => t.getSubject(fakeToken({ sub: 'u', exp: 946684800 }))]
].map(([label, run]) => {
  try { return { label, value: 'returned ' + JSON.stringify(run()) } } catch (e) { return { label, value: e.constructor.name + ': ' + e.message } }
})
</script>

<template>
  <div v-for="c in cases" :key="c.label" class="row">{{ c.label }} → {{ c.value }}</div>
</template>
```

```text
null → Error: Token cannot be null or blank
undefined → Error: Token cannot be null or blank
empty text → Error: Token cannot be null or blank
spaces only → Error: Token cannot be null or blank
a number → Error: Token cannot be null or blank
one part → Error: Invalid JWT format (must have 3 parts)
two parts → Error: Invalid JWT format (must have 3 parts)
four parts → Error: Invalid JWT format (must have 3 parts)
characters outside Base64Url → Error: Failed to decode JWT
payload is not JSON → Error: Failed to decode JWT
exp is a text → Error: Invalid JWT expiration (exp)
exp is null → returned false
blank claim name → Error: Claim name cannot be null or blank
expired token, strict read → Error: Token has expired
```

The failures fall into four groups, and each has its own message:

- **Not a text, or blank.** `null`, `undefined`, a number, an empty text and a text of only spaces all give "Token cannot be null or blank".
- **Wrong number of parts.** Any number of parts other than three gives "Invalid JWT format (must have 3 parts)".
- **Unreadable parts.** A part with characters outside `Base64Url`, or a header or payload that is not JSON, gives "Failed to decode JWT". The message is the same for all these causes.
- **Invalid `exp`.** If `exp` is a text, it gives "Invalid JWT expiration (exp)". A `null` `exp` is the same as none, so it gives `false`.

Two more messages come from the reading itself: "Claim name cannot be null or blank" for a blank claim name, and "Token has expired" for a strict read of an expired token.

The composable does not sanitize its input. Spaces around the token, a line break inside it, or the word `Bearer` in front of it all end in "Failed to decode JWT", which does not say *why*. When the text comes from outside, sanitize it first:

```js
// Removes the spaces and the "Bearer " prefix that the composable would not accept.
const clean = (raw) => (raw ?? '').trim().replace(/^Bearer\s+/i, '')

getSubjectIgnoringExpiration(clean(headerValue))
```

When a page only wants "an answer or nothing", a small wrapper function turns every exception into `null`:

```js
// One place that turns every problem into "no answer".
function safely(read) {
  try {
    return read()
  } catch {
    return null
  }
}

const id = safely(() => getSubjectIgnoringExpiration(token))
```

The wrapper suits places where an invalid token is normal, such as a page that works for guests and signed-in users alike. Where an invalid token is a real problem, let the message reach the user or the log instead of hiding it.

## 6. What it cannot do

Two limits matter in practice, and both come from the same fact: the composable only decodes. It reads what is written in the token; it does not know whether that writing can be trusted, and it only reads text claims. The page below shows both:

```vue
<script setup>
const b64 = (o) => btoa(String.fromCharCode(...new TextEncoder().encode(JSON.stringify(o)))).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '')
const fakeToken = (payload) => b64({ alg: 'HS256', typ: 'JWT' }) + '.' + b64(payload) + '.signature'

const { getSubject, getClaimAsList, getClaimAsString, isTokenExpired } = useTokenProcessor()

// A token made by hand, with no secret and a meaningless signature.
const forged = b64({ alg: 'none' }) + '.' + b64({ sub: 'admin', roles: ['admin'], exp: 4102444800 }) + '.'

const readPayload = (token) => {
  const text = token.split('.')[1].replace(/-/g, '+').replace(/_/g, '/')
  return JSON.parse(new TextDecoder().decode(Uint8Array.from(atob(text), c => c.charCodeAt(0))))
}

const real = fakeToken({ sub: 'user-42', level: 5, iat: 1700000000, exp: 4102444800 })
const numbers = readPayload(real)
</script>

<template>
  <div class="row" id="forged-sub">forged subject: {{ getSubject(forged) }}</div>
  <div class="row" id="forged-roles">forged roles: {{ JSON.stringify(getClaimAsList(forged, 'roles')) }}</div>
  <div class="row" id="forged-exp">forged token expired: {{ isTokenExpired(forged) }}</div>
  <div class="row" id="num-level">level through the composable: {{ JSON.stringify(getClaimAsString(real, 'level')) }}</div>
  <div class="row" id="num-read">level read by the page: {{ numbers.level }}, exp: {{ numbers.exp }}</div>
  <div class="row" id="num-year">expires in the year: {{ new Date(numbers.exp * 1000).getUTCFullYear() }}</div>
</template>
```

```text
forged subject: admin
forged roles: ["admin"]
forged token expired: false
level through the composable: null
level read by the page: 5, exp: 4102444800
expires in the year: 2100
```

**It cannot tell a real token from a forged one.** The token above is written by hand, with no secret and no signature, and it is read without any error: an `admin` subject, an `admin` role, not expired. The signature can only be checked with a secret that the browser does not have. So the composable is for display and convenience. Never grant access because of what the token says; the server decides from the verified token.

**It cannot read numbers.** `exp`, `iat` and any other numeric or boolean claim are not available through the readers: `getClaimAsString` gives `null` for `level`. The only number-based answer is `isTokenExpired`. When the page needs a number, for a countdown for instance, it can decode the payload itself, as the page above does. Another way is to ask the server to also send the value as a string claim, which `getClaimAsString` can read.

## 7. Putting it together

We build a session bar. The user signs in with a token (pasted, or made by one of the buttons), the bar shows the name, the id and the roles, and when the token expires the page signs the user out by itself with a message. The page does what the composable leaves to it: it sanitizes the input, rejects what is not a token, re-checks on a timer and stops the timer when it is done:

```vue
<script setup>
const b64 = (o) => btoa(String.fromCharCode(...new TextEncoder().encode(JSON.stringify(o)))).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '')
const fakeToken = (payload) => b64({ alg: 'HS256', typ: 'JWT' }) + '.' + b64(payload) + '.signature'

const { isTokenExpired, getSubjectIgnoringExpiration, getClaimAsStringIgnoringExpiration, getClaimAsListIgnoringExpiration } = useTokenProcessor()

const input = ref('')
const token = ref('')
const message = ref('Signed out')
let timer = null

const user = computed(() => {
  if (!token.value) return null
  return {
    id: getSubjectIgnoringExpiration(token.value),
    name: getClaimAsStringIgnoringExpiration(token.value, 'name') ?? 'Guest',
    roles: getClaimAsListIgnoringExpiration(token.value, 'roles') ?? []
  }
})

function check() {
  if (!token.value) return
  try {
    if (isTokenExpired(token.value)) {
      signOut('Your session has ended. Please sign in again.')
    }
  } catch (e) {
    signOut('Rejected: ' + e.message)
  }
}

function signIn(text) {
  token.value = text.trim()
  if (!token.value) return signOut('Paste a token first')
  message.value = 'Signed in'
  check()
  if (token.value && !timer) timer = setInterval(check, 500)
}

function signOut(why = 'Signed out') {
  token.value = ''
  input.value = ''
  message.value = why
  clearInterval(timer)
  timer = null
}

const soon = () => fakeToken({ sub: 'user-7', name: 'رضا کریمی', roles: ['editor', 'reviewer'], exp: Math.floor(Date.now() / 1000) + 2 })
const tomorrow = () => fakeToken({ sub: 'user-7', name: 'رضا کریمی', roles: ['editor'], exp: Math.floor(Date.now() / 1000) + 86400 })

onUnmounted(() => clearInterval(timer))
</script>

<template>
  <input id="paste" v-model="input">
  <button id="paste-go" @click="signIn(input)">Sign in with pasted token</button>
  <button id="soon" @click="signIn(soon())">Token valid for 2 seconds</button>
  <button id="tomorrow" @click="signIn(tomorrow())">Token valid for a day</button>
  <button id="out" @click="signOut()">Sign out</button>
  <p id="message">{{ message }}</p>
  <div v-if="user" id="user">
    <b id="name">{{ user.name }}</b> ({{ user.id }})
    <span v-for="r in user.roles" :key="r" class="role">{{ r }}</span>
  </div>
</template>
```

```text
start | message='Signed out' | user=None
token for a day | message='Signed in' | user=('رضا کریمی', ['editor'])
after sign out | message='Signed out' | user=None
token for 2 seconds, right away | message='Signed in' | user=('رضا کریمی', ['editor', 'reviewer'])
after 3.5 seconds | message='Your session has ended. Please sign in again.' | user=None
pasted garbage | message='Rejected: Invalid JWT format (must have 3 parts)' | user=None
pasted spaces | message='Paste a token first' | user=None
day token after 1.2 s | message='Signed in' | user=('رضا کریمی', ['editor'])
```

- **A token for a day** signs the user in, and the bar shows the name and the roles.
- **A token for two seconds** signs the user in, and when its time is up the page ends the session with a message. The timer here runs every half second; a real page can check every few seconds.
- **Garbage** is rejected with the composable's own message, and **blank input** with a message from the page.
- **The display uses the `...IgnoringExpiration` readers.** The page has already decided with `isTokenExpired`, so a token that expires a moment later cannot make a reader throw in the middle of rendering.
- **The timer is stopped** on sign-out and when the component goes away.

The same check can guard routes. As a rough sketch, a route middleware in a Nuxt project:

```js
// middleware/auth.global.js (a sketch)
export default defineNuxtRouteMiddleware(() => {
  if (import.meta.server) return          // local storage exists only in the browser

  const { isTokenExpired } = useTokenProcessor()

  try {
    const token = useLocalStorage().findByKey('token')

    if (!token || isTokenExpired(token)) return navigateTo('/login')
  } catch {
    return navigateTo('/login')
  }
})
```

The check only saves the visitor a wasted trip. Protecting data is still the server's job.

---

## Appendix A. Quick reference

### A.1 The functions

| Function | Returns | Expired token |
|----------|---------|---------------|
| `getSubject(token)` | The `sub` claim, or `null` | Throws |
| `getSubjectIgnoringExpiration(token)` | The `sub` claim, or `null` | Reads normally |
| `getClaimAsString(token, name)` | A string, or `null` | Throws |
| `getClaimAsStringIgnoringExpiration(token, name)` | A string, or `null` | Reads normally |
| `getClaimAsList(token, name)` | A list of strings, or `null` | Throws |
| `getClaimAsListIgnoringExpiration(token, name)` | A list of strings, or `null` | Reads normally |
| `getClaimAsMap(token, name)` | An object, or `null` | Throws |
| `getClaimAsMapIgnoringExpiration(token, name)` | An object, or `null` | Reads normally |
| `isTokenExpired(token)` | `true` or `false` | Returns `true` |

### A.2 What each claim reader accepts

| Reader | Returns the claim when | Otherwise |
|--------|------------------------|-----------|
| String | It is a string (an empty text too) | `null` |
| List | It is a list and every item is a string (an empty list too) | `null` |
| Map | It is an object, not a list and not `null` (an empty object too) | `null` |

### A.3 When is a token expired

| `exp` in the token | `isTokenExpired` |
|--------------------|------------------|
| Missing or `null` | `false` |
| A number in the future | `false` |
| A number in the past, zero or negative, or equal to now | `true` |
| Not a finite number | Throws `Invalid JWT expiration (exp)` |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Token is `null`, missing, not a text, empty or only spaces | `Error: Token cannot be null or blank` |
| Token does not have three parts | `Error: Invalid JWT format (must have 3 parts)` |
| A part is not Base64Url, or header or payload is not JSON | `Error: Failed to decode JWT` |
| Spaces around the token, or a `Bearer ` prefix | `Error: Failed to decode JWT`; sanitize the text first |
| Payload is the JSON value `null` | `TypeError` while reading `exp` or `sub` |
| `exp` present but not a finite number | `Error: Invalid JWT expiration (exp)` |
| Strict reader, and the token has expired | `Error: Token has expired` |
| Claim name empty or missing | `Error: Claim name cannot be null or blank` |
| Claim absent or of another type | `null` |
| Signature wrong, empty or invented | Not checked; the token is read normally |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Trusting what a token says.** The signature is not checked. Roles and ids read here are for display, not for deciding access.
- **A server that writes `exp` in milliseconds.** The value looks like a date thousands of years away, so the token never seems to expire.
- **Reading numbers.** `exp`, `iat` and other numeric claims cannot be read with the readers.
- **A subject that is not a string.** `getSubject` returns whatever `sub` holds: a number stays a number.
- **Unsanitized input.** Spaces, a line break or a `Bearer ` prefix break the decoding.
- **A wrong clock.** A visitor whose clock is hours off gets wrong answers from `isTokenExpired`.
- **Strict readers between checks.** A token can expire between `isTokenExpired` and the next strict read; use the `...IgnoringExpiration` versions after the check.
- **Mistaking the expiry message for a bad token.** "Token has expired" means the token is readable; the other messages mean it is not.
- **Treating a token without `exp` as safe.** It means "not expired" here, nothing more.

### C.2 Behaviour worth knowing

- Every call splits and decodes the token again, so a list or a map returned twice gives two separate objects.
- Names with Persian letters, and other Unicode text, are decoded correctly.
- Standard Base64 characters (`+`, `/`) and missing padding are accepted as well as Base64Url.
- The header is only checked to be valid JSON. Its content, such as the algorithm, is never used.
- A claim reader checks its arguments in this order: claim name, then the token, then expiry (strict versions only).
- A payload that is valid JSON but not an object is not rejected: a text or a number gives odd answers (`undefined` for the subject), and the value `null` gives a `TypeError`.
- Nothing is sent over the network, and nothing is stored.

### C.3 Limits

| What | Limit |
|------|-------|
| Claims readable as numbers or booleans | None |
| Signature checking | None |
| Time claims looked at | `exp` only |
| Clock tolerance | None |
| Token size | Whatever the browser can decode |

### C.4 Tuning

- Decide once with `isTokenExpired`, then read the details with the `...IgnoringExpiration` versions.
- Sanitize the text before the call when it comes from a header or from the user.
- Check expiry on a timer of several seconds, not on every re-render; each call decodes the whole token.
- Keep a wrapper named `safely` for the places where a bad token is an everyday event.
- Ask the server to send any value the page needs as a string claim, instead of decoding numbers by hand.

## Appendix D. Message reference

### D.1 Error messages

| Message | Cause |
|---------|-------|
| `Token cannot be null or blank` | The token is not a text, or is empty or only spaces |
| `Invalid JWT format (must have 3 parts)` | The token does not have exactly three dot-separated parts |
| `Failed to decode JWT` | A part could not be decoded, or the header or payload is not JSON |
| `Invalid JWT expiration (exp)` | `exp` is present but is not a finite number |
| `Token has expired` | A strict reader was used on an expired token |
| `Claim name cannot be null or blank` | The claim name is empty or missing |
