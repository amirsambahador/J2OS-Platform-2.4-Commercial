# useString

*J2OS Platform 2.4*

A table of news items has a column for the summary, and some summaries run to a paragraph. A notification has room for a line. A tooltip should show a hint, not an essay. In each case the text has to be shortened, and a few dots at the end tell the reader that there is more.

**useString** is a small composable of the Nuxt front end of J2OS that shortens text. It has three features:

- **One function.** `truncate(text, maxLength)` returns either the text itself, or its beginning followed by three dots.
- **Short text is left alone.** Text that already fits, and empty values, come back untouched.
- **A plain cut.** It counts and cuts characters. It knows nothing about words, sentences or HTML.

In a few lines:

```js
const { truncate } = useString()

truncate('Quarterly report for the northern branch', 20)   // 'Quarterly report for...'
truncate('Holiday schedule', 20)                           // 'Holiday schedule'
```

Here is the route through the guide: chapter 1 explains how the composable works and chapter 2 shows the first use. Chapter 3 covers where the cut falls, chapter 4 the inputs that behave unusually and chapter 5 how to cut more neatly. Chapter 6 builds a list of articles with a "Show more" button. The appendices are meant for quick lookups.

## Contents

1. [How useString works](#1-how-usestring-works)
2. [The first use](#2-the-first-use)
3. [Where the cut falls](#3-where-the-cut-falls)
4. [Unusual inputs](#4-unusual-inputs)
5. [Cutting neatly](#5-cutting-neatly)
6. [Everything in one application](#6-everything-in-one-application)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Each example is a page you can drop into a Nuxt project, and each is a single `.vue` file. In Nuxt `ref`, `computed` and `useString` need no `import`, so none is written.

---

## 1. How useString works

Some terms first; later chapters take them for granted:

- **Limit.** The number of characters of the original text that are kept. The default is 100.
- **Dots.** The three characters `...` that are added after a cut text. They are three plain full stops, not the single ellipsis character.
- **Character.** Here, one unit of the text as JavaScript counts it. Latin letters, Persian letters, digits and spaces are one each. An emoji is usually two.

The function of this composable:

| Piece | Role |
|-------|------|
| `truncate(text, maxLength)` | Returns the text itself, or its first `maxLength` characters followed by `...` |

That is the whole interface, and the function itself is short enough to read in full:

```js
function truncate(text, maxLength = 100) {
  if (!text || text.length <= maxLength) {
    return text
  }

  return `${text.slice(0, maxLength)}...`
}
```

Remember five things:

- **The rule has two parts.** A value that is empty (or any falsy value) comes back as it is; so does a text no longer than the limit. Everything else is cut.
- **The dots are added on top of the limit.** The cut text keeps `maxLength` characters and then gets three more. A text just over the limit becomes longer, not shorter (chapter 3).
- **The input is not changed.** The function returns a new text and keeps nothing between calls.
- **The input must be text.** A number or an object that is not a list makes the function throw a `TypeError` (chapter 4).
- **The cut ignores the meaning of the text.** It falls wherever the count says, in the middle of a word or even of an emoji (chapter 5).

## 2. The first use

Lists and cards have little room, but titles have no fixed length. The first job of `truncate` is to cut a long title so that it fits, and to leave a short title alone. The page below gives it a few titles of different lengths, all with a limit of 20:

```vue
<script setup>
const string = useString()

const titles = [
  'Quarterly report for the northern branch',
  'Holiday schedule',
  'گزارش فصلی فروش شعبهٔ شمال کشور'
]

const short = titles.map(title => string.truncate(title, 20))
</script>

<template>
  <div v-for="(title, i) in short" :key="i" class="row">{{ title }}</div>
</template>
```

```text
Quarterly report for...
Holiday schedule
گزارش فصلی فروش شعبه...
```

The first title is longer than 20 characters and loses its end. The second already fits and stays as it is, so the call is safe to apply to every title without checking first. The Persian title is cut like the English one: its letters are single characters, so counting works the same.

## 3. Where the cut falls

Most unexpected behaviors of `truncate` come from one fact: **the limit is the number of characters kept, not the length of the result.** The dots are added after the cut, so a cut text is three characters longer than the limit. The page below tries texts of 9, 10, 11 and 13 characters, all with a limit of 10:

```vue
<script setup>
const { truncate } = useString()

const alphabet = 'abcdefghijklm'

const atTen = [9, 10, 11, 13].map((n) => {
  const text = alphabet.slice(0, n)
  const result = truncate(text, 10)
  return `limit 10, text of ${n}: ${result} (${result.length})`
})

const long = 'x'.repeat(120)
const byDefault = truncate(long)
const explicit = truncate(long, undefined)
</script>

<template>
  <div v-for="line in atTen" :key="line" class="row">{{ line }}</div>
  <div class="row">default limit, text of 120: {{ byDefault.length }} characters, ends with "{{ byDefault.slice(-5) }}"</div>
  <div class="row">limit passed as undefined: {{ explicit.length }} characters</div>
</template>
```

```text
limit 10, text of 9: abcdefghi (9)
limit 10, text of 10: abcdefghij (10)
limit 10, text of 11: abcdefghij... (13)
limit 10, text of 13: abcdefghij... (13)
default limit, text of 120: 103 characters, ends with "xx..."
limit passed as undefined: 103 characters
```

A text at or under the limit comes back whole, and the limit itself is allowed (9 and 10 characters). A text one over the limit (11 characters) is cut to 10 and gets the dots, so the result has 13 characters, longer than the original. This is what "the dots are added on top of the limit" means, and it is the reason a text just over the limit can look worse after the cut.

Without a limit the default 100 applies, and passing `undefined` is the same as passing nothing: a text of 120 characters becomes 103.

If the whole result, dots included, must fit in a space, pass a limit three below it: `truncate(text, 20 - 3)`. This also shortens texts that would have fitted, so compare the lengths first if that matters.

## 4. Unusual inputs

Texts come from servers, forms and other code, and they are not always what you expect. A title can be missing, a count can arrive where a text was expected, a limit can come from a form field. The page below tries these cases on purpose:

```vue
<script setup>
const { truncate } = useString()

const show = (v) => (v === undefined ? 'undefined' : Number.isNaN(v) ? 'NaN' : JSON.stringify(v))

const cases = [
  ['null', () => truncate(null, 3)],
  ['undefined', () => truncate(undefined, 3)],
  ['empty text', () => truncate('', 3)],
  ['the number 0', () => truncate(0, 3)],
  ['false', () => truncate(false, 3)],
  ['the number 12345', () => truncate(12345, 3)],
  ['a short list', () => truncate(['a', 'b'], 3)],
  ['a list of 8 numbers', () => truncate([1, 2, 3, 4, 5, 6, 7, 8], 3)],
  ['limit 0', () => truncate('abcdef', 0)],
  ['limit -2', () => truncate('abcdef', -2)],
  ['limit null', () => truncate('abcdef', null)],
  ['limit NaN', () => truncate('abcdef', NaN)],
  ['limit "3" (a text)', () => truncate('abcdef', '3')],
  ['limit 2.7', () => truncate('abcdef', 2.7)]
].map(([label, run]) => {
  try { return { label, value: show(run()) } } catch (e) { return { label, value: e.constructor.name + ': ' + e.message } }
})
</script>

<template>
  <div v-for="c in cases" :key="c.label" class="row">{{ c.label }} → {{ c.value }}</div>
</template>
```

```text
null → null
undefined → undefined
empty text → ""
the number 0 → 0
false → false
the number 12345 → TypeError: text.slice is not a function
a short list → ["a","b"]
a list of 8 numbers → "1,2,3..."
limit 0 → "..."
limit -2 → "abcd..."
limit null → "..."
limit NaN → "..."
limit "3" (a text) → "abc..."
limit 2.7 → "ab..."
```

The function is simple, and its behavior follows from its few lines of code. Two groups are worth remembering.

**The value.** Empty values pass through unchanged: `null`, `undefined`, an empty text, `0` and `false` all come back as they were. `truncate` never turns them into an empty text, so a caller may get back a value that is not text, and a template that shows it should expect that. Anything else that is not text fails. A number such as `12345` has no `length` to compare and no `slice`, so the call ends with a `TypeError`, and `true` and a plain object do the same. A list (array) works only partly (a short one comes back as a list, a long one is cut and turned into one text such as `"1,2,3..."`), and you should not rely on it.

**The limit.** The function does not check it, and some values give strange but predictable results: a limit of `0` gives just the dots, a negative limit cuts that many characters off the end instead (`-2` turns `abcdef` into `abcd...`), and a limit that is `null`, `NaN` or not a number gives just the dots. A limit written as text works if it looks like a number (`"3"`), and a decimal is rounded down (`2.7` keeps two).

So a value that may not be text must be converted, and a limit that comes from a form or a setting needs a guard:

```js
truncate(String(value ?? ''), 40)
```

```js
// Never below 10, and 50 when the box holds something that is not a number.
const limit = computed(() => Math.max(10, Math.floor(Number(raw.value)) || 50))
```

## 5. Cutting neatly

Because the cut ignores the meaning of the text, three things can look wrong to a reader:

- **A word is cut in half.** "The warehouse opens at nine" with a limit of 20 ends in "The warehouse opens ", and with a limit of 24 in "The warehouse opens at n".
- **A space sits before the dots,** when the cut falls right after a space.
- **An emoji is cut in half.** An emoji takes two characters, and a cut between them leaves a broken symbol.

A small helper on top of `truncate` fixes all three problems. It cuts back to the last space (unless the cut already falls on one), drops half an emoji and trims the end. The page below compares the plain cut with the neat one:

```vue
<script setup>
const string = useString()

function neat(text, max) {
  const cut = string.truncate(text, max)
  if (cut === text) return text

  let body = cut.slice(0, -3)

  const code = body.charCodeAt(body.length - 1)
  if (code >= 0xD800 && code <= 0xDBFF) body = body.slice(0, -1)

  if (!/\s/.test(text[max])) {
    const space = body.lastIndexOf(' ')
    if (space > 0) body = body.slice(0, space)
  }

  return body.trimEnd() + '...'
}

const samples = [
  ['English', 'The warehouse opens at nine every weekday morning', 20],
  ['Persian', 'انبار هر روز هفته ساعت نه صبح باز می‌شود', 18],
  ['one long word', 'Supercalifragilisticexpialidocious', 10],
  ['cut right after a space', 'one two three four', 8],
  ['emoji', 'Sale 🎉🎉🎉 starts today', 6]
]

const rows = samples.map(([label, text, max]) => {
  const plain = string.truncate(text, max)
  return {
    label,
    plain,
    broken: /[\ud800-\udbff]$/.test(plain.slice(0, -3)) ? 'yes' : 'no',
    neat: neat(text, max)
  }
})
</script>

<template>
  <div v-for="r in rows" :key="r.label" class="row">{{ r.label }} | plain: {{ r.plain }} (half an emoji: {{ r.broken }}) | neat: {{ r.neat }}</div>
</template>
```

```text
English | plain: The warehouse opens ... (half an emoji: no) | neat: The warehouse opens...
Persian | plain: انبار هر روز هفته ... (half an emoji: no) | neat: انبار هر روز هفته...
one long word | plain: Supercalif... (half an emoji: no) | neat: Supercalif...
cut right after a space | plain: one two ... (half an emoji: no) | neat: one two...
emoji | plain: Sale �... (half an emoji: yes) | neat: Sale...
```

The English and Persian results now end on a whole word, where the plain version ends with a space before the dots in both. Persian words are separated by spaces, so the same helper works, and a word joined by the half-space (ZWNJ) counts as one word. A single long word has no space to cut back to, so it is cut inside, as before. The emoji case shows the plain cut leaving a broken symbol and the helper removing it.

Any shortening needs two cautions:

- **Cut text, not HTML.** A cut can fall inside a tag or an entity (`<b>`, `&amp;`) and break the markup. Cut the plain text first and escape it after.
- **Do not lose the full text.** The cut version is only a hint. Keep the whole text reachable, for instance in a `title` attribute:

```vue
<p :title="text">{{ truncate(text, 80) }}</p>
```

When the aim is only to fit a line in a box, a CSS rule that clips the text and adds an ellipsis follows the real width of the box and is often the better tool. `truncate` is the right tool when the count of characters matters, such as a fixed length in a table, a message or a log.

## 6. Everything in one application

Let us build a list of articles. Each shows the beginning of its text with the dots, and a "Show more" button appears only on the ones that were actually cut. A box sets the limit, with a guard against nonsense:

```vue
<script setup>
const string = useString()

const articles = [
  { id: 1, title: 'Opening hours', text: 'The main warehouse opens at nine every weekday morning and closes at five. On Fridays the last pickup is at noon.' },
  { id: 2, title: 'انبار مرکزی', text: 'انبار مرکزی هر روز کاری ساعت نه صبح باز می‌شود و ساعت پنج بعدازظهر بسته می‌شود. روزهای جمعه آخرین تحویل ظهر است.' },
  { id: 3, title: 'Contact', text: 'Call us.' }
]

const raw = ref('50')
const limit = computed(() => Math.max(10, Math.floor(Number(raw.value)) || 50))
const open = ref({})

function neat(text, max) {
  const cut = string.truncate(text, max)
  if (cut === text) return text
  let body = cut.slice(0, -3)
  if (!/\s/.test(text[max])) {
    const space = body.lastIndexOf(' ')
    if (space > 0) body = body.slice(0, space)
  }
  return body.trimEnd() + '...'
}

const view = (a) => (open.value[a.id] ? a.text : neat(a.text, limit.value))
const cut = (a) => neat(a.text, limit.value) !== a.text
</script>

<template>
  <input id="limit" v-model="raw">
  <article v-for="a in articles" :key="a.id" class="item" :data-id="a.id">
    <h4>{{ a.title }}</h4>
    <p class="text" :title="a.text">{{ view(a) }}</p>
    <button v-if="cut(a)" class="more" @click="open[a.id] = !open[a.id]">{{ open[a.id] ? 'Show less' : 'Show more' }}</button>
  </article>
</template>
```

```text
limit 50
  Opening hours | The main warehouse opens at nine every weekday... | button=Show more
  انبار مرکزی | انبار مرکزی هر روز کاری ساعت نه صبح باز می‌شود و... | button=Show more
  Contact | Call us. | button=None
after Show more on the first
  Opening hours | The main warehouse opens at nine every weekday morning and closes at five. On Fridays the last pickup is at noon. | button=Show less
  انبار مرکزی | انبار مرکزی هر روز کاری ساعت نه صبح باز می‌شود و... | button=Show more
  Contact | Call us. | button=None
limit 200
  Opening hours | The main warehouse opens at nine every weekday morning and closes at five. On Fridays the last pickup is at noon. | button=None
  انبار مرکزی | انبار مرکزی هر روز کاری ساعت نه صبح باز می‌شود و ساعت پنج بعدازظهر بسته می‌شود. روزهای جمعه آخرین تحویل ظهر است. | button=None
  Contact | Call us. | button=None
limit 3 (page raises it to 10)
  Opening hours | The main... | button=Show more
  انبار مرکزی | انبار... | button=Show more
  Contact | Call us. | button=None
limit abc (page falls back to 50)
  Opening hours | The main warehouse opens at nine every weekday... | button=Show more
  انبار مرکزی | انبار مرکزی هر روز کاری ساعت نه صبح باز می‌شود و... | button=Show more
  Contact | Call us. | button=None
```

How the page applies the points of the earlier chapters:

- **The button depends on the cut itself.** The comparison `neat(...) !== text` decides whether a button is needed, so the button and the cut can never disagree: the short article ("Call us.") never gets one, and a big limit (200) removes every button.
- **The button toggles one article** without touching the others.
- **The limit goes through the guard of chapter 4.** A tiny limit (3) is raised to the page's minimum of 10, and a non-number falls back to 50.

---

## Appendix A. Quick reference

### A.1 The function

| Call | Returns | Notes |
|------|---------|-------|
| `truncate(text, maxLength)` | The text, or its first `maxLength` characters plus `...` | `maxLength` defaults to 100 |

### A.2 What comes back

| Input | Result |
|-------|--------|
| Empty value (`null`, `undefined`, `''`, `0`, `false`) | The same value |
| Text with length up to the limit | The same text |
| Text longer than the limit | First `maxLength` characters plus `...` |
| A number (not 0), `true` or an object that is not a list | `TypeError` |
| A list longer than the limit | One text of the cut list, plus `...` |

### A.3 What a limit does

| Limit | Effect |
|-------|--------|
| Omitted or `undefined` | 100 |
| 0 | Only `...` (for a non-empty text) |
| Negative | Cuts that many characters off the end |
| Decimal | Rounded down |
| `null` or `NaN` | Only `...` |
| `Infinity` | Never cuts |
| A text that looks like a number | Used as that number |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Value is `null`, `undefined`, `''`, `0` or `false` | Returned unchanged, no error |
| Value is a number (not 0), `true` or a plain object | `TypeError: text.slice is not a function` |
| Value is a long list | A single text, not a list |
| Limit is not a usable number | Only `...` is returned for a non-empty text |
| Cut falls inside an emoji | A broken symbol before the dots |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Expecting the result to fit the limit.** The dots add three characters, so a text just over the limit comes back longer.
- **Passing a number.** Convert it to text first.
- **Using a limit from outside.** A negative, `NaN` or empty value gives strange cuts. Guard it.
- **Cutting words and emoji.** The cut ignores the meaning of the text; use a helper when it shows.
- **Cutting HTML.** A cut can break a tag. Shorten plain text.
- **Hiding the only copy.** Anything cut from the screen should still be reachable.
- **Counting on empty values becoming text.** `null` stays `null`.

### C.2 Behavior worth knowing

- The function is pure: it changes nothing and remembers nothing.
- Lengths count UTF-16 units. Persian letters, digits and spaces are one each; an emoji is two.
- The dots are three full stops, the same for every language.
- A text of exactly the limit is left whole.
- The result of a cut is not trimmed, so it may end with a space before the dots.
- No setup is needed and nothing is read from the page.

### C.3 Limits

| What | Limit |
|------|-------|
| Default limit | 100 characters |
| Dots added | 3 characters |
| Word, sentence or HTML awareness | None |
| Validation of the limit | None |
| Text length | Whatever the browser can hold |

### C.4 Tuning

- Pass `limit - 3` when the result, dots included, must fit.
- Compare the lengths before calling when the result will be compared or a button depends on it.
- Convert non-text values with `String(value ?? '')`.
- Guard limits that come from users or settings.
- Use a small helper for word and emoji safety, as in chapter 5.
- Use a style rule instead when only the width of a box matters.

## Appendix D. Message reference

### D.1 Error messages

| Message | Cause |
|---------|-------|
| `text.slice is not a function` (`TypeError`) | The value is neither text nor a list, and it is not empty |
