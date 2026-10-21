# useOperatingSystemNotification

*J2OS Platform 2.4*

A new order arrives while the shop owner is working in another window. A long export finishes while the visitor reads something else. In both cases the page should alert the visitor, and the operating system's own notification suits this: it appears outside the browser, on top of whatever else is open.

**useOperatingSystemNotification** is a small composable of the Nuxt front end of J2OS that shows such a notification. It has three features:

- **One function.** `showNotification(title, message, website)` asks for permission if needed and then shows the notification.
- **A yes or no answer.** It returns `true` if the notification is created and `false` if it cannot be; it does not throw.
- **A click that does something.** Clicking the notification brings the window forward, opens the address that was given (if any) in a new tab and closes the notification.

In code, it comes to this:

```js
const { showNotification } = useOperatingSystemNotification()

const shown = await showNotification(
  'Backup finished',
  'All 24 files were saved.',
  'https://admin.example/backups'      // optional: opened when the notification is clicked
)

// true  - the notification was created
// false - the browser cannot show it, or the visitor refused
```

The path through the guide: chapter 1 explains how the composable works and chapter 2 shows the first notification. Chapter 3 covers permission and every way a notification can fail, and chapter 4 the click. Chapter 5 builds an order alert that groups several orders into one notification. The appendices are meant for quick lookups.

## Contents

1. [How useOperatingSystemNotification works](#1-how-useoperatingsystemnotification-works)
2. [The first notification](#2-the-first-notification)
3. [Permission and failures](#3-permission-and-failures)
4. [Clicking the notification](#4-clicking-the-notification)
5. [The last example](#5-the-last-example)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are full pages from a Nuxt project, and each is a single `.vue` file. In Nuxt `ref`, `onUnmounted` and `useOperatingSystemNotification` need no `import`, so none is written.

---

## 1. How useOperatingSystemNotification works

Before the details, the words this guide keeps coming back to:

- **Notification.** A small message that the operating system shows outside the browser window. The browser makes it on the page's behalf.
- **Permission.** The visitor's answer to "may this site show notifications?". It is one of three values: `default` (not decided yet), `granted` and `denied`.
- **User action.** A click or a key press. Browsers want the permission question to start from one, and may ignore it otherwise.

What the composable offers:

| Piece | Role |
|-------|------|
| `showNotification(title, message, website)` | Asks for permission, shows the notification and returns `true` or `false` |

That is the whole interface. Six facts shape the rest of the guide:

- **`true` means created.** It does not mean that the notification was seen, read or clicked. The operating system may still hide it, for instance in a do-not-disturb mode.
- **Permission is requested on every call.** If the visitor has already answered, the browser gives that answer at once and shows no question.
- **Failures are quiet.** An unsupported browser, a refusal and a creation error all return `false` and write a message to the console. No exception is thrown.
- **Only a title and a text.** The composable sets no icon, sound, buttons or tag, so identical notifications pile up (chapter 2).
- **The click is ready-made.** It focuses the window, opens `website` in a new tab if one was given, and closes the notification.
- **The browser and the device have the final say.** The page must be served from a secure address (HTTPS), and some mobile browsers refuse to create notifications from a page at all. The project's own example notes that in Firefox the visitor must change a setting first.

## 2. The first notification

A button that shows a notification. The page keeps the answer:

```vue
<script setup>
const { showNotification } = useOperatingSystemNotification()

const result = ref('')

async function notify() {
  const shown = await showNotification('Backup finished', 'All 24 files were saved.')
  result.value = String(shown)
}
</script>

<template>
  <button id="go" @click="notify">Notify me</button>
  <p id="result">{{ result }}</p>
</template>
```

```text
result: true
notification: {"title": "Backup finished", "options": {"body": "All 24 files were saved."}, "closed": false}
permission requests: 1
after a second click: notifications 2, permission requests 2
```

- **The answer** is `true`, and one notification is created with the given title and a text, and nothing else in its options.
- **The permission is requested once** for this call.
- **A second click** asks again (the browser now answers at once) and creates a second notification. Nothing is merged or replaced: two identical notifications are two notifications. A page that can fire often should count and group, as chapter 5 does.

Call `showNotification` in a click handler, as the example does. At that moment browsers accept a permission question without problems.

## 3. Permission and failures

The same button, tried in every situation the composable has to deal with. The page also asks the browser why a notification did not appear, because the composable only says `false`:

```vue
<script setup>
const { showNotification } = useOperatingSystemNotification()

const result = ref('')
const reason = ref('')
const permission = ref('')

function whyNot() {
  if (typeof Notification === 'undefined') return 'This browser cannot show notifications.'
  if (Notification.permission === 'denied') return 'Notifications are blocked. Allow them in the browser settings.'
  if (Notification.permission === 'default') return 'You did not answer the permission question.'
  return 'The notification could not be created on this device.'
}

async function notify() {
  const shown = await showNotification('Report ready', 'The monthly report can be opened now.')
  result.value = String(shown)
  reason.value = shown ? '' : whyNot()
  permission.value = typeof Notification === 'undefined' ? 'none' : Notification.permission
}
</script>

<template>
  <button id="go" @click="notify">Notify me</button>
  <p id="result">{{ result }}</p>
  <p id="reason">{{ reason }}</p>
  <p id="permission">{{ permission }}</p>
</template>
```

```text
permission already granted
  result: true | permission afterwards: granted | notifications shown: 1
  console: nothing
not asked yet, the visitor allows
  result: true | permission afterwards: granted | notifications shown: 1
  console: nothing
not asked yet, the visitor refuses
  result: false | permission afterwards: denied | notifications shown: 0
  console: warning: Notification permission was not granted (status: denied)
  page says: Notifications are blocked. Allow them in the browser settings.
prompt closed without an answer
  result: false | permission afterwards: default | notifications shown: 0
  console: warning: Notification permission was not granted (status: default)
  page says: You did not answer the permission question.
blocked earlier
  result: false | permission afterwards: denied | notifications shown: 0
  console: warning: Notification permission was not granted (status: denied)
  page says: Notifications are blocked. Allow them in the browser settings.
browser without notifications
  result: false | permission afterwards: none | notifications shown: 0
  console: error: This browser does not support operating system notifications.
  page says: This browser cannot show notifications.
the browser refuses to create it
  result: false | permission afterwards: granted | notifications shown: 0
  console: error: Failed to show notification (likely a mobile browser restriction): TypeError: Failed to construct 'Noti...
  page says: The notification could not be created on this device.
```

- **Permission already granted,** or **not asked yet and the visitor allows it:** `true`, and one notification.
- **Not asked yet and the visitor refuses,** or **blocked earlier:** `false`, permission `denied`, and a console warning with the status. No notification.
- **The question is closed without an answer:** `false`, the permission stays `default`, and the warning says so. The question can be asked again later.
- **A browser without notifications:** `false`, with a console error. The page can see this itself, because `Notification` does not exist.
- **A browser that refuses to create the notification** (typical of mobile browsers): `false`, with a console error that includes the browser's own message, although the permission is `granted`.

The composable's `false` hides these differences, but the page can tell most of them apart by looking at `Notification.permission` afterwards, as the page above does in its `whyNot` function:

- `denied` means the visitor must change the browser's setting; asking again does not help.
- `default` means the question has not been answered; ask again on the next click.
- `granted` together with `false` means the device refused the notification; use an in-page message instead.
- A missing `Notification` means the browser does not support notifications.

A page that needs a notification to be seen should always have a second way: a message in the page itself, as chapter 5 does.

## 4. Clicking the notification

The click is the only interaction with a notification. The page below makes one notification with a link and one without, and clicks each:

```vue
<script setup>
const { showNotification } = useOperatingSystemNotification()

const sent = ref(0)

async function withLink() {
  await showNotification('Order 305', 'Paid, ready to pack.', 'https://shop.example/orders/305')
  sent.value++
}

async function withoutLink() {
  await showNotification('Heads up', 'Nothing to open.')
  sent.value++
}
</script>

<template>
  <button id="with" @click="withLink">With a link</button>
  <button id="without" @click="withoutLink">Without a link</button>
  <p id="sent">{{ sent }}</p>
</template>
```

```text
with a link: notification Order 305 | focus calls 1 | opened [['https://shop.example/orders/305', '_blank']] | closed True
without a link: notification Heads up | focus calls 2 | windows opened in total 1 | closed True
```

- **With a link,** the click focuses the window, opens the address in a new tab (`_blank`) and closes the notification.
- **Without a link,** the click focuses the window and closes the notification, but no tab is opened.
- **The address is not checked.** Whatever text is passed goes to `window.open`. Never pass an address that came from a visitor or from an outside source without checking it. For example, allow only web addresses:

```js
// Only web addresses are allowed to be opened by a click.
const safe = (url) => (/^https?:\/\//i.test(url) ? url : undefined)

showNotification(title, text, safe(link))
```

The composable offers no way to react to the notification being closed or ignored, and no way to close it from the page. If the page needs that, it has to use the browser's `Notification` directly.

## 5. The last example

Let us build an order alert. A checkbox turns desktop alerts on; turning it on is the user action that asks for permission, and it shows a test notification so that the visitor knows what to expect. If that fails, the checkbox goes back off and a message says why. Each new order is always listed in the page; with alerts on, orders that arrive within a short moment are grouped into one notification, so a rush does not become a stack of them:

```vue
<script setup>
const { showNotification } = useOperatingSystemNotification()

const alerts = ref(false)
const notice = ref('')
const inPage = ref([])

let nextId = 101
let waiting = []
let timer = null

async function toggle() {
  if (!alerts.value) return

  const ok = await showNotification('Desktop alerts are on', 'New orders will appear here.')
  if (!ok) {
    alerts.value = false
    notice.value = 'Desktop alerts could not be turned on. Check the browser settings.'
  } else {
    notice.value = ''
  }
}

function newOrder() {
  const order = { id: nextId++, total: 20 + (nextId % 7) * 5 }
  inPage.value.push(`Order #${order.id}`)

  if (!alerts.value) return

  waiting.push(order)
  clearTimeout(timer)
  timer = setTimeout(flush, 300)
}

async function flush() {
  const batch = waiting
  waiting = []

  const ok = batch.length === 1
    ? await showNotification(`Order #${batch[0].id}`, `Total: ${batch[0].total}`, `https://shop.example/orders/${batch[0].id}`)
    : await showNotification(`${batch.length} new orders`, `From #${batch[0].id} to #${batch.at(-1).id}`, 'https://shop.example/orders')

  if (!ok) notice.value = 'Desktop alert failed; the order is on this page.'
}

onUnmounted(() => clearTimeout(timer))
</script>

<template>
  <label><input id="alerts" v-model="alerts" type="checkbox" @change="toggle"> Desktop alerts</label>
  <button id="order" @click="newOrder">New order</button>
  <p id="notice">{{ notice }}</p>
  <div v-for="line in inPage" :key="line" class="row">{{ line }}</div>
</template>
```

```text
alerts on: notifications ['Desktop alerts are on'], checkbox True
one order: {"title": "Order #101", "options": {"body": "Total: 40"}, "closed": false}
clicked: opened [['https://shop.example/orders/101', '_blank']]
three quick orders: 1 new notification(s): {"title": "3 new orders", "options": {"body": "From #102 to #104"}, "closed": false}
orders listed on the page: ['Order #101', 'Order #102', 'Order #103', 'Order #104']
blocked: checkbox False | notice='Desktop alerts could not be turned on. Check the browser settings.' | notifications 0
order while off: notifications 0, listed 1
```

- **Turning alerts on** asks for permission and shows one notification.
- **One order** produces a notification with its number and total, and a click opens its page.
- **Three orders in quick succession** produce a single notification ("3 new orders") that points to the list of orders.
- **With permission blocked,** the checkbox switches itself off and a message explains why, and **new orders** still appear in the page, with no notification.
- **The timer is cleared** when the component goes away, so a notification is never sent for a page that is gone.

---

## Appendix A. Quick reference

### A.1 The function

| Call | Returns | Notes |
|------|---------|-------|
| `showNotification(title, message, website)` | A promise of `true` or `false` | `website` is optional; never throws an exception |

### A.2 What each permission leads to

| Permission | Result of the call |
|------------|--------------------|
| `granted` | The notification is created (`true`), unless the browser refuses |
| `default` and the visitor allows | Becomes `granted`; as above |
| `default` and the question stays open | `false` |
| `denied` | `false`; the question is not shown again |
| Browser without `Notification` | `false`, with a console error |

### A.3 What the notification has

| Part | Value |
|------|-------|
| Title | The first argument |
| Text | The second argument (`body`) |
| Icon, sound, buttons, tag | None |
| On click | Focus the window, open `website` in a new tab if given, close the notification |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Browser without notifications | `false`, console error |
| Permission denied | `false`, console warning with the status |
| Permission question left open | `false`, console warning with the status |
| The browser refuses to create the notification | `false`, console error with the browser's message |
| Any other error while creating it | `false`, the same console error |
| Notification created | `true` |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Asking at page load.** Browsers may ignore a permission question that no user action caused. Ask in a click handler.
- **Treating `true` as "the user saw it".** It only means the notification was created.
- **Treating `false` as one problem.** It can mean unsupported, refused, unanswered or blocked by the device.
- **Passing an unchecked address.** Whatever is given is opened on click.
- **Sending many identical notifications.** They stack; group them.
- **Counting on it in every browser.** Some mobile browsers refuse to create notifications, and a page on plain HTTP is usually not allowed to ask for permission.
- **No fallback.** A page that depends on being noticed needs an in-page message too.
- **Expecting an icon or sound.** None is set.
- **A missing or non-text title or message.** The browser turns the value into text: `undefined` becomes the word "undefined", and an object becomes "[object Object]". Pass text.

### C.2 Behavior worth knowing

- The composable asks for permission on each call; an already given answer returns immediately.
- A refusal or failure is written to the console, never thrown.
- The notification object is not returned, so the page cannot close it or react to it being closed.
- Clicking focuses the window, opens `website` in a new tab (if given) and closes the notification.
- Nothing is stored between calls, and nothing is sent over the network.
- The title and text are shown as plain text.

### C.3 Limits

| What | Limit |
|------|-------|
| Notification content | Title and text only |
| Click action | Open one address, if given |
| Control after creation | None |
| Grouping, limits, repeats | None; done by the page |
| Where it works | Browsers with notifications, on secure pages, where the device allows |

### C.4 Tuning

- Ask for permission in the click that turns alerts on, and show a first notification at once.
- Look at `Notification.permission` after a `false` to give the visitor the right advice.
- Group notifications that arrive close together.
- Always keep the information in the page too.
- Check any address before passing it as `website`.
- Clear timers when the component is removed.

## Appendix D. Message reference

### D.1 Console messages

| Message | Cause |
|---------|-------|
| `This browser does not support operating system notifications.` (error) | `Notification` does not exist in the browser |
| `Notification permission was not granted (status: ...)` (warning) | The permission is `denied` or `default` after the question |
| `Failed to show notification (likely a mobile browser restriction): ...` (error) | Creating the notification threw an error |
