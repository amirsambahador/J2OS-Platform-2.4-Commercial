# useNumber

*J2OS Platform 2.4*

An invoice form has a price field. The user types `1250000` and wants to see `1,250,000`. A Persian-speaking user wants `۱,۲۵۰,۰۰۰`. The cheque needs the amount in words. The server, meanwhile, wants none of this: it wants the plain number `1250000`. Somebody has to keep the display and the value apart, and do it the same way on every form.

**useNumber** is a composable of the Nuxt front end of J2OS that does this. **useNumberRefBased** is its twin for inputs that come and go (modals, wizard steps), and this guide covers both. They have three properties:

- **Formatting while typing.** A number box shows commas and, if asked, Persian digits, keeps the cursor where it was and cleans pasted text.
- **A raw value for the program.** Whatever is shown, the callback receives Latin digits with a dot as the decimal mark, and nothing else.
- **Numbers in words.** `getEnglishWords` and `getPersianWords` write a number out as text, up to 36 digits.

The whole thing in a few lines:

```vue
<script setup>
const { setupDoubleNumberBox, getPersianWords } = useNumber()

const amount = ref('0')                       // raw: "1250000.5"

onMounted(() => {
  // 12 digits before the point, 2 after; the callback gets the raw number.
  setupDoubleNumberBox('amount-field', 12, 2, (raw) => { amount.value = raw })
})
</script>

<template>
  <input id="amount-field">                   <!-- shows 1,250,000٫5 -->
  <p>{{ getPersianWords(amount) }}</p>
</template>
```

This guide starts with how the composable works (chapter 1). Chapter 2 writes numbers as words and chapter 3 formats them for display. Chapter 4 covers number boxes that find their input by `id`, and chapter 5 the ref-based boxes. Chapter 6 builds a price and quantity form. The appendices work as a reference.

## Contents

1. [How useNumber works](#1-how-usenumber-works)
2. [Numbers as words](#2-numbers-as-words)
3. [Numbers as display text](#3-numbers-as-display-text)
4. [Number boxes by id](#4-number-boxes-by-id)
5. [Ref-based boxes](#5-ref-based-boxes)
6. [Putting it together](#6-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are complete pages of a Nuxt project: each is one `.vue` file. In Nuxt `ref`, `computed`, `onMounted` and `useNumber` need no `import`, so none is written.

---

## 1. How useNumber works

First, a few terms, because they come up throughout the guide:

- **Raw value.** The number as the program should keep it: Latin digits, a dot for the decimal mark, no commas, no leading zeros. It is always a string, for example `"1250000.5"`.
- **Display text.** What the visitor sees in the input: commas between groups of three digits, `٫` as the decimal mark, and Persian digits in the Persian boxes.
- **Number box.** The behaviour attached to one input: it reformats the text on every keystroke and reports the raw value.

The functions of this composable:

| Piece | Role |
|-------|------|
| `getEnglishWords(value)`, `getPersianWords(value)` | A number as words |
| `getPersianNumber(value)`, `getEnglishNumber(value)` | A number as display text |
| `getEnglishNumberWithoutCommas(value)` | Any number text as a raw value |
| `setupNumberBox(id, maxLength, onChange)` | Whole-number box, Latin digits |
| `setupPersianNumberBox(id, maxLength, onChange)` | Whole-number box, Persian digits |
| `setupDoubleNumberBox(id, maxLength, decimalPlaces, onChange)` | Decimal box, Latin digits |
| `setupPersianDoubleNumberBox(id, maxLength, decimalPlaces, onChange)` | Decimal box, Persian digits |
| `useNumberRefBased()` | The same nine functions; the boxes take a ref instead of an `id` |

Keep these six points in mind:

- **Values are strings.** Pass numbers as text and keep them as text. A box never gives back a JavaScript number.
- **The box keeps the raw value in two places:** the callback argument and the `data-raw` attribute of the input.
- **The input is never empty.** Emptying it gives `0`.
- **Typing is forgiving.** Letters are dropped, Persian and Latin digits are both accepted, and pasted text is cleaned the same way.
- **`maxLength` counts the digits before the decimal mark,** and `decimalPlaces` the digits after it. Extra digits are cut off, not refused.
- **The `id` versions need the input to exist** when they are called. The ref-based versions do not (chapter 5).

## 2. Numbers as words

Cheques, invoices and contracts write the amount in words as well as in digits, and a page that shows the words next to the input helps the visitor catch a mistyped zero. Two functions do it, one per language. They take the number as text:

```vue
<script setup>
const { getEnglishWords, getPersianWords } = useNumber()

const show = (read, value) => {
  try { return JSON.stringify(read(value)) } catch (e) { return `${e.constructor.name}: ${e.message}` }
}

const values = [
  ['0', '0'], ['7', '7'], ['15', '15'], ['21', '21'], ['100', '100'], ['123', '123'],
  ['1001', '1001'], ['12345', '12345'], ['2500000', '2500000'], ['1,234 (with a comma)', '1,234'],
  ['1.5', '1.5'], ['0.05', '0.05'], ['007', '007'],
  ['the number 0', 0], ['an empty text', ''], ['minus 5', '-5'], ['abc', 'abc'], ['Persian digits', '۱۲۳']
]

const rows = values.map(([label, value]) => ({
  label,
  en: show(getEnglishWords, value),
  fa: show(getPersianWords, value)
}))

const biggest = '9'.repeat(36)
const tooBig = '1' + '0'.repeat(36)
</script>

<template>
  <div v-for="r in rows" :key="r.label" class="row">{{ r.label }} → {{ r.en }} | {{ r.fa }}</div>
  <div class="row">36 digits → {{ show(getEnglishWords, biggest).slice(0, 60) }}…</div>
  <div class="row">37 digits → {{ show(getEnglishWords, tooBig) }}</div>
  <div class="row">37 digits (Persian) → {{ show(getPersianWords, tooBig) }}</div>
</template>
```

```text
0 → "Zero" | "صفر"
7 → "Seven" | "هفت"
15 → "Fifteen" | "پانزده"
21 → "Twenty One" | "بیست و یک"
100 → "One Hundred" | "صد"
123 → "One Hundred Twenty Three" | "صد و بیست و سه"
1001 → "One Thousand One" | "یک هزار و یک"
12345 → "Twelve Thousand Three Hundred Forty Five" | "دوازده هزار و سیصد و چهل و پنج"
2500000 → "Two Million Five Hundred Thousand" | "دو میلیون و پانصد هزار"
1,234 (with a comma) → "One Thousand Two Hundred Thirty Four" | "یک هزار و دویست و سی و چهار"
1.5 → "One Point Five" | "یک ممیز پنج"
0.05 → "Zero Point Zero Five" | "صفر ممیز صفر پنج"
007 → "Seven" | "هفت"
the number 0 → "" | ""
an empty text → "" | ""
minus 5 → "Five" | "پنج"
abc → "" | ""
Persian digits → "" | ""
36 digits → "Nine Hundred Ninety Nine Decillion Nine Hundred Ninety Nine…
37 digits → RangeError: Number is too large to convert to words (exceeds supported magnitude).
37 digits (Persian) → RangeError: عدد برای تبدیل به حروف بسیار بزرگ است (خارج از محدوده‌ی پشتیبانی‌شده).
```

Whole numbers are written out the way a person would say them. English words start with capitals (`Twenty One`) and Persian words are joined with `و`. Commas in the input are ignored, and so are leading zeros (`007` is seven). Decimals are read digit by digit after the word for the point: `1.5` is "One Point Five" and `0.05` is "Zero Point Zero Five".

The edge cases are worth knowing, because a live input meets all of them while the visitor types:

- **Zero** is "Zero" (`صفر`) when it is passed as the text `"0"`. The JavaScript number `0` and an empty text give an empty string instead, so the page shows nothing, rather than "zero", while the box is empty.
- **A minus sign is dropped:** `-5` is read as five. Negative numbers are not supported.
- **Text with no digits gives an empty string,** and so does **Persian digits**: these two functions expect Latin digits. Run Persian input through `getEnglishNumberWithoutCommas` first (chapter 3).
- **A number of 37 digits or more throws a `RangeError`,** because the largest group name that exists covers up to 36 digits. The message is in English or Persian, like the function.

A page that shows words next to a live input should therefore catch the error:

```js
const { getPersianWords } = useNumber()

const words = (value) => {
  try {
    return getPersianWords(String(value))
  } catch {
    return ''                          // more than 36 digits
  }
}
```

## 3. Numbers as display text

The same number is written in several ways: for the program (`1250000.5`), for an English reader (`1,250,000.5`), for a Persian reader (`۱,۲۵۰,۰۰۰٫۵`). Three functions convert between them, and knowing which one produces what avoids a whole class of mistakes:

```vue
<script setup>
const { getPersianNumber, getEnglishNumber, getEnglishNumberWithoutCommas } = useNumber()

const inputs = [
  '1234567.891', '0001234', '1,234,567', '۱۲۳۴۵۶۷', '۱۲۳۴.۵', '12.3.4', 'abc 12 def', '0', '.5', ''
]

const rows = inputs.map(value => ({
  value,
  fa: getPersianNumber(value),
  en: getEnglishNumber(value),
  plain: getEnglishNumberWithoutCommas(value)
}))
</script>

<template>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ JSON.stringify(r.value) }} → persian: {{ JSON.stringify(r.fa) }} | english: {{ JSON.stringify(r.en) }} | plain: {{ JSON.stringify(r.plain) }}</div>
</template>
```

```text
"1234567.891" → persian: "۱,۲۳۴,۵۶۷٫۸۹۱" | english: "1,234,567٫891" | plain: "1234567.891"
"0001234" → persian: "۱,۲۳۴" | english: "1,234" | plain: "1234"
"1,234,567" → persian: "۱,۲۳۴,۵۶۷" | english: "1,234,567" | plain: "1234567"
"۱۲۳۴۵۶۷" → persian: "۰" | english: "1,234,567" | plain: "1234567"
"۱۲۳۴.۵" → persian: "۰٫" | english: "1,234٫5" | plain: "1234.5"
"12.3.4" → persian: "۱۲٫۳۴" | english: "12٫3" | plain: "12.34"
"abc 12 def" → persian: "۱۲" | english: "abc 12 def" | plain: "12"
"0" → persian: "۰" | english: "0" | plain: "0"
".5" → persian: "۰٫۵" | english: "0٫5" | plain: "0.5"
"" → persian: "" | english: "" | plain: ""
```

- **`getPersianNumber`** gives Persian digits, commas and `٫`. It expects **Latin digits**: Persian digits in the input are thrown away (the rows with Persian digits show `۰` and `۰٫`).
- **`getEnglishNumber`** gives Latin digits with commas, and accepts Persian digits. Its decimal mark is `٫`, not a dot, so its result cannot be passed to `Number()`.
- **`getEnglishNumberWithoutCommas`** is the cleaner. It accepts commas, `٫` and Persian digits, keeps only digits and the first dot, and removes leading zeros. This is the function that produces a **raw value**, the form a program should keep.
- **An empty input** gives an empty string from all three.
- **A second dot** is handled differently: the raw value joins the digits (`12.3.4` becomes `12.34`), while `getEnglishNumber` keeps only the first two parts (`12٫3`).

The rule of thumb is to clean once, with `getEnglishNumberWithoutCommas`, and to produce display text from the clean value. Display text is for eyes; it should never be parsed back.

## 4. Number boxes by id

Showing a number is one task; letting the visitor *type* one is a harder one. The visitor types digits, commas, Persian letters, pastes a number from a spreadsheet, deletes everything. A number box takes over one input and handles all of it: it reformats the text on every keystroke and reports the raw value to a callback.

The `id` versions look the input up with `document.getElementById`, so call them once the page is mounted. The first example has a whole-number box with a maximum of 5 digits, a Persian one, and a button that detaches the first:

```vue
<script setup>
const { setupNumberBox, setupPersianNumberBox } = useNumber()

const plainRaw = ref('')
const persianRaw = ref('')
const calls = ref(0)
let stopPlain = null
const missing = ref('')

onMounted(() => {
  stopPlain = setupNumberBox('plain', 5, (raw) => { plainRaw.value = raw; calls.value++ })
  setupPersianNumberBox('persian', 36, (raw) => { persianRaw.value = raw })
  missing.value = typeof setupNumberBox('nowhere', 5, () => {})
})
</script>

<template>
  <input id="plain">
  <input id="persian">
  <button id="stop" @click="stopPlain()">Detach the first box</button>
  <p id="plain-raw">{{ plainRaw }}</p>
  <p id="persian-raw">{{ persianRaw }}</p>
  <p id="calls">{{ calls }}</p>
  <p id="missing">{{ missing }}</p>
</template>
```

```text
missing id returns: undefined
typed '1234567'      -> shown '12,345'               raw '12345'
typed '12a34'        -> shown '1,234'                raw '1234'
typed '123456789'    -> shown '12,345'               raw '12345'
callback raw: '12345', calls: 25
pasted 98765432 -> '98,765' raw '98765'
cleared -> shown '0' raw '0'
typed '1234567'      -> shown '۱,۲۳۴,۵۶۷'            raw '1234567'
typed '۱۲۳۴a'        -> shown '۱,۲۳۴'                raw '1234'
persian callback raw: '1234'
after detach, typed 77 -> shown '077' raw '0'
```

The box is forgiving about what is typed, and strict about what is kept:

- **Extra digits are cut, not refused.** Typing `1234567` into the box with a maximum of 5 showed `12,345`.
- **Letters are dropped,** and **pasting** works exactly like typing. Pasted text replaces the whole value wherever the caret is: the pasted `98765432` became `98,765`.
- **Text already in the input** is formatted when the box attaches (`1234567.891` became `1,234,567٫89` with two places), with one callback call for it.
- **The input is never empty.** Emptying it shows `0`, and the callback receives `0`.
- **The Persian box** shows Persian digits and accepts Latin ones.
- **A missing `id` returns `undefined`** and nothing else happens, not even an error. A misspelt `id` therefore fails quietly; check the first run in the browser.
- **The returned function** removes the box from that input. Afterwards the input is a plain input again (typing `77` after `0` gave `077`).

Release the box when the page can outlive the input, or when the component is unmounted while the input stays:

```js
let stop

onMounted(() => {
  stop = setupNumberBox('count-field', 4, (raw) => { count.value = raw })
})

onBeforeUnmount(() => stop?.())
```

A value set from code is not formatted, because the box listens to the `input` event. After assigning, fire the event yourself:

```js
// A value set from code is not formatted, because no input event is fired.
const el = document.getElementById('amount-field')
el.value = '1234567'
el.dispatchEvent(new Event('input'))
```

Prices and measurements need decimals. Decimal boxes take one more argument, `decimalPlaces`:

```vue
<script setup>
const { setupDoubleNumberBox, setupPersianDoubleNumberBox } = useNumber()

const plainRaw = ref('')
const persianRaw = ref('')

onMounted(() => {
  setupDoubleNumberBox('plain', 8, 2, (raw) => { plainRaw.value = raw })
  setupPersianDoubleNumberBox('persian', 8, 2, (raw) => { persianRaw.value = raw })
})
</script>

<template>
  <input id="plain">
  <input id="persian">
  <p id="plain-raw">{{ plainRaw }}</p>
  <p id="persian-raw">{{ persianRaw }}</p>
</template>
```

```text
typed '1234.5678'    -> shown '1,234٫56'             raw '1234.56'
typed '.5'           -> shown '0٫5'                  raw '0.5'
typed '12.3.4'       -> shown '12٫34'                raw '12.34'
typed '00012.50'     -> shown '12٫50'                raw '12.50'
typed '123456789.1'  -> shown '12,345,678٫1'         raw '12345678.1'
callback raw: '12345678.1'
typed '1234.5678'    -> shown '۱,۲۳۴٫۵۶'             raw '1234.56'
typed '12.3'         -> shown '۱۲٫۳'                 raw '12.3'
persian callback raw: '12.3'
```

The decimal rules are easiest to remember as one picture: *the whole part and the decimals are limited separately.*

- **A dot is the decimal mark.** `.5` becomes `0.5`, and the display shows `٫` in place of the dot. A second dot is ignored (`12.3.4` gave `12.34`).
- **Extra decimals are cut:** `1234.5678` with two places gave `1234.56`.
- **Leading zeros go, trailing zeros stay:** `00012.50` gave `12.50`.
- **The limit on the whole part** (here 8) is applied apart from the decimals: `123456789.1` gave `12345678.1`.

Whatever the display, send the raw value to the server and never the display text:

```js
// The display text is for people, the raw value is for the server.
await $fetch('/api/invoices', {
  method: 'POST',
  body: { amount: amount.value }       // "1250000.5", never "1,250,000٫5"
})
```

## 5. Ref-based boxes

In forms with modals, tabs and wizard steps, the input often does not exist yet when the page sets up, or is created again later. The `id` versions would have nothing to attach to. `useNumberRefBased()` solves this: it has the same nine functions, but the box functions take a ref (or a component ref) instead of an `id`, and the rest of the argument list stays as before, with an optional `options` object at the end.

The input does not have to exist when the function is called. The box attaches when the input appears, attaches again if it is created again, and lets go when it disappears or the component is unmounted. This is exactly what modals and wizard steps need:

```vue
<script setup>
const { setupPersianNumberBox } = useNumberRefBased()

const open = ref(false)
const input = ref()
const code = ref('0')

// Called once, in setup. The input does not exist yet, and that is fine.
setupPersianNumberBox(input, 6, (raw) => { code.value = raw })
</script>

<template>
  <button @click="open = !open">Toggle</button>
  <input v-if="open" ref="input">
</template>
```

The next page shows the life of one box. It starts with a stored value `500`, then the value is set from code, the input is hidden and shown again, and the box is finally destroyed:

```vue
<script setup>
const { setupNumberBox } = useNumberRefBased()

const input = ref()
const visible = ref(true)
const raw = ref('')
const stored = reactive({ amount: '500' })
const log = ref([])

// Called in setup(): the input does not exist yet, and that is fine.
const box = setupNumberBox(input, 10, (value) => { raw.value = value }, {
  initialValue: () => stored.amount
})

const report = (label) => log.value.push(`${label}: box.value=${JSON.stringify(box.value)}, shown=${JSON.stringify(input.value?.value ?? null)}`)

onMounted(() => report('after mount'))

function setFromCode() {
  box.value = '1234567'
  report('after box.value = 1234567')
}

function toggle() {
  if (visible.value) stored.amount = box.value
  visible.value = !visible.value
  setTimeout(() => report(visible.value ? 'input back' : 'input removed'), 50)
}
</script>

<template>
  <input v-if="visible" id="field" ref="input">
  <button id="set" @click="setFromCode">Set from code</button>
  <button id="toggle" @click="toggle">Hide or show</button>
  <button id="destroy" @click="box.destroy()">Destroy</button>
  <p id="raw">{{ raw }}</p>
  <div v-for="(l, i) in log" :key="i" class="row">{{ l }}</div>
</template>
```

```text
shown at start: '500'
typed 4200 -> raw '4200'
input value now '1,234,567', raw '1234567'
after hide and show: shown '1,234,567'
after destroy, typed 99 -> shown '99' raw '1234567'
after mount: box.value="500", shown=""
after box.value = 1234567: box.value="1234567", shown="1,234,567"
input removed: box.value="1234567", shown=null
input back: box.value="1234567", shown="1,234,567"
```

Reading the output as a lesson in the box's life:

- **The start value** `500` appeared in the input as soon as the box was attached. Right inside `onMounted` the input is still empty, because the box attaches just after the first render. Read `box.value` rather than the input there.
- **Assigning `box.value = '1234567'`** reformats the input at once (`1,234,567`), with no event needed. Reading `box.value` gives the raw value.
- **Hiding the input** keeps `box.value`, and **showing it again** puts the last value back, with its format. The value belongs to the box, not to the element.
- **`box.destroy()`** stops the formatting for good. The input keeps whatever text it had, and typing `99` shows `99`.

The ref can point at an `<input>`, a `<textarea>`, a `<select>`, or a component. For a component the box uses the element it exposes as `inputRef` or `textareaRef`, and otherwise the first input inside its root element.

To start from a value the page keeps itself, give a function as `initialValue`. It is read every time the input is attached, so the form data stays the single source of truth:

```js
const { setupNumberBox } = useNumberRefBased()

const input = ref()
const form = reactive({ quantity: '3' })

// The function is read again each time the input appears.
setupNumberBox(input, 6, (raw) => { form.quantity = raw }, {
  initialValue: () => form.quantity
})
```

The returned box has `value` (read and write), `instance` (the attached box, or `null`), `isComplete()` and `destroy()`. For number boxes `isComplete()` always returns `false`, because the check belongs to masked inputs; do not use it to validate a number.

## 6. Putting it together

A price and quantity line, the heart of every order form. The price is a Persian decimal box with two places, the quantity a whole-number box. The total is calculated from the raw values, shown in Persian digits, written in words, and the payload for the server is built from raw values only:

```vue
<script setup>
const { setupPersianDoubleNumberBox, setupNumberBox, getPersianNumber, getPersianWords, getEnglishNumberWithoutCommas } = useNumberRefBased()

const priceInput = ref()
const quantityInput = ref()
const price = ref('0')
const quantity = ref('1')

setupPersianDoubleNumberBox(priceInput, 12, 2, (raw) => { price.value = raw })
setupNumberBox(quantityInput, 4, (raw) => { quantity.value = raw })

// Money is added up in the smallest unit (hundredths), not as floating point.
const totalText = computed(() => {
  const [whole, fraction = ''] = price.value.split('.')
  const hundredths = BigInt(whole + fraction.padEnd(2, '0').slice(0, 2)) * BigInt(quantity.value)
  const text = hundredths.toString().padStart(3, '0')
  const cut = text.length - 2
  const decimals = text.slice(cut)
  return decimals === '00' ? text.slice(0, cut) : `${text.slice(0, cut)}.${decimals.replace(/0$/, '')}`
})

const payload = computed(() => JSON.stringify({
  unitPrice: getEnglishNumberWithoutCommas(price.value),
  quantity: quantity.value,
  total: totalText.value
}))
</script>

<template>
  <input id="price" ref="priceInput" dir="ltr">
  <input id="quantity" ref="quantityInput" dir="ltr">
  <p id="total">{{ getPersianNumber(totalText) }}</p>
  <p id="words">{{ getPersianWords(totalText) }}</p>
  <p id="payload">{{ payload }}</p>
</template>
```

```text
price 1250.5 x 3
  total ۳,۷۵۱٫۵
  words سه هزار و هفتصد و پنجاه و یک ممیز پنج
  payload {"unitPrice":"1250.5","quantity":"3","total":"3751.5"}
price 0.1 x 3
  total ۰٫۳
  words صفر ممیز سه
  payload {"unitPrice":"0.1","quantity":"3","total":"0.3"}
price 19.99 x 12
  total ۲۳۹٫۸۸
  words دویست و سی و نه ممیز هشت هشت
  payload {"unitPrice":"19.99","quantity":"12","total":"239.88"}
```

The page uses almost everything from this guide:

- **Money is added up in hundredths** with `BigInt`. This keeps `19.99 × 12` at exactly `239.88` and not a floating-point neighbour of it:

```js
// 19.99 x 12, added up in hundredths so no floating point is involved.
const hundredths = BigInt('1999') * BigInt('12')     // 23988n
const total = `${hundredths / 100n}.${String(hundredths % 100n).padStart(2, '0')}`   // "239.88"
```

- **The payload carries raw values,** and the total as text, never the display text (chapter 4).
- **The words follow the total as it is typed** (chapter 2). Decimals of the total are read digit by digit.
- **The boxes are set up in `setup()`,** with no `onMounted`, because the ref-based functions wait for the inputs by themselves (chapter 5).

---

## Appendix A. Quick reference

### A.1 Reading and formatting

| Call | Returns | Notes |
|------|---------|-------|
| `getEnglishWords(v)` | `"One Hundred Twenty Three"` | Latin digits only; `""` for empty or `0`; throws over 36 digits |
| `getPersianWords(v)` | `"صد و بیست و سه"` | Same rules |
| `getPersianNumber(v)` | `"۱,۲۳۴٫۵"` | Expects Latin digits |
| `getEnglishNumber(v)` | `"1,234٫5"` | Accepts Persian digits; decimal mark is `٫` |
| `getEnglishNumberWithoutCommas(v)` | `"1234.5"` | The raw value |

### A.2 Number boxes

| Function | Digits shown | Decimals |
|----------|--------------|----------|
| `setupNumberBox` | Latin | No |
| `setupPersianNumberBox` | Persian | No |
| `setupDoubleNumberBox` | Latin | Yes |
| `setupPersianDoubleNumberBox` | Persian | Yes |

| Argument | Meaning | Default |
|----------|---------|---------|
| `id` or ref | The input | none |
| `maxLength` | Digits before the decimal mark | 36 |
| `decimalPlaces` | Digits after it (decimal boxes only) | 36 |
| `onChange` | Called with the raw value on every change | none |
| `options` | Ref-based only; `{ initialValue }` | none |

### A.3 What the ref-based box returns

| Member | Meaning |
|--------|---------|
| `value` | The raw value; assigning to it reformats the input |
| `instance` | The attached box, or `null` while there is no input |
| `isComplete()` | Always `false` for number boxes |
| `destroy()` | Detaches for good |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| `id` not found | `undefined` returned, nothing attached, no error |
| Words for 37 digits or more | `RangeError` |
| Words for empty text, `0` or text without digits | `""` |
| Words for Persian digits | `""` |
| `getPersianNumber` for Persian digits | The number `۰`; the digits are lost |
| Input emptied | `0` |
| Ref-based, no input yet | Waits; attaches when the input appears |
| Ref-based, input removed | Detaches; `box.value` keeps the last value |
| Ref-based, component unmounted | Detaches by itself |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Sending the display text to the server.** Send the raw value.
- **Using `Number()` on display text.** `1,234٫5` is not a number to JavaScript. Use the raw value.
- **Calculating money with floating point.** Use whole units (hundredths) or `BigInt`.
- **Calling an `id` version too early or with a wrong `id`.** The call quietly does nothing. Use the ref-based version when the input comes and goes.
- **The same `id` twice on a page.** Only the first element is found.
- **Forgetting the cleanup** of an `id` box when the input outlives its component.
- **Passing Persian digits to the words functions or to `getPersianNumber`.** Clean them first with `getEnglishNumberWithoutCommas`.
- **Passing the JavaScript number `0`** to the words functions and expecting "Zero". It gives an empty string; pass `"0"`.
- **Not catching the `RangeError`** when the words come from a live input with a large `maxLength`.
- **Expecting a minus sign.** Boxes and words drop it.
- **Using `isComplete()`** to validate. It is always `false` for number boxes.
- **Setting `input.value` from code** on an `id` box without firing `input`.

### C.2 Behaviour worth knowing

- A box reformats on `input` and `paste`; pasted text is cleaned like typed text and replaces the whole value.
- The caret is kept in place while the text is reformatted.
- Leading zeros are removed; trailing zeros after the decimal mark are kept.
- The raw value is stored in the `data-raw` attribute of the input and passed to the callback.
- The callback is also called when the box is attached, with the start value (`0` for an empty input).
- A ref-based box remembers its last value across the input being removed and created again, unless `initialValue` is a function.

### C.3 Limits

| What | Limit |
|------|-------|
| Digits before the decimal mark | `maxLength`, default 36 |
| Digits after it | `decimalPlaces`, default 36 |
| Words | Up to 36 digits (groups up to the largest name), then `RangeError` |
| Negative numbers | Not supported |
| Languages for words | English and Persian |
| Digits accepted when typing | Latin and Persian |

### C.4 Tuning

- Keep numbers as strings from the box to the request body.
- Choose `maxLength` and `decimalPlaces` from the database column, not from habit.
- Use the ref-based version inside modals, tabs and wizard steps.
- Give a function as `initialValue` when the form keeps its own data.
- Show words next to the box only for amounts that people read aloud, and catch the `RangeError`.
- Set `dir="ltr"` on number inputs in right-to-left pages, so the digits and commas keep their order.

## Appendix D. Message reference

### D.1 Errors

| Message | Cause |
|---------|-------|
| `RangeError: Number is too large to convert to words (exceeds supported magnitude).` | `getEnglishWords` with 37 digits or more |
| `RangeError: عدد برای تبدیل به حروف بسیار بزرگ است (خارج از محدوده‌ی پشتیبانی‌شده).` | `getPersianWords` with 37 digits or more |
