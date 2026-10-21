# useDate

*J2OS Platform 2.4*

A registration form asks for a birth date. The visitor expects to type it in the calendar they use every day, which in Iran is the Jalaali (Persian) calendar, and the form expects a date that exists: no 31st of Mehr, no 30th of Esfand in a year that does not have one, nothing outside the years the form allows. A plain text field accepts anything, and a native date picker speaks Gregorian. Checking the text after the fact means error messages and frustrated visitors; it is better to make wrong dates impossible to type.

**useDate** is a composable of the Nuxt front end of J2OS that turns a text field into such a date box. **useDateRefBased** is its twin that finds the field through a ref instead of an id, and this guide covers both. They have three main features:

- **A date box that checks as you type.** `setupProDateBox` accepts only digits that can still lead to a valid date, keeps days inside the month, and offers keyboard shortcuts for stepping through days.
- **A range.** Every box has an earliest and a latest date, and the page is told only about dates inside it.
- **Calendar helpers.** Conversion between Jalaali and Gregorian dates, today's date and the name of the weekday.

Here it is in a nutshell:

```vue
<script setup>
const date = useDate()
const birthDate = ref('')

onMounted(() => {
  date.setupProDateBox(
    'birth',                  // id of the <input>
    1300, 1, 1,               // earliest date: year, month, day
    1410, 1, 1,               // latest date
    (value) => { birthDate.value = value }      // '1403/09/15', or '' when there is no valid date
  )
})
</script>

<template>
  <input id="birth" dir="ltr">
  <p>{{ birthDate || 'not entered' }}</p>
</template>
```

We begin with how the composable works (chapter 1) and the first form (2). Chapter 3 shows what the visitor can do inside the box, and chapter 4 what the page is told, which is the part that most often surprises. Chapter 5 covers the simpler box, chapter 6 working with the box from code, and chapter 7 the calendar helpers. Chapter 8 is the ref-based version, and chapter 9 builds a booking form. The appendices are meant for quick lookups.

## Contents

1. [How useDate works](#1-how-usedate-works)
2. [The first date boxes](#2-the-first-date-boxes)
3. [Typing in the box](#3-typing-in-the-box)
4. [What the page is told](#4-what-the-page-is-told)
5. [The simple box](#5-the-simple-box)
6. [Working with the box from code](#6-working-with-the-box-from-code)
7. [Converting dates and naming days](#7-converting-dates-and-naming-days)
8. [The ref-based version](#8-the-ref-based-version)
9. [A booking form](#9-a-booking-form)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Every example is a self-contained page of a Nuxt project, and each is a single `.vue` file. In Nuxt `ref`, `computed`, `onMounted` and the composables need no `import`, so none is written. The output under a page is what the visitor sees on it as the page is used. Pages that depend on today's date were run with the date set to 1405/01/01 (21 March 2026).

---

## 1. How useDate works

Quick definitions first:

- **Date text.** A Jalaali date written as `YYYY/MM/DD`, such as `1403/09/15`, with two digits for month and day. This is what the boxes show and report.
- **Segment.** One of the three parts of the date text: year, month or day. The visitor edits one segment at a time.
- **Range.** The earliest and latest date a box accepts, given as six numbers: from year, month, day and to year, month, day.
- **Report.** The call to the page's `onChange` function. It receives the date text, or an empty text `''` when the box holds no valid date.

What the composable offers:

| Function | Role |
|----------|------|
| `setupProDateBox(inputId, ..., onChange, initialValue)` | Turns an `<input>` into the full date box |
| `setupDateBox(inputId, ..., onChange, initialValue)` | The simpler box, built on a text mask |
| `getPersianDate(text, inSeparator, outSeparator)` | Gregorian text to Jalaali text |
| `getGregorianDate(text, inSeparator, outSeparator)` | Jalaali text to Gregorian text |
| `getGregorianNowDate()` | Today, as `YYYY-MM-DD` |
| `getWeekDayName(text, language, calendar)` | The weekday of a date, in English or Persian |

The two box functions take the id of an input, the range as six numbers, an optional `onChange` and an optional starting value. They return an object that represents the box. Six facts shape the rest of the guide:

- **The input must exist when the function is called,** so the place for it is `onMounted`.
- **The box works in the Jalaali calendar only.** Gregorian dates exist as helpers, not as input.
- **The page is told about complete, valid, in-range dates, and about nothing else.** Everything else is reported as `''` (chapter 4).
- **The plain version attaches once.** The ref-based version follows the input as it comes and goes (chapter 8).
- **Two libraries must be available as globals.** `jalaali` (the `jalaali-js` package) does the calendar arithmetic and `IMask` (the `imask` package) drives the simple box. The composable does not import them.
- **Latin digits only.** The visitor types `0` to `9`; Persian digits are ignored (chapter 3).

## 2. The first date boxes

A registration form has two dates: the birth date, which the visitor types, and the date of the visit, which starts as today. The page also shows the weekday next to the birth date, because that is a good way for a visitor to notice a typing mistake. The page below is modeled on the project's own example:

```vue
<script setup>
const date = useDate()
const birthDate = ref('')
const visitDate = ref('')
const weekday = ref('')

onMounted(() => {
  // birth date: empty at the start, the visitor types it
  date.setupProDateBox('birth', 1300, 1, 1, 1410, 1, 1, (value) => {
    birthDate.value = value
    weekday.value = date.getWeekDayName(value, 'en', 'jalaali')
  })

  // visit date: starts as today. The box shows the starting value but does not
  // report it, so the page keeps its own copy
  visitDate.value = date.getPersianDate(date.getGregorianNowDate(), '-', '/')
  date.setupProDateBox('visit', 1400, 1, 1, 1410, 1, 1, (value) => {
    visitDate.value = value
  }, visitDate.value)
})
</script>

<template>
  <label>Birth date <input id="birth" dir="ltr"></label>
  <span id="weekday">{{ weekday }}</span>
  <label>Visit date <input id="visit" dir="ltr"></label>
  <p id="summary">Birth date: {{ birthDate || 'not entered' }}. Visit date: {{ visitDate || 'not entered' }}.</p>
</template>
```

```text
Page opens:                            Birth date: not entered. Visit date: 1405/01/01.
Visitor types a birth date:            Birth date: 1403/09/15. Visit date: 1405/01/01. Weekday shown: Thursday.
Visitor replaces the visit date:       Birth date: 1403/09/15. Visit date: 1406/01/01.
Visitor erases part of the visit date: Birth date: 1403/09/15. Visit date: not entered.
Visitor clears the birth date:         Birth date: not entered. Visit date: not entered. Weekday shown: "".
```

Both boxes come from `setupProDateBox`. The first argument is the `id` of the input, then come the earliest date (1300/01/01) and the latest (1410/01/01) as year, month, day, then the function that is called when the date changes. The page copies the value into its own state and asks `getWeekDayName` for the weekday.

Read the output as a sequence of what the page knows. After the birth date was typed in full, the page knew it, and the weekday appeared. When the visit date was replaced, the page followed. When the visitor erased part of it, the page was told `''` and showed "not entered": the box never reports half a date.

The visit date shows the one thing to remember from this chapter. The starting value (today) is **shown** by the box, but the box **does not report it**. That is why the page computes today itself and keeps its own copy:

```js
// The box shows the starting value but does not call onChange for it,
// so the page keeps its own copy.
const visitDate = ref(date.getPersianDate(date.getGregorianNowDate(), '-', '/'))

date.setupProDateBox('visit', 1400, 1, 1, 1410, 1, 1,
  (value) => { visitDate.value = value },
  visitDate.value                 // or the word 'today'
)
```

`initialValue` can also be the word `'today'`, or any date text; if it is not a valid date inside the range, the box starts empty.

## 3. Typing in the box

The date box does not look like a normal text field when it is used, and that is the point. The text always has the shape `____/__/__`, and the visitor edits one segment at a time. Clicking a segment selects it, and typing replaces the selected segment. Tab is left to the browser and moves the focus on as usual.

What the visitor can do, with the real result of each action:

| The visitor | What happens |
|-------------|--------------|
| Types a digit that can still lead to an allowed value | It is written, and the cursor moves on when the segment is full |
| Types a digit that cannot lead to an allowed value | Nothing; the digit is refused. A month cannot start with 13, and a range that starts in 1300 refuses a year starting with 12 |
| Types a month digit from 2 to 9 | Written as `02` to `09`, and the cursor moves on. A `1` waits for the second digit, since 10, 11 and 12 exist |
| Types a day that the month does not have (31 in Mehr) | Corrected to the last day of the month (`1404/07/31` becomes `1404/07/30`) |
| Types 30 Esfand in a year that has only 29 | Corrected to 29. In a leap year such as 1403 the 30th stays |
| Presses the left or right arrow | Moves to the previous or next segment |
| Presses the up or down arrow | Raises or lowers the segment by one; on an empty segment the up arrow starts at the largest value (12 for months, 31 for days). The result stays inside the range, and the day is kept inside the month |
| Presses `+` or `-` | Moves to the next or previous day, across months and years, and stops at the edges of the range |
| Presses `T` | Fills in today's date, if today is inside the range |
| Presses Backspace | Deletes the character before the cursor; in an empty segment it moves to the previous segment instead |
| Presses Delete | Deletes the character after the cursor |
| Presses Home or End | Selects the first (year) or last (day) segment |
| Presses Ctrl+A and then Backspace | Clears the box |
| Pastes text with exactly eight digits (`14030915`, `1403-09-15`) | Written, if it is a valid date inside the range; otherwise nothing happens |
| Presses Enter | Completes pending segments and reports; it does not move the focus or submit a form |
| Types letters, spaces, or Persian digits | Ignored |
| Presses Ctrl+X, Ctrl+Z or other shortcuts | Blocked. Only Ctrl+A, Ctrl+C and Ctrl+V work |

Four of these rows have consequences for a form:

- **Years that do not fit the range cannot be typed.** In a range that starts in 1405, the digits `1404` stop after `140_`. This is better than an error message, but the visitor may wonder why the box ignores them; a hint next to the field helps.
- **Short entries are completed when the visitor leaves.** A single day digit (`1`) becomes `01`, and a single month digit becomes `01`, when the other two segments are filled. A year typed with two digits is completed to the first year of the range that ends with them: in a range from 1300 to 1410, `13` becomes 1313. Tell visitors to type four digits.
- **Persian digits are ignored.** A visitor with a Persian keyboard layout who types `۱۴۰۳` sees nothing happen. Add a hint, or switch the visitor's layout with the page, since the composable does not convert digits.
- **Typing very fast loses keys.** The box moves the cursor a moment after each digit; keys that arrive a few milliseconds apart (an automated test, not a person) land in the wrong segment. In tests, leave about 30 milliseconds between keys.

## 4. What the page is told

Everything the visitor does inside the box is the box's own business. The page hears about it only through `onChange`, and the rule is strict: **the page receives the date text when the box holds a complete, valid date inside the range, and `''` in every other case.**

That includes half-finished dates. While the visitor types `1405/09/`, the page receives `''` on every key press. It also includes a case that is easy to miss: **a complete date outside the range can be typed.** The box refuses digits one segment at a time (a month must be 1 to 12), but it cannot know that `1405/09/15` is outside a range that ends on 1405/06/31 until the last digit is in. The text stays in the box, and the page is told `''`.

The page below allows trips in the first half of 1405. It shows what a visitor who types a date outside the range sees:

```vue
<script setup>
const date = useDate()
const picked = ref('')
const message = ref('Please enter a date.')
let box

function describe(value) {
  if (value) return `Accepted: ${value}.`
  // an empty report is either "not finished" or "finished but not allowed"
  return box.isComplete()
    ? 'That date is outside 1405/01/01 to 1405/06/31.'
    : 'Please enter a date.'
}

onMounted(() => {
  box = date.setupProDateBox('trip', 1405, 1, 1, 1405, 6, 31, (value) => {
    picked.value = value
    message.value = describe(value)
  })
})
</script>

<template>
  <label>Trip date <input id="trip" dir="ltr"></label>
  <p id="message">{{ message }}</p>
</template>
```

```text
Page opens:                                  Please enter a date.
Visitor types the year 1405:                 Please enter a date.  Box: 1405/__/__
Visitor types 09/15:                         That date is outside 1405/01/01 to 1405/06/31.  Box: 1405/09/15
Visitor types 1405/03/15 instead:            Accepted: 1405/03/15.  Box: 1405/03/15
Visitor types 1405/06/31 (last allowed day): Accepted: 1405/06/31.  Box: 1405/06/31
Visitor presses +:                           Accepted: 1405/06/31.  Box: 1405/06/31
```

The output shows three states that all report `''`. At the start the box is empty. After `1405` it is still being typed. After `09/15` it is complete but not allowed. The page can tell these apart by asking the box, which is what `describe` does:

```js
let box

function describe(value) {
  if (value) return 'Accepted: ' + value
  // '' means "no valid date", and there are two reasons for it
  return box.isComplete()
    ? 'That date is not allowed.'        // all digits typed, but outside the range
    : 'Please enter a date.'             // still being typed, or empty
}

box = date.setupProDateBox('trip', 1405, 1, 1, 1405, 6, 31, (value) => {
  message.value = describe(value)
})
```

`box.isComplete()` is `true` only when every digit is in place. Together with the empty report it means "typed, but not allowed", and that is the moment to show a message. The last two lines of the output show the edges of the range: the last allowed day was accepted, and `+` on it did not move the date.

Three smaller rules complete the picture:

- **A starting value is shown, not reported.** The page keeps its own copy (chapter 2).
- **Invalid starting values are dropped.** `1402/13/06`, a date outside the range, or text that is not a date leave the box empty, without a report.
- **Reports happen on every change,** so `onChange` can run many times for one date. Keep it cheap, and do not use it to submit.

## 5. The simple box

`setupDateBox` is the older, simpler sibling. It uses the `IMask` library to force the shape `0000/00/00`, and then checks the result. There are no segments, no arrows, no shortcuts, and no corrections: the visitor types digits, and the box either accepts the whole date or empties itself.

```vue
<script setup>
const date = useDate()
const message = ref('Waiting for a date.')

onMounted(() => {
  date.setupDateBox('invoice', 1400, 1, 1, 1410, 1, 1, (value) => {
    message.value = value ? `Accepted: ${value}.` : 'No valid date.'
  })
})
</script>

<template>
  <label>Invoice date <input id="invoice" dir="ltr"></label>
  <button id="other">Next field</button>
  <p id="message">{{ message }}</p>
</template>
```

```text
Page opens:                                     Waiting for a date.
Visitor types 14030915:                         Accepted: 1403/09/15.  Box: 1403/09/15
Visitor types 14031315 (month 13):              No valid date.  Box: ____/__/__
Visitor types 1403 and moves on:                No valid date.  Box: ____/__/__
Visitor types 14041230 (1404 has no 30 Esfand): No valid date.  Box: ____/__/__
```

A complete, valid date was accepted. The date with month 13 was wiped as soon as the last digit made it invalid, and so was a half-typed date when the visitor left the box. The last line is the difference that matters most: `14041230` is not a date (1404 has only 29 days in Esfand), and the simple box empties itself, while `setupProDateBox` would have corrected the day.

| | `setupProDateBox` | `setupDateBox` |
|---|-------------------|----------------|
| Typing | Segment by segment; impossible digits refused | Any digits; the mask fixes the shape |
| Invalid complete date | Day corrected to the month's length | Whole text wiped |
| Date outside the range | Stays in the box, reported as `''` | Wiped |
| Half-typed date on leaving | Stays; reported as `''` | Wiped |
| Arrows, `+`, `-`, `T` | Yes | No |
| Starting value reported | No | Yes |
| What it returns | `{ value, isComplete }` | The `IMask` object |

For new forms use `setupProDateBox`. The simple box is for places where a plain mask is all that is wanted.

## 6. Working with the box from code

The object returned by `setupProDateBox` is the page's handle on the box. It has two members:

- **`value`** is the text shown by the box (`____/__/__` when empty). Assigning to it sets the box.
- **`isComplete()`** tells whether every digit is in place.

Assigning is where a handle is useful: a "Today" button, a "Reset" button, or filling the box from data loaded from the server. The assignment accepts a valid in-range date or an empty text. Anything else is ignored, silently:

```vue
<script setup>
const date = useDate()
const status = ref('Nothing pressed yet.')
let box

onMounted(() => {
  box = date.setupProDateBox('due', 1400, 1, 1, 1410, 1, 1)
})

const report = label => { status.value = `${label} Box shows ${box.value}, complete: ${box.isComplete()}.` }

function today() {
  box.value = date.getPersianDate(date.getGregorianNowDate(), '-', '/')
  report('Today pressed.')
}
function setDate(text) {
  box.value = text
  report(`Set to ${text || 'an empty text'}.`)
}
</script>

<template>
  <input id="due" dir="ltr">
  <button id="today" @click="today">Today</button>
  <button id="bad" @click="setDate('1405/12/30')">Set 1405/12/30</button>
  <button id="old" @click="setDate('1250/01/01')">Set 1250/01/01</button>
  <button id="clear" @click="setDate('')">Clear</button>
  <p id="status">{{ status }}</p>
</template>
```

```text
Page opens:                     Nothing pressed yet.
Visitor presses Today:          Today pressed. Box shows 1405/01/01, complete: true.
Visitor presses Set 1405/12/30: Set to 1405/12/30. Box shows 1405/01/01, complete: true.
Visitor presses Set 1250/01/01: Set to 1250/01/01. Box shows 1405/01/01, complete: true.
Visitor presses Clear:          Set to an empty text. Box shows ____/__/__, complete: false.
```

"Today" filled the box. The two invalid assignments (a 30th of Esfand in a year that has 29 days, and a year outside the range) were ignored, and the box kept today's date. The empty text cleared the box and `isComplete()` became `false`.

Two details are worth knowing. Assigning does **not** call `onChange`; if the page uses the value, update its own state in the same place as the assignment. And a rejected assignment leaves no trace, so check `box.value` afterwards if the value came from outside.

## 7. Converting dates and naming days

The box speaks Jalaali. The server usually does not, and a few screens want the name of the day. The helpers are plain functions that take text and return text, and when the text is not a date they return an empty text instead of throwing.

```js
// The picker works in Jalaali, the server usually wants Gregorian.
const gregorian = date.getGregorianDate('1405/01/01', '/', '-')      // '2026-03-21'
const jalaali = date.getPersianDate('2026-03-21', '-', '/')          // '1405/01/01'
const dayName = date.getWeekDayName('1405/01/01', 'en')             // 'Saturday'
```

A table of calls and what they return, from a real run on 21 March 2026:

| Call | Result |
|------|--------|
| `getPersianDate('2026-03-21', '-', '/')` | `1405/01/01` |
| `getPersianDate('2026/03/21', '/', '.')` | `1405.01.01` |
| `getPersianDate('2026-3-5', '-', '/')` | `1404/12/14` (no zero padding needed) |
| `getGregorianDate('1405/01/01', '/', '-')` | `2026-03-21` |
| `getGregorianDate('1405/01/01', '/', ',')` | `2026,03,21` |
| `getGregorianDate('1403/12/30', '/', '-')` | `2025-03-20` |
| `getGregorianDate('1405/12/30', '/', '-')` | empty text (1405 has no 30 Esfand) |
| `getGregorianDate('1405-01-01', '/', '-')` | empty text (wrong separator) |
| `getGregorianNowDate()` | `2026-03-21` |
| `getWeekDayName('1405/01/01')` | `Saturday` |
| `getWeekDayName('1405/01/01', 'fa')` | `شنبه` |
| `getWeekDayName('2026/03/21', 'en', 'gregorian')` | `Saturday` |
| `getWeekDayName('1405/12/30')` | empty text |

Some rules follow from the table. The separators describe the **input** and **output**; getting the input separator wrong gives an empty text, not an error. `getWeekDayName` always splits on `/`. Its language is `'en'` (the default) or `'fa'`; any other language gives English. `getGregorianNowDate` uses the visitor's own clock and time zone. Empty results should be treated as "not a date", the same way as `''` from the box.

## 8. The ref-based version

The plain functions find the input by `id`, once, when called. That is a problem for inputs that come and go, the most common case being a dialog: the input does not exist until the dialog opens, and a new one is made every time it opens. `useDateRefBased()` has the same two box functions, but they take a **ref** instead of an id, and the box follows the input:

```vue
<script setup>
const date = useDateRefBased()
const dateInput = ref()                  // a plain <input>, or a component such as UInput
const picked = ref('')

// Called once in setup(): the box attaches itself whenever the input exists.
const box = date.setupProDateBox(
  dateInput,
  1300, 1, 1, 1410, 1, 1,
  (value) => { picked.value = value },
  () => picked.value                     // read every time the input appears
)
</script>

<template>
  <UInput v-if="showDialog" ref="dateInput" dir="ltr" />
</template>
```

```vue
<script setup>
const date = useDateRefBased()
const open = ref(false)
const dateInput = ref()
const picked = ref('')

// set up once; the box attaches itself whenever the input exists
const box = date.setupProDateBox(dateInput, 1300, 1, 1, 1410, 1, 1, (value) => {
  picked.value = value
}, () => picked.value)

const describe = () => `Dialog ${open.value ? 'open' : 'closed'}. Page holds ${picked.value || 'nothing'}.`
</script>

<template>
  <button id="toggle" @click="open = !open">{{ open ? 'Close' : 'Choose a date' }}</button>
  <div v-if="open" style="border: 1px solid #999; padding: 8px; width: 240px">
    <UInput ref="dateInput" dir="ltr" />
  </div>
  <p id="status">{{ describe() }}</p>
  <p id="boxvalue">{{ box.value }}</p>
</template>
```

```text
Page opens:                Dialog closed. Page holds nothing. Box value: .
Visitor opens the dialog:  Dialog open. Page holds nothing. Input shows: ____/__/__
Visitor types a date:      Dialog open. Page holds 1403/09/15. Input shows: 1403/09/15
Visitor closes the dialog: Dialog closed. Page holds 1403/09/15. Box value: 1403/09/15.
Visitor opens it again:    Dialog open. Page holds 1403/09/15. Input shows: 1403/09/15
Visitor changes the date:  Dialog open. Page holds 1404/10/20. Input shows: 1404/10/20
```

Call the setup function once, in `setup()`, not in `onMounted`. The input does not have to exist at that time. The ref may point at a plain `<input>`, or at a component such as `UInput`, in which case the input inside it is used. Whenever the input appears the box is attached, whenever it disappears the box is detached, and when the component is unmounted everything is removed. Nothing has to be called by hand.

The output shows how the page keeps working across the dialog's life. While the dialog was closed there was no input, and the page held the typed date. When the dialog opened again, the new input showed the same date, because the starting value was a **function** (`() => picked.value`) that is read every time an input appears. That is the pattern for any form that has its own data, such as a wizard: the form stays the single source of truth, and the box is only a view.

The returned object has the same `value` and `isComplete()`, plus two members:

- **`instance`** is the box that is attached right now, or `null` while the input does not exist.
- **`destroy()`** detaches the box and stops following the input. After it the input is an ordinary text field again; it keeps whatever text it showed at that moment, placeholder underscores included, and the same happens when the component that made the box is unmounted.

Two behaviors help when the input is not there. Reading `value` returns the last known value (or the starting value) instead of failing, and assigning to `value` is remembered and shown when the input appears. If the starting value is a fixed text instead of a function, the last value reported is shown again when the input is re-created.

## 9. A booking form

Here the pieces are combined in one form. A visitor gives a birth date and a travel date. The travel date cannot be before today, so the range of the second box starts at today's date. The Book button is enabled only when both dates are valid, the weekday of the trip is shown, and the dates are sent in the form the server expects, Gregorian and written `YYYY-MM-DD`:

```vue
<script setup>
const date = useDate()
const birth = ref('')
const travel = ref('')
const weekday = ref('')
const sent = ref('')

onMounted(() => {
  date.setupProDateBox('birth', 1300, 1, 1, 1410, 1, 1, (value) => { birth.value = value })

  // travel dates start today
  const [year, month, day] = date.getPersianDate(date.getGregorianNowDate(), '-', '/').split('/').map(Number)
  date.setupProDateBox('travel', year, month, day, 1410, 1, 1, (value) => {
    travel.value = value
    weekday.value = date.getWeekDayName(value, 'en', 'jalaali')
  })
})

const ready = computed(() => Boolean(birth.value && travel.value))

function submit() {
  // the server expects Gregorian dates in ISO form
  sent.value = JSON.stringify({
    birthDate: date.getGregorianDate(birth.value, '/', '-'),
    travelDate: date.getGregorianDate(travel.value, '/', '-')
  })
}
</script>

<template>
  <label>Birth date <input id="birth" dir="ltr"></label>
  <label>Travel date <input id="travel" dir="ltr"></label>
  <span id="weekday">{{ weekday }}</span>
  <button id="submit" :disabled="!ready" @click="submit">Book</button>
  <p id="sent">{{ sent }}</p>
</template>
```

```text
Page opens:                                     Book button disabled
Visitor types a birth date:                     Book button disabled
Visitor tries to type 1404/12/01, before today: Box shows 140_/__/__. Book button disabled
Visitor types 1405/02/10:                       Book button enabled. Weekday shown: Thursday.
Visitor presses Book:                           server payload {"birthDate":"1999-09-05","travelDate":"2026-04-30"}
```

The range of the travel box is built from today's date. The helper `getPersianDate` returns today in Jalaali text, which is split into three numbers:

```js
// "from today on": the earliest date is today, written as three numbers
const [year, month, day] = date.getPersianDate(date.getGregorianNowDate(), '-', '/')
  .split('/')
  .map(Number)

date.setupProDateBox('travel', year, month, day, 1410, 1, 1, onTravelChange)
```

Follow the output. The button stays disabled while a date is missing, because the page is only told about valid dates. A visitor who tries to type a day before today is stopped by the box (the digits of 1404 are refused), so the page never has to show an error. Once both dates were valid the button was enabled, the weekday appeared, and the payload held the Gregorian dates, converted by `getGregorianDate`.

For your own forms the recipe is the same: give each box a range that fits the question, build "from today" ranges from `getGregorianNowDate`, enable actions only from the reported values, keep your own copy of starting values, show a message when the box is complete but the report is empty, and convert to Gregorian at the last moment.

---

## Appendix A. Quick reference

### A.1 Functions

| Call | Returns | Notes |
|------|---------|-------|
| `setupProDateBox(id, fy, fm, fd, ty, tm, td, onChange, initialValue)` | `{ value, isComplete }` | Full box; needs the input to exist; ref-based version also returns `instance` and `destroy` |
| `setupDateBox(id, fy, fm, fd, ty, tm, td, onChange, initialValue)` | The `IMask` object | Simple box; reports the starting value |
| `getPersianDate(text, inSep, outSep)` | Text | Gregorian to Jalaali; `''` if not a date |
| `getGregorianDate(text, inSep, outSep)` | Text | Jalaali to Gregorian; `''` if not a date |
| `getGregorianNowDate()` | Text | Today as `YYYY-MM-DD`, in the visitor's time zone |
| `getWeekDayName(text, language, calendar)` | Text | `'en'` or `'fa'`; `'jalaali'` (default) or `'gregorian'`; `''` if not a date |

### A.2 Keys of the full box

| Key | Action |
|-----|--------|
| Digits | Fill the current segment |
| Left, Right | Previous, next segment |
| Up, Down | Raise, lower the segment |
| `+`, `-` | Next, previous day |
| `T` | Today |
| Backspace, Delete | Delete characters |
| Home, End | First, last segment |
| Enter | Complete and report |
| Ctrl+A, Ctrl+C, Ctrl+V | Select all, copy, paste |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Input id not found | The setup call fails with a `TypeError` |
| Starting value invalid or outside the range | The box starts empty; nothing is reported |
| Assigned value invalid or outside the range | Ignored; the box keeps its text |
| Complete date outside the range typed | Stays in the box (full box), reported as `''`; wiped (simple box) |
| Digit that cannot lead to an allowed value | Refused |
| Day beyond the month's length | Corrected to the last day (full box); wiped (simple box) |
| Paste of something that is not a valid in-range date | Ignored |
| Conversion helper given a bad date or wrong separator | Returns `''` |
| `jalaali` or `IMask` not available | Calls fail with a `ReferenceError` |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Calling a setup function while the component is being set up** (plain version). The input is not there yet; use `onMounted`.
- **Expecting the starting value to be reported.** It is shown only; keep your own copy.
- **Treating `''` as "the visitor left it empty".** It also means unfinished, and complete but outside the range.
- **Assigning to `value` and expecting `onChange` to run.** It does not.
- **Trusting an assignment.** Invalid values are ignored without any sign.
- **Persian digits.** They are ignored by the box.
- **Letting the range exclude the digits the visitor expects.** A year below the range cannot be typed; say why.
- **Submitting from `onChange`.** It runs on every change, including for half-typed dates.
- **Passing the wrong separator to a converter.** The result is an empty text.
- **Using the simple box where corrections are wanted.** It wipes invalid dates.
- **Sending keys too fast in tests.** Leave about 30 milliseconds.

### C.2 Behavior worth knowing

- Keys are handled on `keydown`; Tab and Shift are left alone.
- The cursor is placed a moment after the text changes, which is why very fast key sequences go wrong.
- Reports are made on every change, not on leaving the field.
- When the input loses focus, a single-digit day or month and a two-digit year are completed, and the day is kept inside the month.
- `getGregorianNowDate` and the `T` key use the browser's clock, so tests need a fixed clock.

### C.3 Limits

| What | Limit |
|------|-------|
| Calendar in the box | Jalaali only |
| Digits | `0` to `9` only |
| Date text | `YYYY/MM/DD`, always with the full width of each part |
| Boxes per setup call | One input |
| Time of day | Not handled |
| Information outside `onChange` | `value` and `isComplete()` |

### C.4 Tuning

- Give each box the narrowest range that is true for the question, and show it near the field.
- Build "from today" ranges from `getGregorianNowDate` and `getPersianDate`.
- Keep a copy of the value in page state, including the starting value.
- Show the "not allowed" message only when `isComplete()` is true and the report is empty.
- Convert to Gregorian when sending, and keep Jalaali in the interface.
- Use the ref-based version for dialogs, wizards and any input that is re-created.
- Set `dir="ltr"` on the input so that the date does not reverse in right-to-left pages.

## Appendix D. Message reference

### D.1 Errors

| Message | Cause |
|---------|-------|
| A `TypeError` when the setup function runs | The id given to a setup function matches no element |
| `ReferenceError: jalaali is not defined` / `IMask is not defined` | The libraries are not available as globals |

### D.2 Return values that mean "no date"

| Value | Meaning |
|-------|---------|
| `''` from `onChange` | No complete, valid, in-range date |
| `''` from a converter or `getWeekDayName` | The text is not a valid date, or the separator is wrong |
