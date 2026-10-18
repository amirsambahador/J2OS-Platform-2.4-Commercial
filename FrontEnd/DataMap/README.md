# DataMap

*J2OS Platform 2.4*

Maps appear in applications for a few simple reasons. A form asks where the delivery should go. A page shows where an office is. A list shows all the branches at once. An order screen shows the way from the courier to the customer. Each of these needs a map that loads when needed, puts markers on it and tells the page what the visitor did.

**DataMap** is a Vue component for the Nuxt front end of J2OS that does these jobs. It is built on *Leaflet*, a small open-source map library, and *OpenStreetMap*. It has three main features:

- **One component, three modes.** `pick` lets the visitor click to choose a place, `view` shows one place, and `multiview` shows many. Routes and live tracking are added to `view`.
- **No key and no account.** The map tiles, the place search and the routes come from open services, so nothing has to be registered.
- **The page decides.** DataMap never sends a position anywhere. It reports what happened through events (a place was picked, a label was clicked, a route was computed), and the page does what it likes with them.

The library at a glance:

```vue
<DataMap v-model="place" mode="pick" searchable />                  <!-- the visitor clicks to choose a place -->
<DataMap :model-value="place" marker-text="Our office" />           <!-- one place, with a label -->
<DataMap mode="multiview" :markers="branches" />                    <!-- many places -->
```

We begin with how the component works (chapter 1) and the first map (2). It then covers choosing a place (3), search and "my location" (4), many markers (5), routes (6), live tracking (7), size, zoom and tiles (8) and failures (9). Chapter 10 builds a small delivery page. The appendices are meant for quick lookups.

## Contents

1. [How DataMap works](#1-how-datamap-works)
2. [The first map](#2-the-first-map)
3. [Picking a place](#3-picking-a-place)
4. [Search and "my location"](#4-search-and-my-location)
5. [Many markers](#5-many-markers)
6. [Routes](#6-routes)
7. [Live tracking](#7-live-tracking)
8. [Size, zoom and tiles](#8-size-zoom-and-tiles)
9. [When loading fails](#9-when-loading-fails)
10. [A worked summary](#10-a-worked-summary)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are real pages, not fragments, and each is a single `.vue` file. In Nuxt the functions `ref` and the component `DataMap` need no `import`, so none is written. The texts that DataMap shows by itself (the search hint, the loading message) are Persian by default; each can be replaced by a property.

---

## 1. How DataMap works

Words worth defining up front:

- **Coordinates.** A place as an object `{ lat, lng }`: latitude and longitude in degrees. This is the shape DataMap takes and gives everywhere.
- **Mode.** What the map is for: `view` (the default), `pick` or `multiview`.
- **Marker.** The pin on the map. In `view` and `multiview` a pin can carry a *label*, a small button above it.
- **Tiles.** The square pictures that make up the map. They come from a tile server.
- **Route.** The road line from one place to another, drawn on the map, with its length and time.

The parts of this component:

| Piece | Role |
|-------|------|
| `mode`, `modelValue`, `markers`, `center`, `route` | What to show: the mode, the place (`v-model`), the many places, the center and the route |
| `zoom`, `minZoom`, `maxZoom`, `width`, `height` | How it looks |
| `searchable`, `geolocationEnabled` | The search box and the "my location" button of `pick` |
| `liveTracking`, `liveTrackingInterval`, `liveTrackingZoom` | Following the device |
| Events | What happened: `update:modelValue`, `marker-click`, `multi-marker-click`, `route-computed`, `location-update` and the error events |
| Slots `loading`, `error` | The screens for loading and for a failed load |
| Methods `flyTo`, `retryLoad` | Move the map; load again |

Five points are worth remembering:

- **Leaflet is loaded when the map appears.** The component fetches `leaflet.js` and `leaflet.css` from `/plugins/leaflet/` of your site the first time a map is mounted (chapter 2). Nothing is fetched on the server.
- **Three outside services.** Tiles come from OpenStreetMap, place search from Nominatim and routes from the public OSRM server. Without internet the tiles stay empty, and search and routes fail.
- **Positions go through the page.** DataMap does not store, send or keep any position. Everything leaves through events.
- **Labels are plain text.** The text of a label is cleaned of HTML before it is shown.
- **Pages can hold many maps.** Each is independent, and Leaflet is loaded once for all of them.

## 2. The first map

Two things must exist before the first map. First, the Leaflet files must be served from the `public` folder of the project, under exactly this path:

```text
public/
└── plugins/
    └── leaflet/
        ├── leaflet.js
        ├── leaflet.css
        └── images/
            ├── marker-icon.png
            ├── marker-icon-2x.png
            └── marker-shadow.png
```

Second, the project must provide the global `DOMPurify` object, which DataMap uses to clean label text. J2OS projects have both.

A place is given as coordinates:

```js
{ lat: 35.7219, lng: 51.3347 }       // a place: latitude and longitude, in degrees
```

Showing one place with a label is the smallest use:

```vue
<script setup>
const clicks = ref(0)
</script>

<template>
  <DataMap
    :model-value="{ lat: 35.7219, lng: 51.3347 }"
    marker-text="Our office"
    height="300px"
    @marker-click="clicks++"
  />

  <p id="clicks">Clicks on the label: {{ clicks }}</p>
</template>
```

```text
markers: 1 | labels: ['Our office'] | centre/zoom: [35.722, 51.335, 16]
Clicks on the label: 2 | markers still: 1
```

- **`:model-value`.** The place. The map opens on it and puts a pin there.
- **`marker-text`.** The label above the pin. It is a button: a click on it fires `marker-click`. In the output above it was clicked twice.
- **`height`.** The map needs a height. It is `400px` when not given.
- **The zoom** of a `view` map is 16 unless `zoom` says otherwise.
- **Only the label is clickable.** Without `marker-text` the pin cannot be clicked, and `marker-click` never fires.

## 3. Picking a place

In `pick` mode a click on the map chooses a place. The place lives in a `v-model`, and it starts as `null` when nothing is chosen:

```vue
<script setup>
const map = ref(null)
const place = ref(null)
</script>

<template>
  <DataMap ref="map" v-model="place" mode="pick" height="300px" />

  <p id="place">{{ place ? `${place.lat}, ${place.lng}` : 'Nothing picked' }}</p>

  <button id="set" @click="place = { lat: 35.7, lng: 51.4 }">Set from code</button>
  <button id="jump" @click="map.flyTo(35.6892, 51.389, 15)">Jump</button>
  <button id="clear" @click="place = null">Clear</button>
</template>
```

```text
start: markers 0 | Nothing picked | centre/zoom: [35.722, 51.335, 14]
click 1 -> 35.724705, 51.339884 | markers 1 | centre/zoom: [35.722, 51.335, 14]
click 2 -> 35.719828, 51.327868 | markers 1
set from code -> 35.7, 51.4 | markers 1 | centre/zoom: [35.7, 51.4, 14]
flyTo -> [35.689, 51.389, 15] | 35.7, 51.4
clear -> Nothing picked | markers 0
```

- **A click** puts the pin at the click and writes the coordinates to the `v-model`, rounded to six decimals. A second click moves the pin; there is only ever one.
- **Without a place** the map opens on the default center (35.7219, 51.3347, Tehran) at zoom 14.
- **Setting the `v-model` from code** moves the pin and also moves the view to it. The map's own click does not move the view, because the visitor is already looking there.
- **`flyTo(lat, lng, zoom)`** moves the view and nothing else. The place stays as it was.
- **Setting the `v-model` to `null`** removes the pin.

## 4. Search and "my location"

`searchable` adds a toolbar above the map of a `pick` map: a search box and a search button. `geolocation-enabled` adds a third button that asks the device for its position:

```vue
<script setup>
const place = ref(null)
const notes = ref([])
</script>

<template>
  <DataMap
    v-model="place"
    mode="pick"
    searchable
    geolocation-enabled
    height="300px"
    @search-not-found="q => notes.push(`not found: ${q}`)"
    @search-error="e => notes.push(`search error: ${e.message}`)"
    @geolocation-error="e => notes.push(`location error: ${e.message}`)"
  />

  <p id="place">{{ place ? `${place.lat}, ${place.lng}` : 'Nothing picked' }}</p>
  <p id="notes">{{ notes.join(' | ') }}</p>
</template>
```

```text
toolbar buttons: 2 | icons: ['i-lucide-search', 'i-lucide-locate-fixed']
search "azadi" -> centre/zoom [35.7, 51.338, 14] | picked: Nothing picked | markers 0
Enter again at once -> requests so far: ['azadi']
search "nowhere" -> hint: مکانی با این عنوان پیدا نشد | notes: not found: nowhere
search box emptied -> hint shown: False
locate button -> picked: 35.7, 51.4 | centre/zoom [35.7, 51.4, 16] | markers 1
```

When the device refuses, the button reports through an event:

```text
without permission -> notes: location error: Failed to get data from the geolocation sensor: User denied Geolocation | picked: Nothing picked
```

- **Search moves the view, it does not pick.** After a search the map shows the place, and the visitor still clicks to choose. The `v-model` stays `null` until then.
- **Not found** shows a hint under the toolbar and fires `search-not-found` with the text that was searched. The hint goes away when the box is emptied.
- **Once a second.** A search sent less than a second after the previous one is ignored. Place search is a shared public service, and this keeps the page polite.
- **The location button** moves the view to the device (zoom 16), puts the pin there and writes the `v-model`, exactly as a click would.
- **Both are part of the toolbar.** The toolbar exists only in `pick` mode with `searchable`. `geolocation-enabled` alone shows nothing.
- **Failures** arrive as `search-error` (a network or HTTP problem) and `geolocation-error` (the device refused or does not know).

## 5. Many markers

`multiview` shows an array of places. Each can have a label, and a click on a label reports the index in the array:

```vue
<script setup>
const branches = ref([
  { lat: 35.7219, lng: 51.3347, text: 'North branch' },
  { lat: 35.6892, lng: 51.389, text: 'Centre branch' },
  { lat: 35.6997, lng: 51.338, text: 'West branch' }
])
const chosen = ref(null)
</script>

<template>
  <DataMap
    mode="multiview"
    :markers="branches"
    :zoom="12"
    height="300px"
    @multi-marker-click="i => (chosen = branches[i].text)"
  />

  <p id="chosen">{{ chosen ?? 'Nothing chosen' }}</p>
  <button id="add" @click="branches.push({ lat: 35.74, lng: 51.37, text: 'New branch' })">Add</button>
</template>
```

```text
labels: ['North branch', 'Centre branch', 'West branch'] | centre/zoom: [35.722, 51.335, 12]
click -> Centre branch
after Add -> labels: ['North branch', 'Centre branch', 'West branch', 'New branch']
```

- **`markers`** is an array of `{ lat, lng, text }`. The `text` is optional.
- **`multi-marker-click`** carries the index, so the page finds its own data with `branches[i]`.
- **The array may change.** Adding an item to it drew the new marker, because DataMap watches the array deeply.
- **The map does not fit the markers.** It opens on the default center unless `center` is given, and the zoom is the one you set. Choose both so that the markers are in sight.

## 6. Routes

A `route` is an object with two places. DataMap asks the public OSRM server for the driving route, draws it in blue with a white outline, marks the start with a dot and zooms the map to fit it:

```js
{ from: { lat: 35.6997, lng: 51.338 }, to: { lat: 35.7219, lng: 51.3347 } }
```

```vue
<script setup>
const summary = ref('')
</script>

<template>
  <DataMap
    :model-value="{ lat: 35.7219, lng: 51.3347 }"
    marker-text="Delivery point"
    :route="{ from: { lat: 35.6997, lng: 51.338 }, to: { lat: 35.7219, lng: 51.3347 } }"
    height="300px"
    @route-computed="r => (summary = `${(r.distanceMeters / 1000).toFixed(1)} km, ${Math.round(r.durationSeconds / 60)} min`)"
    @route-error="e => (summary = `no route: ${e.message}`)"
  />

  <p id="summary">{{ summary }}</p>
</template>
```

```text
while the route was computed, the overlay said: ['در حال بارگذاری نقشه...']
request (lng,lat;lng,lat): 51.338,35.6997;51.3347,35.7219?overview=full&geometries=geojson
summary: 3.2 km, 9 min | lines: 2 | overlay: 0 | zoom changed to fit the route: True
router fails -> no route: HTTP 500 | lines: 0 | markers: 1
```

- **The loading screen stays until the route is ready.** The whole map is covered while the route is asked for, so the text of `loading-text` is a good place to say "calculating the route".
- **`route-computed`** brings the length in meters and the time in seconds.
- **`route-error`** brings the exception. The map then shows no line and the other markers remain.
- **A new `route`** (a different start or end) replaces the line. Only the coordinates matter, not the identity of the object.
- **The server is a public demo.** It has no promise of availability, so the page should always handle `route-error`.

## 7. Live tracking

With `live-tracking` the map follows the device. Every interval it asks for the position, moves a pulsing dot there, centers the map on it and fires `location-update`. DataMap sends the position nowhere; the page decides, here with a `fetch`:

```vue
<script setup>
const destination = { lat: 35.7219, lng: 51.3347 }
const last = ref(null)

function onUpdate(position) {
  last.value = position
  fetch('/api/live-location', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(position)
  })
}
</script>

<template>
  <DataMap
    :model-value="destination"
    marker-text="Destination"
    :route="{ from: destination, to: destination }"
    live-tracking
    :live-tracking-interval="2000"
    height="300px"
    @location-update="onUpdate"
  />

  <p id="last">{{ last ? `${last.lat.toFixed(4)}, ${last.lng.toFixed(4)} (±${last.accuracy} m)` : 'Waiting' }}</p>
</template>
```

```text
first fix: 35.7000, 51.4000 (±12 m) | route requests: 1 | 51.4,35.7;51.3347,35.7219?overview=full&geometries=geojson | lines: 2
camera: [35.7, 51.4, 17] | posted fields: ['accuracy', 'lat', 'lng', 'timestamp']
moved about 7 m -> route requests: 1
moved about 280 m -> route requests: 2 | now: 35.7020, 51.4020 (±12 m) | camera: [35.702, 51.402, 17]
live marker: 1 | posts so far >= 3: True
```

- **`location-update`** carries `lat`, `lng`, `accuracy` (meters) and `timestamp`.
- **The interval** is `live-tracking-interval` milliseconds, never below 2000. The default is 3000.
- **With a route**, the line is drawn from the device to `route.to`. The `from` of the route is not used. The first position fix takes its place. After that the route is recalculated only when the device has moved at least 15 meters or the destination has changed. In the example the device first moves 7 meters (no request) and then 280 meters (one request).
- **The camera follows every fix** at `live-tracking-zoom` (17 by default), so a visitor cannot pan away; the next fix brings the map back.
- **Permission and HTTPS.** Browsers allow the position only on secure pages, and only after the visitor agrees. A refusal arrives as `geolocation-error` at every interval.

## 8. Size, zoom and tiles

Where the map opens is decided by the first of these that is given:

| Order | Source |
|-------|--------|
| 1 | `modelValue` |
| 2 | `center` |
| 3 | `route.to` |
| 4 | The default, 35.7219, 51.3347 |

The zoom is `zoom`, or 14 in `pick` and 16 in the other modes. `min-zoom` (3) and `max-zoom` (19) limit what the visitor can do. `width` (`100%`) and `height` (`400px`) are CSS lengths as strings. The map follows the size of its box if the box changes.

The tiles come from `tile-url`. OpenStreetMap's public tile server is meant for light use; a busy site should use its own server or a provider, and says so in the attribution. The credit line is hidden by default, so show it whenever the tile source requires it:

```vue
<DataMap
  tile-url="/tiles/{z}/{x}/{y}.png"
  attribution="&copy; Your company"
  show-attribution
/>
```

## 9. When loading fails

If `leaflet.js` or `leaflet.css` cannot be loaded, the map shows a message and a retry button, and fires `load-error`. The example below first makes the script return 404, then repairs it and presses the button:

```vue
<script setup>
const problem = ref('')
</script>

<template>
  <DataMap
    :model-value="{ lat: 35.7219, lng: 51.3347 }"
    height="200px"
    load-error-text="The map could not be loaded"
    retry-button-text="Try again"
    @load-error="e => (problem = e.message)"
  />

  <p id="problem">{{ problem }}</p>
</template>
```

```text
event: Failed to load JS: /plugins/leaflet/leaflet.js
screen: The map could not be loaded / Try again
after Try again -> overlay: 0 | markers: 1
```

- **`load-error`** carries an exception whose message names the file that failed.
- **`retryLoad()`** does the same as the button, and can be called from code.
- **The three texts** `loading-text`, `load-error-text` and `retry-button-text` are properties.
- **Whole screens** can be replaced with the slots `loading` and `error`. The `error` slot receives `message` and `retry`:

```vue
<DataMap :model-value="place">
  <template #loading><MySpinner /></template>
  <template #error="{ message, retry }">
    <p>{{ message }}</p>
    <button @click="retry">Again</button>
  </template>
</DataMap>
```

## 10. A worked summary

A delivery address page: the visitor searches or uses "my location", adjusts the pin by clicking, and sends the choice. The button stays disabled until a place exists, and a new click after saving makes the page unsaved again:

```vue
<script setup>
const place = ref(null)
const saved = ref(false)

async function save() {
  await fetch('/api/address', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(place.value)
  })
  saved.value = true
}
</script>

<template>
  <DataMap
    v-model="place"
    mode="pick"
    searchable
    geolocation-enabled
    search-placeholder="Street or place name..."
    height="300px"
    @update:model-value="saved = false"
  />

  <button id="save" :disabled="!place" @click="save">Deliver here</button>
  <p id="state">
    {{ saved ? 'Saved' : place ? `Selected ${place.lat}, ${place.lng}` : 'Click the map to choose' }}
  </p>
</template>
```

```text
start: Click the map to choose | button disabled: True
click -> Selected 35.720524, 51.337309 | button disabled: False
save -> Saved | sent: {"lat":35.720524,"lng":51.337309}
click again -> Selected 35.719131, 51.330442
```

- **`v-model="place"`** is the only state the map needs.
- **`@update:model-value`** is the place to react to a new choice, here by clearing the "saved" mark.
- **`save`** is ordinary page code. DataMap has no part in it.

## Appendix A. Quick reference

### A.1 Properties

| Property | Type | Default | Meaning |
|----------|------|---------|---------|
| `mode` | String | `'view'` | `view`, `pick` or `multiview` |
| `modelValue` | Object | `null` | The place `{ lat, lng }` (`v-model`) |
| `markers` | Array | `[]` | `multiview`: `{ lat, lng, text }` items |
| `center` | Object | `null` | Where the map opens when there is no `modelValue` |
| `route` | Object | `null` | `{ from, to }` |
| `markerText` | String | `''` | The label of the pin in `view` |
| `zoom` | Number | `null` | Opening zoom: 14 in `pick`, otherwise 16 |
| `minZoom`, `maxZoom` | Number | `3`, `19` | Zoom limits |
| `tileUrl` | String | OpenStreetMap | Tile address with `{z}`, `{x}`, `{y}` |
| `attribution` | String | OpenStreetMap credit | Credit text |
| `showAttribution` | Boolean | `false` | Show the credit line |
| `width`, `height` | String | `'100%'`, `'400px'` | Size of the map |
| `searchable` | Boolean | `false` | Toolbar with search box (`pick` only) |
| `geolocationEnabled` | Boolean | `false` | "My location" button, inside the toolbar |
| `liveTracking` | Boolean | `false` | Follow the device |
| `liveTrackingInterval` | Number | `3000` | Milliseconds between fixes, at least 2000 |
| `liveTrackingZoom` | Number | `17` | Zoom while following |
| `searchPlaceholder`, `searchNotFoundText` | String | Persian | Texts of the search |
| `loadingText`, `loadErrorText`, `retryButtonText` | String | Persian | Texts of loading and failure |
| `geolocationErrorText` | String | Persian | Declared but not used in this version; read the error from the event |

### A.2 Events

| Event | Argument | When |
|-------|----------|------|
| `update:modelValue` | `{ lat, lng }` | A click or the location button in `pick` |
| `marker-click` | none | The label of the `view` pin was clicked |
| `multi-marker-click` | index | A label in `multiview` was clicked |
| `route-computed` | `{ distanceMeters, durationSeconds }` | A route is drawn |
| `route-error` | exception | The route failed |
| `location-update` | `{ lat, lng, accuracy, timestamp }` | A live fix arrived |
| `search-not-found` | the query | Search found nothing |
| `search-error` | exception | Search failed |
| `geolocation-error` | exception | The device position failed |
| `load-error` | exception | Leaflet could not be loaded |

### A.3 Slots and methods

| Name | Meaning |
|------|---------|
| Slot `loading` | Replaces the loading screen |
| Slot `error` | Replaces the failure screen; receives `{ message, retry }` |
| `flyTo(lat, lng, zoom)` | Moves the view; `zoom` is optional |
| `retryLoad()` | Loads Leaflet and builds the map again |

### A.4 Files and services

| What | Where |
|------|-------|
| Leaflet files | `/plugins/leaflet/` of the site |
| Tiles | `tileUrl`; OpenStreetMap by default |
| Place search | Nominatim (OpenStreetMap), one result |
| Routes | The public OSRM server, driving |

## Appendix B. Failure handling

| Situation | What the visitor sees | What happens |
|-----------|-----------------------|--------------|
| Leaflet cannot be loaded | The failure screen with the retry button | `load-error`; the button or `retryLoad()` tries again |
| A route fails | The map without a line | `route-error` |
| Search finds nothing | A hint under the toolbar | `search-not-found` |
| Search fails | Nothing | `search-error` |
| The device refuses its position | Nothing | `geolocation-error` (at every fix in live tracking) |
| The browser has no geolocation | Nothing | `geolocation-error` |
| A search comes within a second of the last | Nothing | The search is ignored |
| The component is removed | Nothing | Timers and requests stop; the Leaflet files are removed when no other map uses them |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Missing Leaflet files.** Without `/plugins/leaflet/leaflet.js` and `leaflet.css` every map shows the failure screen. The path is fixed in the component.
- **No height, no map.** A parent that gives the map no height collapses it. Keep `height` set.
- **A label that is not a label.** `marker-click` needs `marker-text`; a pin without text cannot be clicked.
- **HTML in labels.** Tags are removed, so `<b>Hi</b>` shows as `Hi`.
- **`geolocation-enabled` without `searchable`.** The button is part of the toolbar and does not appear.
- **Expecting search to pick.** Search only moves the view; the `v-model` changes when the visitor clicks or presses the location button.
- **`multiview` out of sight.** The map does not fit the markers; give `center` and `zoom`.
- **A map that cannot be panned.** In live tracking the camera returns at every fix.
- **The credit line is hidden.** Show it with `show-attribution` when the tile provider requires it.
- **Waiting for `geolocation-error-text`.** That property does nothing in this version.

### C.2 Behavior worth knowing

- **Echo is handled.** When the page writes back the same coordinates the map just sent, the view does not jump.
- **The mode can change.** Switching `mode` at runtime changes the markers and the click behavior.
- **Pick coordinates are rounded** to six decimals, about ten centimeters.
- **Search returns one place** only.
- **The pin is not draggable.** In `pick` the place changes by clicking again.

### C.3 Limits

| What | Limit |
|------|-------|
| Interval of live tracking | At least 2000 milliseconds |
| Route recalculation while following | After 15 meters, or when the destination changes |
| Time for one route | 10 seconds |
| Search rate | One per second |
| Route type | Driving |
| Results of a search | One |

### C.4 Tuning

- **Tiles.** For a busy site, use your own tile server or a provider and set `tile-url` and `attribution`.
- **Routes.** For production, run your own OSRM or use a service with a promise of availability; the address is in the component's constants.
- **Many markers.** Every labeled pin is a page element, so for a crowded map group the places in the page before passing them.
- **Live tracking.** A longer interval saves battery and requests.

## Appendix D. Message reference

### D.1 Texts on the screen

| Text | Meaning | Property |
|------|---------|----------|
| نام مکان را جستجو کنید... | Search for a place name... | `searchPlaceholder` |
| مکانی با این عنوان پیدا نشد | No place with this name was found | `searchNotFoundText` |
| در حال بارگذاری نقشه... | Loading the map... | `loadingText` |
| بارگذاری نقشه با خطا مواجه شد | Loading the map failed | `loadErrorText` |
| تلاش مجدد | Try again | `retryButtonText` |

### D.2 Error messages

| Message | Event | Cause |
|---------|-------|-------|
| `Failed to load JS: <path>` | `load-error` | `leaflet.js` could not be loaded |
| `Failed to load CSS: <path>` | `load-error` | `leaflet.css` could not be loaded |
| `HTTP <status>` | `route-error`, `search-error` | The service answered with an error status |
| The message of the router, or `OSRM: <code>` | `route-error` | The router found no route |
| `Browser does not support geolocation` | `geolocation-error` | The browser has no position feature |
| `Failed to get data from the geolocation sensor: <reason>` | `geolocation-error` | The device refused or failed |
