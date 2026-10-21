# useRichText

*J2OS Platform 2.4*

An article page, a product description, a comment box with a little formatting: in all of them the visitor writes text that has headings, bold words and lists, and the page has to store it as HTML. A plain `<textarea>` cannot do that, and writing a formatting editor from scratch is a project of its own. The J2OS answer is to put a ready-made editor, SunEditor, in place of the textarea and to hide its details behind a few functions.

**useRichText** is a composable of the Nuxt front end of J2OS that does this, and **useRichTextRefBased** is its twin that finds the textarea through a ref instead of an id; this guide covers both. They have three main features:

- **One call to get an editor.** `init` loads the editor's files on demand, builds a right-to-left editor in the Vazirmatn font and gives it a sensible toolbar.
- **Content in and out as HTML or text.** `getHtml`, `getText`, `setHtml` and `clear`.
- **Small conveniences.** Locking the editor, downloading the content as a file, and a `destroy` that gives everything back.

Here is the shape of it:

```vue
<script setup>
const body = useRichText()

onMounted(() => body.init('articleBody'))      // turns the <textarea> into an editor
onBeforeUnmount(() => body.destroy())          // removes it again
// later: body.getHtml(), body.setHtml('<p>...</p>')
</script>

<template>
  <textarea id="articleBody" />
</template>
```

The guide begins with how the composable works (chapter 1) and a first editor with a Publish button (2). Chapter 3 is about the content, what goes in and what comes out, and chapter 4 about the options of `init`. Chapters 5 and 6 cover locking and exporting. Chapter 7 deals with several editors and with leaving the page, chapter 8 with the ref-based version, and chapter 9 builds a dialog that edits a saved text.

## Contents

1. [How useRichText works](#1-how-userichtext-works)
2. [The first editor](#2-the-first-editor)
3. [What goes in and what comes out](#3-what-goes-in-and-what-comes-out)
4. [Options](#4-options)
5. [Locking the editor](#5-locking-the-editor)
6. [Exporting the content](#6-exporting-the-content)
7. [Several editors, leaving the page and failures](#7-several-editors-leaving-the-page-and-failures)
8. [The ref-based version](#8-the-ref-based-version)
9. [A dialog that edits a saved text](#9-a-dialog-that-edits-a-saved-text)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Every example is a self-contained page of a Nuxt project, and each is a single `.vue` file. In Nuxt `ref`, `onMounted` and the composables need no `import`, so none is written. What appears under a page is what the visitor sees when the page is in use. The editor needs its files under `/plugins/suneditor/` in the project's public folder, which is where the project keeps them.

---

## 1. How useRichText works

A short glossary before we start:

- **Editor.** The SunEditor object that replaces the textarea. `init` gives it back, and the composable keeps it in the `editor` ref.
- **Toolbar.** The row of buttons above the writing area. Its content is the `buttonList` option.
- **Contents.** What the visitor has written, as HTML (`<p>Hello</p>`). The text of it without the tags is called the **text**.
- **Editor files.** SunEditor's script and stylesheet. They are loaded by `init` and released by `destroy`, through `useFileLoader`.

The members of this composable:

| Member | Role |
|--------|------|
| `init(id, options)` | Builds the editor in place of the textarea with that id; resolves with the editor |
| `editor` | A ref holding the editor (empty before `init`, `null` after `destroy`) |
| `getHtml()` | The contents as HTML |
| `getText()` | The contents as text |
| `setHtml(html)` | Replaces the contents |
| `clear()` | Empties the editor |
| `setReadOnly(flag)` | Locks or unlocks the editor |
| `exportHTML(name)` | Downloads the HTML as a file |
| `exportText(name)` | Downloads the text as a file |
| `destroy()` | Removes the editor and releases its files |

Six points are worth remembering:

- **`init` is asynchronous.** It first loads the editor's files, and the other functions work only after it has finished (`await` it).
- **The textarea must exist when `init` is called,** so the place for it is `onMounted`.
- **The editor is always right to left,** and always uses the Vazirmatn font. There is no option for either.
- **The files are shared.** Every `init` asks for them and every `destroy` gives them back, so they stay loaded as long as one editor needs them (chapter 7).
- **The editor cleans what it is given.** HTML passed to `setHtml` comes back changed (chapter 3).
- **Each call of the composable makes one editor.** A page with two editors calls it twice.

## 2. The first editor

An article page has a title, a body and two buttons: one publishes the article, one loads the last published version into the editor. A line under the editor tells the visitor what is going on, and shows how much has been written so far.

```vue
<script setup>
const body = useRichText()
const title = ref('')
const length = ref(0)
const message = ref('Nothing written yet.')

onMounted(async () => {
  const editor = await body.init('articleBody', { placeholder: 'Write the article here', height: '260px' })
  editor.onChange = () => {
    length.value = body.getText().length
    message.value = length.value ? `Draft: ${length.value} characters.` : 'Nothing written yet.'
  }
})
onBeforeUnmount(() => body.destroy())

async function publish() {
  const form = new FormData()
  form.append('title', title.value)
  form.append('body', body.getHtml())
  const response = await fetch('/api/articles', { method: 'POST', body: form })
  message.value = await response.text()
}

async function loadLast() {
  const response = await fetch('/api/articles/last')
  body.setHtml(await response.text())
  message.value = 'Loaded the last published version.'
}
</script>

<template>
  <input id="title" v-model="title" placeholder="Title">
  <textarea id="articleBody" />
  <button id="publish" @click="publish">Publish</button>
  <button id="last" @click="loadLast">Load last version</button>
  <p id="message">{{ message }}</p>
</template>
```

```text
Page opens:                        Nothing written yet.
Visitor types a short text:        Draft: 20 characters.
Visitor presses Publish:           Published as article 17.
Visitor presses Load last version: Loaded the last published version. Editor now shows "Version one of the article.".
```

The page puts a plain `<textarea id="articleBody">` in the template and calls `init` with that id in `onMounted`. The textarea disappears and the editor takes its place. `onBeforeUnmount` calls `destroy`, so that leaving the page removes the editor and its files.

Two things in the page are worth a closer look. The first is the line "Draft: 20 characters". `init` resolves with the editor, and the editor has its own events; `onChange` runs a fraction of a second after the visitor stops typing (not on every key), and the page uses it to refresh the counter:

```js
onMounted(async () => {
  // init() resolves with the editor itself once it is ready
  const editor = await body.init('articleBody', { placeholder: 'Write the article here' })
  editor.onChange = () => {            // called a moment after the visitor stops typing
    length.value = body.getText().length
  }
})
```

The second is Publish. `getHtml()` gives the HTML string, which is what the server should store; the page sends it in a form with the title. "Load last version" does the opposite: `setHtml` puts the stored HTML back into the editor.

## 3. What goes in and what comes out

The editor is not a text box with a different look. It owns the markup, and what comes out is not always what went in. These are results of real calls:

| Call or action | Result |
|----------------|--------|
| `getHtml()` on an empty editor | `<p><br></p>` |
| `getText()` on an empty editor | empty text |
| The visitor types `Hello world` | `getHtml()` gives `<p>Hello world</p>` |
| `setHtml('<p>one</p><p>two</p><p>three</p>')` | `getText()` gives `onetwothree` |
| `setHtml('plain text no tags')` | `getHtml()` gives `<p>plain text no tags</p>` |
| `setHtml('<b>x</b>')` | comes back as `<strong>x</strong>` |
| `setHtml` with a `<script>` tag | the tag is dropped |
| `setHtml` with `onclick="..."` or `onerror="..."` | the attributes are dropped |
| `setHtml` with `style="color:red"` | the style attribute is dropped |
| `setHtml` with `<a href="javascript:...">` | the `href` is dropped |
| `setHtml` with a table | each cell's content is wrapped in a `<div>` |
| `clear()` | the same as an empty editor: `<p><br></p>` |

Four consequences follow:

- **"Empty" is not `''`.** An empty editor holds `<p><br></p>`. To test whether the visitor wrote anything, use `getText().trim()`, as the comment box in chapter 4 does.
- **`getText()` has no separators.** Paragraphs, list items and table cells are glued together. It is good for counting characters and for a plain-text export of short content; for words, count from the HTML:

```js
// getText() glues paragraphs together ("onetwothree"), so count words from the HTML
function countWords(html) {
  const text = html.replace(/<[^>]+>/g, ' ').replace(/&nbsp;/g, ' ')
  return text.split(/\s+/).filter(Boolean).length
}
```

- **The editor's cleaning helps but does not protect.** The page that shows stored HTML to other visitors, and the server that stores it, must still treat the HTML as untrusted input (see also `useXssProtector`).
- **Round trips change the markup a little.** `<b>` becomes `<strong>`, and a pasted style disappears. Compare content by text when you need to know whether something changed.

## 4. Options

`init` takes the id and an options object. All four options are optional:

| Option | Default | Meaning |
|--------|---------|---------|
| `width` | `'100%'` | Width of the editor |
| `height` | `'400px'` | Height of the writing area |
| `placeholder` | empty | Hint shown while the editor is empty |
| `buttonList` | the default toolbar | The toolbar, as an array of groups |

The default toolbar has, in order, one group each of: the block format menu (`formatBlock`), `bold`, `underline`, `italic`, `strike`, the color pickers (`fontColor`, `hiliteColor`), `removeFormat`, `align`, `list`, `blockquote`, `table`, `codeView` and `fullScreen`.

A comment box needs less. This one is small, has only bold, italic and lists, and keeps the visitor's attention on the text:

```vue
<script setup>
const comment = useRichText()
const posted = ref([])

onMounted(async () => {
  await comment.init('commentBox', {
    width: '420px',
    height: '120px',
    placeholder: 'Add a comment',
    buttonList: [['bold', 'italic'], ['list']]
  })
})
onBeforeUnmount(() => comment.destroy())

function post() {
  if (!comment.getText().trim()) return
  posted.value.push(comment.getHtml())
  comment.clear()
}
</script>

<template>
  <textarea id="commentBox" />
  <button id="post" @click="post">Post comment</button>
  <p id="count">{{ posted.length }} comment(s) posted.</p>
  <div v-for="(html, i) in posted" :key="i" class="posted" v-html="html" />
</template>
```

```text
Page opens:                                0 comment(s) posted. Toolbar offers: list, bold, italic. Editor size: 420 x 120 px.
Placeholder shown:                         Add a comment
Visitor presses Post with nothing written: 0 comment(s) posted.
Visitor writes a comment and presses Post: 1 comment(s) posted. Posted HTML: <p>Looks good</p>
Editor afterwards:                         empty again, placeholder back
```

The options are in the call:

```js
await comment.init('commentBox', {
  width: '420px',                        // default '100%'
  height: '120px',                       // default '400px'
  placeholder: 'Add a comment',          // default: none
  buttonList: [['bold', 'italic'], ['list']]   // each inner array is one group of buttons
})
```

The output confirms what the visitor gets: three buttons, the size that was asked for and the placeholder. Posting with nothing written did nothing, because the page tested `getText().trim()` and not the HTML. After posting, `clear()` empties the editor and the placeholder returns.

Names for `buttonList` come from SunEditor, which has many more buttons (image, link, video, font size and others) than the defaults use. Some of them need extra configuration in SunEditor itself, such as an upload address for images; this composable does not pass any, so stay with the buttons that work on their own.

## 5. Locking the editor

A reviewer should be able to read a note and not change it. `setReadOnly(true)` locks the editor, and `setReadOnly(false)` opens it again:

```vue
<script setup>
const note = useRichText()
const locked = ref(false)
const status = ref('Editing is open.')

onMounted(async () => {
  await note.init('reviewNote', { height: '120px' })
  note.setHtml('<p>Approved by finance.</p>')
})
onBeforeUnmount(() => note.destroy())

function toggle() {
  locked.value = !locked.value
  note.setReadOnly(locked.value)
  status.value = locked.value ? 'Note is locked for review.' : 'Editing is open.'
}
</script>

<template>
  <textarea id="reviewNote" />
  <button id="lock" @click="toggle">{{ locked ? 'Unlock' : 'Lock for review' }}</button>
  <p id="status">{{ status }}</p>
</template>
```

```text
Page opens:                         Editing is open. Note reads "Approved by finance.".
Visitor presses Lock for review:    Note is locked for review. Toolbar buttons disabled: True
Visitor tries to type while locked: Note reads "Approved by finance.".
Visitor unlocks and types again:    Editing is open. Note reads "Approved by finance. Extra words.".
```

While locked, the toolbar buttons are disabled, the caret is hidden and typing changes nothing. The content stays readable, and the page itself can still change it with `setHtml`; the lock only concerns the visitor. After unlocking, typing works again from where it stopped.

## 6. Exporting the content

`exportHTML` and `exportText` download the current content as a file, through a temporary link. The name is optional: the defaults are `document.html` and `document.txt`.

```vue
<script setup>
const note = useRichText()
onMounted(async () => {
  await note.init('meetingNote', { height: '140px', placeholder: 'Meeting notes' })
  note.setHtml('<h2>Standup</h2><p>Release moved to <b>Sunday</b>.</p><ul><li>Docs</li><li>Tests</li></ul>')
})
onBeforeUnmount(() => note.destroy())
</script>

<template>
  <textarea id="meetingNote" />
  <button id="html" @click="note.exportHTML('standup.html')">Download HTML</button>
  <button id="text" @click="note.exportText('standup.txt')">Download text</button>
  <button id="default" @click="note.exportText()">Download text (no name)</button>
</template>
```

```text
Visitor presses Download HTML:           browser saves standup.html containing '<h2>Standup</h2><p>Release moved to <strong>Sunday</strong>.</p><ul><li>Docs</li><li>Tests</li></ul>'
Visitor presses Download text:           browser saves standup.txt containing 'StandupRelease moved to Sunday.DocsTests'
Visitor presses Download text (no name): browser saves document.txt containing 'StandupRelease moved to Sunday.DocsTests'
```

The HTML file holds exactly what `getHtml()` returns, a fragment and not a full page with `<html>` and `<head>`. The text file holds `getText()`, so the paragraphs are glued together as chapter 3 explained. If a download should be readable, build the file on the page instead (for example, from the HTML with line breaks between blocks) and keep these two for quick exports.

## 7. Several editors, leaving the page and failures

A newsletter page has an introduction and a footer, each its own editor. Two editors need two calls of the composable:

```vue
<script setup>
const intro = useRichText()
const footer = useRichText()
const showFooter = ref(true)
const status = ref('Both parts are open.')

onMounted(async () => {
  await intro.init('introBox', { height: '100px', placeholder: 'Introduction' })
  await footer.init('footerBox', { height: '100px', placeholder: 'Footer' })
})
onBeforeUnmount(() => {
  intro.destroy()
  if (showFooter.value) footer.destroy()
})

function removeFooter() {
  footer.destroy()
  showFooter.value = false
  status.value = 'Footer removed. The introduction is still editable.'
}
</script>

<template>
  <textarea id="introBox" />
  <textarea v-if="showFooter" id="footerBox" />
  <button id="remove" @click="removeFooter">Remove footer</button>
  <p id="status">{{ status }}</p>
</template>
```

```text
Page opens:                                        Both parts are open. Editors on the page: 2; editor files loaded.
Visitor presses Remove footer:                     Footer removed. The introduction is still editable. Editors on the page: 1; editor files loaded.
Visitor keeps typing in the introduction:          it reads "Hello readers!".
Visitor leaves the page (both editors still open): Editors left: 0; editor files removed.
```

Removing the footer destroyed its editor and nothing else: the introduction kept working, and the editor files stayed loaded because the introduction still needed them. When the visitor left the page, the second `destroy` removed the files too. The rule is a counter: **every `init` takes a share of the files and every `destroy` gives one back.**

That rule also explains the failures:

| Situation | Result |
|-----------|--------|
| `init` with an id that has no textarea | The promise is rejected: `[SUNEDITOR.create.fail] The element for that id was not found` |
| `init` twice on the same id | The second call is rejected: `The ID of the suneditor you are trying to create already exists` |
| `getHtml()` and the other functions before `init` has finished | `TypeError`: the editor is still `undefined` |
| The same functions after `destroy()` | `TypeError`: the editor is `null` |
| `init` after `destroy` | Works; the editor is built again |

A rejected `init` has already asked for the files. If the page then calls `destroy`, the share is returned, but an extra `init` that failed leaves one share behind. Do not call `init` twice for one editor, and wrap it in `try` when the id comes from data.

After `destroy`, three small things stay behind: the page's own textarea stays hidden, the font style that `init` added to the page stays, and the global `SUNEDITOR` stays defined. None of them hurts when the same page builds a new editor later. If the textarea must be visible again, show it yourself.

A button that can be pressed before the editor is ready is the most common source of the `TypeError`. A flag solves it:

```js
// getHtml() before init() has finished, or after destroy(), throws a TypeError.
// Keep your own flag when a button can be pressed early.
const ready = ref(false)
onMounted(async () => { await body.init('articleBody'); ready.value = true })
onBeforeUnmount(() => { ready.value = false; body.destroy() })
```

## 8. The ref-based version

`useRichTextRefBased()` has the same members, with one difference: `init` takes a **ref** to the textarea instead of an id. It also accepts the element itself. The rest, from `getHtml` to `destroy`, is the same.

```vue
<script setup>
const note = useRichTextRefBased()
const noteEl = ref()                     // a ref instead of an id

onMounted(() => note.init(noteEl, { placeholder: 'Write a note' }))
onBeforeUnmount(() => note.destroy())
</script>

<template>
  <textarea ref="noteEl" />
</template>
```

The ref is better in three situations: the textarea sits inside a `v-if` or a `v-for` and has no fixed id, the same component is used several times on a page and ids would clash, or the textarea belongs to a child component. The ref also removes the first failure in chapter 7, because there is no id to get wrong.

One behavior is specific to this version. If the ref has no element when `init` is called, `init` does not fail: it finishes quietly with no editor, and `editor` stays empty. The files have already been requested by then, so the share is taken all the same. Make sure the element exists before `init`, which in a dialog means waiting for the next tick (chapter 9).

## 9. A dialog that edits a saved text

All the earlier chapters come together in one case that appears in almost every application: a profile with a bio. The bio is shown as formatted text. A button opens a dialog with an editor, **Save** keeps the text and closes the dialog, **Cancel** throws the changes away. The textarea exists only while the dialog is open, so this is where the ref-based version belongs:

```vue
<script setup>
const bio = useRichTextRefBased()
const open = ref(false)
const bioEl = ref()
const saved = ref('')
const status = ref('Bio is empty.')

watch(open, async (isOpen) => {
  if (isOpen) {
    await nextTick()
    await bio.init(bioEl, { height: '140px', placeholder: 'Tell us about yourself' })
    bio.setHtml(saved.value)
  } else {
    bio.destroy()
  }
})

function save() {
  saved.value = bio.getHtml()
  status.value = `Bio saved (${bio.getText().length} characters).`
  open.value = false
}
</script>

<template>
  <button id="edit" @click="open = true">Edit bio</button>
  <div v-if="open" class="dialog">
    <textarea ref="bioEl" />
    <button id="save" @click="save">Save</button>
    <button id="cancel" @click="open = false">Cancel</button>
  </div>
  <p id="status">{{ status }}</p>
  <div id="shown" v-html="saved" />
</template>
```

```text
Page opens:                                Bio is empty. Editors on the page: 0; editor files removed.
Visitor presses Edit bio:                  Editors on the page: 1; editor files loaded. Placeholder: Tell us about yourself
Visitor types a bio and presses Save:      Bio saved (30 characters). Page shows "Backend developer from Shiraz.". Editors on the page: 0; editor files removed.
Visitor opens the dialog again:            editor shows "Backend developer from Shiraz.".
Visitor adds more text and presses Cancel: Bio saved (30 characters). Page still shows "Backend developer from Shiraz.". editor files removed.
```

The page owns the text: `saved` holds the HTML, and the editor is only a window onto it. When the dialog opens, the page waits for the next tick so that the textarea exists, calls `init`, and puts the saved text in with `setHtml`. **Save** copies `getHtml()` into `saved` and closes the dialog; **Cancel** just closes it, and the unsaved words never reach `saved`. Closing the dialog calls `destroy`, so while it is closed, no editor and no editor files are on the page.

The recipe carries over to your own screens: keep the text in page state, build the editor when its container appears, fill it with `setHtml`, copy it back with `getHtml`, and always pair `init` with `destroy`.

---

## Appendix A. Quick reference

### A.1 Members

| Call | Returns | Notes |
|------|---------|-------|
| `init(idOrRef, options)` | Promise of the editor | Loads files, builds the editor; the ref-based version takes a ref or element |
| `editor` | Ref | The editor; empty before `init`, `null` after `destroy` |
| `getHtml()` | Text | HTML; `<p><br></p>` when empty |
| `getText()` | Text | Text without tags, blocks glued together |
| `setHtml(html)` | | Replaces the contents; the editor cleans the markup |
| `clear()` | | Empties the editor |
| `setReadOnly(flag)` | | Locks or unlocks |
| `exportHTML(name)` | | Downloads the HTML; default name `document.html` |
| `exportText(name)` | | Downloads the text; default name `document.txt` |
| `destroy()` | | Removes the editor and releases its files |

### A.2 Options of init

| Option | Default |
|--------|---------|
| `width` | `'100%'` |
| `height` | `'400px'` |
| `placeholder` | empty |
| `buttonList` | the default toolbar (chapter 4) |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| No element for the id | `init` is rejected |
| `init` twice on one id | The second `init` is rejected |
| No element in the ref (ref-based) | `init` finishes without an editor |
| Content functions before `init` or after `destroy` | `TypeError` |
| Markup the editor does not allow | Dropped from the contents |
| Failed `init` followed by no `destroy` | One share of the files stays loaded |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Calling `init` outside `onMounted`.** The textarea does not exist yet.
- **Not awaiting `init`.** The editor is not ready, and the other calls throw.
- **Forgetting `destroy`.** The editor and its files stay on the page after the visitor leaves it.
- **Testing emptiness with `=== ''`.** An empty editor holds `<p><br></p>`.
- **Counting words with `getText()`.** Blocks are glued together.
- **Trusting the editor to make HTML safe.** Treat stored HTML as untrusted.
- **Calling `init` twice for one editor.** The second call is rejected and costs a share of the files.
- **Expecting `setHtml` to keep every attribute.** Styles and scripts are dropped.
- **Expecting the textarea to come back after `destroy`.** It stays hidden.
- **Looking for an option for direction or font.** There is none.

### C.2 Behavior worth knowing

- The editor's files are loaded the first time and reused afterwards.
- `onChange` runs after a short pause in typing, not on every key.
- Downloads are made with a temporary link and need no server.
- Each composable call has its own editor; the loaded files are shared by all of them.

### C.3 Limits

| What | Limit |
|------|-------|
| Direction | Right to left only |
| Font | Vazirmatn only |
| Editors per call | One |
| Images and files inside the content | Need extra SunEditor configuration that the composable does not offer |
| Plain text export | Blocks are not separated |

### C.4 Tuning

- Give comment boxes a short toolbar and a small height; give articles the default.
- Keep the text in page state and treat the editor as a window onto it.
- Build editors when their container appears and destroy them when it goes.
- Test for content with `getText().trim()`.
- Store the HTML on the server and sanitize it there too.

## Appendix D. Message reference

### D.1 Errors

| Message | Cause |
|---------|-------|
| `[SUNEDITOR.create.fail] The element for that id was not found (ID:"...")` | No textarea has that id |
| `[SUNEDITOR.create.fail] The ID of the suneditor you are trying to create already exists (ID:"...")` | An editor was already built for that textarea |
| `TypeError: Cannot read properties of undefined (reading 'getContents')` | A content function was called before `init` finished |
| `TypeError: Cannot read properties of null (reading 'getContents')` | A content function was called after `destroy` |

### D.2 Values worth recognizing

| Value | Meaning |
|-------|---------|
| `<p><br></p>` | The editor is empty |
| `document.html`, `document.txt` | Default names of exported files |
