# DataWizard

*J2OS Platform 2.4*

Some forms are too long for one screen or depend on earlier answers: creating a user, opening an account, placing an order. Cutting them into steps makes each screen small, and a check after each step catches mistakes while the visitor still remembers what they typed.

**DataWizard** is a Vue component for the Nuxt front end of J2OS that shows such a form as a dialog with steps. It is built on the modal, stepper and alert components of Nuxt UI. It has three main features:

- **The page owns the content.** DataWizard draws the frame (heading, step list, buttons, messages). What goes inside each step is the page's own markup, in a slot.
- **One data object for all steps.** Every step reads and writes the same `data` object. Checks and the final action receive a copy of it.
- **Checks and the final action can be slow.** They may be `async`. While they run the form is locked, the button spins and Back becomes "cancel operation".

A first look:

```vue
<button @click="wizard.start()">New contact</button>

<DataWizard ref="wizard" :steps="steps" :initial-data="() => ({ name: '' })" :on-complete="save">
  <template #step-name="{ data }"><UInput v-model="data.name" /></template>
  <template #step-done="{ data }"><p>Save {{ data.name }}?</p></template>
</DataWizard>
```

The guide begins with how the component works (chapter 1) and the first wizard (2). It then covers moving between steps (3), checking a step (4), the final action (5), starting with data (6), appearance (7) and wizards that open wizards (8). Chapter 9 builds a user-creation wizard.

## Contents

1. [How DataWizard works](#1-how-datawizard-works)
2. [The first wizard](#2-the-first-wizard)
3. [Moving between steps](#3-moving-between-steps)
4. [Checking a step](#4-checking-a-step)
5. [Finishing](#5-finishing)
6. [Starting with data](#6-starting-with-data)
7. [Appearance](#7-appearance)
8. [Wizards that open wizards](#8-wizards-that-open-wizards)
9. [A user-creation wizard](#9-a-user-creation-wizard)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Each example is a whole page you can use in a Nuxt project, and each is a single `.vue` file. In Nuxt `ref` and the component `DataWizard` need no `import`, so none is written. The texts that DataWizard shows by itself (the buttons, "step 2 of 3") are Persian and fixed.

---

## 1. How DataWizard works

To keep later chapters short, here are the terms they rely on:

- **Step.** One screen of the wizard, described by an object in the `steps` list.
- **Data.** The object that holds everything the visitor has entered. It is shared by all steps.
- **Trail.** The row of buttons at the top with the names of the steps already done and the current one.
- **Validate.** The optional check of a step, run when the visitor presses Next.
- **Complete.** The final action, run when the visitor confirms on the last step.
- **Busy.** The state while a check or the final action is running.

Its parts at a glance:

| Piece | Role |
|-------|------|
| `steps` | Required: the list of step objects (A.2) |
| `initialData` | A function that returns the starting data |
| `onComplete` | A function run on the last step; it saves the result |
| `title`, `size`, `fullscreen` | Heading, width and full-screen mode of the dialog |
| Slots `step-<id>`, `default` | The content of each step |
| Event `cancel` | The dialog is closed without finishing |
| Methods `start`, `cancel`, `back`, `next`, `goTo` | Control from the page |

Five things to hold on to:

- **It opens only when asked.** The page calls `start()`; there is no `v-model` that opens it.
- **Every start is fresh.** `start()` goes to step 1 and rebuilds the data from `initialData`.
- **Checks need exactly `true`.** A step passes only when `validate` returns `true`; anything else stops the wizard on that step.
- **Finishing is a function, not an event.** There is no "done" event. `onComplete` does the work, and when it succeeds the dialog closes.
- **Going back is always free.** Back never runs a check.

The dialog is drawn with Nuxt UI (`UModal`, `UStepper`, `UAlert`), with Lucide icons, as in the rest of the J2OS front end.

## 2. The first wizard

The smallest wizard is a list of steps, a slot for the content of each, and a function that receives the result. A step is an object with an `id` and a `title`, and the `id` names the slot that holds the step's content. A registration in three short steps:

```vue
<script setup>
const wizard = ref(null)
const saved = ref('nothing yet')

const steps = [
  { id: 'name', title: 'Your name' },
  { id: 'mail', title: 'Your email' },
  { id: 'confirm', title: 'Check and send' }
]

const initialData = () => ({ name: '', email: '' })

const finish = data => {
  saved.value = `${data.name} <${data.email}>`
}
</script>

<template>
  <button id="start" @click="wizard.start()">New contact</button>
  <p id="saved">Saved: {{ saved }}</p>

  <DataWizard ref="wizard" :steps="steps" :initial-data="initialData" :on-complete="finish" title="New contact">
    <template #step-name="{ data }">
      <UInput v-model="data.name" placeholder="Name" />
    </template>
    <template #step-mail="{ data }">
      <UInput v-model="data.email" placeholder="Email" />
    </template>
    <template #step-confirm="{ data }">
      <p>{{ data.name }} / {{ data.email }}</p>
    </template>
  </DataWizard>
</template>
```

```text
before start: closed
start: مرحله 1 از 3 | trail: Your name* | shows: <input Name> | buttons: ['انصراف', 'بعدی']
Next: مرحله 2 از 3 | trail: Your name > Your email* | shows: <input Email> | buttons: ['قبلی', 'بعدی']
Next: مرحله 3 از 3 | trail: Your name > Your email > Check and send* | shows: Sara / sara@example.com | buttons: ['قبلی', 'تأیید نهایی']
Back: مرحله 2 از 3 | trail: Your name > Your email* | shows: <input Email> | buttons: ['قبلی', 'بعدی']
Final button: closed | Saved: Sara <sara@example.com>
start again: مرحله 1 از 3 | trail: Your name* | shows: <input Name> | buttons: ['انصراف', 'بعدی']
Cancel on the first step: closed
```

Three lines of the page do the work. `wizard.start()` opens the dialog; before it, nothing is shown. `#step-name="{ data }"` is the content of the step with `id: 'name'`, and `data` is the shared object, so `v-model="data.name"` is all a field needs. `onComplete` is called with the data when the visitor confirms on the last step.

The texts around the content are drawn by the component, in Persian. On the first step the left button says "انصراف" (cancel); later it says "قبلی" (previous). The right button says "بعدی" (next), and on the last step "تأیید نهایی" (final confirmation). A line such as "مرحله 2 از 3" (step 2 of 3) says where the visitor is, and the trail at the top grows as they advance.

The output shows three rules of the wizard. The text typed on step 1 is still there after going back from step 3, so **going back keeps the data**. `finish` receives the data when the last button is pressed, and the dialog closes. **Starting again** begins on step 1, with fresh data.

## 3. Moving between steps

The visitor moves with the two buttons, but a page often needs to move the wizard itself: to open it on a given step, or to cancel it when something else happens. The methods of the wizard are reached through a template ref:

| Method | What it does |
|--------|--------------|
| `start(override)` | Open at step 1 with fresh data (chapter 6) |
| `next()` | Same as the right button: check, then go on or finish |
| `back()` | Same as the left button: one step back, or cancel on step 1 |
| `goTo(index)` | Go to a step by its position, starting from 0 |
| `cancel()` | Close without finishing |

The trail at the top is a way to move as well: clicking a name goes back to that step. The trail shows only the steps already done and the current one, so it can lead backwards only. Going back never runs checks and never deletes data, which is why the visitor can always return to correct an earlier answer.

`goTo(index)` is the only way to jump **forward**. It does not skip the rules: it runs the check of every step between the current one and the target, and if one fails, the wizard stops on that step and shows its message:

```vue
<script setup>
const wizard = ref(null)
const out = ref('')

const steps = [
  { id: 's1', title: 'One', pathLabel: 'First', validate: d => (d.a ? true : 'Fill in A.') },
  { id: 's2', title: 'Two', pathLabel: 'Second', validate: d => (d.b ? true : 'Fill in B.') },
  { id: 's3', title: 'Three', pathLabel: 'Third' }
]
</script>

<template>
  <button id="start" @click="wizard.start({ a: 'x' })">Start</button>
  <button id="jump" @click="wizard.goTo(2).then(() => (out = 'index ' + wizard.currentIndex))">Jump to 3</button>
  <p id="out">{{ out }}</p>

  <DataWizard ref="wizard" :steps="steps" :initial-data="() => ({ a: '', b: '' })" @cancel="out = 'cancelled'">
    <template #step-s1="{ data }"><UInput v-model="data.a" placeholder="A" /></template>
    <template #step-s2="{ data }"><UInput v-model="data.b" placeholder="B" /></template>
    <template #step-s3><p>Last</p></template>
  </DataWizard>
</template>
```

```text
start: مرحله 1 از 3 | trail: First* | shows: <input A> | buttons: ['انصراف', 'بعدی']
goTo(2): مرحله 2 از 3 | trail: First > Second* | shows: <input B> | alert: Fill in B. | buttons: ['قبلی', 'بعدی'] | index 1
goTo(2) again: مرحله 3 از 3 | trail: First > Second > Third* | shows: Last | buttons: ['قبلی', 'تأیید نهایی'] | index 2
click First in the trail: مرحله 1 از 3 | trail: First* | shows: <input A> | buttons: ['انصراف', 'بعدی']
close with the X: closed | cancelled
```

`goTo(2)` runs the check of step 1 (fine) and of step 2 (B is empty), so the wizard stops on step 2 and shows "Fill in B.". After B is filled in, the same call reaches step 3. The trail lets the visitor go back to step 1 with one click, and the X of the dialog counts as a cancel: the `cancel` event is emitted.

Note that the dialog cannot be closed by clicking outside it or by pressing Escape. A half-filled long form is not thrown away by accident; the only ways out are the cancel button, the X and `cancel()`.

## 4. Checking a step

A form that is long enough for steps is long enough to contain mistakes. The wizard checks a step when the visitor presses Next, so that an error is reported while the visitor still remembers what they typed. A step with a `validate` function is checked: the function receives the data and returns `true` when all is well, or a text that explains the problem:

```js
validate: data => true                           // the step is fine
validate: data => 'Please type a name.'          // the step is not fine; the text is shown
validate: data => data.name.trim() !== ''        // WRONG: false gives only the general message
validate: data => { if (!data.name) return 'Name?' }   // WRONG: forgot to return true at the end
```

The first two lines are the right shapes. A step passes only when the result is exactly `true`. Returning `false`, `undefined` (a forgotten `return`) or any other value stops the wizard with the general message "اطلاعات این مرحله معتبر نیست." ("The data of this step is not valid."); a text result is shown instead, so return text whenever the visitor can do something about the problem.

The function may be `async`, for example to ask the server whether a username is free. The page below does that:

```vue
<script setup>
const wizard = ref(null)
const log = ref([])

const taken = ['admin', 'root']

const steps = [
  {
    id: 'name',
    title: 'Name',
    validate: data => (data.name.trim() ? true : 'Please type a name.')
  },
  {
    id: 'user',
    title: 'Username',
    description: 'Pick one nobody else has.',
    icon: 'i-lucide-user',
    validate: async (data, ctx) => {
      await new Promise(r => setTimeout(r, 150))
      log.value.push(`checked ${data.user}, aborted: ${ctx.signal.aborted}`)
      return taken.includes(data.user) ? 'That username is taken.' : true
    }
  },
  { id: 'done', title: 'Done' }
]
</script>

<template>
  <button id="start" @click="wizard.start()">Start</button>
  <p id="log">{{ log.join(' | ') }}</p>

  <DataWizard ref="wizard" :steps="steps" :initial-data="() => ({ name: '', user: '' })">
    <template #step-name="{ data }">
      <UInput v-model="data.name" placeholder="Name" />
    </template>
    <template #step-user="{ data }">
      <UInput v-model="data.user" placeholder="Username" />
    </template>
    <template #step-done>
      <p>All good.</p>
    </template>
  </DataWizard>
</template>
```

```text
title and description on step 1: 'Name' / ''
Next with an empty name: مرحله 1 از 3 | trail: Name* | shows: <input Name> | alert: Please type a name. | buttons: ['انصراف', 'بعدی']
Next: مرحله 2 از 3 | trail: Name > Username* | shows: <input Username> | buttons: ['قبلی', 'بعدی']
title and description on step 2: 'Username' / 'Pick one nobody else has.' | stepper: ['Name[i-lucide-check]', 'Username*[i-lucide-user]', 'Done[i-lucide-circle]']
while checking: مرحله 2 از 3 | trail: Name > Username* | shows: <input Username> | form disabled | buttons: ['لغو عملیات', 'بعدی (disabled) (loading)']
answer: مرحله 2 از 3 | trail: Name > Username* | shows: <input Username> | alert: That username is taken. | buttons: ['قبلی', 'بعدی']
checking again, press Back: مرحله 2 از 3 | trail: Name > Username* | shows: <input Username> | form disabled | buttons: ['لغو عملیات', 'بعدی (disabled) (loading)']
after Back: مرحله 1 از 3 | trail: Name* | shows: <input Name> | buttons: ['انصراف', 'بعدی']
log: checked admin, aborted: false | checked sara, aborted: true
```

A text result appears in a red box above the form, and the wizard stays on the step. While the check runs, the wizard protects itself: the form is disabled, the Next button spins, and the left button says "لغو عملیات" (cancel operation). Pressing it abandons the check and moves one step back (on step 1 it closes the wizard instead, like Cancel). The check function still runs to its end, but its signal becomes `aborted: true` (the output above shows this), so a check that has side effects must look at the signal (next chapter).

Two more facts about `validate`. It gets a **copy** of the data, so changing the copy does not change the real data; a check should check, not repair. And if it throws, the message of the error is shown; when the error has no message, the general text "انجام عملیات ناموفق بود. دوباره تلاش کنید." ("The operation failed. Try again.") appears.

The last step usually has no `validate`: pressing the right button there starts the final action.

## 5. Finishing

On the last step the right button says "تأیید نهایی". Pressing it runs the last step's check (if any) and then calls `onComplete(data, ctx)`. The dialog closes when `onComplete` ends without an error. This is where the data is saved, and saving is exactly where things go wrong: the server may be slow, down, or asked twice.

Both `validate` and `onComplete` get a `ctx` object to help with that:

```js
ctx = {
  signal,           // an AbortSignal: it fires when the visitor goes Back or the wizard is restarted
  generation,       // a number that grows with every start()
  idempotencyKey    // a new random text for every attempt; send it to the server
}
```

The page below makes the first attempt fail and the second work:

```vue
<script setup>
const wizard = ref(null)
const sent = ref([])
let attempt = 0
let lastKey = ''

const steps = [
  { id: 'one', title: 'Amount' },
  { id: 'two', title: 'Send' }
]

async function send(data, ctx) {
  attempt++
  sent.value.push(`attempt ${attempt}: amount ${data.amount}, new key: ${ctx.idempotencyKey !== lastKey}`)
  lastKey = ctx.idempotencyKey
  await new Promise(r => setTimeout(r, 300))
  if (attempt === 1) throw new Error('The bank is not answering.')
  if (ctx.signal.aborted) return
  sent.value.push('done')
}
</script>

<template>
  <button id="start" @click="wizard.start()">Pay</button>
  <p id="sent">{{ sent.join(' | ') }}</p>

  <DataWizard ref="wizard" :steps="steps" :initial-data="() => ({ amount: 10 })" :on-complete="send">
    <template #step-one="{ data }">
      <UInput v-model="data.amount" placeholder="Amount" />
    </template>
    <template #step-two="{ data }">
      <p>Pay {{ data.amount }}?</p>
      <UInput v-model="data.note" placeholder="Note" />
    </template>
  </DataWizard>
</template>
```

```text
step two: مرحله 2 از 2 | trail: Amount > Send* | shows: Pay 10? <input Note> | buttons: ['قبلی', 'تأیید نهایی']
sending: مرحله 2 از 2 | trail: Amount > Send* | shows: Pay 10? <input Note> | form disabled | buttons: ['لغو عملیات', 'تأیید نهایی (disabled) (loading)']
failed: مرحله 2 از 2 | trail: Amount > Send* | shows: Pay 10? <input Note> | alert: The bank is not answering. | buttons: ['قبلی', 'تأیید نهایی']
second try: closed
log: attempt 1: amount 10, new key: true | attempt 2: amount 10, new key: true | done
Back while sending: مرحله 1 از 2 | trail: Amount* | shows: <input Amount> | buttons: ['انصراف', 'بعدی']
log: attempt 1: amount 10, new key: true | attempt 2: amount 10, new key: true | done | attempt 3: amount 10, new key: true
```

While sending, the form is locked and the button spins. A failure (a thrown error) puts its message in the red box; the wizard stays on the last step with the data intact, and the visitor can simply press the button again. Every attempt has a new `idempotencyKey`, which a server can use to ignore a request it has already seen.

The last two lines of the output are the delicate part. If the visitor presses **Back during sending**, the wizard returns to the previous step and the late answer is ignored: no "done" line appears for attempt 3 and the dialog does not close. But going back does not stop a request that is already on its way. Pass the signal to `fetch` so the browser cancels it, and send the key so the server can tell a repeat from a new request:

```js
async function save(data, ctx) {
  await fetch('/api/users', {
    method: 'POST',
    signal: ctx.signal,                                     // stops the request on Back
    headers: { 'Idempotency-Key': ctx.idempotencyKey },     // lets the server ignore a repeat
    body: JSON.stringify(data)
  })
}
```

`onComplete` may return anything; the value is ignored, and there is no event for success. If the page needs to react (refresh a list, show a message), `onComplete` is the place for it.

## 6. Starting with data

A wizard is used for editing as often as for creating. The visitor changes a record in the same steps that created it, and the steps should start with the record's values. `initialData` is a function that returns the starting data. A function is used so that every start gets a fresh object, and `start()` can lay values over it:

```vue
<script setup>
const wizard = ref(null)
const after = ref('')

const steps = [
  { id: 'a', title: 'Edit' },
  { id: 'b', title: 'Check' }
]
</script>

<template>
  <button id="blank" @click="wizard.start()">Add</button>
  <button id="edit" @click="wizard.start({ title: 'Old title', tags: ['a', 'b'] })">Edit</button>
  <button id="peek" @click="after = JSON.stringify(wizard.data)">Peek</button>
  <p id="after">{{ after }}</p>

  <DataWizard ref="wizard" :steps="steps" :initial-data="() => ({ title: '', tags: [] })">
    <template #default="{ data, step, index }">
      <p>Step {{ index }}: {{ step.id }}</p>
      <UInput v-model="data.title" placeholder="Title" />
    </template>
  </DataWizard>
</template>
```

```text
Add: مرحله 1 از 2 | trail: Edit* | shows: Step 0: a <input Title> | buttons: ['انصراف', 'بعدی']
data: {"title":"Draft","tags":[]}
Next: مرحله 2 از 2 | trail: Edit > Check* | shows: Step 1: b <input Title> | buttons: ['قبلی', 'تأیید نهایی']
cancelled: closed
data after cancel: {"title":"Draft","tags":[]}
Edit: مرحله 1 از 2 | trail: Edit* | shows: Step 0: a <input Title> | buttons: ['انصراف', 'بعدی']
data: {"title":"Old title","tags":["a","b"]}
Add again, data: {"title":"","tags":[]}
```

This page has no `#step-...` slots. The `default` slot is used for every step, and it receives `data`, `step` and `index` (the position, from 0), which is convenient when the steps look alike.

`start()` gives blank data, from `initialData`. `start({ ... })` is for editing: the values passed are laid over the starting data, key by key, so a wizard that edits a record starts with the record. A nested object is replaced as a whole, not merged. After the dialog closes the data stays, readable as `wizard.data`, until the next `start()`, which clears it. Canceling fires the `cancel` event, and the page decides what to do.

One technical rule follows from "fresh". Both `initialData()` and the argument of `start()` are copied with `structuredClone`, so they must hold plain data: text, numbers, dates, arrays and plain objects, not functions and not reactive objects. A row taken from a `ref` is reactive, and passing it fails:

```js
wizard.value.start(row)             // fails when row is reactive: DataCloneError
wizard.value.start({ ...row })      // works when row holds plain values
```

For a flat record a spread is enough. For a record with nested objects, make a deep copy first (for example `structuredClone(toRaw(row))`).

## 7. Appearance

The wizard draws its own frame, but each part can be tuned. The settings are on the component and on each step:

- **Heading.** `title` is the heading of the dialog. Without it the title of the current step is used, so the heading changes as the visitor advances. A step's `description` appears under the heading.
- **Step list.** The stepper shows each step's `title` and `description`. A step's `icon` replaces the default (a circle, or a check once the step is done). The stepper is only a picture; it cannot be clicked.
- **Trail.** `pathLabel` is the short name of the step in the trail. Without it, the `title` is used. A short label keeps the trail on one line.
- **Size.** `size` is `md` (the default), `lg`, `xl` or `auto`. `fullscreen` makes the dialog cover the screen.
- **Content.** A step's content may be as large as needed; it scrolls sideways if it is wider than the dialog.

The wizard of chapter 4 has an icon and a description on its step, and its output shows both in the step list and under the heading. When the content changes from one step to another, it fades and slides slightly; the animation is turned off for visitors who prefer reduced motion.

## 8. Wizards that open wizards

Sometimes a step needs a whole wizard of its own: while ordering, the visitor chooses an address, and the address needs several steps. Two `DataWizard`s can live on the same page, and the child writes its result into the parent's `data` when it finishes:

```vue
<script setup>
const order = ref(null)
const address = ref(null)

const orderSteps = [
  { id: 'item', title: 'Item' },
  { id: 'ship', title: 'Shipping' }
]
const addressSteps = [
  { id: 'street', title: 'Street' },
  { id: 'check', title: 'Check' }
]

function saveAddress(data) {
  order.value.data.address = `${data.street}`
}
</script>

<template>
  <button id="start" @click="order.start()">Order</button>

  <DataWizard ref="order" :steps="orderSteps" :initial-data="() => ({ item: '', address: '' })" title="Order">
    <template #step-item="{ data }"><UInput v-model="data.item" placeholder="Item" /></template>
    <template #step-ship="{ data }">
      <p id="shown">Address: {{ data.address || '(none)' }}</p>
      <button id="pick" @click="address.start()">Choose address</button>
    </template>
  </DataWizard>

  <DataWizard ref="address" :steps="addressSteps" :initial-data="() => ({ street: '' })" :on-complete="saveAddress" title="Address">
    <template #step-street="{ data }"><UInput v-model="data.street" placeholder="Street" /></template>
    <template #step-check="{ data }"><p>{{ data.street }}</p></template>
  </DataWizard>
</template>
```

```text
order: مرحله 2 از 2 | trail: Item > Shipping* | shows: Address: (none) Choose address | buttons: ['قبلی', 'تأیید نهایی']
modals open: 2
modals open: 1 | order: مرحله 2 از 2 | trail: Item > Shipping* | shows: Address: Main St Choose address | buttons: ['قبلی', 'تأیید نهایی']
```

The button `Choose address` starts the second wizard from inside the first one's step, and both dialogs stay open at the same time. `saveAddress` runs when the child finishes; it writes into `order.value.data`, and the step of the parent shows the new value at once. If the child is canceled, the parent's data is not touched, so a half-finished address never leaks into the order.

The pattern generalizes: a child wizard is a function from "nothing" to "one value", and the parent step is only a button and a place to show the value.

## 9. A user-creation wizard

All the earlier chapters come together in one realistic page. A wizard creates a user: the first step checks the username against the server, the second checks the email, the last one confirms. The page lists the users and counts cancellations:

```vue
<script setup>
const wizard = ref(null)
const users = ref(['admin'])
const cancelled = ref(0)

const steps = [
  {
    id: 'account',
    title: 'Account',
    pathLabel: 'Account',
    validate: async (data, ctx) => {
      if (data.username.length < 3) return 'The username needs at least 3 letters.'
      await new Promise(r => setTimeout(r, 150))
      return users.value.includes(data.username) ? 'That username is taken.' : true
    }
  },
  {
    id: 'contact',
    title: 'Contact',
    pathLabel: 'Contact',
    validate: data => (data.email.includes('@') ? true : 'The email needs an @.')
  },
  { id: 'confirm', title: 'Check and create', pathLabel: 'Check' }
]

async function create(data, ctx) {
  await new Promise(r => setTimeout(r, 200))
  if (ctx.signal.aborted) return
  users.value.push(data.username)
}
</script>

<template>
  <button id="add" @click="wizard.start()">New user</button>
  <p id="users">Users: {{ users.join(', ') }} | cancelled: {{ cancelled }}</p>

  <DataWizard
    ref="wizard"
    :steps="steps"
    :initial-data="() => ({ username: '', email: '' })"
    :on-complete="create"
    title="New user"
    @cancel="cancelled++"
  >
    <template #step-account="{ data }"><UInput v-model="data.username" placeholder="Username" /></template>
    <template #step-contact="{ data }"><UInput v-model="data.email" placeholder="Email" /></template>
    <template #step-confirm="{ data }"><p>{{ data.username }} ({{ data.email }})</p></template>
  </DataWizard>
</template>
```

```text
short name: مرحله 1 از 3 | trail: Account* | shows: <input Username> | alert: The username needs at least 3 letters. | buttons: ['انصراف', 'بعدی']
taken name: مرحله 1 از 3 | trail: Account* | shows: <input Username> | alert: That username is taken. | buttons: ['انصراف', 'بعدی']
good name: مرحله 2 از 3 | trail: Account > Contact* | shows: <input Email> | buttons: ['قبلی', 'بعدی']
bad email: مرحله 2 از 3 | trail: Account > Contact* | shows: <input Email> | alert: The email needs an @. | buttons: ['قبلی', 'بعدی']
contact done: مرحله 3 از 3 | trail: Account > Contact > Check* | shows: sara (sara@example.com) | buttons: ['قبلی', 'تأیید نهایی']
Account in the trail: مرحله 1 از 3 | trail: Account* | shows: <input Username> | buttons: ['انصراف', 'بعدی']
create: closed | Users: admin, sara | cancelled: 0
second start and cancel: Users: admin, sara | cancelled: 1
checking on step 1: مرحله 1 از 3 | trail: Account* | shows: <input Username> | form disabled | buttons: ['لغو عملیات', 'بعدی (disabled) (loading)']
Back while checking on step 1: closed | Users: admin, sara | cancelled: 2
```

Read the output as a story of the earlier chapters. The checks go from cheap to costly (chapter 4): the length of the username is tested before the slow lookup. The trail is used to go back to the first step, and the data stays (chapter 3). `create` respects `ctx.signal` (chapter 5), so if the visitor presses Back during the save, the new user is not added. Success is silent: the new user appears in the list and the `cancel` counter does not change. Canceling the second start leaves the list as it was and counts one cancellation, and Back on step 1 during a check cancels the whole wizard, so the counter goes to 2.

To build your own wizard, follow the same order: write the steps and their slots, add a check to each step that can be wrong (cheap tests first), put the saving in `onComplete` with the signal and the key, and decide what a cancellation means for the page.

---

## Appendix A. Quick reference

### A.1 Properties

| Property | Type | Default | Meaning |
|----------|------|---------|---------|
| `steps` | Array, required | - | The step objects (A.2) |
| `initialData` | Function | `() => ({})` | Returns the starting data; must be plain data |
| `title` | String | `''` | Heading; empty means the title of the current step |
| `size` | String | `'md'` | `md`, `lg`, `xl` or `auto` |
| `fullscreen` | Boolean | `false` | Cover the screen |
| `onComplete` | Function | none | `async (data, ctx)`, run on the last step |

### A.2 The step object

| Field | Meaning |
|-------|---------|
| `id` | Required and unique; names the slot `step-<id>` |
| `title` | Heading and name in the step list |
| `description` | Line under the heading and in the step list |
| `pathLabel` | Name in the trail; defaults to `title` |
| `icon` | Icon in the step list |
| `validate` | `(data, ctx) => true or text`, may be async |

### A.3 Slots, events and methods

| Name | Meaning |
|------|---------|
| Slot `step-<id>` | Content of that step; gets `data`, `step`, `index` |
| Slot `default` | Content of any step without its own slot; same arguments |
| Event `cancel` | Closed by Cancel, Back on step 1, the X or `cancel()` |
| `start(override)` | Open at step 1; data is `initialData()` with `override` laid over it |
| `next()`, `back()`, `goTo(index)`, `cancel()` | Control the wizard |
| `data` | The shared data object |
| `open`, `busy`, `error` | Whether it is open; whether a check or the action is running; the message shown |
| `currentIndex`, `currentStep`, `isFirst`, `isLast` | Where the wizard is |

### A.4 The ctx argument

| Field | Meaning |
|-------|---------|
| `signal` | An `AbortSignal`, aborted when the operation is abandoned |
| `generation` | The number of the current session; grows with every `start()` |
| `idempotencyKey` | A new random text for each attempt |

## Appendix B. Failure handling

| Situation | What the visitor sees | What happens |
|-----------|-----------------------|--------------|
| `validate` returns a text | The text in a red box | Stays on the step |
| `validate` returns anything but `true` or text | "اطلاعات این مرحله معتبر نیست." | Stays on the step |
| `validate` throws | The error's message, or "انجام عملیات ناموفق بود. دوباره تلاش کنید." | Stays on the step |
| `goTo` meets a failing step | Its message, or "لطفاً ابتدا این مرحله را کامل کنید." | Stops on that step |
| `onComplete` throws | The error's message, or the general text above | Stays on the last step; data intact |
| Back during a check or the action | Nothing | The operation is abandoned; one step back (on step 1 the wizard is canceled) |
| `start()` while open | Step 1 | The running operation is abandoned; data reset |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **A check that returns `false` or nothing.** Only `true` passes. A missing `return` at the end of a function that returns a text on failure blocks the wizard for everyone.
- **Passing reactive data to `start()`.** It throws `DataCloneError` and the wizard does not open. Pass a copy.
- **Functions in `initialData`.** They cannot be copied; keep the data plain and keep functions in the page.
- **Back does not stop the server.** A request already sent may still be processed. Use `ctx.signal` and send `ctx.idempotencyKey`.
- **Slow checks that ignore `signal`.** They keep running after Back. Their result is dropped, but their work is not undone.
- **A missing slot.** A step without its own slot shows the `default` slot, or nothing at all. A mistyped `id` is the usual cause.
- **Repeated ids.** Ids name slots and mark steps in the trail; two steps with the same id confuse both.
- **Data stays after closing.** The shared data remains in memory until the next `start()`. Clear sensitive fields in `onComplete` or on `cancel`.
- **Expecting a "done" event.** There is none; use `onComplete`.
- **Texts in other languages.** The buttons and messages of the wizard are Persian and cannot be changed by a property; your own texts (titles, validation messages) are free.

### C.2 Behavior worth knowing

- The form is inside a `fieldset` that is disabled while busy, so the inputs of the step cannot be edited during a check. Any other button inside the step is disabled as well.
- A check runs on a copy of the data, so a long check is not confused by edits (the form is locked anyway).
- Going back from the last step clears the message; going forward does too.
- The line "مرحله n از m" is announced to screen readers as it changes.
- Pressing Next while busy does nothing.
- The stepper is hidden from screen readers; the trail and the buttons are the controls.
- Opening a second wizard does not close the first.

### C.3 Limits

| What | Limit |
|------|-------|
| Number of steps | No limit; the trail wraps onto more lines |
| Dialog widths | `md` 2xl, `lg` 4xl, `xl` and `auto` up to 90 percent of the screen |
| Closing | Cancel, Back on step 1, the X, `cancel()`; not by clicking outside or Escape |
| `goTo` forward | Runs the checks one after another |
| Data | Must survive `structuredClone` |

### C.4 Tuning

- Put the cheap, local checks before the slow, remote ones in the same `validate`.
- Keep a step small: one idea, a few fields.
- Give long step names a short `pathLabel`.
- Choose `lg` or `xl` for steps that hold tables.
- Use `fullscreen` on phones for wizards with many fields.

## Appendix D. Message reference

### D.1 Texts on the screen

| Text | Meaning | Where |
|------|---------|-------|
| مرحله n از m | Step n of m | Under the step list |
| انصراف | Cancel | Left button on step 1 |
| قبلی | Previous | Left button on later steps |
| لغو عملیات | Cancel operation | Left button while busy |
| بعدی | Next | Right button |
| تأیید نهایی | Final confirmation | Right button on the last step |

### D.2 Error messages

| Message | When |
|---------|------|
| اطلاعات این مرحله معتبر نیست. | `validate` returned something other than `true` or a text |
| لطفاً ابتدا این مرحله را کامل کنید. | `goTo` stops on a step whose check failed without a text |
| خطایی در اعتبارسنجی رخ داد. | A check run by `goTo` threw an error without a message |
| انجام عملیات ناموفق بود. دوباره تلاش کنید. | A check or `onComplete` threw an error without a message |
