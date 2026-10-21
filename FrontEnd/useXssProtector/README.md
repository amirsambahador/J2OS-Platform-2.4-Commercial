# useXssProtector

*J2OS Platform 2.4*

A comment box, a product description, a profile "about" field: wherever visitors can write text, someone writes `<script>` instead. If the page shows that text as HTML, the script runs in other visitors' browsers and can read their session, click buttons for them or redirect them to another page. This attack is called cross-site scripting, or XSS.

**useXssProtector** is a small composable of the Nuxt front end of J2OS that sanitizes text before the page shows it as HTML. It wraps the DOMPurify library and has three main features:

- **Two functions.** `toDisplayHtml` keeps safe markup (bold, lists, links) and removes the dangerous parts; `toPlainText` removes all markup.
- **Works on any input.** It accepts text, numbers and even `null` or `undefined`, and gives back a string.
- **No state.** Each call stands alone. Nothing is configured, stored or remembered.

The basic use, briefly:

```js
const { toDisplayHtml, toPlainText } = useXssProtector()

toDisplayHtml('<b>Hi</b><img src=x onerror=steal()>')   // '<b>Hi</b><img src="x">'
toPlainText('<b>Hi</b> there')                           // 'Hi there'
```

The order is this: chapter 1 explains how the composable works. Chapter 2 covers showing HTML, chapter 3 removing tags, chapter 4 options, chapter 5 unusual values and links, and chapter 6 the places where J2OS components use it. Chapter 7 builds a small comment list. The appendices are meant for quick lookups.

## Contents

1. [How useXssProtector works](#1-how-usexssprotector-works)
2. [Showing HTML safely](#2-showing-html-safely)
3. [Removing tags](#3-removing-tags)
4. [Options](#4-options)
5. [Unusual values and links](#5-unusual-values-and-links)
6. [Where J2OS uses it](#6-where-j2os-uses-it)
7. [From start to finish](#7-from-start-to-finish)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Every example is a self-contained page of a Nuxt project, and each is a single `.vue` file. In Nuxt `ref` and `useXssProtector` need no `import`, so none is written.

---

## 1. How useXssProtector works

A few words need pinning down before we go on:

- **Sanitizing.** Examining a piece of HTML and removing everything that could run code, while keeping the rest.
- **Allow list.** The set of tags and attributes that are kept. DOMPurify has a default list; options change it.
- **HTML context.** A place where text is read as markup, such as `v-html` or a column with `render: true`. This is where sanitizing is needed.
- **Text context.** A place where text is shown as plain characters, such as `{{ value }}` in a template. Vue escapes text there by itself.

Its functions at a glance:

| Function | Role |
|----------|------|
| `toDisplayHtml(value, options)` | Sanitizes HTML: safe tags stay, dangerous parts are removed |
| `toPlainText(value)` | Removes every tag; only text stays |

Hold on to these five points:

- **The page needs DOMPurify.** The composable calls a global `DOMPurify`. The project must load that library in the browser; without it, both functions throw an error (Appendix B).
- **It works only in the browser.** DOMPurify needs a real document. Call the functions in the browser, not while the server renders the page.
- **Everything is converted to a string.** `null` and `undefined` give `''`; other values go through `String()`. The result is a string unless an option such as `RETURN_DOM` asks for something else (chapter 4).
- **It sanitizes markup, not meaning.** A link to a malicious site, or an offensive word, is not removed.
- **Sanitizing belongs at the point of display.** Sanitize the text where it is shown as HTML, even if the server sanitized it before.

## 2. Showing HTML safely

When text contains markup you want to show, `v-html` is the way, and it is also the most dangerous way. The first pane below uses `v-html` on the raw text; the second sanitizes the text first:

```vue
<script setup>
const { toDisplayHtml } = useXssProtector()

const comment = '<b>Nice!</b> <img src="x" onerror="window.__fired = (window.__fired || []).concat(\'img\')">'
</script>

<template>
  <div id="raw" v-html="comment" />
  <div id="clean" v-html="toDisplayHtml(comment)" />
</template>
```

```text
raw pane html: <b>Nice!</b> <img src="x" onerror="window.__fired = (window.__fired || []).concat('img')">
clean pane html: <b>Nice!</b> <img src="x">
handlers that fired: ['img'] | dialogs/errors: []
```

The comment has an image that cannot load, with an `onerror` handler. In the raw pane the handler runs (`['img']`); in the sanitized pane the image stays but the handler is removed.

In a template, the pattern is just one line:

```vue
<div v-html="toDisplayHtml(comment.text)" />
```

How `toDisplayHtml` treats common input:

```vue
<script setup>
const { toDisplayHtml } = useXssProtector()

const samples = [
  '<b>Bold</b> and <i>italic</i>',
  '<a href="https://example.com" target="_blank">link</a>',
  '<a href="javascript:alert(1)">bad link</a>',
  '<img src="x" onerror="alert(1)">',
  '<script>alert(1)<' + '/script><p>after</p>',
  '<p style="color:red" onclick="alert(1)">styled</p>',
  '<iframe src="https://example.com"></iframe>text',
  '<ul><li>one</li><li>two</li></ul>'
]
</script>

<template>
  <ul>
    <li v-for="(html, i) in samples" :key="i" class="row">{{ html }}  =>  {{ toDisplayHtml(html) }}</li>
  </ul>
</template>
```

```text
<b>Bold</b> and <i>italic</i> => <b>Bold</b> and <i>italic</i>
<a href="https://example.com" target="_blank">link</a> => <a href="https://example.com">link</a>
<a href="javascript:alert(1)">bad link</a> => <a>bad link</a>
<img src="x" onerror="alert(1)"> => <img src="x">
<script>alert(1)</script><p>after</p> => <p>after</p>
<p style="color:red" onclick="alert(1)">styled</p> => <p style="color:red">styled</p>
<iframe src="https://example.com"></iframe>text => text
<ul><li>one</li><li>two</li></ul> => <ul><li>one</li><li>two</li></ul>
```

- **Formatting stays:** `b`, `i`, `p`, `ul` and `li` pass through unchanged.
- **Scripts are removed,** with their content. The paragraph after the script stays.
- **Event handlers are removed:** `onclick`, `onerror` and the like are removed from any tag.
- **`javascript:` links lose their address.** The link text stays, but it leads nowhere.
- **`iframe` is removed;** the text around it stays.
- **`style` stays.** Colors and sizes written in a style attribute are kept. If visitors must not restyle the page, restrict the attributes (chapter 4).
- **`target` is removed** from links by default.

## 3. Removing tags

`toPlainText` removes all markup and keeps the text:

```vue
<script setup>
const { toPlainText } = useXssProtector()

const samples = [
  '<b>Hello</b> <i>world</i>',
  '<script>alert(1)<' + '/script>Safe text',
  '<img src=x onerror=alert(1)>Photo',
  'Tom & Jerry',
  'Tom & <b>Jerry</b>',
  'a < b and c > d',
  null
]
</script>

<template>
  <ul>
    <li v-for="(text, i) in samples" :key="i" class="row">{{ JSON.stringify(text) }} => {{ JSON.stringify(toPlainText(text)) }}</li>
  </ul>
</template>
```

```text
"<b>Hello</b> <i>world</i>" => "Hello world"
"<script>alert(1)</script>Safe text" => "Safe text"
"<img src=x onerror=alert(1)>Photo" => "Photo"
"Tom & Jerry" => "Tom & Jerry"
"Tom & <b>Jerry</b>" => "Tom &amp; Jerry"
"a < b and c > d" => "a &lt; b and c &gt; d"
null => ""
```

- **Tags and scripts are removed,** and only the readable text remains.
- **A string without `<` comes back unchanged.** `Tom & Jerry` is the same text.
- **A string with `<` comes back HTML-escaped.** `Tom & <b>Jerry</b>` becomes `Tom &amp; Jerry`, and `a < b` becomes `a &lt; b`. The result is safe to put into HTML, but if it is shown with `{{ }}` the visitor sees `&amp;` and `&lt;` as characters.

That last point shows where `toPlainText` fits. To show visitor text as characters in a template, write `{{ text }}`: Vue escapes it by itself and nothing more is needed. `toPlainText` is for places where text must lose its tags first and is then read as HTML, or where tags are not expected, such as a short excerpt or a heading in a `v-text`.

## 4. Options

`toDisplayHtml` takes a second argument, passed to DOMPurify as its configuration:

```js
toDisplayHtml(html, { ALLOWED_TAGS: ['b', 'i'] })       // only these tags survive
toDisplayHtml(html, { FORBID_TAGS: ['a'] })             // everything default, except links
toDisplayHtml(html, { ADD_ATTR: ['target'] })           // also keep the target attribute
```

```vue
<script setup>
const { toDisplayHtml } = useXssProtector()

const html = '<h1>Title</h1><p>Hello <b>bold</b> <a href="https://example.com" target="_blank">link</a></p>'

const results = [
  ['default', toDisplayHtml(html)],
  ['only b and i', toDisplayHtml(html, { ALLOWED_TAGS: ['b', 'i'] })],
  ['without links', toDisplayHtml(html, { FORBID_TAGS: ['a'] })],
  ['keep target', toDisplayHtml(html, { ADD_ATTR: ['target'] })]
]
</script>

<template>
  <ul>
    <li v-for="[name, out] in results" :key="name" class="row">{{ name }}: {{ out }}</li>
  </ul>
</template>
```

```text
default: <h1>Title</h1><p>Hello <b>bold</b> <a href="https://example.com">link</a></p>
only b and i: TitleHello <b>bold</b> link
without links: <h1>Title</h1><p>Hello <b>bold</b> link</p>
keep target: <h1>Title</h1><p>Hello <b>bold</b> <a href="https://example.com" target="_blank">link</a></p>
```

- **`ALLOWED_TAGS`** makes the allow list exactly what is given. Tags outside the list lose their markup, but their text stays: "Title" and "Hello" run together.
- **`FORBID_TAGS`** removes some tags from the default list; the link is removed and its text stays.
- **`ADD_ATTR`** adds attributes to the default list. `target` is kept this time.

The full list of options is in the DOMPurify documentation. `toPlainText` takes no options. Options that change the type of the result, such as `RETURN_DOM`, make `toDisplayHtml` return a DOM node instead of a string, even for text without tags; such a result cannot be put into `v-html`.

## 5. Unusual values and links

The functions accept any value:

```vue
<script setup>
const { toDisplayHtml, toPlainText } = useXssProtector()

const values = [undefined, null, 0, 42, true, '', { a: 1 }, 'javascript:alert(1)']
</script>

<template>
  <ul>
    <li v-for="(v, i) in values" :key="i" class="row">{{ String(v) }} => html {{ JSON.stringify(toDisplayHtml(v)) }}, text {{ JSON.stringify(toPlainText(v)) }}</li>
  </ul>
</template>
```

```text
undefined => html "", text ""
null => html "", text ""
0 => html "0", text "0"
42 => html "42", text "42"
true => html "true", text "true"
 => html "", text ""
[object Object] => html "[object Object]", text "[object Object]"
javascript:alert(1) => html "javascript:alert(1)", text "javascript:alert(1)"
```

- **`undefined`, `null` and `''`** give an empty string.
- **Numbers, booleans and objects** are converted to text first: `0` is `'0'`, an object is `'[object Object]'`. Zero is not treated as missing.
- **A bare `javascript:` address comes back unchanged.** The text has no tags, so there is nothing to remove. If the same text is used as the `href` of a link, it will run when clicked.

The last point is about context. These functions sanitize HTML. A value that goes into an attribute, such as `:href="user.website"`, needs its own check:

```js
const safe = url => /^https?:\/\//i.test(url) ? url : '#'
// <a :href="safe(user.website)">
```

## 6. Where J2OS uses it

Two components insert HTML, and both use this composable:

- **DataTable.** A column with `render: true` inserts its result as HTML, as it is. The page must sanitize the user-written parts when it builds that result, for example with `toPlainText` as in the column below.
- **DataMap.** The text of a marker (`markerText`, and the `text` of each marker in a list) goes through `toPlainText` before it is placed in the marker's HTML.

```js
{ field: 'title', label: 'Title', render: true, processor: row => `<b>${toPlainText(row.title)}</b>` }
```

A good habit: sanitize on the way into HTML, not on the way into the database. Stored text stays exactly as the visitor wrote it, and every place that shows it decides how to sanitize it.

## 7. From start to finish

We build a small comment list. The author's name is shown as plain text, the body as sanitized HTML, and a short excerpt is cut from the plain text. A draft box shows a live preview:

```vue
<script setup>
const { toDisplayHtml, toPlainText } = useXssProtector()

const draft = ref('<b>Great</b> post! <img src="x" onerror="window.__fired = 1">')
const comments = ref([
  { id: 1, author: 'Sara', text: '<p>Loved it.</p><script>window.__fired = 2<' + '/script>' },
  { id: 2, author: '<i>Tom</i>', text: 'Please <a href="javascript:window.__fired=3">click</a> or <a href="https://example.com">visit</a>.' }
])

function add() {
  comments.value.push({ id: comments.value.length + 1, author: 'You', text: draft.value })
}
</script>

<template>
  <textarea id="draft" v-model="draft" />
  <div id="preview" v-html="toDisplayHtml(draft)" />
  <button id="add" @click="add">Send</button>

  <article v-for="c in comments" :key="c.id" class="comment">
    <h4 v-text="toPlainText(c.author)" />
    <div class="body" v-html="toDisplayHtml(c.text)" />
    <small class="excerpt">{{ toPlainText(c.text).slice(0, 20) }}</small>
  </article>
</template>
```

```text
preview html: <b>Great</b> post! <img src="x">
comment 1 body: <p>Loved it.</p> | author text: Sara | excerpt: Loved it.
comment 2 body: Please <a>click</a> or <a href="https://example.com">visit</a>. | author text: Tom | excerpt: Please click or visi
after Send: comments=3, last body: <b>Great</b> post! <img src="x">
click the bad link: None
window.__fired = None (1/2/3 would mean a handler ran)
```

- **The preview** is `toDisplayHtml(draft)`: the draft's `onerror` is removed.
- **The first comment** has a script; only the paragraph remains.
- **The second comment's author** is `<i>Tom</i>`; `v-text` with `toPlainText` shows `Tom`. The unsafe link loses its address and the safe one keeps it.
- **The excerpt** is cut from the plain text, so it never contains half a tag.
- **The new comment** (added with Send) is sanitized the same way.
- **No handler runs:** `window.__fired` stays empty, even after clicking the unsafe link.

---

## Appendix A. Quick reference

### A.1 Functions

| Function | Returns | Notes |
|----------|---------|-------|
| `toDisplayHtml(value, options = {})` | String | Safe HTML; `options` go to DOMPurify (`RETURN_DOM` and similar change the result type) |
| `toPlainText(value)` | String | No tags; text may be HTML-escaped when the input had `<` |

### A.2 Common options

| Option | Effect |
|--------|--------|
| `ALLOWED_TAGS` | Exactly these tags are kept |
| `ALLOWED_ATTR` | Exactly these attributes are kept |
| `FORBID_TAGS` | Removes these tags from the default list |
| `FORBID_ATTR` | Removes these attributes from the default list |
| `ADD_TAGS`, `ADD_ATTR` | Adds to the default lists |

### A.3 Behavior of the default list

| Input | Result |
|-------|--------|
| `b`, `i`, `p`, `ul`, `li`, `a`, `img` | Kept |
| `script`, `iframe` | Removed, with their content |
| `onclick`, `onerror` and other `on...` attributes | Removed |
| `javascript:` in `href` | The `href` is removed |
| `style` attribute | Kept |
| `target` attribute | Removed |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| DOMPurify is not loaded | Both functions throw `ReferenceError: DOMPurify is not defined` |
| Called while the server renders | DOMPurify has no document there; the call fails or does not sanitize |
| `null` or `undefined` | Returns `''` |
| An object | Returns `'[object Object]'` |
| Input is completely unsafe | Dangerous parts are removed; no exception is thrown |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Showing user text with `v-html` and no sanitizing.** This is the XSS hole itself. Always pass the text through `toDisplayHtml` first.
- **Changing text after sanitizing it.** Building HTML from sanitized parts with your own string joins can undo the sanitizing. Sanitize the final string.
- **Using these functions for attributes.** `javascript:` addresses and similar values are plain text to them. Check URLs yourself (chapter 5).
- **Showing `toPlainText` with `{{ }}` and seeing `&amp;`.** The result may be HTML-escaped; for plain display write `{{ text }}`.
- **Expecting the content to be moderated.** Only code is removed. Links to malicious sites stay.
- **`style` staying allowed by accident.** A visitor can place a large or invisible element. Use `ALLOWED_ATTR` when that matters.
- **Options that return DOM nodes.** `RETURN_DOM` and similar options make `toDisplayHtml` return an object, which `v-html` cannot show.
- **Calling it before DOMPurify is loaded.** Both functions fail at once.
- **Calling it on the server.** Use it only in the browser (after mount, in event handlers, or inside `<ClientOnly>`).

### C.2 Behavior worth knowing

- `toPlainText` always returns a string, and so does `toDisplayHtml` unless an option such as `RETURN_DOM` is given.
- Text with no `<` in it is normally returned by DOMPurify unchanged, whatever the options.
- Tags that are not allowed lose their markup but usually keep their text.
- The composable keeps no state; it can be called as often as needed and from any component.
- A DOMPurify setting made elsewhere in the project (hooks, a global configuration) affects these functions too.

### C.3 Limits

| What | Limit |
|------|-------|
| Input types | Anything; it goes through `String()` |
| Output | A string (a DOM node with `RETURN_DOM` and similar options) |
| Options | Only for `toDisplayHtml` |
| Environment | Browser; needs a global `DOMPurify` |

### C.4 Tuning

- Choose the strictest allow list that the content needs. Comments often need only `b`, `i`, `a` and `br`.
- Sanitize once, at the place where text enters the template, and not in several layers.
- For long lists, sanitize the visible rows only, or keep the sanitized text next to the original.
- For pages that show user text, write a test with a few malicious samples (a script, an `onerror` image, a `javascript:` link).

## Appendix D. Message reference

### D.1 Messages

The composable shows no text of its own. The only message you may see comes from the browser when the library is missing:

| Message | Cause |
|---------|-------|
| `ReferenceError: DOMPurify is not defined` | The DOMPurify library is not loaded on the page |
