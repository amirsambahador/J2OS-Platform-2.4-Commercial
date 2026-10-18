# DataTree

*J2OS Platform 2.4*

Folders inside folders are everywhere: departments in a company, categories in a shop, pages in a site map. Loading the whole thing at once is slow, and most of it is never opened. A better tree asks the server only for what the visitor opens.

**DataTree** is a Vue component for the Nuxt front end of J2OS that draws such a tree and fetches each level when it is first needed. It has three main features:

- **Lazy by default.** Only the top level is loaded at start. The children of a folder arrive when it is opened, and are kept, so closing and opening again costs nothing.
- **A menu on right-click.** The page gives a list of actions; the component shows them in a small menu, runs the chosen one and shows a spinner meanwhile.
- **Plain data in, plain functions out.** The server answers with a simple array. The page reacts through function properties, not events.

The whole thing at a glance:

```vue
<DataTree list-url="/api/folders" />                                   <!-- a tree that loads itself -->
<DataTree list-url="/api/folders" :on-node-click="open" />             <!-- react to a click on a leaf -->
<DataTree list-url="/api/folders" :actions="actions" />                <!-- right-click menu -->
```

Start with how the component works (chapter 1). It then covers the first tree (2), clicks (3), field names (4), the context menu (5), updating the tree after a change (6), reloading (7) and failures (8). Chapter 9 builds a small folder manager. The appendices are meant for quick lookups.

## Contents

1. [How DataTree works](#1-how-datatree-works)
2. [The first tree](#2-the-first-tree)
3. [Clicking](#3-clicking)
4. [Field names and the parent parameter](#4-field-names-and-the-parent-parameter)
5. [The context menu](#5-the-context-menu)
6. [Changing the tree after an action](#6-changing-the-tree-after-an-action)
7. [Reload, reset and starting by hand](#7-reload-reset-and-starting-by-hand)
8. [When it fails](#8-when-it-fails)
9. [Everything in one application](#9-everything-in-one-application)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Every example is a self-contained page of a Nuxt project, and each is a single `.vue` file. In Nuxt `ref` and the component `DataTree` need no `import`, so none is written.

---

## 1. How DataTree works

Let us settle the vocabulary first:

- **Node.** One line of the tree, built from one object the server sent.
- **Folder and leaf.** A node whose `hasChildren` is true is a folder: it can be opened. Any other node is a leaf.
- **Lazy loading.** The children of a folder are requested the first time it is opened, not before.
- **Action.** An entry of the right-click menu, described by an object with a `handler` function.
- **ctx.** A small toolbox, given to `onNodeClick` and to every action handler, for changing the tree.

The parts of this component:

| Piece | Role |
|-------|------|
| `listUrl` | Where the nodes come from (required) |
| `parentIdParam` | Name of the query parameter that carries the parent id |
| `fields` | Names of `id`, `label` and `hasChildren` in the server's objects |
| `actions` | The right-click menu |
| `onNodeClick` | Function called when a leaf is clicked |
| `onError` | Function called when something fails |
| `autoInit` | Whether the tree loads itself on mount |

Remember five things:

- **One URL, one parameter.** The top level is `listUrl` as it is; the children of a node are `listUrl` with the parent id added as a query parameter.
- **The server sends a plain array.** No wrapper object, no paging.
- **Children are fetched once.** After the first load they are cached until a reload or a refresh.
- **Clicking a folder opens or closes it.** `onNodeClick` is for leaves only.
- **Functions, not events.** `onNodeClick`, `onError` and the action handlers are properties that hold functions.

The tree is drawn with the Nuxt UI `UTree` component, with folder and file icons from Lucide, as in the rest of the J2OS front end.

## 2. The first tree

The server answers a request for the top level with an array:

```json
GET /api/tree

[
  { "id": 1, "label": "Documents", "hasChildren": true },
  { "id": 2, "label": "Pictures",  "hasChildren": true },
  { "id": 3, "label": "Notes",     "hasChildren": false }
]
```

Each folder is asked for its children by adding the parent id:

```json
GET /api/tree?parentId=1

[
  { "id": 4, "label": "Reports",  "hasChildren": true },
  { "id": 5, "label": "Invoices", "hasChildren": false }
]
```

The page only names the URL:

```vue
<script setup>
const clicked = ref('nothing')
</script>

<template>
  <DataTree list-url="/api/tree" :on-node-click="node => (clicked = node.label)" />

  <p id="clicked">Clicked: {{ clicked }}</p>
</template>
```

```text
start: ['GET /api/tree']
    [+] Documents
    [+] Pictures
     -  Notes
click Documents -> ['GET /api/tree?parentId=1']
    [-] Documents
      [+] Reports
       -  Invoices
    [+] Pictures
     -  Notes
click Reports -> ['GET /api/tree?parentId=4']
    [-] Documents
      [-] Reports
         -  Q1
         -  Q2
       -  Invoices
    [+] Pictures
     -  Notes
click Documents again -> []
    [+] Documents
    [+] Pictures
     -  Notes
click Documents a third time -> []
click leaf Q1 -> Clicked: Q1
click folder Reports -> Clicked: Q1
click leaf Notes -> Clicked: Notes | requests: []
```

Folders show a closed or an open folder icon; leaves show a file icon. Opening Documents sends one request, and so does opening Reports. Closing and opening Documents again sends none, because the children are already there.

## 3. Clicking

A click means one of two things. On a folder it opens or closes the folder. On a leaf it calls `onNodeClick` with the node and `ctx`. The last lines of the output above show it: clicking the leaf `Q1` sets the text, clicking the folder Reports leaves it as it was.

The node given to the function looks like this:

```js
{
  id: 4,
  label: 'Reports',
  parentId: 1,        // null for top-level nodes
  loaded: true,       // children have been fetched
  children: [ ... ],  // present only for folders
  raw: { ... }        // the object the server sent
}
```

`raw` is useful when the server sends more than the three basic fields, for example a path or a type. `onNodeClick` may be `async`; if it throws, the error goes to `onError` with the phase `nodeClick`.

## 4. Field names and the parent parameter

Servers do not always say `id`, `label` and `hasChildren`, and `parentId` is just a default. Both can be renamed:

```vue
<template>
  <DataTree
    list-url="/api/tree-legacy"
    parent-id-param="NODE_PARENT_ID"
    :fields="{ id: 'NODE_ID', label: 'NODE_NAME', hasChildren: 'NODE_HAS_CHILDREN' }"
  />
</template>
```

```text
['GET /api/tree-legacy']
['GET /api/tree-legacy?NODE_PARENT_ID=2']
    [+] Documents
    [-] Pictures
       -  Holiday
     -  Notes
```

`fields` lists only the names that differ; the others keep their defaults. The component reads `fields` once, when it is created, so they cannot be changed afterwards.

## 5. The context menu

A right-click on a node opens a menu next to the pointer. Its first line is the node's label; below it come the actions that apply to this node. Each action is an object:

```js
{
  key: 'rename',                 // unique inside the list
  label: 'Rename',               // the text in the menu
  icon: 'i-lucide-pencil',       // icon name
  color: 'text-blue-500',        // class for the icon and the text
  hoverClass: 'hover:bg-blue-50',
  visible: node => true,         // optional: hide the action for some nodes
  keepSelection: false,          // optional: keep the node selected afterwards
  handler: async (node, ctx) => { /* the work */ }
}
```

An action can be hidden for some nodes with `visible`. Here the root nodes cannot be deleted:

```vue
<script setup>
const actions = [
  {
    key: 'add',
    label: 'Add child',
    icon: 'i-lucide-plus',
    color: 'text-green-500',
    handler: async (node, tree) => {
      await fetch('/api/tree/add', { method: 'POST', body: JSON.stringify({ parentId: node.id }) })
      await tree.refreshNode(node)
    }
  },
  {
    key: 'delete',
    label: 'Delete',
    icon: 'i-lucide-trash',
    color: 'text-red-500',
    visible: node => node.parentId !== null,
    handler: async (node, tree) => {
      const res = await fetch('/api/tree/remove', { method: 'POST', body: JSON.stringify({ id: node.id }) })
      const { parentId } = await res.json()
      await tree.refreshNode(tree.findNodeById(parentId))
    }
  }
]
</script>

<template>
  <DataTree list-url="/api/tree" :actions="actions" />
</template>
```

```text
right-click Reports -> menu: ['Reports', 'Add child', 'Delete']
click elsewhere -> menu: None
right-click Documents (a root) -> menu: ['Documents', 'Add child']
Add child on Reports -> ['POST /api/tree/add', 'GET /api/tree?parentId=4']
    [-] Documents
      [-] Reports
         -  Q1
         -  Q2
         -  New 9
       -  Invoices
    [+] Pictures
     -  Notes
menu after the action: None
Delete Q1 -> ['POST /api/tree/remove', 'GET /api/tree?parentId=4']
    [-] Documents
      [-] Reports
         -  Q2
         -  New 9
       -  Invoices
    [+] Pictures
     -  Notes
right-click Notes (a leaf) -> menu: ['Notes', 'Add child']
Add child on Notes -> ['POST /api/tree/add', 'GET /api/tree?parentId=3']
    [-] Documents
      [-] Reports
         -  Q2
         -  New 9
       -  Invoices
    [+] Pictures
    [+] Notes
after clicking Notes:
    [-] Documents
      [-] Reports
         -  Q2
         -  New 9
       -  Invoices
    [+] Pictures
    [-] Notes
       -  New 10
```

While a handler runs, its node is dimmed and shows a spinner, and it ignores clicks. When the handler finishes the menu is gone and the selection is cleared; `keepSelection: true` keeps the node selected. A click outside the menu, or another right-click, closes it without doing anything.

Note the last step of the output: Notes was a leaf, and after it got a child it became a folder. The tree follows what the server says now, not what it said before.

## 6. Changing the tree after an action

An action usually changes something on the server, and the tree must show it. `ctx` has four tools for this:

| Tool | Use |
|------|-----|
| `refreshNode(node)` | Fetch the children of the node again |
| `findNodeById(id)` | Find a node that is already loaded (or `null`) |
| `purgeSubtree(node)` | Forget a node and everything below it |
| `reload()` | Reload the whole tree, keeping open folders open |

The pattern is the same in nearly every action: send the change, then refresh the parent. For a new child, the parent is the node itself, as in the example above. For a deleted node, the parent is found by id, as the server returns it.

The tree also has two shortcuts that do the same, available through the template ref (chapter 7) and not through `ctx`: `addReload(node)` is `refreshNode(node)`, and `changeReload(node, parentId)` forgets the node and refreshes its parent, or reloads the whole tree when `parentId` is empty. Chapter 9 uses `changeReload`.

## 7. Reload, reset and starting by hand

The methods of the tree are reached through a template ref. Three of them rebuild everything:

- `init()` loads the top level. It runs by itself on mount.
- `reload()` loads the top level again and re-opens the folders that were open.
- `reset()` loads the top level again and closes everything.

```vue
<script setup>
const tree = ref(null)
const found = ref('')
</script>

<template>
  <button id="reload" @click="tree.reload()">Reload</button>
  <button id="reset" @click="tree.reset()">Reset</button>
  <button id="find" @click="found = tree.findNodeById(7)?.label ?? 'not loaded'">Find 7</button>
  <span id="found">{{ found }}</span>

  <DataTree ref="tree" list-url="/api/tree" />
</template>
```

```text
Find 7 while Reports is open -> Q1
Reload -> ['GET /api/tree', 'GET /api/tree?parentId=1', 'GET /api/tree?parentId=4']
    [-] Documents
      [-] Reports
         -  Q1
         -  Q2
       -  Invoices
    [+] Pictures
     -  Notes
Reset -> ['GET /api/tree']
    [+] Documents
    [+] Pictures
     -  Notes
Find 7 after Reset -> not loaded
```

`reload` asks for the top level and then for every folder that was open, so the tree comes back looking the same. `reset` asks for the top level only; after it, `Q1` is no longer loaded, and `findNodeById` shows that.

While one of these runs, the tree is dimmed and ignores clicks and right-clicks.

With `:auto-init="false"` nothing is requested until the page calls `init()`. This is useful when the URL depends on something chosen first.

```vue
<script setup>
const tree = ref(null)
</script>

<template>
  <button id="start" @click="tree.init()">Start</button>
  <DataTree ref="tree" list-url="/api/tree" :auto-init="false" />
</template>
```

```text
before Start: requests [] | rows: 0
after Start: ['GET /api/tree'] | rows: 3
```

## 8. When it fails

Every failure goes to `onError`, once, and also to the browser console. The function receives the error and a context:

```js
function onError(error, context) {
  // error.message  the text
  // error.status   the HTTP status, when the server answered
  // context.phase  'init' | 'lazyLoad' | 'refresh' | 'action' | 'nodeClick'
  // context.node   the node involved (all phases except 'init')
  // context.action the action object (only 'action')
}
```

The next page has two trees. One lives on a server that answers 401 for the top level, the other on a server whose first request for the children of Documents fails:

```vue
<script setup>
const notes = ref([])
const report = (error, context) => notes.value.push(`${context.phase}: ${error.message} (${error.status})`)
</script>

<template>
  <DataTree list-url="/api/tree-flaky" :on-error="report" />
  <DataTree list-url="/api/tree-private" :on-error="report" />

  <p id="notes">{{ notes.join(' | ') }}</p>
</template>
```

```text
start: ['GET /api/tree-flaky', 'GET /api/tree-private']
notes: init: عدم احراز هویت؛ لطفاً دوباره وارد شوید (401)
click Documents -> ['GET /api/tree-flaky?parentId=1']
notes: init: عدم احراز هویت؛ لطفاً دوباره وارد شوید (401) | lazyLoad: خطا در دریافت اطلاعات (503)
    [-] Documents
    [+] Pictures
     -  Notes
click again (collapse) -> ['GET /api/tree-flaky?parentId=1']
click again (expand) -> []
    [-] Documents
      [+] Reports
       -  Invoices
    [+] Pictures
     -  Notes
```

Three things to notice:

- **A 401 has its own message** (`عدم احراز هویت؛ لطفاً دوباره وارد شوید`, "not signed in, please sign in again"). The page can look at `error.status` and send the visitor to the login page.
- **A failed folder stays open and empty.** Nothing is shown to the visitor on their own; `onError` is the only signal. The next click on the folder tries again, so closing and opening it is a retry.
- **A failed top-level request leaves the tree empty.** The tree stays empty until the page calls `init()` or `reload()` again.

## 9. Everything in one application

A small folder manager: clicking a leaf shows it above, the menu adds and deletes, a button reloads, and errors are collected for the page to show.

```vue
<script setup>
const tree = ref(null)
const current = ref('')
const problems = ref([])

async function post(url, body) {
  const res = await fetch(url, { method: 'POST', body: JSON.stringify(body) })
  if (!res.ok) throw new Error('Request failed: ' + res.status)
  return res.json()
}

const actions = [
  {
    key: 'add',
    label: 'Add child',
    icon: 'i-lucide-plus',
    handler: async (node, ctx) => {
      await post('/api/tree/add', { parentId: node.id })
      await ctx.refreshNode(node)
    }
  },
  {
    key: 'delete',
    label: 'Delete',
    icon: 'i-lucide-trash',
    visible: node => node.parentId !== null,
    handler: async (node, ctx) => {
      const { parentId } = await post('/api/tree/remove', { id: node.id })
      await tree.value.changeReload(node, parentId)
    }
  }
]

const onProblem = (error, context) => problems.value.push(context.phase + ': ' + error.message)
</script>

<template>
  <button id="reload" @click="tree.reload()">Reload</button>

  <DataTree
    ref="tree"
    list-url="/api/tree"
    :actions="actions"
    :on-node-click="node => (current = node.label)"
    :on-error="onProblem"
  />

  <p id="current">Open: {{ current || '-' }}</p>
  <p id="problems">{{ problems.join(' | ') }}</p>
</template>
```

```text
click leaf Q1 -> Open: Q1
Add child on Reports -> ['POST /api/tree/add', 'GET /api/tree?parentId=4']
[-] Documents
  [-] Reports
     -  Q1
     -  Q2
     -  New 9
   -  Invoices
[+] Pictures
 -  Notes
Delete Q2 -> ['POST /api/tree/remove', 'GET /api/tree?parentId=4']
[-] Documents
  [-] Reports
     -  Q1
     -  New 9
   -  Invoices
[+] Pictures
 -  Notes
Reload -> ['GET /api/tree', 'GET /api/tree?parentId=1', 'GET /api/tree?parentId=4']
[-] Documents
  [-] Reports
     -  Q1
     -  New 9
   -  Invoices
[+] Pictures
 -  Notes
problems: ''
```

Some choices in this page are worth copying:

- Deleting calls `changeReload` through the template ref. The node is forgotten and its parent is refreshed, using the `parentId` that the server returned.
- The `post` helper throws on a bad answer. An error thrown in a handler reaches `onError` with the phase `action`, and the spinner stops.
- Reload works without breaking anything: the open folders were opened again, and the new data is in them.

---

## Appendix A. Quick reference

### A.1 Properties

| Property | Type | Default | Meaning |
|----------|------|---------|---------|
| `listUrl` | String | (required) | URL of the nodes; must not contain `?` |
| `parentIdParam` | String | `parentId` | Query parameter for the parent id |
| `fields` | Object | `{}` | Names of `id`, `label`, `hasChildren` |
| `actions` | Array | `[]` | Right-click menu actions |
| `onNodeClick` | Function | none | `(node, ctx)`, called for leaves |
| `onError` | Function | none | `(error, context)` |
| `autoInit` | Boolean | `true` | Load the tree on mount |

### A.2 The action object

| Field | Meaning |
|-------|---------|
| `key` | Unique name of the action |
| `label` | Text in the menu |
| `icon` | Icon name |
| `color` | Class for the icon and the text |
| `hoverClass` | Class for the hover effect |
| `visible` | `(node) => boolean`; the action is shown when it returns true or is missing |
| `keepSelection` | `true` to keep the node selected after the action |
| `handler` | `async (node, ctx) => ...` |

### A.3 Tools and methods

| Name | In `ctx` | Through ref | Meaning |
|------|:--------:|:-----------:|---------|
| `refreshNode(node)` | yes | yes | Fetch the children again |
| `findNodeById(id)` | yes | yes | Find a loaded node or `null` |
| `purgeSubtree(node)` | yes | yes | Forget a node and its subtree |
| `reload()` | yes | yes | Reload the top level, keep open folders |
| `init()` | no | yes | First load |
| `reset()` | no | yes | Reload and close all folders |
| `addReload(node)` | no | yes | Same as `refreshNode` |
| `changeReload(node, parentId)` | no | yes | Forget the node, refresh its parent or reload everything |

### A.4 Server contract

| Request | Answer |
|---------|--------|
| `listUrl` | Array of objects for the top level |
| `listUrl?parentId=ID` | Array of objects for the children of `ID` |

Each object needs an id, a label and, for folders, a true `hasChildren`. Cookies are sent with every request.

## Appendix B. Failure handling

| What failed | Phase | What the visitor sees | Recovery |
|-------------|-------|-----------------------|----------|
| First load | `init` | An empty tree | Call `init()` or `reload()` |
| Opening a folder | `lazyLoad` | An open, empty folder | Click the folder again |
| `refreshNode` | `refresh` | An empty folder that loads again on click | Click the folder, or refresh again |
| An action's handler | `action` | The spinner stops | Run the action again |
| `onNodeClick` | `nodeClick` | Nothing | None needed |
| `reload()` or `reset()` | none | Nothing; the call rejects | Catch the rejection |

`onError` is called once per error, even when the error passes through several layers, for example when `refreshNode` fails inside a handler.

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **`onNodeClick` does not fire for folders.** A click on a folder only opens it. To react to folders, use an action.
- **`reload()` and `reset()` do not report to `onError`.** A failure rejects the promise they return. Without a `catch` it ends up as an unhandled rejection. Only `init()` reports.
- **`ctx` has four tools, not all of them.** `changeReload`, `addReload`, `init` and `reset` are only on the template ref. In a handler, use the ref, or combine `purgeSubtree`, `findNodeById` and `refreshNode`.
- **Ids must be unique in the whole tree.** Nodes are found by id, so the same id on two levels makes one hide the other.
- **`listUrl` must not contain `?`.** The parent id is added with a `?`, so an existing query string breaks the URL. Put the extra parameters in the path.
- **A delete needs the parent's id.** After a delete, the page must refresh the parent. The server can return its id, as in the examples, or the page can read `node.parentId` before the call.
- **`fields` is read once.** Changing the property later has no effect.
- **A root operation ignores clicks.** During `init`, `reload` and `reset` the tree is dimmed at 60 percent and does not react.

### C.2 Behavior worth knowing

- Closing a folder and opening it again does not send a request. A refresh is the only way to see new children without reloading.
- Reload keeps the open folders open and loads them again; Reset closes them.
- A leaf that gets children after a refresh turns into a folder.
- A folder whose children failed to load is retried on every click until it succeeds.
- The menu is 192 pixels wide and opens to the left of the pointer, so it stays on the screen near the right edge.
- A right-click while the tree is loading does nothing.
- The tree sends cookies (`credentials: 'include'`). Token headers are not sent by default.

### C.3 Limits

| Item | Value |
|------|-------|
| Levels | No limit; each level is a request |
| Siblings per level | All are drawn; no paging |
| Menu width | 192 pixels |
| Reserved id | `__dummy__` |
| Requests per folder | One until a refresh |

### C.4 Tuning

- Return only one level per request; the component asks for deeper levels itself.
- For very wide levels, group the nodes on the server (by letter, by date) so no level has thousands of rows.
- Keep the labels short; they are cut with an ellipsis in one line.
- Refresh the smallest part you can: `refreshNode(parent)` instead of `reload()`.

## Appendix D. Message reference

### D.1 Texts

| Text | When |
|------|------|
| `عدم احراز هویت؛ لطفاً دوباره وارد شوید` | The server answered 401 |
| `خطا در دریافت اطلاعات` | The server answered any other error status |

Both are the `message` of the error given to `onError`; `status` has the number. A network failure gives the browser's own error instead, without `status`.
