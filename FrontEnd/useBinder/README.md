# useBinder

*J2OS Platform 2.4*

Many of the tools of a Nuxt page want an **element id**: a number box that formats the digits as they are typed, a rich text editor, a code editor, a date picker. They look the element up with `document.getElementById`. But a page built from Nuxt UI components rarely has ids. A `UInput` is a component, not an input: you get a ref to the component, the real `<input>` sits inside it, and it may not exist yet, because the step of the form that holds it has not been opened.

**useBinder** is a composable of the Nuxt front end of J2OS that bridges this gap. It waits until the field is on the page, makes sure its input has an id, and then calls your function with that id. It has three main features:

- **From a ref to an id.** Give it the ref of a component or an element; it finds the input inside and returns its id to your function.
- **It follows the field.** If the field appears later or disappears and appears again, the function is called again each time.
- **It cleans up.** If your function returns another function, that one is called when the field goes away or the page is left.

What it looks like in use:

```vue
<script setup>
const binder = useBinder()
const price = ref()                       // the UInput

binder.bindId(price, (elementId) => {
  // the input inside UInput now has an id, and it is on the page
  return someLibrary.attach(elementId)    // optional: a function that undoes it
})
</script>

<template>
  <UInput ref="price" />
</template>
```

First comes how the composable works (chapter 1) and a first binding (2). Chapter 3 is about fields that come and go, chapter 4 about which element is chosen and which id it gets, and chapter 5 about cleaning up. Chapter 6 lists the cases where nothing happens, and chapter 7 builds an expense form that uses everything. The appendices are meant for quick lookups.

## Contents

1. [How useBinder works](#1-how-usebinder-works)
2. [A first binding](#2-a-first-binding)
3. [Fields that come and go](#3-fields-that-come-and-go)
4. [Which element, and which id](#4-which-element-and-which-id)
5. [Cleaning up](#5-cleaning-up)
6. [When nothing happens](#6-when-nothing-happens)
7. [An expense form](#7-an-expense-form)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Each example is a page you can drop into a Nuxt project, and each is a single `.vue` file. In Nuxt `ref`, `watch` and the composables need no `import`, so none is written. The output under each page is what the visitor sees while using it. The number box used in the examples is `useNumber` (it has its own guide); here it is only a tool that asks for an id.

---

## 1. How useBinder works

Terms used throughout:

- **Element id.** The `id` attribute of an HTML element. Libraries that work on an input find it by this id.
- **Component ref.** The ref you get on a component such as `UInput`: not the input, but the component, whose root element is reached as `$el`.
- **Binding.** Connecting a tool to a field: the function you give to `bindId`, called with the field's id.
- **Cleanup.** A function returned by the binding, which undoes it.

The function of this composable:

```js
binder.bindId(elementRef, callback, { selector })

// elementRef : a ref to a component (UInput ...) or to an HTML element
// callback   : (elementId) => nothing, or a function that cleans up, or a promise of either
// selector   : which element inside the component gets the id (default: input:not([type="hidden"]))
```

| Part | Role |
|------|------|
| `elementRef` | A ref to a component or an HTML element; `bindId` watches it |
| `callback` | Called with the id of the chosen element; may return a cleanup function, or a promise of one |
| `selector` | Which element inside the component is chosen; the default is the first input that is not hidden |

Keep these five points in mind:

- **It runs when the field is really there.** `bindId` waits for the page to be updated, so `document.getElementById(id)` finds the element when your function runs.
- **It runs every time the ref changes.** The first time the field appears, and again each time a new field takes its place.
- **The id is only added when needed.** An existing, unique id is kept; otherwise a new one starting with `x-` is made.
- **Cleanup is automatic.** The previous cleanup runs before a new binding, and when the page that called `bindId` is left.
- **Call it while the page is being set up**, at the top of `<script setup>` or inside `onMounted`, the way you call any composable.

## 2. A first binding

A payroll form has a salary field. The number box tool formats digits as they are typed and keeps the plain value ready for the server. It needs an id, and the field is a `UInput`.

```vue
<script setup>
const jnumber = useNumber()
const binder = useBinder()

const salary = ref()          // the UInput component
const raw = ref('')

binder.bindId(salary, (elementId) => {
  const box = document.getElementById(elementId)
  if (!box.value) box.value = '14000'                 // a starting value
  return jnumber.setupNumberBox(elementId, 12, (value) => { raw.value = value })
})
</script>

<template>
  <label>Monthly salary</label>
  <UInput ref="salary" />
  <p id="raw">Value for the server: {{ raw || 'nothing yet' }}</p>
</template>
```

```text
Page opens:                   the box shows 14,000; Value for the server: 14000.
Visitor types 1500000:        the box shows 1,500,000; Value for the server: 1500000.
Visitor types abc12x instead: the box shows 12; Value for the server: 12.
```

The page has a ref, `salary`, on the `UInput`, and one call: `binder.bindId(salary, ...)`. When the field is on the page, the function receives its id. The page puts a starting value in the box, and hands the id to `setupNumberBox`, which returns its own cleanup function. The page returns it, and that is all `useBinder` needs.

Read the output: the box shows `14,000` as soon as the page opens, typing `1500000` gives `1,500,000`, and the value for the server stays plain digits. Typing letters leaves only the digits. None of this needed the page to know the id.

## 3. Fields that come and go

The strength of `useBinder` shows when a field is not always there. A payment wizard asks who the payment is for on step 1 and the amount on step 2. The amount field exists only on step 2.

```vue
<script setup>
const jnumber = useNumber()
const binder = useBinder()

const step = ref(1)
const amount = ref()
const setups = ref(0)
const stops = ref(0)

binder.bindId(amount, (elementId) => {
  setups.value++
  const stop = jnumber.setupNumberBox(elementId, 12, () => {})
  return () => { stop(); stops.value++ }
})
</script>

<template>
  <div v-if="step === 1">
    <p>Step 1: who is the payment for?</p>
    <button id="next" @click="step = 2">Next</button>
  </div>
  <div v-else>
    <p>Step 2: how much?</p>
    <UInput ref="amount" />
    <button id="back" @click="step = 1">Back</button>
  </div>
  <p id="status">Number formatting set up {{ setups }} time(s), switched off {{ stops }} time(s).</p>
</template>
```

```text
Page opens on step 1:                       Number formatting set up 0 time(s), switched off 0 time(s).
Visitor goes to step 2 and types 12345:     the box shows 12,345. Number formatting set up 1 time(s), switched off 0 time(s).
Visitor goes back to step 1:                Number formatting set up 1 time(s), switched off 1 time(s).
Visitor returns to step 2 and types 987654: the box shows 987,654. Number formatting set up 2 time(s), switched off 1 time(s).
```

The binding is written once. Nothing happens on step 1, because the ref is empty. When the visitor reaches step 2 the field appears, the ref fills, and the function runs: "set up 1 time". When the visitor goes back, the field is removed, the cleanup runs: "switched off 1 time". On the second visit a new field is built, and the whole sequence runs again: "set up 2 times", and the new box formats digits as well.

Compare with the usual approach of calling the tool in `onMounted` with a fixed id: if the field is not on the page at that moment, the call finds nothing and the box stays plain for good. `useBinder` removes the question of timing.

## 4. Which element, and which id

The default is the first input that is not hidden. `bindId` looks at the root of the component first, then inside it:

| What the ref points at | Chosen element |
|------------------------|----------------|
| An `input` (the root itself) | That input |
| A wrapper with inputs inside (like `UInput`) | The first input inside that is not `type="hidden"` |
| An HTML element from `<input ref="x">` | That element |
| A component with no matching element inside | Nothing; the callback is not called |

For anything else, give a `selector`; it is an ordinary CSS selector:

```js
binder.bindId(note, attach, { selector: 'textarea' })           // a textarea
binder.bindId(pick, attach, { selector: 'select' })             // a select
binder.bindId(card, attach, { selector: '.editor [contenteditable]' })
```

The id follows three rules:

- **An unused id is kept.** If the element already has an `id` and no other element on the page has it, nothing changes.
- **A missing id is made.** The element gets `x-` followed by a time stamp and a random part.
- **A repeated id is replaced.** If another element has the same id, the one being bound gets a new id and the other is untouched.

A component with **two root elements** is the one case to avoid. Its ref does not point at a single element, so `bindId` looks in the parent of the component and takes the first match it finds there, which may be another field.

```vue
<!-- a component with two root elements: bindId cannot know which one you mean -->
<template>
  <label>Amount</label>
  <input>
</template>

<!-- give it one root and the ref points at it -->
<template>
  <div>
    <label>Amount</label>
    <input>
  </div>
</template>
```

## 5. Cleaning up

A binding often adds something: an event listener, a number box, an editor. When the field goes, it has to be taken off. The function you give to `bindId` can return a function that does this, and `bindId` calls it at the right time:

| When | What happens |
|------|--------------|
| The field is removed (the ref becomes empty) | The cleanup runs |
| A different field replaces it | The old cleanup runs, then the function runs for the new field |
| The page that called `bindId` is left | The cleanup runs |
| The function returned nothing | There is nothing to run |

A note field shows it: the page counts its listeners.

```vue
<script setup>
const binder = useBinder()
const note = ref()
const left = ref(100)
const shown = ref(true)
const active = ref(0)

binder.bindId(note, (elementId) => {
  const box = document.getElementById(elementId)
  const update = () => { left.value = 100 - box.value.length }
  box.addEventListener('input', update)
  update()
  active.value++
  return () => {                                  // runs when the field goes away
    box.removeEventListener('input', update)
    active.value--
  }
}, { selector: 'textarea' })
</script>

<template>
  <div v-if="shown">
    <UTextarea ref="note" placeholder="A note for the driver" />
    <p id="left">{{ left }} characters left</p>
  </div>
  <p id="active">Active listeners: {{ active }}</p>
  <button id="toggle" @click="shown = !shown">{{ shown ? 'Remove the note' : 'Add a note' }}</button>
</template>
```

```text
Page opens:                           100 characters left. Active listeners: 1.
Visitor types "Leave it at the door": 80 characters left.
Visitor presses Remove the note:      Active listeners: 0.
Visitor presses Add a note:           100 characters left. Active listeners: 1.
Visitor types "Ring twice":           90 characters left.
```

Here the field is a `textarea` inside the component, so the page passes `selector: 'textarea'`. The function adds an `input` listener and returns a function that removes it. The count of active listeners goes `1`, `0`, `1` as the note is removed and added; it never grows, so nothing is left behind. The new note starts with all 100 characters free because it is a new field.

The function can be `async`. If the field leaves while it is still working, its result is cleaned up as soon as it arrives:

```js
binder.bindId(editorBox, async (elementId) => {
  const editor = await createEditor(elementId)     // takes a while
  return () => editor.destroy()                    // still cleaned up if the field left meanwhile
})
```

## 6. When nothing happens

Most of the following cases are silent, so it is useful to know them.

| Situation | Result |
|-----------|--------|
| The ref is still empty | Nothing; the callback is not called |
| The ref is an array (a ref in a `v-for`) | Nothing; the callback is not called |
| The component has no element that matches the selector | Nothing |
| The element is not on the page any more when the page has updated | Nothing |
| The component has several roots | The first match in the parent is used, which may be another field |
| The callback throws | The error is not caught; it appears as an unhandled error in the console |
| The selector is not valid CSS | The same kind of error |
| The page re-renders but the same element stays | Nothing; the binding is not repeated |
| The element is replaced (for example by changing its `key`) | The cleanup runs and the function runs for the new element |

For a list of fields, call `bindId` once per field with a ref of its own, or move the field and its binding into a small component, as the next chapter does.

## 7. An expense form

All of the above in one screen. The fields are their own component, `ExpenseFields`: an amount box and a note with a character count. Each binds itself and reports to the page, which shows how many are connected and can close and reopen the form.

```vue
<script setup>
const emit = defineEmits(['bound', 'unbound', 'amount'])
const jnumber = useNumber()
const binder = useBinder()

const amount = ref()
const note = ref()
const left = ref(100)

binder.bindId(amount, (elementId) => {
  emit('bound')
  const stop = jnumber.setupNumberBox(elementId, 12, value => emit('amount', value))
  return () => { stop(); emit('unbound') }
})

binder.bindId(note, (elementId) => {
  const box = document.getElementById(elementId)
  const update = () => { left.value = 100 - box.value.length }
  box.addEventListener('input', update)
  emit('bound')
  return () => { box.removeEventListener('input', update); emit('unbound') }
}, { selector: 'textarea' })
</script>

<template>
  <label>Amount</label>
  <UInput ref="amount" />
  <label>What was it for?</label>
  <UTextarea ref="note" />
  <p id="left">{{ left }} characters left</p>
</template>
```

```vue
<script setup>
const open = ref(true)
const amount = ref('0')
const active = ref(0)
</script>

<template>
  <ExpenseFields
    v-if="open"
    @bound="active++"
    @unbound="active--"
    @amount="value => amount = value"
  />
  <p id="summary">Amount for the server: {{ amount }}. Fields connected: {{ active }}.</p>
  <button id="toggle" @click="open = !open">{{ open ? 'Close the form' : 'Open the form' }}</button>
</template>
```

```text
Page opens:                                   Amount for the server: 0. Fields connected: 2.
Visitor types 2500 and "Taxi to the airport": amount box shows 2,500, 81 characters left; Amount for the server: 2500. Fields connected: 2.
Visitor presses Close the form:               Amount for the server: 2500. Fields connected: 0.
Visitor presses Open the form:                Amount for the server: 0. Fields connected: 2.
```

The component never mentions an id. Each field has a ref and a `bindId`, and the tool in it gets the id when it needs it. The page closes the form and the page's count falls to `0`: both cleanups ran. Opening the form again builds new fields, binds them again, and the count is back at `2` with a fresh amount.

For your own screens the recipe is the same: put a ref on the component, call `bindId` with the ref and a function that sets up your tool, return the function that takes it down, pass a `selector` if the field is not an input, and keep each component to a single root element.

---

## Appendix A. Quick reference

### A.1 Call

| Call | Role |
|------|------|
| `bindId(elementRef, callback, options)` | Connects a tool to the field of the ref, now and each time the ref changes |

### A.2 Options and callback

| Part | Content |
|------|---------|
| `options.selector` | CSS selector of the element to use; default `input:not([type="hidden"])` |
| `callback(elementId)` | Receives the id of the chosen element |
| Return value | A cleanup function, or a promise of one; anything else is ignored |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| No element, no match | The callback is not called |
| Callback throws | Unhandled error in the console |
| Invalid selector | Unhandled error in the console |
| The field leaves while an `async` callback is still running | The result is cleaned up when it arrives |
| Several roots | The first match in the parent |
| Two elements with the same id | The bound element gets a new id |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Calling `bindId` from a click or a timer.** Vue warns that it has no page to attach to, and the binding is no longer stopped when the page is left. Call it in `<script setup>` or `onMounted`.
- **Skipping the cleanup.** Listeners and editors pile up.
- **Using a ref in `v-for`.** An array of elements is not understood.
- **A component with two roots.** The wrong field can be chosen.
- **Looking for the element with a hand-made id.** Use the id the callback receives.
- **Expecting the id to be stable between visits.** A new `x-` id is made for every new field.
- **Using the default selector for a `textarea` or a `select`.** Pass the `selector`.
- **Doing slow work and not returning its cleanup.** Return it, even from an `async` function.

### C.2 Behavior worth knowing

- The callback runs after the page has updated.
- It also runs for a field that is already there when `bindId` is called.
- The cleanup of the old field runs before the callback for the new one.
- Only the ref is watched, so changes inside a field do not repeat the binding.

### C.3 Limits

| What | Limit |
|------|-------|
| Fields per call | One |
| Elements chosen | The first match |
| Ids | Kept when unique, otherwise `x-...` |
| Ref in `v-for` | Not supported |

### C.4 Tuning

- Keep one root element per component.
- Put a field and its binding in a small component.
- Pass `selector` for anything that is not an input.
- Return the cleanup function every time.

## Appendix D. Message reference

### D.1 Errors

`useBinder` has no messages of its own. Errors thrown by your callback, or by an invalid selector, reach the console unchanged.

### D.2 Values worth recognizing

| Value | Meaning |
|-------|---------|
| `x-...` | An id made by `useBinder` |
| `input:not([type="hidden"])` | The default selector |
