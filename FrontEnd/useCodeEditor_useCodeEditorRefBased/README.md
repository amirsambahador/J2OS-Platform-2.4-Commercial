# useCodeEditor

*J2OS Platform 2.4*

A rule written as a small script, a query for an admin screen, a template for an email, a configuration in JSON: some forms ask the visitor to write code. A plain `<textarea>` makes that painful, because it has no colors, no indentation help and no suggestions. A good code editor is a large piece of software, and the J2OS answer is to use an existing one, Ace, and to hide its setup behind a few functions.

**useCodeEditor** is a composable of the Nuxt front end of J2OS that does this, and **useCodeEditorRefBased** is its twin that finds the element through a ref instead of an id; this guide covers both. They have three properties:

- **One call to get an editor.** `init` loads Ace's files on demand, builds the editor in a `<div>` with a language, a theme and suggestions.
- **Content in and out as text.** `get` and `set`, plus `reformatCode` to tidy the code and `downloadCode` to save it as a file.
- **Small conveniences.** Locking the editor, and a `destroy` that gives everything back.

Short version:

```vue
<script setup>
const code = useCodeEditor()

onMounted(() => code.init('codeBox', { language: 'javascript' }))   // the <div> becomes an editor
onBeforeUnmount(() => code.destroy())
// later: code.get(), code.set('...')
</script>

<template>
  <div id="codeBox" style="height: 300px;" />
</template>
```

To begin: how the composable works (chapter 1) and a first editor (2). Chapter 3 is about the options of `init`, chapter 4 about suggestions and snippets, and chapter 5 about tidying code. Chapter 6 covers locking and downloading, and chapter 7 several editors, leaving the page and failures. Chapter 8 is the ref-based version, and chapter 9 builds a dialog that edits a saved script. The appendices are meant for quick lookups.

## Contents

1. [How useCodeEditor works](#1-how-usecodeeditor-works)
2. [The first editor](#2-the-first-editor)
3. [Language, theme and font size](#3-language-theme-and-font-size)
4. [Suggestions and snippets](#4-suggestions-and-snippets)
5. [Tidying code](#5-tidying-code)
6. [Locking and downloading](#6-locking-and-downloading)
7. [Several editors, leaving the page and failures](#7-several-editors-leaving-the-page-and-failures)
8. [The ref-based version](#8-the-ref-based-version)
9. [A dialog that edits a saved script](#9-a-dialog-that-edits-a-saved-script)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are real pages, not fragments, and each is a single `.vue` file. In Nuxt `ref`, `onMounted`, `watch` and the composables need no `import`, so none is written. What appears under a page is what the visitor sees when the page is in use. The editor needs Ace's files under `/plugins/ace/` in the project's public folder, which is where the project keeps them.

---

## 1. How useCodeEditor works

Before the details, the words this guide keeps coming back to:

- **Editor.** The Ace object that is built inside the `<div>`. `init` gives it back, and the composable keeps it in the `editor` ref.
- **Mode.** Ace's name for a language: how to color it and what its words are. `language: 'java'` selects `ace/mode/java`.
- **Theme.** The colors of the editor, such as `ace/theme/monokai`.
- **Suggestions.** The list that appears while the visitor types. It holds the language's own words, and the words the page adds.
- **Editor files.** Ace's script, its suggestion extension and a code formatter. They are loaded by `init` and released by `destroy`, through `useFileLoader`.

What the composable exposes:

| Member | Role |
|--------|------|
| `init(id, options)` | Builds the editor in the element with that id; resolves with the Ace editor |
| `editor` | A ref holding the Ace editor (empty before `init`, `null` after `destroy`) |
| `get()` | The code as text |
| `set(text)` | Replaces the code |
| `reformatCode()` | Tidies the code with the formatter |
| `downloadCode(name)` | Saves the code as a file |
| `readOnly()` | Locks the editor |
| `readWrite()` | Unlocks it |
| `destroy()` | Removes the editor and releases its files |

Hold on to these six points:

- **`init` is asynchronous.** It first loads the files, and the other functions work only after it has finished (`await` it).
- **The element must exist, and have a height.** The editor fills its element, and an element without a height shows nothing; give it one in a style.
- **The editor takes the focus** when it is built.
- **The files are shared.** Every `init` asks for them and every `destroy` gives them back, so they stay loaded as long as one editor needs them.
- **Some settings are fixed:** no print margin, suggestions on while typing, and a background syntax checker for the languages that have one.
- **Every tooltip of the editor is hidden.** The composable adds a style that switches off Ace's tooltips, including the ones on error marks.

## 2. The first editor

A shop's admin screen lets a manager edit a discount rule as a small script. The page shows the editor, a line that counts lines and characters, and four buttons: load an example, tidy up, lock and download.

```vue
<script setup>
const script = useCodeEditor()
const status = ref('Empty script.')
const locked = ref(false)

onMounted(async () => {
  const editor = await script.init('scriptBox', { language: 'javascript', fontSize: 14 })
  editor.session.on('change', () => {
    const code = script.get()
    status.value = code ? `${code.split('\n').length} line(s), ${code.length} characters.` : 'Empty script.'
  })
})
onBeforeUnmount(() => script.destroy())

function loadExample() {
  script.set('function discount(total){if(total>100){return total*0.9}else{return total}}')
}

function toggleLock() {
  locked.value = !locked.value
  if (locked.value) script.readOnly()
  else script.readWrite()
}
</script>

<template>
  <div id="scriptBox" style="height: 220px;" />
  <button id="example" @click="loadExample">Load example</button>
  <button id="tidy" @click="script.reformatCode()">Tidy up</button>
  <button id="lock" @click="toggleLock">{{ locked ? 'Unlock' : 'Lock' }}</button>
  <button id="download" @click="script.downloadCode('discount.js')">Download</button>
  <p id="status">{{ status }}</p>
</template>
```

```text
Page opens:                                              Empty script.
Visitor presses Load example:                            1 line(s), 75 characters.
Visitor presses Tidy up:                                 7 line(s), 118 characters.
Visitor presses Lock and tries to type:                  the script is unchanged; button now reads Unlock.
Page loads the example while locked:                     1 line(s), 75 characters.
Visitor presses Unlock and types two slashes at the end: 1 line(s), 77 characters.
Visitor presses Download:                                the browser saves discount.js (77 characters, ends with "l}}//").

After Load example the editor shows:
    function discount(total){if(total>100){return total*0.9}else{return total}}
After Tidy up the editor shows:
    function discount(total) {
        if (total > 100) {
            return total * 0.9
        } else {
            return total
        }
    }
```

The page puts an empty `<div id="scriptBox">` with a height in the template and calls `init` in `onMounted`. `init` resolves with the Ace editor, and the page uses it to listen to changes (`editor.session.on('change', ...)`), which is how the status line stays current; everything else goes through the composable's own functions. `onBeforeUnmount` calls `destroy`.

Read the output as a conversation. The status line follows the code: empty at first, one line after the example was loaded, seven lines after tidying. Chapters 5 and 6 explain the rest of the lines. The two blocks at the end of the output show the code in the editor before and after `reformatCode`.

`set(text)` puts the cursor at the start of the new text, and `get()` returns exactly what the editor holds, with `\n` between lines.

## 3. Language, theme and font size

`init` takes the id and an options object. All options are optional:

| Option | Default | Meaning |
|--------|---------|---------|
| `language` | `'java'` | The Ace mode: `'javascript'`, `'java'`, `'python'`, `'sql'`, `'json'`, ... |
| `theme` | `'ace/theme/monokai'` | The colors, written as the path of an Ace theme |
| `fontSize` | `16` | Text size in pixels |
| `snippets` | none | Code templates (chapter 4) |
| `autocompletes` | none | Extra suggestions (chapter 4) |

```js
await code.init('codeBox', {
  language: 'java',               // default 'java'; the name of an Ace mode
  theme: 'ace/theme/github',      // default 'ace/theme/monokai'
  fontSize: 14,                   // default 16
  snippets: [...],                // see chapter 4
  autocompletes: [...]
})
```

Ace loads the file of the chosen mode and theme when they are first needed, from the same `/plugins/ace/` folder, so the folder must contain the modes and themes the pages use. The settings that are not options are fixed: no print margin, suggestions while typing on, and the background syntax checker. For JavaScript, for example, a half-written function gets a warning such as `Missing name in function declaration.` as a mark in the gutter.

## 4. Suggestions and snippets

Two extra lists can be given to `init`. **Autocompletes** are words that join the suggestions the language already offers. **Snippets** are templates: a name and some text that expands to several words.

```vue
<script setup>
const editor = useCodeEditor()

const snippets = [
  { name: 'sout', tabTrigger: 'sout', content: 'System.out.println();' },
  { name: 'psvm', tabTrigger: 'psvm', content: 'public static void main(String[] args) {}' }
]

const jpaWords = [
  { caption: '@Entity', value: 'Entity', meta: 'JPA Annotation', score: 1000 },
  { caption: '@Table', value: 'Table(name = "")', meta: 'JPA Annotation', score: 1000 }
]

onMounted(() => editor.init('javaBox', { language: 'java', snippets, autocompletes: jpaWords }))
onBeforeUnmount(() => editor.destroy())
</script>

<template>
  <p>Type <code>Ent</code>, <code>sout</code> or <code>psvm</code> and pick from the list.</p>
  <div id="javaBox" style="height: 200px;" />
</template>
```

```text
Visitor types Ent:                                   the list starts with "@Entity JPA Annotation"
Visitor presses Enter:                               the editor shows "Entity".
Visitor clears the editor and types sout:            the list starts with "sout snippet"
Visitor presses Enter:                               the editor shows "System.out.println();".
Visitor types psvm, closes the list and presses Tab: the editor shows "psvm".
```

The lists are plain data:

```js
const snippets = [
  { name: 'sout', tabTrigger: 'sout', content: 'System.out.println();' }
]

const jpaWords = [
  // caption: shown in the list, value: inserted, meta: grey label, score: higher is listed earlier
  { caption: '@Entity', value: 'Entity', meta: 'JPA Annotation', score: 1000 }
]

await editor.init('javaBox', { language: 'java', snippets, autocompletes: jpaWords })
```

Notice how each entry reaches the visitor:

- **An autocomplete** appears in the list with its `caption` and `meta` (here `@Entity JPA Annotation`), above the language's own words when its `score` is high. Choosing it inserts its `value`, and that is why the output shows `Entity` and not `@Entity`: write the `value` the way it should land in the code.
- **A snippet** appears in the list as its `name` followed by the word `snippet`. Choosing it with Enter inserts the whole `content`.
- **Tab does not expand snippets.** The composable turns snippet expansion off in the editor, so typing a trigger and pressing Tab does nothing special (the last line of the output, where the editor still shows `psvm`). Tell visitors to pick the snippet from the list.
- **The lists belong to the language, not to the editor.** The extra suggestions are offered only in editors whose language is the one given to `init`, but any editor of that language shows them, including the ones that were given a different list (chapter 7).

## 5. Tidying code

`reformatCode()` runs the code through a JavaScript formatter: four spaces, braces on the same line, at most two empty lines. The cursor stays on the same row and column afterwards. For the script in chapter 2 this is exactly what a developer wants:

| Before | After |
|--------|-------|
| `function discount(total){if(total>100){return total*0.9}else{return total}}` | seven lines, indented, with spaces around operators |

The formatter understands the JavaScript family, so what it does to other languages depends on their syntax:

| Language | Result |
|----------|--------|
| JavaScript | Tidied |
| Java | Tidied |
| JSON | Works as JavaScript does |
| SQL | Only the spacing changes |
| Python | Damaged: the indentation, which is part of the syntax, is rewritten and the program changes meaning |

For a page whose editor can be in several languages, only offer the button for the ones that suit:

```js
// the formatter understands JavaScript-style code: JavaScript, Java, JSON, C-like languages
const formattable = ['javascript', 'java', 'json', 'typescript']

function tidy() {
  if (!formattable.includes(language.value)) return
  code.reformatCode()
}
```

Two details. If the formatter file is missing, `reformatCode()` throws `Formatter library is not loaded`; and when there is no editor (before `init` or after `destroy`) it does nothing at all, without an error.

## 6. Locking and downloading

`readOnly()` locks the editor: typing changes nothing, the cursor is hidden and the focus leaves the editor. `readWrite()` undoes this and puts the focus back. The output shows the visitor's attempt to type while locked and the script staying as it was and the button changing to Unlock.

Locking is for the visitor. The page can still call `set`, as the line "Page loads the example while locked" shows: the text changes even though the editor is locked.

`downloadCode(name)` saves the current code as a text file through a temporary link. The default name is `code.txt`, the file is plain text, and the name is used as given, so write the extension that matches the language. The last line of the output shows `discount.js` with exactly the characters of the editor at that moment, including the two slashes the visitor added after unlocking.

## 7. Several editors, leaving the page and failures

Each call of the composable has its own editor, so a page with two editors calls it twice. Because the files are shared, destroying one editor does not remove them while another still needs them. The rule is a counter: **every `init` takes a share of the files and every `destroy` gives one back.**

| Situation | Result |
|-----------|--------|
| `init` with an id that has no element | The promise is rejected: `ace.edit can't find div #...` |
| `init` twice on the same id | No error; Ace returns the same editor, but a second share of the files is taken |
| `get()`, `set()`, `downloadCode()` before `init` has finished | `TypeError`: the editor is still `undefined` |
| The same after `destroy()` | `TypeError`: the editor is `null` |
| `reformatCode()` in either state | Nothing happens |
| `init` after `destroy` | Works; the editor is built again, with the options given the second time |
| Two editors, or an editor built again, with different `autocompletes` in the same language | Every editor of that language shows all the lists, and the lists of editors already destroyed stay until the page is reloaded |

After `destroy` the element is left empty but keeps the classes Ace gave it, and the global `ace` stays defined; none of this hurts when the page builds a new editor in the same place.

A button that can be pressed before the editor is ready is the most common source of the `TypeError`. A flag solves it:

```js
// get(), set() and downloadCode() throw a TypeError before init() has finished
// and after destroy(); reformatCode() does nothing in those states.
const ready = ref(false)
onMounted(async () => { await code.init('codeBox'); ready.value = true })
onBeforeUnmount(() => { ready.value = false; code.destroy() })
```

## 8. The ref-based version

`useCodeEditorRefBased()` has the same members, with one difference: `init` takes a **ref** to the element instead of an id. It also accepts the element itself. The rest, from `get` to `destroy`, is the same.

```vue
<script setup>
const code = useCodeEditorRefBased()
const box = ref()                          // a ref instead of an id

onMounted(() => code.init(box, { language: 'javascript' }))
onBeforeUnmount(() => code.destroy())
</script>

<template>
  <div ref="box" style="height: 300px;" />
</template>
```

The ref is better when the element sits inside a `v-if` or a `v-for` and has no fixed id, when the same component is used several times on a page and ids would clash, or when the element belongs to a child component. It also removes the first failure of chapter 7, because there is no id to get wrong.

One behavior is specific to this version. If the ref has no element when `init` is called, `init` does not fail: it finishes quietly with no editor, and `editor` stays empty. The files have already been requested by then, so the share is taken all the same. Make sure the element exists before `init`, which in a dialog means waiting for the next tick (chapter 9).

## 9. A dialog that edits a saved script

The earlier chapters now meet in one case that appears in almost every admin application: a saved script that is edited in a dialog. A button opens the dialog with an editor, **Save** keeps the code and closes the dialog, **Cancel** throws the changes away. The element exists only while the dialog is open, so this is where the ref-based version belongs:

```vue
<script setup>
const script = useCodeEditorRefBased()
const open = ref(false)
const box = ref()
const saved = ref('return 1 + 1')
const status = ref('Saved script: return 1 + 1')

watch(open, async (isOpen) => {
  if (isOpen) {
    await nextTick()
    await script.init(box, { language: 'javascript', fontSize: 14 })
    script.set(saved.value)
  } else {
    script.destroy()
  }
})

function save() {
  saved.value = script.get()
  status.value = `Saved script: ${saved.value.replace(/\n/g, ' ')}`
  open.value = false
}
</script>

<template>
  <button id="edit" @click="open = true">Edit script</button>
  <div v-if="open" class="dialog">
    <div ref="box" style="height: 140px;" />
    <button id="save" @click="save">Save</button>
    <button id="cancel" @click="open = false">Cancel</button>
  </div>
  <p id="status">{{ status }}</p>
</template>
```

```text
Page opens:                                        Saved script: return 1 + 1 Editors on the page: 0.
Visitor presses Edit script:                       editors on the page: 1; it shows "return 1 + 1".
Visitor adds " + 1" and presses Save:              Saved script: return 1 + 1 + 1 Editors on the page: 0.
Visitor reopens, adds " + 100" and presses Cancel: Saved script: return 1 + 1 + 1 Editors on the page: 0.
```

The page owns the code: `saved` holds the text and the editor is only a window onto it. When the dialog opens, the page waits for the next tick so that the element exists, calls `init`, and puts the saved code in with `set`. **Save** copies `get()` into `saved` and closes the dialog; **Cancel** just closes it, and the unsaved addition never reaches `saved`, as the last two lines of the output show. Closing the dialog calls `destroy`, so while it is closed there is no editor on the page and no editor files.

On your own screens, follow the same recipe: keep the code in page state, build the editor when its container appears, fill it with `set`, copy it back with `get`, offer `reformatCode` only for languages the formatter understands, and always pair `init` with `destroy`.

---

## Appendix A. Quick reference

### A.1 Members

| Call | Returns | Notes |
|------|---------|-------|
| `init(idOrRef, options)` | Promise of the Ace editor | Loads files, builds the editor; the ref-based version takes a ref or element |
| `editor` | Ref | The editor; empty before `init`, `null` after `destroy` |
| `get()` | Text | The code |
| `set(text)` | | Replaces the code; the cursor goes to the start |
| `reformatCode()` | | Tidies the code and keeps the cursor row and column |
| `downloadCode(name)` | | Saves a text file; default name `code.txt` |
| `readOnly()` | | Locks; hides the cursor |
| `readWrite()` | | Unlocks; focuses the editor |
| `destroy()` | | Removes the editor and releases its files |

### A.2 Options of init

| Option | Default |
|--------|---------|
| `language` | `'java'` |
| `theme` | `'ace/theme/monokai'` |
| `fontSize` | `16` |
| `snippets` | none |
| `autocompletes` | none |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| No element for the id | `init` is rejected |
| `init` twice on one id | Same editor; an extra share of the files |
| No element in the ref (ref-based) | `init` finishes without an editor |
| `get`, `set`, `downloadCode` before `init` or after `destroy` | `TypeError` |
| `reformatCode` before `init` or after `destroy` | Nothing happens |
| Formatter file missing | `Error`: `Formatter library is not loaded` |
| `reformatCode` on a language that is not JavaScript-like | The code may be damaged |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **An element without a height.** The editor is invisible; give the `<div>` a height.
- **Calling `init` outside `onMounted`.** The element does not exist yet.
- **Not awaiting `init`.** The editor is not ready, and `get` and `set` throw.
- **Forgetting `destroy`.** The editor and its files stay on the page.
- **Expecting Tab to expand snippets.** Snippets are chosen from the list.
- **Writing `@` in the `value` of a suggestion** when the visitor has typed it already; the `value` is what is inserted.
- **Tidying Python.** The formatter breaks it.
- **Relying on tooltips.** They are switched off for every editor.
- **Calling `init` twice for one editor.** A second share of the files is taken.
- **Leaving out the extension in `downloadCode`.** The name is used as given.

### C.2 Behavior worth knowing

- The editor takes the focus when it is built.
- Suggestions appear while typing, without a shortcut.
- JavaScript editors check syntax in the background.
- `set` moves the cursor to the start; `reformatCode` keeps it where it was.
- `readOnly` does not stop the page from calling `set`.
- Downloads are made with a temporary link and need no server.

### C.3 Limits

| What | Limit |
|------|-------|
| Languages and themes | Those present in `/plugins/ace/` |
| Formatter | JavaScript-style only; four spaces |
| Snippet expansion with Tab | Off |
| Tooltips | Off |
| Editors per call | One |
| Extra suggestion lists | Shared by every editor of the language; never removed |
| Download format | Plain text |

### C.4 Tuning

- Give the editor's `<div>` a fixed height.
- Keep the code in page state and treat the editor as a window onto it.
- Offer the Tidy button only for languages the formatter understands.
- Put the suggestion text in `value` exactly as it should land in the code, and keep one suggestion list per language on a page.
- Build editors when their container appears and destroy them when it goes.

## Appendix D. Message reference

### D.1 Errors

| Message | Cause |
|---------|-------|
| `ace.edit can't find div #...` | No element has that id |
| `TypeError: Cannot read properties of undefined (reading 'getValue')` | `get` was called before `init` finished |
| `TypeError: Cannot read properties of null (reading 'getValue')` | `get` was called after `destroy` |
| `Formatter library is not loaded` | The formatter's file is not available when `reformatCode` runs |

### D.2 Values worth recognizing

| Value | Meaning |
|-------|---------|
| `code.txt` | Default name of the downloaded file |
| `name snippet` | How a snippet appears in the suggestion list |
