# DataCard

*J2OS Platform 2.4*

Almost every application has a page that shows a long list: products, articles, people, orders. Nobody wants all of it at once, so the page loads a few rows, loads more as the visitor scrolls, lets them search, lets them sort, and handles the moments when the network is slow or the server fails. Written by hand, this is the same few hundred lines on every page.

**DataCard** is a Vue component for the Nuxt front end of J2OS that does this work once. It shows rows as *cards* in a responsive grid. It has three properties:

- **The server does the work.** DataCard only sends `page`, `rows`, `q`, `sort` and `order` to an address and shows what comes back. Paging, searching and sorting happen on the server, so the list can be as long as the database.
- **The card is yours.** DataCard draws the search box, the grid and the messages. What one row looks like is a *slot* that you fill with anything: a plain `div`, a Nuxt UI card, an image.
- **It looks after the hard moments.** Loading, an empty result, a failure on the first page, a failure while loading more with a retry button, and a stale answer arriving late are all handled.

In a few lines:

```vue
<DataCard url="/api/items?" :page-size="12" :cols="{ sm: 1, md: 3 }">     <!-- 1. where the rows come from, how many per page, how many columns -->
  <template #default="{ item }">                                         <!-- 2. how ONE row looks -->
    <UCard>{{ item.title }}</UCard>
  </template>
</DataCard>
```

This guide starts with how the component works (chapter 1) and the first card (2). It then covers the server it talks to (3), loading while scrolling (4), search (5), sorting (6), columns (7), slots and methods (8) and failures (9). Chapter 10 builds a small catalog. The appendices work as a reference.

## Contents

1. [How DataCard works](#1-how-datacard-works)
2. [The first card](#2-the-first-card)
3. [What the server must do](#3-what-the-server-must-do)
4. [Loading while the visitor scrolls](#4-loading-while-the-visitor-scrolls)
5. [Searching](#5-searching)
6. [Sorting](#6-sorting)
7. [Columns](#7-columns)
8. [Slots and methods](#8-slots-and-methods)
9. [When it fails](#9-when-it-fails)
10. [Putting it together](#10-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are complete pages of a Nuxt project: each is one `.vue` file. In Nuxt the functions `ref` and `computed` and the components `DataCard`, `USelect`, `UCard`, `UBadge` and `UButton` need no `import`, so none is written. The examples fetch from small addresses such as `/api/items?` that chapter 3 describes.

---

## 1. How DataCard works

First, a few terms, because they come up throughout the guide:

- **Row (item).** One entry of the list, for example one product. On the page it becomes one card.
- **Page.** A group of rows, numbered from 1. The size of a page is the `pageSize` property.
- **Total.** The number of rows the server has for the current search. DataCard compares it with the rows it already holds to know whether more exist.
- **Sentinel.** An invisible one-pixel line under the last card. When the visitor scrolls it into view, the next page is requested.
- **Slot.** A place in the component that the page fills with its own markup.

The parts of this component:

| Piece | Role |
|-------|------|
| Properties `url`, `pageSize`, `itemKey`, `cols`, `emptyText` | The address, the page size, the field that identifies a row, the grid columns and the text for an empty result |
| Models `v-model:sort`, `v-model:order` | The sort field and direction, kept by the page and sent to the server |
| Slots `default`, `before-search`, `after-search` | The card of one row, and two places beside the search box |
| Methods `reload`, `focus` | Start again from page 1; put the cursor in the search box |
| Event `error` | Fired when a request fails |

Keep these four points in mind:

- **The address is a prefix.** DataCard appends the query string to `url` as it is. That is why `url` must end with `?` or `&` (chapter 3).
- **One request at a time.** A new request cancels the one still running, and an answer that arrives after it was replaced is ignored.
- **Rows only grow.** While the visitor scrolls, every loaded row stays on the page. Any change of search, sort, page size or address starts again from page 1.
- **Browser only.** The first request is sent after the component is mounted in the browser, never during server-side rendering.

DataCard sits in the `components` folder of the project, and Nuxt registers it by its file name. It draws its search box, icons and retry button with Nuxt UI (`UInput`, `UIcon`, `UButton`), so that module must be installed:

```text
components/
└── DataCard.vue          (Nuxt registers it by file name: <DataCard> works in every page)
```

## 2. The first card

The smallest useful use is an address and a slot:

```vue
<template>
  <DataCard url="/api/items?">
    <template #default="{ item }">
      <div class="card">{{ item.title }}</div>
    </template>
  </DataCard>
</template>
```

- **`url="/api/items?"`.** Where the rows are. It ends with `?`.
- **`#default="{ item }"`.** The slot is called once per row, and `item` is the row exactly as the server sent it.
- **No more is needed.** The page size is 10, the grid has one column, and the key of a row is its `id`.

On screen the visitor sees a search box, then the cards. The next chapters say what the server must answer and what happens when the visitor scrolls.

## 3. What the server must do

For every request DataCard sends a `GET` with these parameters:

| Parameter | Value | Sent |
|-----------|-------|------|
| `page` | The page number, from 1 | Always |
| `rows` | The page size | Always |
| `q` | The text in the search box (empty when nothing is typed) | Always |
| `sort` | The sort field | Only when the page has set it |
| `order` | `ASC` or `DESC`, whatever the page set | Only when the page has set it |

A request for the second page of a search looks like this:

```text
GET /api/items?page=2&rows=10&q=shirt&sort=price&order=DESC
```

The server answers with JSON that has two fields:

```json
{
  "rows": [ { "id": 11, "title": "Item 11" }, { "id": 12, "title": "Item 12" } ],
  "total": 25
}
```

- **`rows`** is the list of the rows of this page. If it is missing or is not an array, it counts as an empty page.
- **`total`** is the number of all matching rows, not only those of this page. If it is missing, it counts as `0`, and only the first page is ever shown.

Any server that answers this way works. In J2OS the back end already has this shape: the `page2` module (`PageDataJPQL`) reads exactly these parameters and answers exactly this JSON.

```java
@GetMapping("/api/items")
public Object items(@RequestParam Map<String, Object> params) {
    PageDataJPQL page = new PageDataJPQL(entityManager);
    page.searchAndSortOn("id", "title", "price");                 // the fields q searches and sort may use
    return page.getResult("select o from Item o", "select count(o) from Item o",
                          "o", Item.class, null, params);        // answers { rows, total, page }
}
```

Two rules of `page2` matter here. `sort` is **mandatory** there: a request without it is rejected, so a DataCard in front of `page2` must have a sort field set (chapter 6). And `sort` may only name a field that was registered in `searchAndSortOn`.

### The address ends with `?` or `&`

DataCard builds the request as `url` followed directly by the parameters. It does not look at what `url` contains:

```text
url="/api/items?"   ->   GET /api/items?page=1&rows=10&q=
url="/api/items&"   ->   GET /api/items&page=1&rows=10&q=      (no "?": a different address)
url="/api/items"    ->   GET /api/itemspage=1&rows=10&q=       (the first parameter is glued to the path)
```

Extra parameters of your own go in front, and the address then ends with `&`. When the address changes, DataCard forgets what it holds and starts again from page 1:

```vue
<script setup>
const category = ref('ring')
const url = computed(() => `/api/items?category=${category.value}&`)
</script>

<template>
  <select v-model="category">
    <option>ring</option>
    <option>necklace</option>
  </select>

  <DataCard :url="url">
    <template #default="{ item }">
      <div class="card">{{ item.title }}</div>
    </template>
  </DataCard>
</template>
```

```text
GET /api/items?category=ring&page=1&rows=10&q=
after scroll -> 20 cards; GET /api/items?category=ring&page=2&rows=10&q=
category changed -> GET /api/items?category=necklace&page=1&rows=10&q= | cards: 10
```

- **The first request** carries `category=ring` and then the three parameters of DataCard.
- **Scrolling** asks for page 2 of the same category.
- **Choosing another category** changes `url`, so the cards are replaced and the request is for page 1 again.

## 4. Loading while the visitor scrolls

The same card of chapter 2, in a window that is too short to show all ten cards. The example below scrolls to the bottom four times:

```text
cards: 10
GET /api/items?page=1&rows=10&q=
after scroll 1 -> cards: 20 | GET /api/items?page=2&rows=10&q=
after scroll 2 -> cards: 25 | GET /api/items?page=3&rows=10&q=
after scroll 3 -> cards: 25 | no request
after scroll 4 -> cards: 25 | no request
```

- **The first request** is page 1, sent as soon as the component is on the page.
- **Reaching the bottom** brings the sentinel into view and asks for the next page. Ten cards become twenty.
- **The last page** has five rows. Now the 25 loaded rows equal `total`, so DataCard knows nothing is left. Scrolling further sends no request.
- **A tall screen** may show the sentinel at once, even before any scrolling. DataCard checks again after every page and keeps loading until the sentinel leaves the screen or the rows run out, so a short first page never leaves the visitor stuck.

The page size trades requests for weight: a small `page-size` needs more requests, a large one makes each request slower. The default is 10.

## 5. Searching

The search box sends what is typed as `q`. It does not send a request for every key. After the last key it waits 500 milliseconds, and only then asks the server, from page 1:

```text
typing done, 300 ms later, requests: 0
after the pause, requests: 1
GET /api/items?page=1&rows=10&q=Item+2
cards: 6 | Item 20, Item 21, Item 22, Item 23, Item 24, Item 25
search "zzz" -> داده‌ای وجود ندارد
```

- **Typing `Item 2`** (six keys, 100 milliseconds apart) sends no request while typing. Even 300 milliseconds after the last key there is still none.
- **After the pause** one request is sent with `q=Item+2`, and the six matching cards replace the old ones.
- **A search with no match** shows the empty text (here the default one, "no data").

What `q` means is up to the server. `page2` searches all fields registered in `searchAndSortOn`.

## 6. Sorting

DataCard has no sort buttons of its own. It has two models, `sort` and `order`, and sends them when they are set. The page decides how the visitor changes them; the `after-search` slot is a natural place for two selects:

```vue
<script setup>
const sort = ref('id')
const order = ref('ASC')

const sortItems = [
  { label: 'Number', value: 'id' },
  { label: 'Title', value: 'title' },
  { label: 'Price', value: 'price' }
]
const orderItems = [
  { label: 'Ascending', value: 'ASC' },
  { label: 'Descending', value: 'DESC' }
]
</script>

<template>
  <DataCard url="/api/items?" v-model:sort="sort" v-model:order="order">
    <template #after-search>
      <USelect v-model="sort" :items="sortItems" />
      <USelect v-model="order" :items="orderItems" />
    </template>

    <template #default="{ item }">
      <div class="card">{{ item.title }} · {{ item.price }}</div>
    </template>
  </DataCard>
</template>
```

```text
GET /api/items?page=1&rows=10&q=&sort=id&order=ASC
first: Item 01 · 10
GET /api/items?page=1&rows=10&q=&sort=price&order=ASC | first: Item 01 · 10
GET /api/items?page=1&rows=10&q=&sort=price&order=DESC | first: Item 18 · 99
```

- **The first request** already carries `sort=id&order=ASC`, because the page gave them initial values.
- **Every change** of either model starts again from page 1 with the new parameters.
- **Without initial values** `sort` and `order` are left out of the request. This is fine for a server that has a default order, but not for `page2` (chapter 3).

## 7. Columns

`cols` says how many cards stand side by side at each screen width. It is an object with the keys `sm`, `md`, `lg`, `xl` and `2xl`:

```vue
<template>
  <DataCard
    url="/api/items?"
    item-key="id"
    :page-size="8"
    :cols="{ sm: 1, md: 2, xl: 4 }"
  >
    <template #default="{ item }">
      <div class="card">{{ item.title }}</div>
    </template>
  </DataCard>
</template>
```

```text
width 390 -> 1 column(s)
width 800 -> 2 column(s)
width 1100 -> 2 column(s)
width 1300 -> 4 column(s)
width 1600 -> 4 column(s)
```

- **The keys** start at 768, 1024, 1280 and 1536 pixels for `md`, `lg`, `xl` and `2xl`. `sm` is everything below 768 pixels.
- **A missing key** takes the value of the nearest smaller one. Here `lg` is missing, so it uses `md`: 2 columns at 1100 pixels.
- **`cols` is optional.** Without it the grid has one column everywhere.
- **The widths** are those of the browser window, not of the area the grid sits in.

## 8. Slots and methods

The component has three slots. The one without a name is the card of a row, and `item` is the row. The other two are places on the line with the search box, one before it and one after it. A parent can also keep a reference to the component and call its two methods:

```vue
<script setup>
const grid = ref(null)
</script>

<template>
  <DataCard ref="grid" url="/api/items?">
    <template #before-search>
      <strong>Catalog</strong>
    </template>

    <template #after-search>
      <button id="reload" @click="grid.reload()">Reload</button>
      <button id="find" @click="grid.focus()">Find</button>
    </template>

    <template #default="{ item }">
      <div class="card">{{ item.title }}</div>
    </template>
  </DataCard>
</template>
```

```text
Reload -> GET /api/items?page=1&rows=10&q=
Find -> focused element: u-input
```

- **`before-search`** and **`after-search`** hold anything: a title, selects, buttons.
- **`reload()`** starts again from page 1 with the same search and sort. Use it after the data changed, for example after a delete (chapter 10).
- **`focus()`** puts the cursor in the search box.
- **During a reload** the cards disappear and the loading message takes their place until the first page arrives. The search box stays.

## 9. When it fails

Besides the list and the loading message, DataCard has three more screens. An empty answer shows the text of `empty-text`:

```vue
<template>
  <DataCard url="/api/empty?" empty-text="No items yet">
    <template #default="{ item }">
      <div class="card">{{ item.title }}</div>
    </template>
  </DataCard>
</template>
```

```text
cards: 0 | text: No items yet
```

A failure of the first page shows an error message and no cards. The `error` event carries the exception:

```vue
<script setup>
const messages = ref([])
</script>

<template>
  <DataCard url="/api/broken?" @error="e => messages.push(e.message)">
    <template #default="{ item }">
      <div class="card">{{ item.title }}</div>
    </template>
  </DataCard>

  <p id="log">{{ messages.join(', ') }}</p>
</template>
```

```text
GET /api/broken?page=1&rows=10&q= | cards: 0
screen: ['خطا در دریافت اطلاعات وجود دارد']
error events: HTTP 500
```

A failure while loading more keeps the cards, shows a message and a retry button. In this example the server fails the first request for page 2:

```vue
<script setup>
const messages = ref([])
const onError = (e) => messages.value.push(e.message)
</script>

<template>
  <DataCard url="/api/flaky?" :page-size="10" @error="onError">
    <template #default="{ item }">
      <div class="card">{{ item.title }}</div>
    </template>
  </DataCard>

  <p id="log">{{ messages.join(', ') }}</p>
</template>
```

```text
page 2 -> GET /api/flaky?page=2&rows=10&q= | cards: 10
screen: ['خطا در دریافت موارد بیشتر تلاش مجدد']
error events: HTTP 503
retry -> GET /api/flaky?page=2&rows=10&q= | cards: 20
```

- **What counts as a failure** is a network error, an HTTP status outside 200 to 299, or an answer that is not JSON. The exception message of a bad status is `HTTP <status>`.
- **The retry button** asks for the same page again. It does not skip it.
- **A cancelled request** is not a failure, and fires no event.

### Signing in

Requests are sent with `credentials: 'include'`, so the browser attaches the session cookie. A server that answers 401 without it gives the error screen:

```vue
<template>
  <DataCard url="/api/private?" empty-text="Nothing here yet">
    <template #default="{ item }">
      <div class="card">{{ item.title }}</div>
    </template>
  </DataCard>
</template>
```

```text
no cookie -> GET /api/private?page=1&rows=10&q= | cards: 0 | ['خطا در دریافت اطلاعات وجود دارد']
cookie -> GET /api/private?page=1&rows=10&q= | cards: 10 | []
```

DataCard does not redirect to a login page; the `error` event is the place to do that. A project that signs in with a token instead of a cookie edits the call in `fetchPage` (the source keeps this variant as a comment):

```js
const token = localStorage.getItem('token')
const response = await fetch(buildUrl(), {
  signal: currentController.signal,
  headers: { ...(token && { Authorization: `Bearer ${token}` }) }
})
```

## 10. Putting it together

A small catalog: six cards per page, three columns, sorting by number or price, and a delete button on each card. After the delete, the page asks the component to start again:

```vue
<script setup>
const grid = ref(null)
const sort = ref('id')
const order = ref('ASC')
const failed = ref('')

const sortItems = [
  { label: 'Number', value: 'id' },
  { label: 'Price', value: 'price' }
]
const orderItems = [
  { label: 'Ascending', value: 'ASC' },
  { label: 'Descending', value: 'DESC' }
]

async function remove(item) {
  await fetch(`/api/items/${item.id}`, { method: 'DELETE', credentials: 'include' })
  grid.value.reload()
}
</script>

<template>
  <DataCard
    ref="grid"
    url="/api/items?"
    :page-size="6"
    :cols="{ sm: 1, md: 3 }"
    v-model:sort="sort"
    v-model:order="order"
    @error="e => (failed = e.message)"
  >
    <template #after-search>
      <USelect v-model="sort" :items="sortItems" />
      <USelect v-model="order" :items="orderItems" />
    </template>

    <template #default="{ item }">
      <UCard>
        <h3>{{ item.title }}</h3>
        <UBadge>{{ item.price }}</UBadge>
        <UButton @click="remove(item)">Delete</UButton>
      </UCard>
    </template>
  </DataCard>

  <p v-if="failed">Problem: {{ failed }}</p>
</template>
```

```text
before: ['Item 01', 'Item 02', 'Item 03', 'Item 04', 'Item 05', 'Item 06']
['DELETE /api/items/2?', 'GET /api/items?page=1&rows=6&q=&sort=id&order=ASC']
after: ['Item 01', 'Item 03', 'Item 04', 'Item 05', 'Item 06', 'Item 07']
```

- **`UCard`** gives the row a frame; `UBadge` shows the price. DataCard does not care what is inside the slot.
- **`remove`** sends the delete to the server and then calls `reload()`. The second card is gone from the new first page, and the others move up.
- **`v-model:sort` and `v-model:order`** are shared with the selects, so one change of a select reloads the list.
- **`@error`** keeps the last message for the page, in case something else on the page wants to show it.

## Appendix A. Quick reference

### A.1 Properties

| Property | Type | Default | Meaning |
|----------|------|---------|---------|
| `url` | String, required | - | The address; must end with `?` or `&` |
| `pageSize` | Number | `10` | Rows per request (`rows`) |
| `itemKey` | String | `'id'` | The field that identifies a row; must be unique |
| `cols` | Object | `{}` | Columns at `sm`, `md`, `lg`, `xl`, `2xl`; `1` when nothing is given |
| `emptyText` | String | `داده‌ای وجود ندارد` | Text when there are no rows |

### A.2 Models, slots, methods and event

| Name | Meaning |
|------|---------|
| `v-model:sort` | Sort field; sent as `sort` when set |
| `v-model:order` | Direction; sent as `order` when set |
| Slot `default` | The card of one row, with `{ item }` |
| Slot `before-search`, `after-search` | Content before and after the search box |
| `reload()` | Start again from page 1 |
| `focus()` | Put the cursor in the search box |
| Event `error` | The exception of a failed request |

### A.3 What is sent and what is expected

| Direction | Content |
|-----------|---------|
| Request | `GET <url>page=<n>&rows=<pageSize>&q=<text>[&sort=<s>][&order=<o>]` with cookies |
| Answer | `{ "rows": [...], "total": <number> }` |

### A.4 What changes start again from page 1

| Change | |
|--------|---|
| `url`, `pageSize`, `sort` or `order` changes | At once |
| The search text changes | After 500 milliseconds without typing |
| `reload()` is called | At once |

## Appendix B. Failure handling

| Situation | What the visitor sees | What happens |
|-----------|-----------------------|--------------|
| The first page fails | The error message, no cards | The rows are cleared; `error` fires |
| A later page fails | The cards and a retry message | The page number goes back; `error` fires; the button asks again |
| The answer is not JSON | Same as a failure | The parse error reaches `error` |
| `rows` is missing or not an array | The empty text | Counts as no rows |
| `total` is missing | Only the first page | Counts as `0`, so nothing more is requested |
| A request is replaced by a newer one | Nothing | The old request is cancelled and its answer ignored |
| The component is removed from the page | Nothing | The running request and the timer are cancelled |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **An address without `?` or `&` at the end.** The parameters are glued to the path and the server sees a different address (chapter 3).
- **A `total` that is larger than what the server really returns.** DataCard decides that more exists because it holds fewer rows than `total`. If the next page then comes back empty, it asks for the one after, and so on without end. Make `total` the count of the same query that `rows` comes from.
- **A server that needs `sort` and a card without one.** `page2` rejects a request without `sort`; the card shows its error screen. Give `v-model:sort` an initial value.
- **A sort field the server does not know.** `page2` refuses fields that are not in `searchAndSortOn`.
- **Expecting `reload()` to keep the scroll position.** It starts again from page 1, so a visitor who had scrolled far lands at the top.
- **A key that repeats.** `itemKey` must name a field that is different for every row, otherwise Vue cannot tell the cards apart.
- **Expecting the texts to be configurable.** Only the empty text is a property; the other messages are fixed (Appendix D).

### C.2 Behaviour worth knowing

- **The search keeps its focus.** Loading never replaces the search box, so typing is not interrupted when the cards are replaced.
- **Sort and search work together.** A change of either one keeps the other.
- **`sort` and `order` are optional for the component.** They are omitted from the request while they are empty.
- **Nothing is fetched on the server.** With server-side rendering the page first shows the loading message, and the request is sent in the browser.
- **`error` is not fired for a cancelled request.** Only real failures reach it.

### C.3 Limits

| What | Limit |
|------|-------|
| Wait after typing | 500 milliseconds, fixed |
| Loaded rows | All stay on the page; there is no recycling of cards |
| Sort fields | One field and one direction at a time |
| Texts | Persian, fixed, except `emptyText` |
| Breakpoints | 768, 1024, 1280 and 1536 pixels, fixed |
| Total on screen | Not shown; the page can show it from its own data |

### C.4 Tuning

- **`pageSize`.** Raise it when the cards are small and the server is fast; lower it when each row is heavy, such as an image card.
- **Columns.** Give `cols` for the widths you really use; the fallback fills the others.
- **Images.** Load them lazily in the card, so the images of cards far below are not requested at once.
- **Very long lists.** If thousands of cards must stay on the page, expect slowness; use a search or a filter in `url` instead of scrolling to the end.

## Appendix D. Message reference

### D.1 Texts on the screen

| Text | Meaning | When |
|------|---------|------|
| جستجو... | Search... | The placeholder of the search box |
| در حال بارگذاری... | Loading... | The first page is being fetched |
| در حال بارگذاری موارد بیشتر... | Loading more... | A later page is being fetched |
| خطا در دریافت اطلاعات وجود دارد | There was an error getting the data | The first page failed |
| خطا در دریافت موارد بیشتر | There was an error getting more | A later page failed |
| تلاش مجدد | Try again | The retry button |
| داده‌ای وجود ندارد | There is no data | No rows, when `emptyText` is not set |

### D.2 The error event

| Message | Cause |
|---------|-------|
| `HTTP <status>` | The server answered with a status outside 200 to 299 |
| The text of the browser's error | The network failed, or the answer was not JSON |
