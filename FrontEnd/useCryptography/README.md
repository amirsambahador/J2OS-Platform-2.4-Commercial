# useCryptography

*J2OS Platform 2.4*

A private note that should be unreadable on the server, a value kept in the browser that nobody else should see, a token passed from one page to another: sometimes text has to be locked with a password before it leaves the page. The browser can do this with its built-in cryptography (Web Crypto), but using it directly means choosing algorithms and parameters, generating salts and random values, converting between text and bytes, and putting the pieces into one string. Each choice is a chance to get something quietly wrong.

**useCryptography** is a composable of the Nuxt front end of J2OS that makes those choices once, with safe settings, and exposes two functions. It has three main features:

- **Two functions.** `encryptStringByAES` locks a text with a password, `decryptStringByAES` opens it.
- **A single Base64 text as result.** It holds everything needed to open it again, except the password, so it can be stored or sent as it is.
- **Compatible with the server.** The J2OS Java class `Cryptography` reads and writes the same format, so a text locked in the page can be opened on the server and the other way round.

The basic use, briefly:

```js
const cryptography = useCryptography()

const token = await cryptography.encryptStringByAES('Meeting with the bank at 9', 'blue-river-42')
// 'MGbuBRYm...'  a Base64 text that is safe to store or send

const text = await cryptography.decryptStringByAES(token, 'blue-river-42')
// 'Meeting with the bank at 9'
```

The guide begins with how the composable works (chapter 1) and a first locked note (2). Chapter 3 looks at what the locked text contains, chapter 4 at what happens when something is wrong, and chapter 5 at speed. Chapter 6 says what the tool is for and what it is not for, and chapter 7 builds a vault that shares its notes with a Java server. The appendices are meant for quick lookups.

## Contents

1. [How useCryptography works](#1-how-usecryptography-works)
2. [The first locked note](#2-the-first-locked-note)
3. [What the token contains](#3-what-the-token-contains)
4. [When something is wrong](#4-when-something-is-wrong)
5. [Speed](#5-speed)
6. [What this is for, and what it is not](#6-what-this-is-for-and-what-it-is-not)
7. [A vault shared with the server](#7-a-vault-shared-with-the-server)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Nothing here is a fragment; each example is a page of a Nuxt project, and each is a single `.vue` file. In Nuxt `ref` and the composables need no `import`, so none is written. Each page is followed by its output, which is exactly what the visitor sees.

---

## 1. How useCryptography works

A short glossary before we start:

- **Password.** The secret the visitor knows. It is never stored by the composable.
- **Key.** The value that actually locks the text. It is made from the password, and it differs for every locked text.
- **Salt.** Random bytes mixed into the key making, so that the same password gives a different key each time.
- **Token.** The Base64 text that `encryptStringByAES` returns.

This composable provides these functions:

| Function | Role |
|----------|------|
| `encryptStringByAES(plainText, password)` | Locks the text; returns the token (a promise) |
| `decryptStringByAES(token, password)` | Opens a token; returns the text (a promise) |

Behind the two functions the composable uses AES with a 256-bit key in GCM mode, which locks the text and also detects any change to it. The key comes from the password with PBKDF2 (SHA-256, 210000 rounds), a deliberately slow method that makes guessing passwords expensive. Salt and the second random value (the IV) are generated fresh for every call. None of these settings can be changed.

Five points matter most:

- **Both functions are asynchronous.** Use `await`.
- **Everything is text.** The input is a text, the token is a text, and the opened result is a text. Numbers and objects have to be turned into text first (for example with `JSON.stringify`).
- **The same text gives a different token every time,** because salt and IV are random.
- **The password is the only secret.** Whoever has the token and the password has the text; whoever has only the token cannot read the text without guessing the password.
- **The browser must allow Web Crypto.** It is available on `https` pages and on `localhost`; on a plain `http` page of another address the browser does not offer it.

## 2. The first locked note

A page keeps a private note. The visitor writes it, types a password and presses Lock; the note disappears from the page and a token appears in its place. Unlock turns the token back into the note, if the password is right.

```vue
<script setup>
const cryptography = useCryptography()
const note = ref('Meeting with the bank at 9')
const password = ref('blue-river-42')
const token = ref('')
const shown = ref('')
const message = ref('Write a note and press Lock.')
const busy = ref(false)

async function lock() {
  busy.value = true
  message.value = 'Locking...'
  token.value = await cryptography.encryptStringByAES(note.value, password.value)
  note.value = ''
  shown.value = ''
  busy.value = false
  message.value = `Note locked (${token.value.length} characters).`
}

async function unlock() {
  busy.value = true
  message.value = 'Unlocking...'
  try {
    shown.value = await cryptography.decryptStringByAES(token.value, password.value)
    message.value = 'Note unlocked.'
  } catch (e) {
    shown.value = ''
    message.value = e.name === 'InvalidCharacterError'
      ? 'This is not a locked note.'
      : e.message
  }
  busy.value = false
}
</script>

<template>
  <textarea id="note" v-model="note" />
  <input id="password" v-model="password" type="text">
  <button id="lock" :disabled="busy" @click="lock">Lock</button>
  <button id="unlock" :disabled="busy" @click="unlock">Unlock</button>
  <textarea id="token" v-model="token" />
  <p id="shown">{{ shown }}</p>
  <p id="message">{{ message }}</p>
</template>
```

```text
Page opens:                                       Write a note and press Lock.
Visitor presses Lock:                             Note locked (96 characters). The note box is empty; the token box holds a text of 96 characters (70 bytes).
Visitor presses Unlock:                           Note unlocked. The page shows "Meeting with the bank at 9".
Visitor mistypes the password and presses Unlock: AES decryption failed - wrong password or corrupted data Nothing is shown.
Visitor changes the last characters of the token: AES decryption failed - wrong password or corrupted data
Visitor pastes ordinary text into the token box:  This is not a locked note.
Visitor pastes a very short valid-looking text:   Invalid AES payload: too short
Visitor locks the same note again:                the new token is different and has 96 characters, like the first (96).
```

Notice what the page does and does not do. `lock` encrypts the note and then empties the note box: from now on the page holds the token, not the text. `unlock` decrypts the token that is in the token box, so the visitor could just as well have pasted a token from somewhere else. The password is typed again for unlocking, because the composable does not remember it.

The rest of the output shows what happens when things go wrong, which is the subject of chapter 4.

## 3. What the token contains

The token is not mysterious: it is the Base64 form of four pieces placed one after the other.

```text
Base64 of:   salt (16 bytes) + iv (12 bytes) + encrypted text + tag (16 bytes)

a 26-byte note  ->  16 + 12 + 26 + 16 = 70 bytes  ->  96 Base64 characters
```

| Piece | Size | Purpose |
|-------|------|---------|
| Salt | 16 bytes | Makes the key different for every token |
| IV | 12 bytes | Makes every locking different, even with the same key |
| Encrypted text | As long as the text in UTF-8 | The note itself, unreadable |
| Tag | 16 bytes | Proves that nothing was changed |

Things worth knowing, all from real runs:

- **Length follows the text.** A single letter gives 45 bytes (60 characters of Base64), an empty text 44 bytes (also 60 characters), and a hundred letters 144 bytes (192 characters). The 26-byte note of the example gave 70 bytes, which is the 96 characters in the output. The token does not hide how long the note is.
- **Persian and other scripts work.** The text is converted to UTF-8 first; a Persian note and a Persian password were locked and opened again without change. Note that a letter in Persian takes two bytes, so the token is longer than the character count suggests.
- **Two tokens of the same note differ,** and have the same length. The output shows this: locking the note again gave another token.
- **An empty text is allowed,** and so is an empty password. The composable only refuses a missing value (`null` or `undefined`). Checking the password's length is the page's job.

## 4. When something is wrong

Opening a token can fail for four different reasons, and two of them look the same from the outside:

| Situation | What the page receives |
|-----------|------------------------|
| Wrong password | `Error`: `AES decryption failed - wrong password or corrupted data` |
| The token was changed (even one character) | the same message |
| The token is too short to hold salt and IV | `Error`: `Invalid AES payload: too short` |
| The text is not Base64 at all | `InvalidCharacterError` from the browser, with the browser's own message |
| A missing argument (`null` or `undefined`) | `Error`: `plainText must not be null`, `password must not be null` or `aesBase64Text must not be null` |

Two points deserve attention. The first is that a wrong password and a damaged token cannot be told apart. This is intended: with GCM the only thing the composable can say is that the check failed. The page should phrase its message in a way that covers both, as the output does. The second is that the "not Base64" case comes from the browser and does not have the composable's friendly sentence. A page that opens tokens typed or pasted by visitors should handle it:

```js
try {
  shown.value = await cryptography.decryptStringByAES(token.value, password.value)
} catch (e) {
  message.value = e.name === 'InvalidCharacterError'
    ? 'This is not a locked note.'      // the text is not Base64 at all
    : e.message                          // our own short sentences
}
```

The output shows all of them: the mistyped password, the changed token, ordinary text pasted instead of a token, and a very short text that is valid Base64.

## 5. Speed

Making a key from a password is slow on purpose. The calls above take about a tenth of a second each, and every call pays that price, both for locking and for opening. For one note this is invisible. It matters in two cases: a page that locks many values in a loop (each one costs the same), and a button that can be pressed twice while the first call is still running.

For the second case the fix is simple: disable the buttons during the call, as the example page does, and restore them in `finally` so that a failure does not leave them locked:

```js
// every call derives a key from the password, which takes about a tenth of a second
busy.value = true
try {
  token.value = await cryptography.encryptStringByAES(note.value, password.value)
} finally {
  busy.value = false           // disables the buttons meanwhile
}
```

For the first case, lock one bigger text instead of many small ones. For example, put the values in an object, convert it with `JSON.stringify` and lock the result once.

## 6. What this is for, and what it is not

The tool locks a text with a password so that someone who only sees the token cannot read it. That is a narrow promise, and it helps to know where it ends:

| Question | Answer |
|----------|--------|
| Can the server read a note locked in the page? | Not without the password. If the server never receives the password, it cannot |
| Can the page keep the password? | It can, but then the page's memory (and any script on it) can read it. Ask for it when needed |
| Is it a way to store users' passwords? | No. For that the server hashes passwords (BCrypt or Argon2 in the Java class); this tool is for text that must be read back |
| Does it protect against someone who has the page open and the password typed? | No |
| Does it replace `https`? | No. It protects the text; `https` protects the whole conversation |
| Can it lock files? | No, only text. The Java class has file functions of its own |

Some rules of thumb follow. Choose passwords that are long; the slow key making only slows down guessing. Do not write a fixed password into the page's code: anyone can read the code. Never send the password with the token to the same place, because that undoes the point.

## 7. A vault shared with the server

Now the full picture. A page offers a vault: the visitor writes a note, the page locks it with the visitor's password, and only the token goes to the server. Loading fetches the token and opens it. The server is a Java application that uses the J2OS `Cryptography` class, so it can also read and write the notes.

```vue
<script setup>
const cryptography = useCryptography()
const note = ref('')
const password = ref('blue-river-42')
const message = ref('The vault keeps your note on the server, locked with your password.')

async function save() {
  const token = await cryptography.encryptStringByAES(note.value, password.value)
  await fetch('/api/vault', { method: 'POST', body: token })
  note.value = ''
  message.value = 'Saved to the vault.'
}

async function load() {
  const token = await (await fetch('/api/vault')).text()
  try {
    note.value = await cryptography.decryptStringByAES(token, password.value)
    message.value = 'Loaded from the vault.'
  } catch (e) {
    message.value = `Could not open the vault: ${e.message}`
  }
}
</script>

<template>
  <textarea id="note" v-model="note" />
  <input id="password" v-model="password" type="text">
  <button id="save" @click="save">Save</button>
  <button id="load" @click="load">Load</button>
  <p id="message">{{ message }}</p>
</template>
```

```text
Page opens:                                                                                         The vault keeps your note on the server, locked with your password.
Visitor writes a note and presses Save:                                                             Saved to the vault. The note box is empty.
Visitor presses Load:                                                                               Loaded from the vault. The note box reads "Server room code is 4711".
The server replaces the stored note with one it locked itself (Java), and the visitor presses Load: Loaded from the vault. The note box reads "Backup runs at midnight".
Visitor changes the password and presses Load:                                                      Could not open the vault: AES decryption failed - wrong password or corrupted data
```

The page never sends the note or the password. Save sends the token and clears the note box; Load fetches the token and opens it with the password in the box. When the password is changed, Load fails with the sentence from chapter 4, and the page shows it.

On the server, the same two functions exist with the same names and the same format:

```java
import org.j2os.platform.jsecurity.cryptography.Cryptography;

String note = Cryptography.decryptStringByAES(token, password);       // what the page sent
String back = Cryptography.encryptStringByAES("Backup at midnight", password);   // what the page can open
```

The format is the same in both directions: a token produced by the page opens in the Java code and gives the note back, a token made by the Java code opens in the page (the third line of the output), and Persian text passes both ways. A wrong password on the Java side also fails, with its own message (`AES decryption failed - wrong key/password or corrupted data`, a little different from the page's wording).

For pages of your own, repeat these steps: lock in the page when the server should not see the text, let the visitor type the password, send only the token, handle the four failures of chapter 4, disable the buttons during the call, and use the Java class on the server when it must read or write the same tokens.

---

## Appendix A. Quick reference

### A.1 Functions

| Call | Returns | Notes |
|------|---------|-------|
| `encryptStringByAES(plainText, password)` | Promise of text | Base64 token; different every time |
| `decryptStringByAES(token, password)` | Promise of text | The original text; fails on wrong password or damage |

### A.2 Fixed settings

| Setting | Value |
|---------|-------|
| Cipher | AES-256 in GCM mode |
| Key from password | PBKDF2 with SHA-256, 210000 rounds |
| Salt | 16 random bytes |
| IV | 12 random bytes |
| Tag | 128 bits |
| Token | Base64 of salt, IV, encrypted text and tag |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Missing argument | `Error` naming the argument |
| Wrong password | `Error`: AES decryption failed |
| Token changed | The same error |
| Token too short | `Error`: Invalid AES payload |
| Not Base64 | Browser's `InvalidCharacterError` |
| Web Crypto unavailable (page not on `https` or `localhost`) | A `TypeError` (`Cannot read properties of undefined (reading 'importKey')`), because `crypto.subtle` does not exist |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Forgetting `await`.** The functions return promises.
- **Locking something that is not text.** Convert objects with `JSON.stringify` first.
- **Writing the password into the page's code.** Anyone can read it.
- **Sending the password along with the token.** Nothing is protected then.
- **Treating "wrong password" and "damaged token" as different.** They cannot be told apart.
- **Letting the "not Base64" error reach the visitor unchanged.** Catch it.
- **Calling the functions in a loop.** Every call costs a key derivation.
- **Accepting an empty password.** The composable does; check the length on the page.
- **Expecting the token to hide the length of the text.** It does not.
- **Using it on an `http` page.** Web Crypto is not offered there.

### C.2 Behavior worth knowing

- Salt and IV are fresh for every call, so tokens differ even for the same text and password.
- The tag makes any change to the token detectable.
- Every call derives its own key, so every call costs the same.
- The composable keeps no state; there is nothing to clean up.
- The format matches the J2OS Java class `Cryptography` for text.

### C.3 Limits

| What | Limit |
|------|-------|
| Input | Text only |
| Settings | Fixed |
| Files | Not supported |
| Password rules | None; the page decides |
| Time per call | About a tenth of a second here |
| Hiding length | Not provided |

### C.4 Tuning

- Disable buttons while a call runs and restore them in `finally`.
- Lock one combined text instead of many small ones.
- Ask the visitor for a long password and require a minimum length on the page.
- Show one message that covers both a wrong password and a damaged token.
- Catch `InvalidCharacterError` when the token comes from outside.
- Keep tokens and passwords in different places.

## Appendix D. Message reference

### D.1 Errors

| Message | Cause |
|---------|-------|
| `plainText must not be null` | The text given to `encryptStringByAES` is missing |
| `password must not be null` | The password is missing |
| `aesBase64Text must not be null` | The token given to `decryptStringByAES` is missing |
| `Invalid AES payload: too short` | The token has fewer than 28 bytes |
| `AES decryption failed - wrong password or corrupted data` | Wrong password or a changed token |
| `InvalidCharacterError` (browser) | The token is not valid Base64 |
| `TypeError: Cannot read properties of undefined (reading 'importKey')` | The page is not on `https` or `localhost`, so the browser offers no Web Crypto |

### D.2 Values worth recognizing

| Value | Meaning |
|-------|---------|
| 60 characters | The shortest possible token (an empty or one-byte text) |
| 44 bytes | Salt, IV and tag with nothing between them |
