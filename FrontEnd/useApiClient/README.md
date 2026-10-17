# useApiClient

*J2OS Platform 2.4*

Almost every page of an application talks to the server: it loads a list, sends a form, downloads a report. The browser's `fetch` can do all of it, but each page then repeats the same lines: add the sign-in token, read the answer in the right way, check whether the server said no, dig the server's message out of the reply. When every page does this a little differently, the messages the visitor sees are different too.

**useApiClient** is a composable of the Nuxt front end of J2OS that does these steps once. Six functions cover the common cases, and forms travel as `FormData`, the same shape the J2OS back end reads with `request.getParameter`. It has three properties:

- **Six small calls.** Read with `getText`, `getJSON`, `getFile`; send a form with `postText`, `postJSON`, `postFile`. The name says how the answer comes back.
- **Sign-in and time zone for free.** The token saved in the browser and the visitor's time zone go with every request.
- **One kind of error.** Whatever went wrong, the page catches an `Error` whose message is ready to show and which carries the status code.

In a few lines:

```vue
<script setup>
const api = useApiClient()
const products = ref([])

onMounted(async () => {
  products.value = await api.getJSON('/api/products')        // GET, answer parsed as JSON
})

async function rename(id, name) {
  const formData = new FormData()
  formData.append('id', id)
  formData.append('name', name)
  await api.postJSON('/api/rename', formData)                // POST a form, answer parsed as JSON
}
</script>
```

This guide starts with how the composable works (chapter 1), then reads data (2) and sends a form (3). Chapter 4 builds forms with several rows, using `useDynamicForm` as one example of where a `FormData` comes from. Chapter 5 is about files, chapter 6 about errors, and chapter 7 about the headers that go with every request. Chapter 8 is a sign-in page that ties it together. The appendices work as a reference.

## Contents

1. [How useApiClient works](#1-how-useapiclient-works)
2. [Reading data](#2-reading-data)
3. [Sending a form](#3-sending-a-form)
4. [Forms with several rows](#4-forms-with-several-rows)
5. [Files](#5-files)
6. [When something goes wrong](#6-when-something-goes-wrong)
7. [The headers that go with every call](#7-the-headers-that-go-with-every-call)
8. [A page that signs in](#8-a-page-that-signs-in)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are complete pages of a Nuxt project: each is one `.vue` file. In Nuxt `ref`, `onMounted` and the composables need no `import`, so none is written. The output under a page is what the visitor sees on it as the page is used. The server in these examples is a small stand-in that answers the addresses written in the pages.

---

## 1. How useApiClient works

First, a few terms, because they come up throughout the guide:

- **Request.** A call to the server: an address and a method. The composable offers two methods, GET to read and POST to send.
- **FormData.** The browser's object for a form: names with values, text and files together. It is sent as `multipart/form-data`.
- **Answer type.** How the body of the reply is read: as text, as JSON, or as a file (a `Blob`).
- **Error.** An `Error` the composable throws when the call fails. It is the same object for every kind of failure (chapter 6).

The functions of this composable:

| Call | Method | Sends | Answer comes back as |
|------|--------|-------|----------------------|
| `getText(url)` | GET | nothing | text |
| `getJSON(url)` | GET | nothing | an object or array |
| `getFile(url)` | GET | nothing | a `Blob` |
| `postText(url, formData)` | POST | a `FormData` | text |
| `postJSON(url, formData)` | POST | a `FormData` | an object or array |
| `postFile(url, formData)` | POST | a `FormData` | a `Blob` |

```js
await api.getText('/api/shop-status')          // GET  -> string
await api.getJSON('/api/products')             // GET  -> object or array
await api.getFile('/api/price-list')           // GET  -> Blob

await api.postText('/api/login', formData)     // POST -> string
await api.postJSON('/api/orders', formData)    // POST -> object or array
await api.postFile('/api/invoice', formData)   // POST -> Blob
```

Keep these five points in mind:

- **The "JSON" in `postJSON` is the answer.** What goes up is always a `FormData`; the composable does not send a JSON body.
- **The address is used as given.** Write `/api/products` or a full address; nothing is added in front, and a query string such as `?page=2` is just part of it.
- **A reply with no content gives `null`.** When the server answers with status 204, every call returns `null`, whatever the answer type.
- **Failures are thrown, and also logged.** The page catches them; the browser console gets a line starting with `[ApiClient]` as well.
- **There is no timeout and no way to cancel a call.** A server that never answers keeps the call waiting.

## 2. Reading data

A shop page shows its catalogue and a line saying whether the shop is open. The list is JSON and the line is plain text, which makes it a good first page for `getJSON` and `getText`.

```vue
<script setup>
const api = useApiClient()
const products = ref([])
const shop = ref('')
const message = ref('')

async function load() {
  message.value = ''
  try {
    products.value = await api.getJSON('/api/products')
    shop.value = await api.getText('/api/shop-status')
  } catch (e) {
    products.value = []
    message.value = `Could not load the catalogue: ${e.message}`
  }
}
onMounted(load)
</script>

<template>
  <h3>Catalogue</h3>
  <p id="shop">{{ shop }}</p>
  <ul>
    <li v-for="p in products" :key="p.id" class="product">{{ p.name }} - {{ p.price }}</li>
  </ul>
  <p id="message">{{ message }}</p>
  <button id="reload" @click="load">Reload</button>
</template>
```

```text
Page opens:                              The shop is open until 18:00. Products: Tea - 4 / Honey - 9 / Saffron - 25.
Server fails, visitor presses Reload:    Could not load the catalogue: Catalogue is being rebuilt (products shown: 0).
Server recovers, visitor presses Reload: no error message, 3 products.
```

The page calls `getJSON` and `getText` in a function, `load`, and runs it when the page opens and again when the visitor presses Reload. Everything that can go wrong is caught in one place, and the message is shown. In the output the server fails once: the visitor reads the server's own words, `Catalogue is being rebuilt`, with a short sentence in front. When the server recovers, the same button clears the message and fills the list.

Use `getText` when the answer is a plain string (a version, a status line, a token) and `getJSON` for everything structured. Asking `getText` for a JSON address is not an error: it gives the raw text, which sometimes helps while debugging.

## 3. Sending a form

A contact form asks for a name, a message and, optionally, a file. Sending it is two lines: build a `FormData` from the form, post it.

```vue
<script setup>
const api = useApiClient()
const formEl = ref()
const message = ref('Fill in the form and press Send.')

async function send() {
  const formData = new FormData(formEl.value)     // every named field of the form
  try {
    const ticket = await api.postJSON('/api/tickets', formData)
    message.value = `Thank you! Your ticket number is ${ticket.number}.`
    formEl.value.reset()
  } catch (e) {
    message.value = e.message
  }
}
</script>

<template>
  <form ref="formEl" @submit.prevent="send">
    <input name="name" placeholder="Your name">
    <textarea name="message" placeholder="How can we help?" />
    <input name="attachment" type="file">
    <button id="send">Send</button>
  </form>
  <p id="message">{{ message }}</p>
</template>
```

```text
Page opens:                                      Fill in the form and press Send.
Visitor sends with an empty message:             Please write a message.
Visitor writes a message, adds a file and sends: Thank you! Your ticket number is T-1001.
Form afterwards:                                 name "", message "".
```

`new FormData(formElement)` collects every field that has a `name`, including the file input. The page does not set any header: the browser sees a `FormData` and writes the right `Content-Type` with its boundary. This is the reason the composable never sets one itself, and why you should not either.

The server's answer is JSON here (`{"number": "T-1001"}`), so `postJSON` is the right call. In the output the first try has no message and the server refuses it with its own sentence, which the page shows as it is. The second try works, and the form is cleared by the page after a success.

The server reads the fields by name. In the J2OS Java endpoints that looks like this:

```java
String customer = request.getParameter("customer");
String lines    = request.getParameter("lines");   // the JSON text of the rows
```

## 4. Forms with several rows

A form does not always come from a `<form>` element. An order has a customer and a list of lines, and the lines come from `useDynamicForm`, the composable that keeps a list of rows (it has its own guide). The pattern is the same for any list: a `FormData` is flat, so a list becomes **one field holding JSON text**.

```vue
<script setup>
const api = useApiClient()
const lines = useDynamicForm(() => ({ item: '', qty: 1 }))
const customer = ref('')
const message = ref('')
onMounted(lines.showForm)

async function save() {
  const formData = new FormData()
  formData.append('customer', customer.value)
  formData.append('lines', JSON.stringify(lines.getAllValues()))   // many rows in one field
  try {
    const order = await api.postJSON('/api/orders', formData)
    message.value = `Order ${order.number} saved with ${order.count} line(s).`
  } catch (e) {
    message.value = e.message
  }
}
</script>

<template>
  <input v-model="customer" id="customer" placeholder="Customer">
  <div v-for="row in lines.rows.value" :key="row.id" class="line">
    <input v-model="row.item" class="item" placeholder="Item">
    <input v-model.number="row.qty" class="qty" type="number" min="1">
    <button class="add" @click="lines.addRow">+</button>
    <button class="remove" @click="lines.removeRow(row.id)">-</button>
  </div>
  <button id="save" @click="save">Save order</button>
  <p id="message">{{ message }}</p>
</template>
```

```text
Page opens:                                           no message; 1 line.
Visitor presses Save order with nothing filled in:    Customer is required.
Visitor types a customer only and presses Save order: Add at least one item.
Visitor fills two lines and presses Save order:       Order O-77 saved with 2 line(s).
```

`getAllValues()` returns the rows without their internal `id`, ready to send. The page appends it once as `lines`, next to the plain `customer` field, and the server turns the text back into a list.

There are three common ways to fill a `FormData`:

```js
// 1. from a <form> element: every field with a name, files included
const formData = new FormData(formEl.value)

// 2. by hand: one call per field
const formData = new FormData()
formData.append('customer', 'Nima')
formData.append('photo', fileInput.files[0])     // a File works too

// 3. a list or an object: turn it into one text field
formData.append('lines', JSON.stringify(rows))
```

Notice in the output how the server's two refusals read: the first uses the field `message`, the second `error`. The page does not care; chapter 6 shows why.

## 5. Files

A file travels in the same `FormData` as the text fields, so uploading needs nothing new. Downloading uses `getFile` and `postFile`, which hand back a `Blob`. This chapter does both, starting with the upload.

**Uploading one file.** A profile page lets the visitor choose a picture. The file input gives a `File` in `files[0]`; the page appends it to a `FormData` and posts it. After the server has saved it, the page asks for the saved address with `getFile` and shows the picture the server really kept.

```vue
<script setup>
const api = useApiClient()
const fileEl = ref()
const message = ref('Choose a picture and press Upload.')
const preview = ref('')

async function upload() {
  const file = fileEl.value.files[0]
  if (!file) {
    message.value = 'Please choose a picture first.'
    return
  }
  if (file.size > 1024 * 1024) {
    message.value = 'The picture must be smaller than 1 MB.'
    return
  }
  const formData = new FormData()
  formData.append('avatar', file)                              // the File goes in as it is
  try {
    const saved = await api.postJSON('/api/avatar', formData)
    message.value = `Saved ${saved.name} (${saved.size} bytes).`
    preview.value = URL.createObjectURL(await api.getFile(saved.url))   // show what the server kept
  } catch (e) {
    message.value = e.message
  }
}
</script>

<template>
  <input ref="fileEl" id="file" type="file" accept="image/*">
  <button id="upload" @click="upload">Upload</button>
  <p id="message">{{ message }}</p>
  <img v-if="preview" id="preview" :src="preview" width="48">
</template>
```

```text
Page opens:                                  Choose a picture and press Upload.
Visitor presses Upload with no file:         Please choose a picture first.
Visitor chooses a 1.1 MB picture:            The picture must be smaller than 1 MB.
Visitor chooses notes.txt:                   Only pictures are accepted.
Visitor chooses me.png:                      Saved me.png (70 bytes).
Page then shows the picture from the server: yes
```

Read the output as the visitor's story. The checks before the call are the page's job, and the server has its own as well:

- **No file chosen.** `files[0]` is `undefined`, so the page says so instead of sending an empty form.
- **Too big.** `file.size` is known before anything is sent; refusing here saves a pointless upload.
- **Wrong kind.** The page asks for pictures with `accept="image/*"`, but a visitor can still choose anything, so the server checks too; its sentence is shown as it is.

One detail to remember with `new FormData(formElement)` (chapter 3): a file input with nothing chosen still sends a part, with an empty file name and the type `application/octet-stream`. If the file is optional, the server should ignore a part with an empty name, or the page should check `files.length` first and delete the field. File names travel as UTF-8, so a Persian name such as `عکس من.png` arrives intact, as long as the server decodes it as UTF-8.

**Uploading several files.** The input needs the `multiple` attribute. Append every file under the same name; the server receives one part per file.

```vue
<script setup>
const api = useApiClient()
const fileEl = ref()
const message = ref('')

async function upload() {
  const files = [...fileEl.value.files]
  if (!files.length) {
    message.value = 'Please choose at least one photo.'
    return
  }
  const formData = new FormData()
  formData.append('album', 'Holiday')
  for (const file of files) formData.append('photos', file)   // same name, one part per file
  try {
    const result = await api.postJSON('/api/photos', formData)
    message.value = `Uploaded ${result.count} photo(s) to ${result.album}: ${result.names.join(', ')}.`
  } catch (e) {
    message.value = e.message
  }
}
</script>

<template>
  <input ref="fileEl" id="files" type="file" accept="image/*" multiple>
  <button id="upload" @click="upload">Upload</button>
  <p id="message">{{ message }}</p>
</template>
```

```text
Visitor presses Upload with no file: Please choose at least one photo.
Visitor chooses three photos:        Uploaded 3 photo(s) to Holiday: a.jpg, b.jpg, c.png.
```

The text field `album` goes in the same `FormData`, next to the three files. The form fields are read by name on the server, as in chapter 3; the files arrive as separate parts of the same request.

**Downloading.** A page that offers a price list (a GET) and an invoice for an order (a POST with the order number) uses `getFile` and `postFile`.

```vue
<script setup>
const api = useApiClient()
const message = ref('')
const link = ref('')
const name = ref('')

function offer(blob, fileName) {
  link.value = URL.createObjectURL(blob)
  name.value = fileName
  message.value = `${fileName} is ready (${blob.size} bytes, ${blob.type}).`
}

async function priceList() {
  try { offer(await api.getFile('/api/price-list'), 'prices.csv') }
  catch (e) { message.value = e.message }
}

async function invoice(orderNumber) {
  const formData = new FormData()
  formData.append('order', orderNumber)
  try { offer(await api.postFile('/api/invoice', formData), `invoice-${orderNumber}.pdf`) }
  catch (e) { message.value = e.message }
}
</script>

<template>
  <button id="prices" @click="priceList">Price list</button>
  <button id="invoice" @click="invoice('O-77')">Invoice O-77</button>
  <button id="missing" @click="invoice('O-00')">Invoice O-00</button>
  <p id="message">{{ message }}</p>
  <a v-if="link" id="download" :href="link" :download="name">Save {{ name }}</a>
</template>
```

```text
Page opens:                    no message
Visitor presses Price list:    prices.csv is ready (25 bytes, text/csv).
Visitor presses Invoice O-77:  invoice-O-77.pdf is ready (2048 bytes, application/pdf).
Visitor presses the Save link: the browser saves invoice-O-77.pdf (2048 bytes).
Visitor presses Invoice O-00:  Order O-00 does not exist.
```

A `Blob` knows its `size` and the `type` the server sent, and that is all the page needs to show a line such as `invoice-O-77.pdf is ready (2048 bytes, application/pdf)`. To let the visitor keep the file, the page makes a temporary address for it with `URL.createObjectURL` and puts it in a link with a `download` attribute; the name is the page's choice, not the server's. The same address, put in an `img` tag, is how the profile page showed the picture.

If the server refuses, the visitor reads its message even though the page asked for a file: the last line of the output shows `Order O-00 does not exist.` The composable looks at the status before it reads the body in the requested way, so an error never arrives as a `Blob`.

## 6. When something goes wrong

Every failure arrives as an `Error` thrown by the call, so one `try`/`catch` per action is enough. What differs is which fields are set:

| What happened | `e.message` | `e.status` |
|---------------|-------------|------------|
| The server answered with an error and a JSON body with `error` | The text of `error` | The status code |
| The same with `message` and no `error` | The text of `message` | The status code |
| The same with a JSON body that has neither | The whole JSON text | The status code |
| The same with a plain text or HTML body | The body, as it is | The status code |
| The same with an empty body | `Request failed with status 502` (the number of the status) | The status code |
| The reply is fine but the body is not JSON (`getJSON`, `postJSON`) | The browser's `SyntaxError` message | Not set |
| The server cannot be reached | `Failed to fetch` | Not set |

For failures with a status, two more fields help: `e.data` holds the body already parsed when it was JSON (or `null`), and `e.rawBody` holds the body as text. A server that sends `{"error": "Stock is empty", "code": 7}` gives the page both the sentence to show and the `code` to act on.

```js
try {
  await api.postJSON('/api/orders', formData)
} catch (e) {
  if (e.status === 401) navigateTo('/login')      // the server answered: e.status is set
  else if (e.status)    message.value = e.message // its own words, ready to show
  else                  message.value = 'The server cannot be reached.'   // no answer at all
}
```

Three details are worth knowing:

- **`error` wins over `message`.** If `error` is an empty string, `message` is used.
- **A reply with status 200 is a success, even if its body says `error`.** The composable looks at the status, not at the body.
- **An error page in HTML is shown as HTML.** A gateway that answers `404` with a web page puts that page's markup in `e.message`. Look at `e.status` first and show your own sentence for statuses you expect.

Every failure is also written to the console as `[ApiClient] ✗ GET /api/products` followed by the error. This is useful in development and costs the page nothing.

When many buttons call the server, a small helper keeps the "busy" flag and the message in one place:

```js
const busy = ref(false)
const message = ref('')

async function run(job) {
  busy.value = true
  message.value = ''
  try {
    return await job()
  } catch (e) {
    message.value = e.message
  } finally {
    busy.value = false
  }
}

// run(() => api.postJSON('/api/orders', formData))
```

## 7. The headers that go with every call

The page writes none of them, but every call carries three things:

| Sent | Value | Notes |
|------|-------|-------|
| `Authorization` | `Bearer <token>` | Only when the browser's local storage has a non-empty item named `token` |
| `X-Timezone` | The browser's time zone, such as `Asia/Tehran` | Always |
| Cookies | Those of the site | The calls are made with `credentials: 'include'` |

A POST with a `FormData` also gets its `multipart/form-data` content type from the browser; a GET has no content type. A POST with no `FormData` is sent with an empty body.

The token is read **on every call**, not once, so saving it after sign-in and removing it at sign-out is all a page has to do:

```js
// what a sign-in page does once the server has answered
const storage = useLocalStorage()
storage.persist('token', token)        // from now on every call carries it
storage.removeByKey('token')           // and from now on none does
```

If local storage is not available (blocked by the browser, for example), the token cannot be read and the call goes out without `Authorization`; no error is raised. `X-Timezone` lets the server format dates for the visitor without asking.

## 8. A page that signs in

Everything so far goes into the page that every application has: sign in, and then ask the server who is signed in. The login is a POST of a `FormData` whose answer is the token as plain text (`postText`). The page saves it with `useLocalStorage`, and from then on `/api/me` answers with the user.

```vue
<script setup>
const api = useApiClient()
const storage = useLocalStorage()
const user = ref('')
const message = ref('')

async function whoAmI() {
  try {
    const me = await api.getJSON('/api/me')
    message.value = `Signed in as ${me.name}.`
  } catch (e) {
    message.value = e.status === 401 ? 'Please sign in.' : e.message
  }
}

async function signIn() {
  const formData = new FormData()
  formData.append('user', user.value)
  try {
    storage.persist('token', await api.postText('/api/login', formData))
    await whoAmI()
  } catch (e) {
    message.value = e.message
  }
}

function signOut() {
  storage.removeByKey('token')
  whoAmI()
}
onMounted(whoAmI)
</script>

<template>
  <input v-model="user" id="user" placeholder="User name">
  <button id="signin" @click="signIn">Sign in</button>
  <button id="signout" @click="signOut">Sign out</button>
  <p id="message">{{ message }}</p>
</template>
```

```text
Page opens:                            Please sign in.
Visitor types Ann and presses Sign in: Signed in as Ann.
Visitor reloads the page:              Signed in as Ann.
Visitor presses Sign out:              Please sign in.
```

Read the output from top to bottom. On the first visit the page asks `/api/me`, the server says `401`, and the page turns the status into a friendly sentence. After signing in, the same question succeeds because the token is now sent. The page survives a reload because the token is in local storage, not in the page. Signing out removes the token and the next question fails again.

For your own pages the recipe is the same: `getJSON` to read, a `FormData` and a `post` call to send, one `try`/`catch` per action, `e.status` for the cases you handle and `e.message` for the rest.

---

## Appendix A. Quick reference

### A.1 Calls

| Call | Method | Body | Result |
|------|--------|------|--------|
| `getText(url)` | GET | none | string |
| `getJSON(url)` | GET | none | object or array |
| `getFile(url)` | GET | none | `Blob` |
| `postText(url, formData)` | POST | `FormData` | string |
| `postJSON(url, formData)` | POST | `FormData` | object or array |
| `postFile(url, formData)` | POST | `FormData` | `Blob` |

### A.2 The error object

| Field | Content |
|-------|---------|
| `message` | `error` of the body, else `message` of the body, else the body, else `Request failed with status N` |
| `status` | The status code (not set when there was no answer) |
| `data` | The body parsed as JSON, or `null` |
| `rawBody` | The body as text |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Server answers with an error status | `Error` with `message`, `status`, `data`, `rawBody` |
| Network failure | `TypeError` `Failed to fetch`, no `status` |
| JSON answer that cannot be parsed | `SyntaxError`, no `status` |
| Empty body where JSON is expected (status 200) | `SyntaxError` |
| Status 204 | `null` for every call |
| Local storage unavailable | No `Authorization`; no error |
| Error where a file is expected | `Error` with the server's message, not a `Blob` |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Setting `Content-Type` yourself for a `FormData`.** The boundary is lost; leave it to the browser.
- **Expecting a JSON body from `postJSON`.** It sends a `FormData`; the JSON is the answer.
- **Passing a plain object or a JSON string instead of a `FormData`.** No error is raised: the browser sends the text `[object Object]` (or the JSON string) as `text/plain`, and the server finds no fields.
- **Forgetting that a list needs `JSON.stringify`.** A `FormData` field holds text or a file.
- **Treating 200 with an `error` field as a failure.** Only the status counts.
- **Showing `e.message` for a network failure.** `Failed to fetch` means nothing to a visitor; check `e.status`.
- **Showing `e.message` for a gateway page.** It can be a whole HTML document.
- **Calling `getJSON` for an address that may answer 204.** The result is `null`; check before using it.
- **Passing a `FormData` to a `get` call.** There is no such parameter; it is ignored.
- **Sending the token where it is not wanted.** Every call carries it, including calls to other servers if a full address is used.

### C.2 Behaviour worth knowing

- The token is read for each call, so signing in and out takes effect at once.
- Files are uploaded by putting them in the `FormData`.
- The status is checked before the body is read, so errors are never turned into a `Blob`.
- Failures are logged in the console before they are thrown.
- Only GET and POST exist.

### C.3 Limits

| What | Limit |
|------|-------|
| Methods | GET and POST |
| Request body | `FormData` only |
| Extra headers | None can be passed |
| Timeout, cancel | None |
| Base address | None; the address is used as given |
| Retry | None |

### C.4 Tuning

- Keep one `try`/`catch` per user action and show `e.message`, or a sentence of your own for the statuses you know.
- Put the busy flag and message in a small helper when several buttons call the server.
- Let the server send `error` with a sentence meant for visitors; the page then needs no translation table.
- Handle `401` in one place, for example by sending the visitor to the login page.
- Build lists as one JSON field and read it back on the server.

## Appendix D. Message reference

### D.1 Errors

| Message | Cause |
|---------|-------|
| The body's `error` or `message` | The server's own sentence |
| `Request failed with status N` | An error status with an empty body |
| `Failed to fetch` | The server could not be reached |
| `Unexpected end of JSON input` | A JSON call whose body was empty |
| `... is not valid JSON` | A JSON call whose body was not JSON |

### D.2 Other values

| Value | Meaning |
|-------|---------|
| `[ApiClient] ✗ GET /url` | Console line written for each failed call |
| `Bearer <token>` | The `Authorization` header when a token is saved |
| `X-Timezone` | The visitor's time zone, such as `Asia/Tehran` |
