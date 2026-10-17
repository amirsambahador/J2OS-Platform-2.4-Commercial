# useValidation

*J2OS Platform 2.4*

Every form needs the same checks: is the field filled, is it long enough, does it look like a mobile number or an email? If they are rewritten for every form, the code gets repeated and each copy words its messages a little differently.

**useValidation** is a composable of the Nuxt front end of J2OS that checks a form straight from its HTML. It has three properties:

- **The rules live in the form.** A field states its rules with the ordinary attributes `required`, `minlength` and `maxlength`, plus two of its own: `data-pattern` and `data-title`. No rule list is written in JavaScript.
- **Suited to Iranian forms.** It comes with ready-made patterns for mobile numbers, national codes, postal codes, dates, plates, Sheba and card numbers, together with common ones such as email, URL and password. It converts Persian digits to Latin ones when asked.
- **One list of messages.** All errors are collected in `validator.state.errors`, a reactive list that a template can show directly.

There are two versions. `useValidation` finds the form by its `id`. `useValidationRefBased` takes a template ref instead and is otherwise the same (chapter 8).

In a few lines:

```vue
<script setup>
const validator = useValidation()

function save() {
  if (validator.validateForm('signup')) send(validator.getFormData('signup'))
}
</script>

<template>
  <form id="signup">
    <UInput name="email" data-title="Email" data-pattern="email" required />
  </form>
  <p v-for="error in validator.state.errors" :key="error">{{ error }}</p>
  <UButton label="Save" @click="save" />
</template>
```

This guide goes step by step: chapter 1 explains how the composable works and chapter 2 shows the first check. Chapter 3 covers the rules, chapter 4 the ready-made patterns, chapter 5 Persian digits, and chapter 6 your own patterns and checks across fields. Chapter 7 covers checkboxes, radio buttons and selects, chapter 8 the ref-based version, and chapter 9 builds a complete form. The appendices are for quick reference.

## Contents

1. [How useValidation works](#1-how-usevalidation-works)
2. [The first check](#2-the-first-check)
3. [The rules](#3-the-rules)
4. [Ready-made patterns](#4-ready-made-patterns)
5. [Persian digits](#5-persian-digits)
6. [Your own patterns and checks across fields](#6-your-own-patterns-and-checks-across-fields)
7. [Checkboxes, radio buttons and selects](#7-checkboxes-radio-buttons-and-selects)
8. [The ref-based version](#8-the-ref-based-version)
9. [Putting it together](#9-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are complete pages of a Nuxt project: each is one `.vue` file. In Nuxt `ref`, `useValidation` and the component `UInput` need no `import`, so none is written. The ref-based version lives in a subfolder and is imported by hand, as the example in chapter 8 shows. The messages of the composable are Persian and fixed.

---

## 1. How useValidation works

First, a few terms, because they come up throughout the guide:

- **Field.** An `input`, `textarea` or `select` inside the form.
- **Rule.** A condition written on the field itself, as an attribute.
- **Pattern.** A regular expression with a name. The field writes that name in `data-pattern`.
- **Normalizing.** Trimming the value and converting Persian digits to Latin ones before it is checked.
- **Title.** The name of the field in the messages: `data-title`, or else its `name`, or else the word "فیلد" ("field").

The members of this composable:

| Member | Role |
|--------|------|
| `validateForm(form)` | Checks every field; returns `true` when there are no errors |
| `getFormData(form)` | Collects the values into one object; checks nothing |
| `state.errors` | The reactive list of messages |
| `addError(text)` | Adds your own message to the list |
| `clear()` | Empties the list |
| `addPattern(key, regex, normalize)` | Adds a pattern of your own |

The attributes of a field that it reads:

| Attribute | Meaning |
|-----------|---------|
| `required` | The field must not be empty |
| `minlength`, `maxlength` | Shortest and longest allowed length |
| `data-pattern` | The name of the pattern the value must match |
| `data-normalize` | `"true"` converts Persian digits and removes spaces; any other value has no effect |
| `data-title` | The field's name in messages |
| `name` | The key in `getFormData`; also the title when `data-title` is missing |

Five points to keep in mind:

- **It reads the page itself, not your data.** The values are read from the fields as they are in the document. Nothing is kept in a model.
- **Every call starts clean.** `validateForm` empties the list first and then fills it again.
- **An empty optional field is not checked.** Only `required` makes emptiness an error; rules for length and pattern apply to fields that have a value.
- **Disabled fields are ignored.** A field that is disabled, or sits inside a disabled `fieldset`, is neither checked nor collected.
- **It checks the shape of a value, not its correctness.** A national code only has to be ten digits; whether it is a real code is not checked.

## 2. The first check

Checking a form needs three things: an `id` (or, in chapter 8, a ref), fields that carry their rules as attributes, and a button that calls `validateForm`. The page below has two required fields, an optional nickname and a place that shows the messages. It presses the button several times, with different input:

```vue
<script setup>
const validator = useValidation()
const result = ref('')

function check() {
  const ok = validator.validateForm('signup')
  result.value = ok ? 'OK ' + JSON.stringify(validator.getFormData('signup')) : 'Not OK'
}
</script>

<template>
  <form id="signup">
    <UInput name="name" data-title="Name" placeholder="Name" required minlength="2" maxlength="10" />
    <UInput name="email" data-title="Email" data-pattern="email" placeholder="Email" required />
    <UInput name="nickname" data-title="Nickname" placeholder="Nickname (optional)" />
  </form>
  <button id="check" @click="check">Check</button>
  <p id="result">{{ result }}</p>
  <ul id="errors">
    <li v-for="e in validator.state.errors" :key="e">{{ e }}</li>
  </ul>
</template>
```

```text
all empty -> Not OK | ['مقداری برای فیلد Name مشخص نشده است', 'مقداری برای فیلد Email مشخص نشده است']
short name, bad email -> Not OK | ['مقدار وارد شده برای Name کوتاه است', 'مقدار وارد شده برای Email معتبر نیست']
typing 12 letters into maxlength=10 keeps: 'abcdefghij'
a value set by code, 12 letters -> Not OK | ['مقدار وارد شده برای Name طولانی است']
good -> OK {"name":"Sara","email":"sara@example.com","nickname":"sa"} | []
check again with new mistakes -> ['مقدار وارد شده برای Name کوتاه است', 'مقدار وارد شده برای Email معتبر نیست']
```

The output shows these points:

- **Every field is checked separately.** With all fields empty, the first press gives two errors, one for each `required` field. The optional nickname causes no error.
- **The messages carry the field's title.** A short name and an invalid email give one message each, and "Name" and "Email" come from `data-title`.
- **A value set by code can exceed `maxlength`; typing cannot.** The "too long" message appears only for a value assigned by the page. Typing cannot do that, because the browser itself stops the visitor at `maxlength`.
- **A clean form returns `true` and an empty list,** and **the list is replaced on every call,** so the last line shows only the new errors, not the old ones as well.
- **`getFormData` returns trimmed values:** the spaces around "Sara" and "sa" are removed.

The attributes of one field are written like this:

```html
<input name="mobile" data-title="Mobile" data-pattern="mobile" data-normalize="true"
       required minlength="10" maxlength="11">
```

The page needs no separate state for the form: the values stay in the fields, and `state.errors` is the only thing to show.

## 3. The rules

For each field the checks run in a fixed order, and knowing this order shows which message a visitor sees first:

1. If the field is `required` and empty, the message is recorded as "مقداری برای فیلد ... مشخص نشده است" ("no value was given for the field ...") and checking of this field ends there.
2. If the field is empty and not required, it is skipped and nothing else is checked.
3. If the value is shorter than `minlength`, the message recorded is "... کوتاه است" ("... is too short").
4. If it is longer than `maxlength`, the message recorded is "... طولانی است" ("... is too long").
5. If `data-pattern` is set and the value does not match, the message recorded is "... معتبر نیست" ("... is not valid").

Steps 3 to 5 do not stop each other: one field can add several messages. Length is counted on the normalized value (trimmed, and with Persian digits converted when normalizing applies; chapter 5).

If `data-pattern` names a pattern that does not exist, the message recorded is "pattern با کلید ... تعریف نشده است" ("the pattern with key ... is not defined"). A mistyped pattern name therefore shows up as an error at once, which is the best way to find it.

The function returns `true` only when the list stays empty.

## 4. Ready-made patterns

Most forms need the same few things: a mobile number, a national code, a postal code, a date. `data-pattern` accepts any of the built-in names (the full list is in A.3). Each is a regular expression that the whole value must match. The page below tries a valid and an invalid value for several of them:

```vue
<script setup>
const validator = useValidation()

const samples = [
  ['mobile', '09123456789', '9123456789'],
  ['nationalCode', '0123456789', '012345678'],
  ['email', 'user@example.com', 'user@example'],
  ['username', 'sara_2024', 'sa'],
  ['password', 'Secret123', 'secret123'],
  ['percentage', '100', '101'],
  ['date', '1403/05/15', '1403-05-15'],
  ['time', '14:30', '24:00']
]

const rows = ref([])

function passes(pattern, value) {
  const form = document.createElement('form')
  form.id = 'tmp'
  form.innerHTML = '<input name="f" data-pattern="' + pattern + '">'
  document.body.appendChild(form)
  form.querySelector('input').value = value
  const ok = validator.validateForm('tmp')
  form.remove()
  return ok
}

function run() {
  rows.value = samples.map(([key, good, bad]) => `${key}: "${good}" ${passes(key, good) ? 'passes' : 'fails'}, "${bad}" ${passes(key, bad) ? 'passes' : 'fails'}`)
}
</script>

<template>
  <button id="run" @click="run">Run</button>
  <ul><li v-for="r in rows" :key="r" class="row">{{ r }}</li></ul>
</template>
```

```text
mobile: "09123456789" passes, "9123456789" fails
nationalCode: "0123456789" passes, "012345678" fails
email: "user@example.com" passes, "user@example" fails
username: "sara_2024" passes, "sa" fails
password: "Secret123" passes, "secret123" fails
percentage: "100" passes, "101" fails
date: "1403/05/15" passes, "1403-05-15" fails
time: "14:30" passes, "24:00" fails
```

The main point is that **patterns check the shape of a value, not its meaning.** A national code only has to be ten digits, and a date has to look like a date. Passing the check does not make the value real; the server has to decide that.

- **`mobile`** accepts `09` followed by nine digits.
- **`nationalCode` and `postalCode`** accept ten digits. The check digit of the national code is not calculated.
- **`date`** accepts `yyyy/mm/dd`. Any digits are accepted; month 99 passes.
- **`password`** needs eight or more characters with at least one capital letter and one digit; `strongPassword` additionally needs a lowercase letter and one of `@$!%*?&`.
- **`username`** accepts three to 36 letters, digits or underscores.
- **`percentage`** accepts a whole number from 0 to 100.

## 5. Persian digits

Visitors in Iran often type Persian digits. A pattern for Latin digits then does not match, although the value is right, and the visitor sees a message they cannot understand. *Normalizing* solves the problem. It converts `۰۱۲۳۴۵۶۷۸۹` to `0123456789`, trims the value and removes every space inside it. It is off by default, and either of two things turns it on:

- the field has `data-normalize="true"`, or
- the field's pattern is registered with normalizing on (the third argument of `addPattern`; no built-in pattern does this).

`data-normalize="false"` is the same as writing nothing: it does not turn off normalizing that a pattern asks for.

The page below checks a Persian mobile number without and with normalizing, a pattern registered with normalizing, and a plain note:

```vue
<script setup>
const validator = useValidation()
const out = ref([])

validator.addPattern('trackingCode', /^TRK\d{6}$/, true)

function check() {
  const ok = validator.validateForm('f')
  out.value = [`valid: ${ok}`, `data: ${JSON.stringify(validator.getFormData('f'))}`, ...validator.state.errors]
}
</script>

<template>
  <form id="f">
    <UInput name="plain" data-title="Plain mobile" data-pattern="mobile" />
    <UInput name="fixed" data-title="Converted mobile" data-pattern="mobile" data-normalize="true" />
    <UInput name="tracking" data-title="Tracking" data-pattern="trackingCode" />
    <UInput name="note" data-title="Note" />
  </form>
  <button id="check" @click="check">Check</button>
  <ul><li v-for="l in out" :key="l" class="row">{{ l }}</li></ul>
</template>
```

```text
Persian digits:
valid: false
data: {"plain":"۰۹۱۲۳۴۵۶۷۸۹","fixed":"09123456789","tracking":"TRK123456","note":"۱۲ ۳"}
مقدار وارد شده برای Plain mobile معتبر نیست
Latin digits, spaces inside the second mobile:
valid: true
data: {"plain":"09123456789","fixed":"09123456789","tracking":"TRK123456","note":"۱۲ ۳"}
Arabic-Indic digits:
valid: false
data: {"plain":"٠٩١٢٣٤٥٦٧٨٩","fixed":"٠٩١٢٣٤٥٦٧٨٩","tracking":"TRK123456","note":"۱۲ ۳"}
مقدار وارد شده برای Plain mobile معتبر نیست
مقدار وارد شده برای Converted mobile معتبر نیست
```

Without normalizing, the Persian mobile number is rejected, and `getFormData` returns the Persian digits as typed. With `data-normalize="true"` the same number passes and comes out in Latin digits, and spaces typed inside it (`09 12 345 6789`) are removed. The tracking pattern is added with normalizing on, so its field needs no attribute: `TRK ۱۲۳ ۴۵۶` becomes `TRK123456`.

Two limits are worth knowing. A field with no pattern and no `data-normalize` (the note) keeps its Persian digits and its inner spaces; only the spaces at both ends are removed. And Arabic-Indic digits (`٠١٢٣٤٥٦٧٨٩`, the Arabic keyboard's) are not converted, so such a value is rejected even with normalizing. Only Persian digits are converted.

The same value is checked and collected, so with normalizing the form gives the server Latin digits. This is the main reason to switch it on for numeric fields.

## 6. Your own patterns and checks across fields

The built-in patterns cover the common cases, but every project has its own: a plate number, an order code, a tracking number. `addPattern(key, regex, normalize)` adds a pattern whose name fields can then write in `data-pattern`:

```js
validator.addPattern('employee', /^EMP-\d{4}$/)         // shape only
validator.addPattern('tracking', /^TRK\d{6}$/, true)    // true: always normalize
```

A `g` or `y` flag is removed from the regular expression, because it would make the same pattern fail on every second check. A value that is not a `RegExp` adds the message "pattern باید RegExp باشد" ("the pattern must be a RegExp") to the list and also throws an exception.

Each field is checked alone, so a rule that compares two fields, such as "the passwords must match", has to be written in the page. `addError` adds a message to the same list, so the page shows all errors together:

```js
const data = validator.getFormData('signup')
if (validator.validateForm('signup') && data.password !== data.again) {
  validator.addError('The two passwords differ.')
}
```

```vue
<script setup>
const validator = useValidation()
const out = ref([])

validator.addPattern('employee', /^EMP-\d{4}$/g)

function check() {
  const data = validator.getFormData('f')
  const ok = validator.validateForm('f')
  if (ok && data.password !== data.again) validator.addError('The two passwords differ.')
  out.value = [`no errors: ${validator.state.errors.length === 0}`, ...validator.state.errors]
}

function bad() {
  try {
    validator.addPattern('oops', 'EMP-\\d{4}')
  } catch (e) {
    out.value = ['threw: ' + e.message, ...validator.state.errors]
  }
}
</script>

<template>
  <form id="f">
    <UInput name="code" data-title="Code" data-pattern="employee" />
    <UInput name="password" data-title="Password" required />
    <UInput name="again" data-title="Repeat" required />
  </form>
  <button id="check" @click="check">Check</button>
  <button id="bad" @click="bad">Bad pattern</button>
  <ul><li v-for="l in out" :key="l" class="row">{{ l }}</li></ul>
</template>
```

```text
bad code -> ['no errors: false', 'مقدار وارد شده برای Code معتبر نیست']
different passwords -> ['no errors: false', 'The two passwords differ.']
same call again (the g flag was removed) -> ['no errors: false', 'The two passwords differ.']
all good -> ['no errors: true']
addPattern with a string -> ['threw: pattern باید RegExp باشد', 'pattern باید RegExp باشد']
```

The output shows three things. An invalid code gives the standard "not valid" message, and the same call again gives the same answer, so the pattern keeps working although it is written with `g`. Different passwords pass the field checks, and the page's own message is added to the list. A string instead of a regular expression makes `addPattern` throw, and its message is also in the list.

One detail matters when adding your own errors: `validateForm` has already returned `true` before `addError` is called. After adding your own errors, decide by `state.errors.length`, not by that result.

## 7. Checkboxes, radio buttons and selects

Forms are not only text boxes. Each kind of field has its own meaning of "filled in", and the validator follows it:

- **A checkbox** is checked only for `required`: it must be ticked. Its value in `getFormData` is `true` or `false`.
- **A radio group** counts as one field. It is required when any button of the group is `required`, and then one button must be chosen. In `getFormData` the group gives the chosen value, and no key at all when nothing is chosen.
- **A select** works like a text field, with `required` meaning the value must not be empty. A `select multiple` gives an array of the chosen values in `getFormData`.
- **A disabled field**, or one inside a disabled `fieldset`, is skipped by both functions. For the validator it does not exist.
- **A field without a `name`** is checked but not collected.

The page below puts all these kinds into one form:

```vue
<script setup>
const validator = useValidation()
const out = ref([])

function check() {
  const ok = validator.validateForm('f')
  out.value = [`valid: ${ok}`, ...validator.state.errors, 'data: ' + JSON.stringify(validator.getFormData('f'))]
}

function fix() {
  document.querySelector('[name=terms]').checked = true
  document.querySelector('[value=m]').checked = true
  document.querySelector('[name=city]').value = 'b'
}
</script>

<template>
  <form id="f">
    <label><input type="checkbox" name="terms" data-title="Terms" required> I agree</label>
    <label><input type="checkbox" name="news" data-title="News"> News</label>
    <label><input type="radio" name="size" value="s" data-title="Size" required> S</label>
    <label><input type="radio" name="size" value="m" data-title="Size"> M</label>
    <select name="city" data-title="City" required>
      <option value="">-</option>
      <option value="a">A</option>
      <option value="b">B</option>
    </select>
    <select name="tags" data-title="Tags" multiple>
      <option value="x" selected>X</option>
      <option value="y" selected>Y</option>
      <option value="z">Z</option>
    </select>
    <fieldset id="extra" disabled>
      <input name="gift" data-title="Gift" required>
    </fieldset>
    <input name="skipped" data-title="Skipped" :disabled="true" required>
    <input data-title="Unnamed" required>
  </form>
  <button id="check" @click="check">Check</button>
  <button id="fix" @click="fix">Fix</button>
  <ul><li v-for="l in out" :key="l" class="row">{{ l }}</li></ul>
</template>
```

```text
untouched form:
valid: false
مقداری برای فیلد Terms مشخص نشده است
مقداری برای فیلد Size مشخص نشده است
مقداری برای فیلد City مشخص نشده است
مقداری برای فیلد Unnamed مشخص نشده است
data: {"terms":false,"news":false,"city":"","tags":["x","y"]}
after ticking, choosing:
valid: false
مقداری برای فیلد Unnamed مشخص نشده است
data: {"terms":true,"news":false,"size":"m","city":"b","tags":["x","y"]}
```

The untouched form shows errors for the unticked terms, the unchosen size and the empty city. The disabled "gift" and "skipped" fields do not appear in the messages. The field with no name is still checked, and it is named by its `data-title` ("Unnamed"); without a `data-title`, the message would say only "فیلد". The collected data shows the checkboxes as `true` and `false`, `size` absent at first and `"m"` once chosen, and the multiple select as an array. The disabled fields and the unnamed field are not in it.

## 8. The ref-based version

`useValidationRefBased` has the same list of messages, patterns and rules. The only difference is how the form is specified: `validateForm` and `getFormData` take a ref to the `<form>` element instead of an id.

```vue
<script setup>
import { useValidationRefBased } from '~/composables/ref-based/useValidationRefBased.js'

const validator = useValidationRefBased()
const form = ref(null)

function save() {
  if (validator.validateForm(form)) send(validator.getFormData(form))
}
</script>

<template>
  <form ref="form"> ... </form>
</template>
```

A ref is useful when two forms on a page could share an id, or when the form lives inside a component that is shown several times. The ref always points to the right element, and no id has to be invented.

```vue
<script setup>
import { useValidationRefBased } from '~/composables/ref-based/useValidationRefBased.js'

const validator = useValidationRefBased()
const form = ref(null)
const out = ref([])

function check() {
  const ok = validator.validateForm(form)
  out.value = [`valid: ${ok}`, ...validator.state.errors, 'data: ' + JSON.stringify(validator.getFormData(form))]
}

function checkElement() {
  const ok = validator.validateForm(form.value)
  out.value = ['with the element itself: ' + ok]
}

function checkMissing() {
  const missing = ref(null)
  const ok = validator.validateForm(missing)
  out.value = [`valid: ${ok}`, ...validator.state.errors]
}
</script>

<template>
  <form ref="form">
    <UInput name="name" data-title="Name" required />
    <UInput name="email" data-title="Email" data-pattern="email" />
  </form>
  <button id="check" @click="check">Check</button>
  <button id="element" @click="checkElement">Element</button>
  <button id="missing" @click="checkMissing">Missing</button>
  <ul><li v-for="l in out" :key="l" class="row">{{ l }}</li></ul>
</template>
```

```text
empty:
valid: false
مقداری برای فیلد Name مشخص نشده است
data: {"name":"","email":""}
filled:
valid: true
data: {"name":"Sara","email":"sara@example.com"}
with the element itself: true
ref with no form:
valid: false
فرم پیدا نشد
```

A ref works like an id, and the element itself also works: `validateForm(form.value)`. A ref that holds nothing (the form is not mounted yet) gives `false` and the message "فرم پیدا نشد" ("form not found"). The id version says the same when no element has that id, and `getFormData` then returns an empty object.

The ref must point to a plain `<form>`. A ref to a component gives an error:

```vue
<script setup>
import { useValidationRefBased } from '~/composables/ref-based/useValidationRefBased.js'

const validator = useValidationRefBased()
const box = ref(null)
const out = ref('')

const Wrapper = { template: '<form><slot /></form>' }

function check() {
  try {
    out.value = 'valid: ' + validator.validateForm(box)
  } catch (e) {
    out.value = e.name + ': ' + e.message
  }
}
</script>

<template>
  <component :is="Wrapper" ref="box"><input name="a" required></component>
  <button id="check" @click="check">Check</button>
  <p class="row">{{ out }}</p>
</template>
```

```text
TypeError: form.querySelectorAll is not a function
```

The error occurs because the ref holds the component itself, not the element. If a component wraps the form, give the inner `<form>` its own ref, or pass its element (for a Nuxt UI component the element is available as `$el`).

## 9. Putting it together

We build a registration form with the ref-based version: field rules in the HTML, a custom pattern for plates, a repeated password checked by the page and a list of errors under the form:

```vue
<script setup>
import { useValidationRefBased } from '~/composables/ref-based/useValidationRefBased.js'

const validator = useValidationRefBased()
const form = ref(null)
const saved = ref([])

validator.addPattern('plate', /^\d{2}[A-Z]\d{3}$/)

function submit() {
  const data = validator.getFormData(form)
  if (!validator.validateForm(form)) return
  if (data.password !== data.again) {
    validator.addError('The two passwords differ.')
    return
  }
  saved.value.push(`${data.name} / ${data.mobile} / ${data.plate}`)
  validator.clear()
}
</script>

<template>
  <form ref="form" @submit.prevent="submit">
    <UInput name="name" data-title="Name" required minlength="2" />
    <UInput name="mobile" data-title="Mobile" data-pattern="mobile" required />
    <UInput name="plate" data-title="Plate" data-pattern="plate" />
    <UInput name="password" data-title="Password" data-pattern="password" required />
    <UInput name="again" data-title="Repeat" required />
    <label><input type="checkbox" name="terms" data-title="Terms" required> I agree</label>
  </form>
  <button id="send" @click="submit">Register</button>
  <ul id="errors"><li v-for="e in validator.state.errors" :key="e">{{ e }}</li></ul>
  <p id="saved">Saved: {{ saved.join(' ; ') || 'nothing' }}</p>
</template>
```

```text
empty -> ['مقداری برای فیلد Name مشخص نشده است', 'مقداری برای فیلد Mobile مشخص نشده است', 'مقداری برای فیلد Password مشخص نشده است', 'مقداری برای فیلد Repeat مشخص نشده است', 'مقداری برای فیلد Terms مشخص نشده است']
terms unticked, passwords differ -> ['مقداری برای فیلد Terms مشخص نشده است']
passwords differ -> ['The two passwords differ.']
bad plate -> ['مقدار وارد شده برای Plate معتبر نیست']
good -> [] | Saved: Sara / 09123456789 / 12A345
```

This form brings the points of the guide together:

- **Rules sit in the HTML,** so the empty form gives an error for every required field and leaves the optional plate alone (chapters 2 and 3).
- **Order of checks matters.** While the terms are unticked, the form is rejected and the password comparison is never reached. The page's own comparison runs only after the field checks pass, so the visitor sees the field errors first.
- **The custom pattern checks the plate** (chapter 6). A lower-case letter does not match `[A-Z]`.
- **A clean form is saved,** and `clear()` empties the list afterwards.

---

## Appendix A. Quick reference

### A.1 Functions

| Function | Returns | Meaning |
|----------|---------|---------|
| `validateForm(form)` | Boolean | Checks all fields and updates `state.errors` |
| `getFormData(form)` | Object | The values keyed by `name`; `{}` if the form is missing |
| `addPattern(key, regex, normalize = false)` | nothing | Adds or replaces a pattern; throws if `regex` is not a `RegExp` |
| `addError(text)` | nothing | Appends a message (empty text is ignored) |
| `clear()` | nothing | Empties `state.errors` |
| `state.errors` | Array | The reactive list of messages |

`form` is an `id` for `useValidation`, and a ref or an element for `useValidationRefBased`.

### A.2 Field attributes

| Attribute | Effect |
|-----------|--------|
| `required` | Empty gives an error (checkbox: must be ticked; radio: one button of the group must be chosen) |
| `minlength`, `maxlength` | Length limits; counted after normalizing |
| `data-pattern` | Name of the pattern |
| `data-normalize` | `"true"` turns normalizing on; no other value turns it off |
| `data-title` | Name in the messages |
| `name` | Key in `getFormData`; fallback title |
| `disabled` | The field is ignored |

### A.3 Built-in patterns

| Key | Accepts | Example |
|-----|---------|---------|
| `mobile` | `09` and nine digits | `09123456789` |
| `nationalCode` | Ten digits | `0123456789` |
| `postalCode` | Ten digits | `1234567890` |
| `date` | `yyyy/mm/dd` | `1403/05/15` |
| `iranianPlate` | Two digits, a Persian letter, three digits, a dash, two digits | `12ب345-67` |
| `shebaNumber` | `IR` and 24 digits | `IR123456789012345678901234` |
| `cardNumber` | Sixteen digits | `1234567890123456` |
| `landline` | `0` and ten digits | `02112345678` |
| `taxCode`, `companyNationalId` | Eleven digits | `12345678901` |
| `email` | Text, `@`, text, a dot, text | `user@example.com` |
| `url` | `http://` or `https://` and more | `https://example.com` |
| `numeric` | Digits only | `12345` |
| `decimal` | Digits with an optional fraction | `12.5` |
| `integer` | Digits with an optional minus | `-5` |
| `positiveNumber` | A whole number from 1 up | `10` |
| `percentage` | A whole number from 0 to 100 | `100` |
| `username` | 3 to 36 letters, digits or `_` | `sara_2024` |
| `password` | 8 or more characters, a capital letter and a digit | `Secret123` |
| `strongPassword` | As above, plus a lowercase letter and one of `@$!%*?&` | `Secret1@x` |
| `ipv4` | Four numbers up to 255 | `192.168.1.1` |
| `hexColor` | `#` and 3 or 6 hex digits | `#ff8800` |
| `time` | `hh:mm`, 24 hours | `14:30` |
| `gregorianDate` | `yyyy-mm-dd` | `2024-01-15` |
| `slug` | Lowercase words joined by dashes | `my-page-title` |
| `alphaEn` | English letters only | `Sara` |
| `alphaFa` | Persian letters and spaces | `سارا` |
| `alphaNumEn` | English letters and digits | `abc123` |
| `noSpecialChar` | English or Persian letters, digits, spaces | `سارا 12` |
| `instagram` | Optional `@`, up to 30 letters, digits, `.` or `_` | `@sara.k` |
| `telegram` | Optional `@`, 5 to 32 letters, digits or `_` | `@sara_k` |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| The form with that id is not found, or the ref is empty | `validateForm` returns `false` with "فرم پیدا نشد"; `getFormData` returns `{}` |
| The ref points to a component | `TypeError: form.querySelectorAll is not a function` |
| `data-pattern` names an unknown pattern | A "pattern ... تعریف نشده است" message; the form is invalid |
| `addPattern` receives a non-RegExp | A message in the list and a thrown `Error` |
| A field is disabled | Ignored by both functions |
| `validateForm` called twice | The second call replaces the first one's messages |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Expecting fields that are not on the page yet to be checked.** Only the DOM is read. A field that has not been rendered yet does not exist for the check.
- **A duplicated `id`.** With `useValidation`, only the first form with that id is found. Use the ref-based version, or unique ids.
- **A ref to a component.** The ref must point to the `<form>` element.
- **`addError` and the result of `validateForm`.** The result does not know about errors added afterwards. Check `state.errors.length`.
- **Arabic digits.** Only Persian digits are converted.
- **Mistaking shape for correctness.** `nationalCode`, `cardNumber`, `shebaNumber` and `date` check shape only; the server must check correctness.
- **A pattern name typo.** It shows up as a visible error, but only when the field has a value; an empty optional field is not checked.
- **A radio group without `required`.** Without it nothing is checked and, if nothing is chosen, the key is absent from the data.
- **The group's title.** The message of a radio group uses the title of the first button; give them all the same `data-title`.
- **Messages in other languages.** The messages are fixed in Persian; translate or replace them in the page if needed (read `state.errors` yourself).

### C.2 Behaviour worth knowing

- `state` is reactive, so a template can show `state.errors` without `.value`.
- `maxlength` is also enforced by the browser while typing, so the "too long" message appears only for values set by code.
- Spaces at both ends of values are removed in both functions; a field of only spaces counts as empty.
- Pattern checks run on the normalized value, so with `data-normalize="true"` a value with Persian digits matches a Latin pattern.
- The list holds each error as a separate string, in the order of the fields in the document.
- A pattern added with `addPattern` replaces a built-in one if it has the same key.

### C.3 Limits

| What | Limit |
|------|-------|
| Messages | Persian, fixed |
| Field types read | `input`, `textarea`, `select` |
| Normalizing | Persian digits, removing spaces at both ends and inside the value; not Arabic digits |
| Cross-field rules | Written by the page with `addError` |
| Asynchronous checks | Not supported; `validateForm` runs synchronously |

### C.4 Tuning

- Put the rules in the HTML and keep JavaScript for comparisons between fields.
- Give every field a `data-title`, so messages name it in words familiar to the visitor.
- Turn normalizing on for numeric fields visitors may type with a Persian keyboard.
- If you do not want old messages to stay, call `clear()` when the visitor starts typing again.
- Keep the server's own validation as well; this one only protects the visitor from simple mistakes.

## Appendix D. Message reference

### D.1 Messages in `state.errors`

| Message | Meaning | When |
|---------|---------|------|
| مقداری برای فیلد (title) مشخص نشده است | No value was given | A required field is empty, a required checkbox is unticked, or a required radio group has no choice |
| مقدار وارد شده برای (title) کوتاه است | Too short | The value is shorter than `minlength` |
| مقدار وارد شده برای (title) طولانی است | Too long | The value is longer than `maxlength` |
| مقدار وارد شده برای (title) معتبر نیست | Not valid | The value does not match the pattern |
| pattern با کلید (key) تعریف نشده است | The pattern is not defined | `data-pattern` names an unknown key |
| فرم پیدا نشد | Form not found | No element has the id, or the ref is empty |
| pattern باید RegExp باشد | The pattern must be a RegExp | `addPattern` receives another type |
