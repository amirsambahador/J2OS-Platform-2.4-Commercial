# useDynamicForm

*J2OS Platform 2.4*

Invoice lines, people to invite, phone numbers, ingredients: a lot of forms ask for "as many of these as needed". The visitor starts with one line and presses a plus button for the next, a minus button removes one. Writing this by hand means an array of lines, a way to give each line a stable identity so that Vue does not mix up the inputs, a rule that the last line cannot be removed, and code to start over. It is not hard, but it is the same code every time.

**useDynamicForm** is a composable of the Nuxt front end of J2OS that keeps those lines for the page. It does not draw anything; the page draws the inputs. It has three properties:

- **A list of lines.** Each line is a reactive object that the page binds its inputs to, with an `id` the composable generates.
- **The usual operations.** Show the form, add a line, remove a line, clear everything, close.
- **The values, ready to send.** `getAllValues()` returns the lines without their ids.

In a few lines:

```vue
<script setup>
// the function says what one empty line looks like
const people = useDynamicForm(() => ({ name: '', email: '' }))
</script>

<template>
  <button @click="people.showForm">Add people</button>
  <div v-if="people.isVisible.value">
    <div v-for="row in people.rows.value" :key="row.id">
      <input v-model="row.name">
      <input v-model="row.email">
      <button @click="people.addRow">+</button>
      <button @click="people.removeRow(row.id)">-</button>
    </div>
  </div>
</template>
```

This guide starts with how the composable works (chapter 1) and a first form (2). Chapter 3 looks at the lines and their ids, chapter 4 at fields that depend on each other, and chapter 5 at checking before sending. Chapter 6 collects the mistakes to avoid, and chapter 7 builds an invoice. The appendices work as a reference.

## Contents

1. [How useDynamicForm works](#1-how-usedynamicform-works)
2. [The first form](#2-the-first-form)
3. [Lines, ids and keys](#3-lines-ids-and-keys)
4. [Fields that depend on each other](#4-fields-that-depend-on-each-other)
5. [Checking before sending](#5-checking-before-sending)
6. [Things to watch](#6-things-to-watch)
7. [An invoice](#7-an-invoice)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are complete pages of a Nuxt project: each is one `.vue` file. In Nuxt `ref`, `computed`, `onMounted` and the composables need no `import`, so none is written. The inputs are plain HTML elements; the same code works with the components of a UI library. The output under a page is what the visitor sees on it as the page is used.

---

## 1. How useDynamicForm works

First, a few terms, because they come up throughout the guide:

- **Line.** One set of fields, for example one invoice item. In the code it is an object in the `rows` list.
- **Empty line function.** The function passed to `useDynamicForm`. It returns a new object with the fields of one empty line, with their starting values.
- **Id.** A text such as `row-1` that the composable adds to every line. Vue needs it as the `key` of each line.
- **Visible.** Whether the form is shown. The page decides what that means; usually a `v-if`.

The members of this composable:

| Member | Role |
|--------|------|
| `useDynamicForm(createEmptyRow)` | Creates the form; takes the empty line function |
| `rows` | A ref with the list of lines |
| `isVisible` | A ref, `false` at first |
| `showForm()` | Starts again with one empty line and makes the form visible |
| `addRow()` | Adds one empty line at the end |
| `removeRow(id)` | Removes the line with that id, unless it is the last one |
| `resetForm()` | Starts again with one empty line |
| `closeForm()` | Hides the form and throws all lines away |
| `getAllValues()` | The lines as plain objects, without ids |

Keep these five points in mind:

- **The composable draws nothing.** The page loops over `rows` and binds `v-model` to the fields of each line.
- **`rows` and `isVisible` are refs,** so in the template of a page that keeps the whole object they are written `form.rows.value` and `form.isVisible.value`.
- **Lines are created by calling your function.** Every new line gets a fresh call.
- **A form always has at least one line** once it is shown; the last line cannot be removed.
- **Nothing is checked or sent by the composable.** Validation and the server call belong to the page (chapter 5).

## 2. The first form

A team page lets a manager invite several people: a name and an email for each. A button opens the form, plus adds a line, minus removes one, Send sends the invitations and closes the form, Clear starts over, Close throws everything away.

```vue
<script setup>
const invite = useDynamicForm(() => ({ name: '', email: '' }))
const message = ref('Nobody invited yet.')

function send() {
  const people = invite.getAllValues()
  message.value = `Invitations sent to ${people.map(p => p.name).join(', ')}.`
  invite.closeForm()
}
</script>

<template>
  <button id="open" @click="invite.showForm">Invite people</button>
  <div v-if="invite.isVisible.value" id="form">
    <div v-for="row in invite.rows.value" :key="row.id" class="line" :data-id="row.id">
      <input v-model="row.name" class="name" placeholder="Name">
      <input v-model="row.email" class="email" placeholder="Email">
      <button class="add" @click="invite.addRow">+</button>
      <button class="remove" @click="invite.removeRow(row.id)">-</button>
    </div>
    <button id="send" @click="send">Send</button>
    <button id="clear" @click="invite.resetForm">Clear</button>
    <button id="close" @click="invite.closeForm">Close</button>
  </div>
  <p id="message">{{ message }}</p>
</template>
```

```text
Page opens:                                                      Nobody invited yet. Form visible: False.
Visitor presses Invite people:                                   1 line(s), row-1.
Visitor presses + twice:                                         3 line(s), row-1,row-2,row-3.
Visitor removes the second line:                                 2 line(s), row-1,row-3; first line still reads Ann, second now reads Cy.
Visitor presses + again:                                         3 line(s), row-1,row-3,row-4.
Visitor removes lines until one is left and presses - once more: 1 line(s), row-1.
Visitor presses Clear:                                           1 line(s), row-1, first line reads "".
Visitor types Eve and presses Close:                             Form visible: False.
Visitor presses Invite people again:                             1 line(s), first line reads "".
Visitor fills two lines and presses Send:                        Invitations sent to Ann, Bob. Form visible: False.
```

The empty line function, `() => ({ name: '', email: '' })`, says what one line looks like. The page loops over `invite.rows.value`, using `row.id` as the key, and binds each input to a field of `row`. The buttons are the composable's functions: `showForm`, `addRow`, `removeRow(row.id)`, `resetForm` and `closeForm` can all be used directly as click handlers.

Follow the output. The form did not exist until the visitor pressed the first button, and then it had one line. Two presses of plus gave three lines, and removing the second one left the first and the third: the names stayed with their lines, because the ids, and not the positions, identify them. The last line could not be removed. Clear and Close both threw away what was typed, and opening the form again started from a single empty line. Send read the values with `getAllValues()`, used them, and closed the form.

## 3. Lines, ids and keys

Ids are the part that surprises people, so here are the rules, observed on a real form:

| Action | Result |
|--------|--------|
| `showForm()` | One line, `row-1`; ids start again |
| `addRow()` | The next number: `row-2`, `row-3`, ... |
| `removeRow('row-2')` | The line goes; the other ids stay as they are |
| `addRow()` after removing | A new number, never one that was used (`row-4` after `row-1`, `row-3`) |
| `removeRow` on the last line | Nothing happens |
| `removeRow` with an unknown id | Nothing happens |
| `resetForm()` | One empty line, `row-1`; ids start again |
| `closeForm()` | `rows` is empty and `isVisible` is `false` |
| `showForm()` while the form is open | Starts again; what was typed is lost |
| `addRow()` while the form is closed | The line is added to `rows` but nothing is shown; `showForm()` clears it again |

Two consequences. First, use `row.id` as the `key`, and do not use the position in the list: after a removal the positions change, and Vue would keep the old inputs for the wrong lines. Second, an id says nothing about order. If the page needs "line number 3", compute it from the position in the list when drawing.

## 4. Fields that depend on each other

A very common need is a field whose choices depend on another field of the same line: a country and its cities. Each line has its own pair. When the country changes, the city chosen earlier may not belong to the new country, so it must be reset.

```vue
<script setup>
const citiesByCountry = {
  iran: ['Tehran', 'Mashhad', 'Isfahan'],
  turkey: ['Istanbul', 'Ankara']
}
const cityOptionsFor = country => citiesByCountry[country] || []

const contacts = useDynamicForm(() => ({ name: '', country: '', city: '' }))
onMounted(() => contacts.showForm())
</script>

<template>
  <div v-for="row in contacts.rows.value" :key="row.id" class="line">
    <input v-model="row.name" class="name" placeholder="Name">
    <select v-model="row.country" class="country" @change="row.city = ''">
      <option value="">Country</option>
      <option value="iran">Iran</option>
      <option value="turkey">Turkey</option>
    </select>
    <select v-model="row.city" class="city" :disabled="!row.country">
      <option value="">City</option>
      <option v-for="city in cityOptionsFor(row.country)" :key="city" :value="city">{{ city }}</option>
    </select>
    <button class="add" @click="contacts.addRow">+</button>
    <button class="remove" @click="contacts.removeRow(row.id)">-</button>
  </div>
  <p id="summary">{{ contacts.getAllValues().map(c => `${c.name || '?'}: ${c.city ? c.city + ', ' + c.country : 'no city yet'}`).join(' | ') }}</p>
</template>
```

```text
Page opens:                            city list is disabled, options: none.
Visitor picks Iran:                    city list is enabled, options: Tehran, Mashhad, Isfahan.
Visitor picks Mashhad:                 Ann: Mashhad, iran
Visitor changes the country to Turkey: city list now offers Istanbul, Ankara; Ann: no city yet
Visitor adds Bob in Iran:              first line offers Istanbul, Ankara, second line offers Tehran, Mashhad, Isfahan.
Both lines together:                   Ann: no city yet | Bob: Tehran, iran
```

The important part is small:

```vue
<select v-model="row.country" @change="row.city = ''">   <!-- a new country invalidates the old city -->
  <option value="iran">Iran</option>
  <option value="turkey">Turkey</option>
</select>
<select v-model="row.city" :disabled="!row.country">
  <option v-for="city in cityOptionsFor(row.country)" :key="city" :value="city">{{ city }}</option>
</select>
```

The city list is built from the country of that line (`cityOptionsFor(row.country)`), and it is disabled until a country is chosen. The `@change` on the country clears the city, which is what the output shows: after the country changed from Iran to Turkey, Ann's city was gone and the list offered Istanbul and Ankara. Bob's line, with Iran, kept offering Iranian cities. Because each line is its own object, lines never interfere with each other.

If the select is a component of a UI library, the idea is the same, but check what the component passes to `update:model-value`. Some versions pass the option object rather than the value, and the page has to take the value out before storing it.

## 5. Checking before sending

`getAllValues()` returns what is in the lines and nothing more. It does not know which fields are required, which lines are empty, or what the server accepts. The page does that, and usually in three steps: drop the lines the visitor left empty, check the rest, then send.

```js
function submit() {
  // ignore lines the visitor left empty, then check the rest
  const lines = invoice.getAllValues().filter(l => l.item.trim() !== '')
  const bad = lines.filter(l => !(Number(l.price) > 0) || !(Number(l.quantity) >= 1))
  if (lines.length === 0) message.value = 'Add at least one item.'
  else if (bad.length) message.value = `Fix ${bad.map(l => l.item).join(', ')} before sending.`
  else send(lines)
}
```

A few rules for this code:

- **Ignore empty lines instead of refusing them.** A visitor who pressed plus once too often should not be blocked.
- **Name the line that is wrong,** by its item, name or number, not just "an error".
- **Convert types yourself.** Fields bound to `type="number"` inputs with `v-model.number` stay numbers; otherwise they are text.
- **Treat the returned list as a copy.** Its objects are shallow copies of the lines, so adding or removing items of the list, or changing a top-level field, does not change the lines on the screen. Nested objects and arrays are the same ones the lines hold.

The values come out as they went in: numbers stay numbers and text stays text.

## 6. Things to watch

Three mistakes come from how the empty line function is written, and they are all about sharing something between lines:

```js
// wrong: every line shares the same nested array
const shared = { tags: [] }
useDynamicForm(() => shared)

// right: build a fresh object for each line
useDynamicForm(() => ({ tags: [] }))

// wrong: an id of your own replaces the generated one
useDynamicForm(() => ({ id: 7, name: '' }))
```

| Mistake | What happens |
|---------|--------------|
| The function returns one shared object that has an array or object inside | Lines get their own top-level fields, but the nested array is the same one in every line: a tag added to one line appears in all of them |
| The function returns a field called `id` | It replaces the generated id. Lines may then have equal ids and equal keys, and `removeRow` with that id removes all of them |
| No function is passed | The form is created, but `showForm()` fails with `createEmptyRow is not a function` |
| `removeRow` called with the line instead of its id | Nothing is removed; pass `row.id` |
| `showForm()` called to "show again" | It also resets; use `isVisible` yourself if you only want to hide and show |

Other points follow from the design. Typing in the inputs changes the lines directly, so there is no "cancel" that restores the earlier values: `closeForm` and `resetForm` drop them, and anything finer is up to the page. And `isVisible` has no effect on its own: the page must use it, normally in a `v-if`.

## 7. An invoice

Everything so far goes into one case that appears in almost every business application: an invoice with a list of items. Each line has an item, a quantity and a price. The page shows the total as the visitor types, ignores empty lines when sending, and names the line that is wrong. Start over returns to one empty line.

```vue
<script setup>
const invoice = useDynamicForm(() => ({ item: '', quantity: 1, price: 0 }))
const message = ref('')

onMounted(() => invoice.showForm())

const total = computed(() =>
  invoice.rows.value.reduce((sum, r) => sum + Number(r.quantity) * Number(r.price), 0)
)

function submit() {
  // empty lines are ignored; a line with an item but no price is a mistake
  const lines = invoice.getAllValues().filter(l => l.item.trim() !== '')
  const bad = lines.filter(l => !(Number(l.price) > 0) || !(Number(l.quantity) >= 1))
  if (lines.length === 0) {
    message.value = 'Add at least one item.'
  } else if (bad.length) {
    message.value = `Fix ${bad.map(l => l.item).join(', ')} before sending.`
  } else {
    message.value = `Invoice with ${lines.length} line(s), total ${total.value}, sent.`
  }
}
</script>

<template>
  <div v-for="row in invoice.rows.value" :key="row.id" class="line">
    <input v-model="row.item" class="item" placeholder="Item">
    <input v-model.number="row.quantity" class="quantity" type="number" min="1">
    <input v-model.number="row.price" class="price" type="number" min="0">
    <button class="add" @click="invoice.addRow">+</button>
    <button class="remove" @click="invoice.removeRow(row.id)">-</button>
  </div>
  <p id="total">Total: {{ total }}</p>
  <button id="submit" @click="submit">Send invoice</button>
  <button id="reset" @click="invoice.resetForm">Start over</button>
  <p id="message">{{ message }}</p>
</template>
```

```text
Page opens:                                                               1 empty line, Total: 0.
Visitor presses Send invoice with nothing filled in:                      Add at least one item.
Visitor fills Paper and Pens (no price yet) and adds an empty third line: Total: 120.
Visitor presses Send invoice:                                             Fix Pens before sending.
Visitor gives Pens a price:                                               Total: 150.
Visitor presses Send invoice:                                             Invoice with 2 line(s), total 150, sent.
Visitor presses Start over:                                               1 empty line, Total: 0.
```

The form is shown as soon as the page opens (`showForm()` in `onMounted`), because an invoice always needs at least one line. The total is a `computed` that goes over `invoice.rows.value`, so it updates with every keystroke, and for the same reason it is wrong if a field holds text instead of a number: `v-model.number` keeps the fields numeric.

Read the output as a conversation. Sending an empty invoice produced a request to add an item. After Paper and Pens were typed (and an empty third line was added by mistake), the total was already 120, and sending named Pens, whose price was still zero; the empty line was ignored. After Pens got a price, the total became 150 and the invoice was sent with two lines. Start over brought back a single empty line and a total of zero.

For your own forms the recipe is the same: describe one empty line, loop over `rows` with `row.id` as the key, bind the fields, use the functions as click handlers, compute totals and checks from the lines, and send what `getAllValues()` returns after checking it.

---

## Appendix A. Quick reference

### A.1 Members

| Call | Returns | Notes |
|------|---------|-------|
| `useDynamicForm(createEmptyRow)` | The form | The function returns a fresh object for one empty line |
| `rows` | Ref | The lines, each with an `id` and the fields |
| `isVisible` | Ref | `false` at first; `showForm` sets it, `closeForm` clears it |
| `showForm()` | | One empty line, visible; loses anything typed |
| `addRow()` | | Adds an empty line at the end |
| `removeRow(id)` | | Removes the line; ignored for the last line and unknown ids |
| `resetForm()` | | One empty line |
| `closeForm()` | | Hidden, no lines |
| `getAllValues()` | Array | Shallow copies of the lines without `id` |

### A.2 Ids

| Event | Id of the lines afterwards |
|-------|----------------------------|
| `showForm()`, `resetForm()` | `row-1` |
| `addRow()` | The next number after the highest ever used since the last `showForm` or `resetForm` |
| `removeRow(id)` | The others unchanged |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| No empty line function | `showForm()` fails with `createEmptyRow is not a function` |
| `removeRow` on the only line | Ignored |
| `removeRow` with an unknown id | Ignored |
| The empty line function returns an `id` | Replaces the generated id; duplicates are possible |
| Shared nested objects in the function | Shared by all lines |
| `getAllValues()` before `showForm()` | An empty list |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Using the position as the `key`.** Inputs get attached to the wrong lines after a removal.
- **Returning a shared object from the empty line function.** Nested values are shared.
- **Putting `id` in the empty line.** It replaces the generated one.
- **Forgetting `.value`.** `form.rows` is a ref; in the template of a page that keeps the whole object write `form.rows.value`.
- **Calling `showForm` to show again.** It starts again and loses the typed values.
- **Expecting validation.** The composable does none.
- **Expecting text from number inputs to be numbers.** Use `v-model.number`.
- **Not ignoring empty lines.** Visitors press plus too often.
- **Passing the line to `removeRow`.** It wants the id.

### C.2 Behaviour worth knowing

- Every new line comes from a new call to your function.
- Line ids are `row-` followed by a counter that restarts at `showForm` and `resetForm`.
- `resetForm` does not change `isVisible`.
- `closeForm` empties `rows`, so the values are lost.
- `addRow` works even when the form is closed, but the line is cleared by the next `showForm`.
- `getAllValues` returns shallow copies: top-level changes do not reach the form, nested objects are shared.

### C.3 Limits

| What | Limit |
|------|-------|
| Minimum lines once shown | One |
| Maximum lines | None; set one yourself if needed |
| Validation | None |
| Sending to a server | None |
| Undo of a removed line | None |
| Ordering of lines | Only by adding; no moving |

### C.4 Tuning

- Give fields sensible starting values in the empty line function (a quantity of 1, an empty text).
- Use `row.id` as the key everywhere.
- Compute totals and checks from `rows`, and keep them as `computed` values.
- Ignore empty lines when sending, and name the incorrect ones.
- Cap the number of lines in the page if the server limits them: hide the plus button when `rows.value.length` reaches the limit.

## Appendix D. Message reference

### D.1 Errors

| Message | Cause |
|---------|-------|
| `createEmptyRow is not a function` | `useDynamicForm` was called without an empty line function and the form was shown or a line added |

### D.2 Values worth recognising

| Value | Meaning |
|-------|---------|
| `row-1` | The first line after `showForm` or `resetForm` |
| `[]` from `getAllValues()` | No lines: the form was closed or never shown |
