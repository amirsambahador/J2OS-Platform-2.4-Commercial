# useKeyListener

*J2OS Platform 2.4*

People who do data entry spend most of their time on the keyboard. They expect Enter to jump to the next field, Ctrl+S to save and Escape to close. A form that makes them reach for the mouse after every field is a slow form. The browser only provides raw key events, and turning them into these habits takes the same repetitive code on every page.

**useKeyListener** is a composable of the Nuxt front end of J2OS that provides three of those habits as ready-made functions. **useKeyListenerRefBased** is its companion for fields that come and go (modals, wizard steps). This guide explains both. Each offers three features:

- **Shortcuts.** `setupKeyBoardShortcuts` runs your functions when a combination such as `Ctrl+s` or `F2` is pressed.
- **Enter as Tab.** `setupEnterNavigation` makes Enter move to the next field and Shift+Enter to the previous one.
- **Tab control.** `setupTabNavigation` moves the focus to a chosen first element on Tab and to a chosen last element on Shift+Tab.

In a few lines:

```vue
<script setup>
const { setupKeyBoardShortcuts, setupEnterNavigation } = useKeyListener()

let stopShortcuts

onMounted(() => {
  stopShortcuts = setupKeyBoardShortcuts({
    'Ctrl+s': (event) => { event.preventDefault(); save() },
    'Escape': () => close()
  })
  setupEnterNavigation(['name', 'email', 'city'])      // Enter moves to the next field
})

onBeforeUnmount(() => stopShortcuts?.())
</script>
```

This guide goes step by step: chapter 1 explains how the composable works, chapter 2 covers shortcuts, chapter 3 Enter navigation and chapter 4 Tab navigation, where it becomes clear what `setupTabNavigation` really does. Chapter 5 is about cleaning up, chapter 6 about the ref-based version, and chapter 7 builds a quick entry form. The appendices are a quick reference.

## Contents

1. [How useKeyListener works](#1-how-usekeylistener-works)
2. [Keyboard shortcuts](#2-keyboard-shortcuts)
3. [Enter navigation](#3-enter-navigation)
4. [Tab navigation](#4-tab-navigation)
5. [Cleaning up](#5-cleaning-up)
6. [The ref-based version](#6-the-ref-based-version)
7. [A quick entry form](#7-a-quick-entry-form)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are complete pages of a Nuxt project: each is one `.vue` file. In Nuxt `ref`, `onMounted`, `onBeforeUnmount` and `useKeyListener` need no `import`, which is why none is written.

---

## 1. How useKeyListener works

First, a few terms, because they come back throughout the guide:

- **Key event.** The event the browser creates when a key goes down. It shows which key (`event.key`) and which modifiers (Ctrl, Shift, Alt) are held.
- **Combination.** A text such as `Ctrl+Shift+z` that names a key with its modifiers.
- **Remover.** The function that every setup function returns. Calling it removes the listener.

The functions of this composable:

| Function | Listens on | Does |
|----------|-----------|------|
| `setupKeyBoardShortcuts(shortcuts)` | The whole page | Calls the function registered for the pressed combination |
| `setupEnterNavigation(elementIds)` | Each listed element | Enter moves the focus to the next one, Shift+Enter to the previous one |
| `setupTabNavigation(firstId, lastId, container)` | The page, or the container | Tab moves the focus to the first element, Shift+Tab to the last |

6 points to keep in mind:

- **The elements are found by `id`,** in the plain version, at the moment of the call. The ref-based version takes refs instead (chapter 6).
- **Every setup function returns a remover.** In the plain version, calling it is up to you.
- **Shortcuts see every key pressed on the page,** including keys typed into fields.
- **The composable does not touch the browser's own behaviour.** To stop what the browser would do, your shortcut function must call `event.preventDefault()`.
- **Missing elements are skipped silently,** but they still count as members of the chain (chapter 3).
- **Only `keydown` is listened to.** Holding a key repeats the event.

## 2. Keyboard shortcuts

The most familiar habits start here: Ctrl+S saves, F2 renames, Escape closes. The browser tells the page about every key pressed, but the page has to work out which presses matter and what to do with each. `setupKeyBoardShortcuts` takes over that job. It takes an object whose keys are combinations and whose values are functions, and it calls the matching function when the visitor presses a combination. The function receives the key event as its argument.

The page below registers several shortcuts and writes into a log which of them run:

```vue
<script setup>
const { setupKeyBoardShortcuts } = useKeyListener()
const rows = ref([])
const hit = name => (event) => rows.value.push(`${name} (key "${event.key}", repeat ${event.repeat})`)

let stop = null
onMounted(() => {
  stop = setupKeyBoardShortcuts({
    'F2': hit('F2'),
    'Escape': hit('Escape'),
    'Ctrl+s': hit('Ctrl+s'),
    'Ctrl+Shift+z': hit('Ctrl+Shift+z'),
    'Ctrl+Shift+Alt+k': hit('Ctrl+Shift+Alt+k'),
    'Alt+ArrowUp': hit('Alt+ArrowUp'),
    's': hit('s'),
    'Ctrl+س': hit('Ctrl+س')
  })
})
function release() { stop(); rows.value.push('listener removed') }
</script>

<template>
  <input id="field" placeholder="type here">
  <button id="release" @click="release">Remove</button>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
</template>
```

```text
press F2                           -> F2 (key "F2", repeat false)
press Escape                       -> Escape (key "Escape", repeat false)
press Control+s                    -> Ctrl+s (key "s", repeat false)
press Control+S (capital)          -> Ctrl+s (key "S", repeat false)
press Control+Shift+Z              -> Ctrl+Shift+z (key "Z", repeat false)
press Shift+Control+z              -> Ctrl+Shift+z (key "z", repeat false)
press Control+Alt+Shift+K          -> Ctrl+Shift+Alt+k (key "K", repeat false)
press Alt+ArrowUp                  -> Alt+ArrowUp (key "ArrowUp", repeat false)
press Control+Alt+k (no match)     -> nothing
press Meta+s                       -> s (key "s", repeat false)
press Control+Shift+s (no match)   -> nothing
type "s" in the text field         -> s (key "s", repeat false)
hold F2 (repeat event)             -> F2 (key "F2", repeat true)
Ctrl + a Persian letter            -> Ctrl+س (key "س", repeat false)
click Remove, then press F2        -> listener removed
```

First, learn how a combination is **written**. The order is fixed: `Ctrl+`, then `Shift+`, then `Alt+`, then the key. If the visitor presses Ctrl, Alt and Shift together with K, the matching combination is `Ctrl+Shift+Alt+k`. Single letters are always lowercase, so `Ctrl+S` and `Ctrl+s` are one combination, and Caps Lock changes nothing. Other keys are written as the browser names them: `F2`, `Escape`, `Enter`, `ArrowUp`, `Tab`. A combination written in another order or with a capital letter never fires, and no error is reported.

```js
// The combination is always written in this order: Ctrl, Shift, Alt, then the key.
'Ctrl+Shift+Alt+k'     // yes
'Alt+Ctrl+k'           // never fires
'Ctrl+K'               // never fires: single letters are lowercase
```

3 more details worth knowing:

- **A combination that is not registered does nothing.** In the page above, `Ctrl+Alt+k` and `Ctrl+Shift+s` are not registered, so pressing them has no effect.
- **The Meta key is not part of the combination.** Command on a Mac and Windows on a PC are ignored, so `Meta+s` matches the plain `s` shortcut, while Command+S does not trigger `Ctrl+s`. If Mac visitors should be able to press Command+S, register another route for them, or accept that Ctrl is the main key.
- **The browser's own action still happens.** Ctrl+S opens the "Save page" dialog unless the function calls `event.preventDefault()`.

**Typing in a field triggers shortcuts too.** The shortcuts listen on the whole page, so typing `s` into a text field runs the `s` shortcut as well. For shortcuts with a modifier this is what you want, since Ctrl+S should save from anywhere. For one-letter shortcuts it is a trap: a visitor typing "/" or "s" into a search box would trigger them. Give those a guard:

```js
const typing = (event) => ['INPUT', 'TEXTAREA', 'SELECT'].includes(event.target.tagName)

setupKeyBoardShortcuts({
  // A one-letter shortcut must not fire while the visitor is typing.
  '/': (event) => { if (!typing(event)) { event.preventDefault(); searchBox.value.focus() } },
  'Ctrl+s': (event) => { event.preventDefault(); save() }       // fine anywhere
})
```

**Holding a key calls the function repeatedly.** The event's `repeat` property is `true` on the repeats, so an action that must run once (opening a dialog, say) should return early when it is `true`.

**Keyboard layouts matter.** The handler is looked up by the exact text of the combination, and the browser reports the letter printed on the key. On a Persian layout, pressing the physical S key with Ctrl gives `Ctrl+س`, which `Ctrl+s` never matches. If the application has Persian-speaking users, register both spellings with the same function:

```js
// On a Persian keyboard layout the same physical key reports a Persian letter.
const save = (event) => { event.preventDefault(); saveForm() }

setupKeyBoardShortcuts({
  'Ctrl+s': save,
  'Ctrl+س': save
})
```

Finally, shortcuts add up and are independent. If two calls register the same combination, both functions run; and if a function throws, the browser shows an uncaught error while later key presses still work. The function that `setupKeyBoardShortcuts` returns is the remover: calling it stops every shortcut of that call, and after "Remove" pressing `F2` logs nothing. Chapter 5 explains why this matters.

## 3. Enter navigation

People who do data entry expect Enter to behave like Tab: finish a field, press Enter, land in the next one. A browser does not do this; in a form, Enter submits. `setupEnterNavigation` takes a list of element `id`s in order; Enter moves the focus to the next field and Shift+Enter to the previous one.

The page below has four chained fields, plus a Send button that is not in the list:

```vue
<script setup>
const { setupEnterNavigation } = useKeyListener()
const rows = ref([])
const active = () => document.activeElement?.id || document.activeElement?.tagName

onMounted(() => {
  setupEnterNavigation(['name', 'family', 'notes', 'phone'])
})

function submitted() { rows.value.push('form submitted') }
</script>

<template>
  <form @submit.prevent="submitted">
    <input id="name">
    <input id="family">
    <textarea id="notes" />
    <input id="phone">
    <button id="send" type="submit">Send</button>
  </form>
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
</template>
```

```text
Enter in name                      -> focus on family  form submitted: no
Enter in family                    -> focus on notes   form submitted: no
Shift+Enter in family              -> focus on name    form submitted: no
Shift+Enter in name (the first)    -> focus on name    form submitted: yes
Enter in the text area             -> focus on phone   form submitted: no
text of the text area: 'line'
Enter in phone (the last)          -> focus on phone   form submitted: yes
Enter on the Send button           -> focus on send    form submitted: yes
```

Enter and Shift+Enter move the focus along the chain. At the two ends the key is left alone: Shift+Enter in the first field and Enter in the last one do what they normally do, which here is submitting the form. For the last field this is good design, because the visitor finishes the row with one more Enter.

The elements in the list **lose the normal behaviour of Enter**. This has two practical consequences:

- **Text areas must stay out of the list.** If the notes field is in the list, Enter in it moves on to the next field, so the visitor can never type a second line.
- **Buttons must stay out of the list too.** In the page above, pressing Enter on Send submits the form, because Send is not in the chain. A button in the list loses that.

```js
// Leave out what should keep its own Enter: text areas and buttons.
setupEnterNavigation(['name', 'family', 'phone'])      // not 'notes', not 'send'
```

The chain works only when all the elements exist. The page below lists three ids, but `b` is not on the page:

```vue
<script setup>
const { setupEnterNavigation } = useKeyListener()

onMounted(() => {
  // 'ghost' is not on the page.
  setupEnterNavigation(['a', 'ghost', 'b'])
})
</script>

<template>
  <input id="a">
  <input id="b">
  <p>x</p>
</template>
```

```text
Enter in a (next id is missing) -> focus on a
Shift+Enter in b (previous id is missing) -> focus on b
```

A missing element is skipped quietly when the listeners are attached, but its neighbours still point at it. Enter in `a` is stopped and the focus stays in `a`, because the next element does not exist; Shift+Enter in `c` behaves the same way. The chain does not jump over the gap. So with the plain version, list only elements that are on the page when `setupEnterNavigation` is called, and for fields that appear and disappear, use the ref-based version (chapter 6).

## 4. Tab navigation

A second need is a small group of controls for which the browser's Tab order does not fit, such as a two-button confirmation. `setupTabNavigation(firstId, lastId)` moves the focus to the first element on Tab and to the last element on Shift+Tab.

Read that literally: it is not a cycle. The page below has four fields, and the rows show where the focus ends when it starts in each field:

```vue
<script setup>
const { setupTabNavigation, setupKeyBoardShortcuts } = useKeyListener()

onMounted(() => {
  // On the whole page: first = #one, last = #four.
  setupTabNavigation('one', 'four')
})
</script>

<template>
  <input id="one">
  <input id="two">
  <input id="three">
  <input id="four">
  <p>x</p>
</template>
```

```text
Tab  from one   -> one
Tab  from two   -> one
Tab  from three -> one
Tab  from four  -> one
Shift+Tab from one   -> four
Shift+Tab from two   -> four
Shift+Tab from four  -> four
```

Wherever the focus is, Tab moves it to the first element and Shift+Tab to the last. The fields between them can no longer be reached with Tab. For two buttons this is exactly right; for a form with ten fields it is exactly wrong.

The third argument limits the effect to keys pressed inside a container:

```vue
<script setup>
const { setupTabNavigation } = useKeyListener()
const box = ref()

onMounted(() => {
  // Only inside #box, so the rest of the page keeps its normal Tab order.
  setupTabNavigation('first', 'last', document.getElementById('box'))
})
</script>

<template>
  <input id="before">
  <div id="box">
    <input id="first">
    <input id="middle">
    <input id="last">
  </div>
  <input id="after">
</template>
```

```text
Tab       from before -> first
Tab       from first  -> first
Tab       from middle -> first
Tab       from last   -> first
Shift+Tab from last   -> last
Shift+Tab from middle -> last
Shift+Tab from after  -> last
```

Inside the box, Tab always goes to `first` and Shift+Tab to `last`, and `middle` is out of reach by keyboard. Outside the box the listener is silent: the `before` and `after` rows show the browser's own order. With this argument the special behaviour stays inside a dialog and the rest of the page is left alone.

What most dialogs really need is a **focus trap**: Tab runs through all the fields in the browser's order and only wraps at the ends. `setupTabNavigation` does not do that, but two shortcuts do:

```js
// A focus trap: Tab on the last element goes to the first, Shift+Tab on the first goes to the last.
// Everything in between keeps the browser's own Tab order.
const first = document.getElementById('first')
const last = document.getElementById('last')

setupKeyBoardShortcuts({
  'Tab': (event) => {
    if (document.activeElement === last) { event.preventDefault(); first.focus() }
  },
  'Shift+Tab': (event) => {
    if (document.activeElement === first) { event.preventDefault(); last.focus() }
  }
})
```

```vue
<script setup>
const { setupKeyBoardShortcuts } = useKeyListener()

onMounted(() => {
  const first = document.getElementById('first')
  const last = document.getElementById('last')
  setupKeyBoardShortcuts({
    'Tab': (event) => {
      if (document.activeElement === last) { event.preventDefault(); first.focus() }
    },
    'Shift+Tab': (event) => {
      if (document.activeElement === first) { event.preventDefault(); last.focus() }
    }
  })
})
</script>

<template>
  <input id="first">
  <input id="middle">
  <input id="last">
  <p>x</p>
</template>
```

```text
Tab       from first  -> middle
Tab       from middle -> last
Tab       from last   -> first
Shift+Tab from first  -> last
Shift+Tab from middle -> first
Shift+Tab from last   -> middle
```

In the middle of the dialog, Tab and Shift+Tab keep the browser's behaviour. Only at the ends does the focus wrap: Tab on `last` goes to `first`, and Shift+Tab on `first` goes to `last`. Use `setupTabNavigation` for tiny groups and the shortcut pair for real dialogs.

## 5. Cleaning up

A listener installed on the page lives until somebody removes it. In a single-page application the page is replaced without a reload, so a listener that is not removed outlives the component that created it: it runs against elements that no longer exist, and one more is added every time the page is visited again.

The plain version gives you the remover and leaves the rest to you. The page below has two child components, one of each version, and removes both from the page:

```vue
<script setup>
const shown = ref(true)
const rows = ref([])

const Plain = {
  setup() {
    const { setupKeyBoardShortcuts } = useKeyListener()
    setupKeyBoardShortcuts({ F2: () => rows.value.push('F2 handled by the id-based component') })
    return () => h('span', 'id-based component')
  }
}
const Tidy = {
  setup() {
    const { setupKeyBoardShortcuts } = useKeyListenerRefBased()
    setupKeyBoardShortcuts({ F3: () => rows.value.push('F3 handled by the ref-based component') })
    return () => h('span', 'ref-based component')
  }
}
</script>

<template>
  <button id="remove" @click="shown = false">Remove both components</button>
  <Plain v-if="shown" />
  <Tidy v-if="shown" />
  <div v-for="(r, i) in rows" :key="i" class="row">{{ r }}</div>
</template>
```

```text
both on screen: F2                 -> F2 handled by the id-based component
both on screen: F3                 -> F3 handled by the ref-based component
components removed: F2             -> F2 handled by the id-based component
components removed: F3             -> nothing
```

While both components are shown, their shortcuts work. After the components are removed, the plain version's shortcut (`F2`) still runs and the ref-based version's (`F3`) does not. The rule is short: keep the remover and call it when the component goes.

```js
let stopShortcuts

onMounted(() => {
  stopShortcuts = setupKeyBoardShortcuts({ F2: rename })
})

// Without this the handler keeps running after the page is gone.
onBeforeUnmount(() => stopShortcuts?.())
```

Put the call in `onBeforeUnmount`. If the setup function runs inside `onMounted`, the remover has to be kept in a variable outside it, as in the code above. The ref-based version (next chapter) does this for you.

## 6. The ref-based version

The plain functions look elements up by `id` at the moment of the call. That is not enough for fields that come and go: a modal that opens later, a wizard that shows one step at a time. `useKeyListenerRefBased()` has the same three functions, but Enter and Tab navigation take refs instead of ids, and everything it sets up is removed automatically when the component is unmounted.

```vue
<script setup>
const { setupEnterNavigation } = useKeyListenerRefBased()

const name = ref()
const email = ref()
const city = ref()

// In setup(), not onMounted: the fields may appear later, and nothing has to be removed.
setupEnterNavigation([name, email, city])
</script>

<template>
  <UInput ref="name" />
  <UInput ref="email" />
  <UInput v-if="showCity" ref="city" />
</template>
```

Call it in `setup()`, not in `onMounted`. The fields do not have to exist when it is called: the listeners attach when they appear, and attach again if a field is re-created. A few more points:

- **Refs may point at components.** A ref to `UInput` or `UButton` works too; the focus goes to the input inside the component, or to its root element if there is no input.
- **`setupTabNavigation(firstRef, lastRef, container)`** takes a ref or an element as the container; the default is the whole page.
- **The returned remover is optional.** It is only for removing the listeners before the unmount.
- **On the server `setupKeyBoardShortcuts` does nothing** and returns an empty remover, so the page can be rendered there without errors.

The page below chains three fields, where the middle one comes and goes:

```vue
<script setup>
const { setupEnterNavigation, setupTabNavigation } = useKeyListenerRefBased()

const Field = {
  props: ['id'],
  render() { return h('div', { class: 'field' }, [h('input', { id: this.id })]) }
}

const first = ref()
const second = ref()
const third = ref()
const button = ref()
const showSecond = ref(false)

// Called in setup(): none of the fields exists yet, and that is fine.
setupEnterNavigation([first, second, third])
setupTabNavigation(first, button)
</script>

<template>
  <Field id="first" ref="first" />
  <Field v-if="showSecond" id="second" ref="second" />
  <Field id="third" ref="third" />
  <button id="button" ref="button" type="button">Go</button>
  <button id="toggle" type="button" @click="showSecond = !showSecond">Toggle the middle field</button>
</template>
```

```text
-- the middle field is not on the page
Enter in first                   -> focus on first
Shift+Enter in third             -> focus on third
Tab in third                     -> focus on first
Shift+Tab in third               -> focus on button
-- the middle field appears
Enter in first                   -> focus on second
Enter in second                  -> focus on third
Shift+Enter in third             -> focus on second
-- the middle field is removed again
Enter in first                   -> focus on first
```

Until the middle field appears, Enter in the first field stays where it is: a missing element is still a member of the chain, as in chapter 3. The Tab control works on the first and last targets (`first`, `button`) regardless. When the middle field appears, the chain connects by itself with no new call, and when it is removed again the chain is interrupted again.

So the ref-based version fixes the *timing* problem but not the *gap* problem. If the fields that come and go are in the middle of a chain, split the chain in two, or make sure the middle field is present whenever the chain is needed.

## 7. A quick entry form

The three habits together make a form that is fast to fill in without the mouse. Enter moves through the fields, Ctrl+Enter saves, and Escape clears the fields and returns to the first one. Everything is set up in `setup()` with the ref-based version, so nothing has to be removed by hand:

```vue
<script setup>
const { setupEnterNavigation, setupKeyBoardShortcuts } = useKeyListenerRefBased()

const Field = {
  props: ['id', 'label'],
  render() { return h('label', [this.label, h('input', { id: this.id })]) }
}

const name = ref()
const email = ref()
const city = ref()
const log = ref([])

setupEnterNavigation([name, email, city])

setupKeyBoardShortcuts({
  'Ctrl+Enter': (event) => {
    event.preventDefault()
    log.value.push('saved: ' + ['name', 'email', 'city'].map(id => document.getElementById(id).value).join(' / '))
  },
  'Escape': () => {
    ['name', 'email', 'city'].forEach((id) => { document.getElementById(id).value = '' })
    document.getElementById('name').focus()
    log.value.push('cleared')
  }
})
</script>

<template>
  <Field id="name" ref="name" label="Name" />
  <Field id="email" ref="email" label="Email" />
  <Field id="city" ref="city" label="City" />
  <div v-for="(l, i) in log" :key="i" class="row">{{ l }}</div>
</template>
```

```text
typed with Enter between fields, focus on city
saved: Sara / sara@example.org / Oslo
cleared
after Escape: focus on name, name ''
```

Typing and pressing Enter between the fields fills all three without the mouse and leaves the focus in the last field. Ctrl+Enter saves, and the output line shows the three values; its handler also cancels the browser's own action (chapter 2). Escape clears the fields and puts the focus back in the first. When the form is removed, the shortcuts and the Enter chain go with it (chapter 5), and the next form starts clean.

For your own forms the pattern is the same: put the text areas and buttons outside the Enter chain, give one-letter shortcuts a typing guard, register Persian spellings next to the Latin ones, and prefer the ref-based version whenever a field may not be on the page from the start.

---

## Appendix A. Quick reference

### A.1 Functions

| Call | Returns | Notes |
|------|---------|-------|
| `setupKeyBoardShortcuts({ combo: fn })` | Remover | `fn(event)`; listens on the whole page |
| `setupEnterNavigation([ids])` | Remover | Plain version: ids; ref-based: refs |
| `setupTabNavigation(first, last, container)` | Remover | Plain: ids and an element (default `document`); ref-based: refs |

### A.2 Combinations

| Written | Matches |
|---------|---------|
| `F2`, `Escape`, `Enter`, `ArrowUp`, `Tab` | The key alone |
| `s` | The letter, in any case; Meta is ignored |
| `Ctrl+s` | Control and the letter |
| `Ctrl+Shift+z` | Order is Ctrl, Shift, Alt, key |
| `Ctrl+س` | The letter the browser reports; layout dependent |

### A.3 Differences between the versions

| | Plain | Ref-based |
|---|-------|-----------|
| Targets | `id` strings | Refs (elements or components) |
| Elements must exist at the call | Yes | No |
| Cleanup | Call the remover | Automatic, remover still returned |
| Server | Needs `document` | Safe |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| Combination written in another order | Never matches |
| Capital letter in a combination | Never matches |
| Element `id` not found (plain) | Skipped at setup; neighbours still point at it |
| Next or previous `id` missing | Enter is cancelled and focus stays |
| Shortcut handler throws | An uncaught error in the console; later key presses still work |
| Remover not called (plain) | The listener keeps running |
| Called on the server (plain) | `document` is not defined |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Writing combinations in the wrong order** (`Alt+Ctrl+k`).
- **Capital letters in combinations.**
- **Relying on Command (Meta) on a Mac.** It is ignored; offer a `Ctrl` combination or add your own listener.
- **Keyboard layouts.** A Persian layout reports Persian letters.
- **One-letter shortcuts while typing.** Guard them.
- **Forgetting `preventDefault()`.** The browser's own action still happens.
- **Not calling the remover** of the plain version.
- **Including a text area or a button in the Enter chain.**
- **Expecting `setupTabNavigation` to cycle.** It moves every Tab to the first element.
- **A missing `id` in the middle of the chain.** The chain stops there.
- **Calling the plain version before the elements exist,** or on the server.
- **Registering the same combination twice in one object.** The last one wins.

### C.2 Behaviour worth knowing

- Only `keydown` is listened to.
- The shortcut object is read at the moment of each key press, by exact text.
- Enter navigation listens on each listed element; shortcuts and the default Tab control listen on the document.
- Enter navigation cancels the browser's action only when it moves the focus (or tries to).
- The Tab functions always cancel the browser's Tab and stop its propagation.
- The ref-based functions attach when the elements appear and clean up with the component.
- Several calls add several listeners; each has its own remover.

### C.3 Limits

| What | Limit |
|------|-------|
| Events | `keydown` only |
| Modifiers | Ctrl, Shift, Alt; not Meta |
| Key names | As the browser reports them |
| Tab control | First and last target only |
| Chain | In list order; stops at a missing element |
| Scope | One page; not other frames |

### C.4 Tuning

- Keep the remover and call it in `onBeforeUnmount`, or use the ref-based version.
- Use the ref-based version for modals, tabs and wizard steps.
- Leave text areas and buttons out of the Enter chain.
- Build real focus traps from shortcuts.
- Register Latin and Persian forms of a letter shortcut.
- Show the shortcuts in the page, next to the actions they trigger.
- Prefer `Ctrl+Enter`, function keys and other combinations that fields do not use.

## Appendix D. Message reference

The composable writes no messages of its own: it does not log and it does not throw. What reaches the console is the error from your own shortcut functions.
