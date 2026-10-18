# DataTable

*J2OS Platform 2.4*

Business screens are mostly tables: orders, customers, invoices. The visitor wants to see a page of rows, sort by a column, search, move between pages, pick a row and then press a button such as "edit" or "delete". Built by hand, each table repeats the same requests, the same page buttons and the same care for slow networks and failures.

**DataTable** is a Vue component for the Nuxt front end of J2OS that does this once. It is built on the table and pagination components of Nuxt UI. It has three main features:

- **The server does the work.** DataTable sends `page`, `rows`, `q`, `sort` and `order` and shows what comes back. Paging, searching and sorting happen on the server, so a table can be as long as the database.
- **Columns are described, not drawn.** A column is a small object: which field, what heading, whether it sorts, and optionally how to compute or render the cell.
- **Selecting is built in.** A row can be chosen with the mouse or the keyboard, one at a time or many, and the page is told through an event.

Here is the shape of it:

```vue
<DataTable
  url="/api/orders?"
  :columns="[{ field: 'id', label: 'No.', sortable: true }, { field: 'total', label: 'Total' }]"
  default-sort="id"
  @selected="row = $event"
/>
```

The `url` says where the rows are, `columns` describes the table, `default-sort` is the sort at the start, and `@selected` brings the chosen row. We start with how the component works (chapter 1) and the first table (2). It then covers sorting, paging and search (3), columns (4), choosing a row (5) and many rows (6), row colors (7), tables that depend on tables (8) and failures (9). Chapter 10 builds a management screen. The appendices are meant for quick lookups.

## Contents

1. [How DataTable works](#1-how-datatable-works)
2. [The first table](#2-the-first-table)
3. [Sorting, paging and search](#3-sorting-paging-and-search)
4. [Columns](#4-columns)
5. [Choosing a row](#5-choosing-a-row)
6. [Choosing many rows](#6-choosing-many-rows)
7. [Row colors](#7-row-colors)
8. [Tables that depend on tables](#8-tables-that-depend-on-tables)
9. [When it fails](#9-when-it-fails)
10. [A complete example](#10-a-complete-example)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Nothing here is a fragment; each example is a page of a Nuxt project, and each is a single `.vue` file. In Nuxt `ref` and `computed` and the component `DataTable` need no `import`, so none is written. The texts that DataTable shows by itself (the search hint, the loading message) are Persian by default.

---

## 1. How DataTable works

Here is the vocabulary in one place:

- **Row.** One object of the list, as the server sent it. On screen it is one line of the table.
- **Column.** An object that describes one column of the table (chapter 4).
- **Page.** A group of rows, numbered from 1; its size is `pageSize`.
- **Selection.** The row (or rows) the visitor has chosen.
- **Row key.** The field that identifies a row; it is `id` unless `row-key` says otherwise.

What the component is made of:

| Piece | Role |
|-------|------|
| `url`, `columns`, `defaultSort` | Required: where the rows are, how to draw them and the starting sort |
| `pageSize`, `defaultOrder`, `rowKey`, `emptyText` | The page size (10), the starting direction (`ASC`), the field that identifies a row (`id`) and the empty text |
| `rowStyle`, `multiSelectable` | A function that styles the cells of a row; many-row selection |
| Event `selected` | The chosen row, or the array of rows, or `null` |
| Event `loaded` | The data the server answered with |
| Event `error` | A request failed |
| Slots `before-search`, `after-search` | Places on the line with the search box |
| Methods `reload`, `focus`, `blur` | Start again; move the keyboard focus |

Five things shape what follows:

- **The address is a prefix.** DataTable appends the query string to `url` as it is, so `url` must end with `?` or `&`.
- **Every request carries the sort.** `sort` and `order` are always sent, starting from `default-sort`. A server that insists on a sort field, like `page2`, is satisfied from the first request.
- **One request at a time.** A new request cancels the one still running, and an answer that is too late is ignored.
- **The page numbers are real.** Unlike the infinite scroll of DataCard, DataTable shows page buttons at the bottom, and always shows them, even when a request failed.
- **Browser only.** The first request is sent after the component is mounted in the browser.

## 2. The first table

A table needs an address, the description of its columns and the starting sort:

```vue
<script setup>
const columns = [
  { field: 'id', label: 'No.', sortable: true },
  { field: 'title', label: 'Title', sortable: true },
  { field: 'price', label: 'Price', sortable: true }
]
</script>

<template>
  <DataTable url="/api/items?" :columns="columns" default-sort="id" :page-size="5" />
</template>
```

- **`url="/api/items?"`.** It ends with `?`.
- **`columns`.** Three columns; `field` is the key in each row and `label` is the heading.
- **`default-sort="id"`.** The sort at the start. The first request already carries `sort=id&order=ASC`, and the heading of that column shows an arrow.
- **The first column** is added by DataTable: it holds the radio button (or the check box) of chapter 5.
- **`:page-size="5"`.** Five rows per request; the 25 rows of the test make five pages.

The server must answer with JSON that has `rows` (this page) and `total` (all matching rows):

```json
{
  "rows": [ { "id": 11, "title": "Item 11", "price": 41 } ],
  "total": 25
}
```

In J2OS the `page2` module reads exactly the parameters above and answers exactly this JSON:

```java
@GetMapping("/api/items")
public Object items(@RequestParam Map<String, Object> params) {
    PageDataJPQL page = new PageDataJPQL(entityManager);
    page.searchAndSortOn("id", "title", "price");                 // what q searches and sort may name
    return page.getResult("select o from Item o", "select count(o) from Item o",
                          "o", Item.class, null, params);        // { rows, total, page }
}
```

## 3. Sorting, paging and search

A heading with `sortable: true` is a button. Clicking it sorts by that column; clicking the same heading again reverses the direction. Page buttons ask for a page, and the search box asks for the rows that match. The output below shows the requests that were sent:

```text
GET /api/items?page=1&rows=5&q=&sort=id&order=ASC
headers: ['گزینه', 'No.↑', 'Title⇅', 'Price⇅']
ids on screen: ['1', '2', '3', '4', '5'] | pages: 5
click Title -> GET /api/items?page=1&rows=5&q=&sort=title&order=ASC | Title↑ | first: Item 01
click Title again -> GET /api/items?page=1&rows=5&q=&sort=title&order=DESC | Title↓ | first: Item 25
click Price -> GET /api/items?page=1&rows=5&q=&sort=price&order=ASC | ['گزینه', 'No.⇅', 'Title⇅', 'Price↑']
page 3 -> GET /api/items?page=3&rows=5&q=&sort=price&order=ASC
sort again while on page 3 -> ['GET /api/items?page=1&rows=5&q=&sort=title&order=ASC']
search "Item 1" from page 3 -> ['GET /api/items?page=1&rows=5&q=Item+1&sort=title&order=ASC'] | rows: 5
ArrowLeft -> ['GET /api/items?page=2&rows=5&q=&sort=title&order=ASC']
ArrowLeft, ArrowRight -> ['GET /api/items?page=3&rows=5&q=&sort=title&order=ASC', 'GET /api/items?page=2&rows=5&q=&sort=title&order=ASC']
ArrowRight three more times -> 1 request(s); on page 1 it stops
```

- **The arrow.** `↑` and `↓` mark the sorted column, which is also bold. Other sortable columns show `⇅`; a column that is not sortable shows nothing.
- **A new sort** goes back to page 1. If the visitor is on page 3 and clicks a heading, the request is for page 1.
- **A new search** also goes back to page 1, and it waits 500 milliseconds after the last key.
- **The keyboard** works when the table has the focus: Arrow Left goes to the next page and Arrow Right to the previous one (the page is right-to-left). On page 1 Arrow Right does nothing.

```text
Arrow Left    next page        (pages are right-to-left)
Arrow Right   previous page
Arrow Down    move to the next row (the first press goes to row 1)
Arrow Up      move to the previous row
Space         select or unselect the row that is marked
```

## 4. Columns

A column is an object:

```js
{
  field: 'price',                    // the key in the row; the column is hidden when no row has it
  label: 'Price',                    // the heading
  sortable: true,                    // the heading becomes a button
  sortName: 'unit_price',            // what is sent as `sort` (default: field)
  processor: row => row.price * 2,   // computes what the cell shows (default: row[field])
  render: false                      // true: the result is inserted as HTML
}
```

Here are the options at work:

```vue
<script setup>
const { toPlainText } = useXssProtector()

const columns = [
  { field: 'id', label: 'No.', sortable: true },
  { field: 'title', label: 'Title', sortable: true, processor: row => `${row.title}!` },
  { field: 'price', label: 'Price (x2)', sortable: true, sortName: 'price', processor: row => row.price * 2 },
  { field: 'title', label: 'Bold', render: true, processor: row => `<b>${toPlainText(row.title)}</b>` },
  { field: 'owner', label: 'Owner' },
  { label: 'Computed', processor: row => `#${row.id}-${row.price}` }
]
</script>

<template>
  <DataTable url="/api/items?" :columns="columns" default-sort="id" :page-size="3" />
</template>
```

```text
headers: ['گزینه', 'No.↑', 'Title⇅', 'Price (x2)⇅', 'Bold', 'Computed']
row 1: ['', '1', 'Item 01!', '20', 'Item 01', '#1-10']
Bold cell html: <div class="px-4 py-3" style=""><b>Item 01</b></div>
click "Price (x2)" -> GET /api/items?page=1&rows=3&q=&sort=price&order=ASC
click "Title" -> GET /api/items?page=1&rows=3&q=&sort=title&order=ASC
buttons disabled in header (not sortable): ['No.⇅:false', 'Title↑:false', 'Price (x2)⇅:false', 'Bold:true', 'Computed:true']
```

- **`processor`** receives the whole row and returns what the cell shows: the title with a `!`, a doubled price, and a value built from two fields.
- **`sortName`** is what is sent as `sort` when the heading is clicked. Clicking the `Price (x2)` heading sends `sort=price`. Without `sortName`, `field` is sent.
- **`render: true`** inserts the result as HTML. The "Bold" column wraps the title in `<b>`. The title is cleaned with `toPlainText` first, because `render` inserts content into the page exactly as it is.
- **A column without `field`** (here "Computed") always shows. It cannot sort unless `sortName` is given.
- **A column whose field is missing** (here "Owner") is hidden. DataTable shows a column with a `field` only if some row of the current page has that key.

## 5. Choosing a row

By default one row can be chosen. The first column holds a radio button, and clicking the row does the same. The page hears about it through `selected`:

```vue
<script setup>
const table = ref(null)
const selected = ref(null)

function reload() {
  table.value.reload()
}
</script>

<template>
  <DataTable
    ref="table"
    url="/api/items?"
    :columns="[{ field: 'id', label: 'No.', sortable: true }, { field: 'title', label: 'Title', sortable: true }]"
    default-sort="id"
    :page-size="5"
    @selected="selected = $event"
  >
    <template #after-search>
      <button id="reload" @click="reload">Reload</button>
      <button id="show" :disabled="!selected">Show</button>
    </template>
  </DataTable>

  <p id="picked">{{ selected ? selected.title : 'Nothing selected' }}</p>
</template>
```

```text
click row 2 -> Item 02 | checked radios: 1 | Show disabled: False
click row 4 -> Item 04 | checked radios: 1
click the same radio again -> Item 04 | checked: 1
page 2 -> Item 04 | checked on screen: 0
back to page 1 -> checked on screen: 1
Reload -> Nothing selected | checked: 0 | Show disabled: True
```

- **`selected`** brings the row object exactly as the server sent it, never the processed cells. It brings `null` when the selection is cleared.
- **A new choice** replaces the old one. A chosen row cannot be un-chosen by clicking it again.
- **The choice survives paging.** On page 2 no row is checked on screen, but the choice made on page 1 is still kept, and the row is checked again when page 1 returns. The `selected` event is not repeated.
- **`reload()` clears it** and fires `selected` with `null`. Changing `url` or `page-size` does the same.
- **The keyboard.** With the table focused, Arrow Down marks the first row, further presses move down, and Space chooses the marked row. The mark moves independently of the mouse.

## 6. Choosing many rows

With `multi-selectable` the first column holds check boxes, `selected` brings an array, and the array can contain rows from several pages:

```vue
<script setup>
const picked = ref([])
</script>

<template>
  <DataTable
    url="/api/items?"
    :columns="[{ field: 'id', label: 'No.' }, { field: 'title', label: 'Title', sortable: true }]"
    default-sort="id"
    :page-size="5"
    multi-selectable
    @selected="picked = $event"
  />

  <p id="picked">Selected: {{ picked.map(r => r.id).join(', ') || 'none' }}</p>
</template>
```

```text
rows 1 and 3 -> Selected: 1, 3
page 2, row 6 -> Selected: 1, 3, 6
page 1, row 3 again -> Selected: 1, 6
after sorting -> Selected: 1, 6
Down, Down -> selected focused - - -
Space -> Selected: 1, 2, 6 | selected selected - - -
```

- **Clicking a row toggles it.** Choosing row 6 on page 2 adds it to rows 1 and 3 of page 1; clicking row 3 again removes it.
- **The array holds the original rows,** not in the order of choice. With numeric keys it comes sorted by key: if you choose 5, 1 and 3 in that order, the array is 1, 3, 5. Do not rely on its order.
- **Sorting and paging keep the choices.** `reload()` and a change of `url` or `page-size` clear them and fire `selected` with `[]`.
- **A marked row** is shaded lightly, a chosen row strongly, and when both are true the chosen look wins. In the output above, rows 1 and 2 are chosen and the mark is on row 2.

## 7. Row colors

`row-style` is a function. It receives a row and returns a CSS string, which DataTable puts on every cell of that row:

```vue
<script setup>
const columns = [
  { field: 'id', label: 'No.' },
  { field: 'price', label: 'Price' }
]
const rowStyle = row => (row.price > 80 ? 'background: #fee; color: #900;' : '')
</script>

<template>
  <DataTable url="/api/items?" :columns="columns" :row-style="rowStyle" default-sort="id" :page-size="25" />
</template>
```

```text
rows: 25 | rows with a style: 6 | their prices: ['84', '89', '94', '99', '83', '88']
```

- **Rows with a price above 80** get the red style (six rows in the output).
- **The style goes on the box inside each cell,** not on the line itself. How it mixes with the component's hover and selection colors depends on the CSS, so look at a chosen row with your colors before using them.
- **Return an empty string** for rows that need no style.

## 8. Tables that depend on tables

An empty `url` means "nothing to load": DataTable does not send a request, clears its rows and shows the empty text. A child table can therefore be driven by the chosen row of a parent:

```vue
<script setup>
const parent = ref(null)
const childUrl = computed(() => (parent.value ? `/api/children?parent=${parent.value.id}&` : ''))
const cols = [{ field: 'id', label: 'No.' }, { field: 'title', label: 'Title' }]
</script>

<template>
  <DataTable url="/api/items?" :columns="cols" default-sort="id" :page-size="3" @selected="parent = $event" />

  <DataTable
    :url="childUrl"
    :columns="cols"
    default-sort="id"
    empty-text="Select a row above"
  />
</template>
```

```text
start: requests: ['GET /api/items?page=1&rows=3&q=&sort=id&order=ASC'] | child says: 1 table(s) | 
text shown below: ['Select a row above']
click parent row 2 -> ['GET /api/children?parent=2&page=1&rows=10&q=&sort=id&order=ASC'] | child rows: ['Part 2-1', 'Part 2-2', 'Part 2-3']
click parent row 1 -> ['GET /api/children?parent=1&page=1&rows=10&q=&sort=id&order=ASC'] | child rows: ['Part 1-1', 'Part 1-2', 'Part 1-3']
```

- **At the start,** while no row is chosen, the child table sends no request and shows its empty text.
- **Choosing a parent row** changes the child's `url`, so the child loads from page 1 with `parent=2` in front of the DataTable parameters.
- **The child's selection** is cleared whenever its `url` changes.
- **`loaded`** is emitted with `null` for the empty address (chapter 9).

## 9. When it fails

If a request fails, the table shows an error message and no rows. The `error` event brings the exception and `loaded` brings every successful answer, with `null` for an empty `url`:

```vue
<script setup>
const notes = ref([])
const cols = [{ field: 'id', label: 'No.' }]
</script>

<template>
  <DataTable
    url="/api/broken?"
    :columns="cols"
    default-sort="id"
    @error="e => notes.push(`error: ${e.message}`)"
    @loaded="d => notes.push(`loaded: ${d && d.total}`)"
  />
  <DataTable
    url="/api/items?"
    :columns="cols"
    default-sort="id"
    :page-size="2"
    @error="e => notes.push(`error: ${e.message}`)"
    @loaded="d => notes.push(`loaded: ${d && d.total}`)"
  />
  <DataTable
    url=""
    :columns="cols"
    default-sort="id"
    empty-text="No address yet"
    @loaded="d => notes.push(`loaded: ${d === null ? 'null' : d.total}`)"
  />

  <p id="notes">{{ notes.join(' | ') }}</p>
</template>
```

```text
notes: loaded: null | error: HTTP 500 | loaded: 25
screens: ['خطا در دریافت اطلاعات وجود دارد', 'No address yet']
```

- **The error screen** says the data could not be fetched. There is no retry button; the visitor changes the page or the page calls `reload()`.
- **The page buttons stay.** They are shown while loading, on an error and when the table is empty, so the visitor can move on.
- **`loaded`** carries the whole JSON, so a page can read more than `rows` and `total`, such as totals computed by the server.
- **A canceled request** is not a failure and fires nothing.

## 10. A complete example

A management screen: choose a row, press Delete, and the table refreshes without a selection. The Delete button is disabled until a row is chosen:

```vue
<script setup>
const table = ref(null)
const row = ref(null)
const columns = [
  { field: 'id', label: 'No.', sortable: true },
  { field: 'title', label: 'Title', sortable: true },
  { field: 'price', label: 'Price', sortable: true }
]

async function remove() {
  await fetch(`/api/items/${row.value.id}`, { method: 'DELETE', credentials: 'include' })
  table.value.reload()
}
</script>

<template>
  <DataTable
    ref="table"
    url="/api/items?"
    :columns="columns"
    default-sort="id"
    :page-size="5"
    @selected="row = $event"
  >
    <template #after-search>
      <button id="refresh" @click="table.reload()">Refresh</button>
      <button id="delete" :disabled="!row" @click="remove">Delete</button>
    </template>
  </DataTable>

  <p id="info">{{ row ? `Selected: ${row.title}` : 'No selection' }}</p>
</template>
```

```text
before: No selection | Delete disabled: True
click row 2 -> Selected: Item 02 | Delete disabled: False
['DELETE /api/items/2', 'GET /api/items?page=1&rows=5&q=&sort=id&order=ASC']
after: No selection | titles: ['Item 01', 'Item 03', 'Item 04', 'Item 05', 'Item 06']
```

- **`@selected="row = $event"`** keeps the chosen row in the page; the buttons look at it.
- **`#after-search`** puts the buttons on the line with the search box.
- **`remove`** sends the delete to the server and calls `reload()`. The table loads the same page again with no selection, and "Item 02" is gone.

## Appendix A. Quick reference

### A.1 Properties

| Property | Type | Default | Meaning |
|----------|------|---------|---------|
| `url` | String, required | - | The address; must end with `?` or `&`; an empty string means nothing to load |
| `columns` | Array, required | - | The column objects (A.2) |
| `defaultSort` | String, required | - | The field to sort by at the start |
| `defaultOrder` | String | `'ASC'` | `ASC` or `DESC` |
| `pageSize` | Number | `10` | Rows per request (`rows`) |
| `rowKey` | String | `'id'` | The field that identifies a row |
| `emptyText` | String | `داده‌ای وجود ندارد` | Text when there are no rows |
| `rowStyle` | Function | none | Row to CSS string |
| `multiSelectable` | Boolean | `false` | Check boxes and an array in `selected` |

### A.2 Column options

| Option | Meaning |
|--------|---------|
| `field` | The key in the row; optional; the column is hidden when no row on the page has it |
| `label` | The heading |
| `sortable` | The heading is a button that sorts |
| `sortName` | The value sent as `sort`; defaults to `field` |
| `processor` | Function from the row to the cell's content |
| `render` | `true`: the content is inserted as HTML |

### A.3 Events, slots and methods

| Name | Meaning |
|------|---------|
| `selected` | The row, or `null`; with `multiSelectable` an array |
| `loaded` | The JSON of a successful answer, or `null` for an empty `url` |
| `error` | The exception of a failed request |
| Slot `before-search`, `after-search` | Content before and after the search box |
| `reload()` | Clear the selection and load again (same page) |
| `focus()`, `blur()` | Give or take the keyboard focus |

### A.4 What is sent and what is expected

| Direction | Content |
|-----------|---------|
| Request | `GET <url>page=<n>&rows=<pageSize>&q=<text>&sort=<s>&order=<o>` with cookies |
| Answer | `{ "rows": [...], "total": <number> }` |

## Appendix B. Failure handling

| Situation | What the visitor sees | What happens |
|-----------|-----------------------|--------------|
| The request fails (network or HTTP status) | The error message; page buttons stay | Rows cleared; `error` fires |
| The answer is not JSON | Same as a failure | The parse error reaches `error` |
| `rows` is missing or not an array | The empty text | Counts as no rows |
| `total` is missing | Rows shown; one page button | Counts as `0` |
| `url` is empty | The empty text | No request; `loaded` with `null` |
| A newer request replaces an older one | Nothing | The old one is canceled |
| The component is removed | Nothing | The request and the timer stop |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **An address without `?` or `&` at the end.** The parameters are glued to the path.
- **A column that vanishes.** A column with a `field` is hidden on a page where no row has that key. A server that leaves out `null` values makes the column disappear whenever a whole page is empty in it.
- **`render: true` with raw text.** The content is inserted as HTML. Clean anything that comes from users, as the examples do with `toPlainText`.
- **A server that does not know the sort field.** `page2` refuses fields not registered in `searchAndSortOn`; give `sortName` the server's name.
- **Hidden selections.** The selection survives paging, so the chosen rows can be off screen. Show how many are chosen.
- **A radio that cannot be cleared.** In single mode only `reload()` empties the choice.
- **Expecting a retry button.** There is none; use `reload()`.
- **Arrow keys in left-to-right pages.** Arrow Left means "next page"; the keys follow right-to-left pages.
- **Color on the line.** `row-style` colors the box inside each cell, so check how a chosen or hovered row looks with your colors.

### C.2 Behavior worth knowing

- **Row clicks select.** A click anywhere on a row chooses it and moves the keyboard focus to the table.
- **The text of the first column's heading** is "گزینه" ("choice") and is fixed.
- **`selected` brings the original row,** not what `processor` made.
- **Several tables can share a page.** Each has its own radio group.
- **`reload()` keeps the page.** It loads the current page again with the same sort and search, and clears the selection.
- **Every request replaces the table.** While rows are loading, even for a page change, the table gives way to the loading message.
- **`reload()` does not wait for the search.** Text typed less than 500 milliseconds earlier is used at once, on the current page.

### C.3 Limits

| What | Limit |
|------|-------|
| Wait after typing in the search box | 500 milliseconds, fixed |
| Sort | One column and one direction at a time |
| Page buttons | Those of Nuxt UI, with two neighbors on each side |
| Texts | Persian, fixed, except `emptyText` |
| Selection | Lives in the component; lost on reload |

### C.4 Tuning

- **`pageSize`.** Use a size that fits the screen, 10 to 50 rows, and let the server do the rest.
- **Search.** Make `q` search the columns people really look at.
- **Heavy cells.** Compute them in `processor` from fields already in the row rather than asking for extra data.
- **Server indexes.** Sorting on a column that is not indexed is the usual reason a table is slow.

## Appendix D. Message reference

### D.1 Texts on the screen

| Text | Meaning | When |
|------|---------|------|
| جستجو... | Search... | The placeholder of the search box |
| در حال بارگذاری... | Loading... | A request is running |
| خطا در دریافت اطلاعات وجود دارد | There was an error getting the data | A request failed |
| داده‌ای وجود ندارد | There is no data | No rows, when `emptyText` is not set |
| گزینه | Choice | The heading of the selection column |

### D.2 The error event

| Message | Cause |
|---------|-------|
| `HTTP <status>` | The server answered with a status outside 200 to 299 |
| The text of the browser's error | The network failed, or the answer was not JSON |
