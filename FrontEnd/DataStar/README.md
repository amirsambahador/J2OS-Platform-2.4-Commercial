# DataStar

*J2OS Platform 2.4*

Stars are the quickest way to say "how good". A product page shows 4.5 stars on average. Under it, a visitor clicks the fourth star to leave their own opinion. Two jobs, one picture.

**DataStar** is a small Vue component for the Nuxt front end of J2OS that draws a row of stars for both jobs. It has three properties:

- **Show or collect.** With `readonly` it only shows a rating; without it, the visitor can click.
- **Fractions are drawn.** A rating of 3.7 fills three stars and most of the fourth, so an average is shown as it is.
- **One value, one `v-model`.** The rating is a plain number. The page reads it and writes it, and the component keeps no other state.

The whole component, in a few lines:

```vue
<DataStar :model-value="3.7" readonly />          <!-- show a rating: 3 full stars and most of the fourth -->
<DataStar v-model="rating" />                     <!-- let the visitor give one: click a star -->
<DataStar v-model="rating" :star-count="10" size="lg" />
```

This guide starts with how the component works (chapter 1). It then covers showing a rating (2), collecting one (3), fractions (4), size and number of stars (5) and right-to-left pages (6). Chapter 7 builds a small review box. The appendices work as a reference.

## Contents

1. [How DataStar works](#1-how-datastar-works)
2. [Showing a rating](#2-showing-a-rating)
3. [Collecting a rating](#3-collecting-a-rating)
4. [Fractions](#4-fractions)
5. [Size and number of stars](#5-size-and-number-of-stars)
6. [Right-to-left pages](#6-right-to-left-pages)
7. [Putting it together](#7-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are complete pages of a Nuxt project: each is one `.vue` file. In Nuxt `ref` and `computed` and the component `DataStar` need no `import`, so none is written.

---

## 1. How DataStar works

First, a few terms, because they come up throughout the guide:

- **Rating.** The number in the `v-model`. `3` means three stars are full; `3.5` means three and a half.
- **Committed rating.** The value now in the `v-model`, not the star under the pointer.
- **Preview.** While the pointer is over a star, the stars up to it show in gold outline. This is what a click would choose.
- **Partial star.** A star filled only in part, drawn when the rating has a fraction.

The parts of this component:

| Piece | Role |
|-------|------|
| `v-model` | The rating, a number |
| `starCount` | How many stars (5) |
| `readonly` | Only show; no clicks |
| `size` | `sm`, `md` (default) or `lg` |

Keep these four points in mind:

- **Two faces.** With `readonly` the component is one picture with a text description. Without it, every star is a button.
- **A click chooses a whole number.** The visitor can pick 1, 2, 3 and so on; fractions come only from the page.
- **A second click on the same star clears.** Clicking the star that is already the rating sets it to `0`.
- **Nothing is sent.** DataStar changes the `v-model` and emits `update:modelValue`. Saving the rating is the page's job.

The stars are drawn with Nuxt UI icons (`i-heroicons-star` and `i-heroicons-star-solid`) and Tailwind classes, as in the rest of the J2OS front end.

## 2. Showing a rating

With `readonly`, DataStar shows a value and nothing else. A typical use is an average:

```vue
<script setup>
const average = 3.7
</script>

<template>
  <DataStar :model-value="average" readonly />
  <span id="avg">({{ average.toFixed(1) }})</span>
</template>
```

```text
solid+gold solid+gold solid+gold partial(30% hidden) outline+grey
aria-label: 3.7 از 5 ستاره | role: img
click does nothing: buttons = 0
```

- **Full, partial and empty.** 3.7 gives three full stars, a fourth that is filled for 70% of its width (the output gives the *hidden* part: 30%) and a fifth that is empty.
- **The description.** The component has the role `img` and a label, here "3.7 از 5 ستاره" ("3.7 of 5 stars"), so a screen reader reads the rating.
- **No buttons.** The stars cannot be clicked, and the page's `v-model` is never written.

## 3. Collecting a rating

Without `readonly` every star is a button. The page keeps the rating in a `ref`:

```vue
<script setup>
const rating = ref(0)
</script>

<template>
  <DataStar v-model="rating" />
  <p id="value">Rating: {{ rating }}</p>
</template>
```

```text
start: Rating: 0 | buttons: 5
click star 4 -> Rating: 4 | outline+gold outline+gold outline+gold outline+gold outline+grey
hover star 2 -> outline+gold outline+gold outline+grey outline+grey outline+grey
mouse leaves -> solid+gold solid+gold solid+gold solid+gold outline+grey
click star 4 again -> Rating: 0
click star 2 -> Rating: 2
focus star 5 + Space -> Rating: 5
labels: ['1 از 5 ستاره/false', '2 از 5 ستاره/false', '3 از 5 ستاره/false', '4 از 5 ستاره/false', '5 از 5 ستاره/true']
```

- **A click** on star *n* sets the rating to *n*.
- **A second click** on the same star sets it back to `0`.
- **Preview.** While the pointer is over star 2, stars 1 and 2 are gold outlines and the others grey, whatever the current rating is. When the pointer leaves, the committed rating (full gold stars) is shown again. This is why the output after a click still shows outlines: the pointer was still on the star.
- **The keyboard works.** The stars are real buttons, so Tab reaches them and Space or Enter chooses. The first Tab moves focus to star 1; two more Tabs and Enter choose star 3, and Space chooses whichever star has focus, for example 5.
- **Accessible names.** Each button says "n از 5 ستاره" and `aria-pressed` is true for the star that equals the rating.

To act on a new rating, listen to the model event or watch the ref:

```vue
<DataStar v-model="rating" @update:model-value="saveRating" />
```

## 4. Fractions

The rating may have a fraction, set by the page. The interactive component draws it, and the first click replaces it with a whole number:

```vue
<script setup>
const rating = ref(3.5)
</script>

<template>
  <DataStar v-model="rating" />
  <p id="value">Rating: {{ rating }}</p>
</template>
```

```text
start 3.5 -> solid+gold solid+gold solid+gold partial(50% hidden) outline+grey
click star 3 -> Rating: 3
click star 4 -> Rating: 4
```

- **3.5** draws three full stars and a half.
- **A click on star 3** gives exactly `3`, and a click on star 4 gives `4`. The fraction disappears with the first click.
- **There is no rounding.** 3.04 draws a fourth star with a 4% sliver; the value is shown as it is. Round it in the page if you want.

## 5. Size and number of stars

`size` has three values and `star-count` is a number. The test below uses readonly stars to show the class of each size and the number of stars:

```vue
<template>
  <DataStar :model-value="3" size="sm" readonly />
  <DataStar :model-value="3" size="md" readonly />
  <DataStar :model-value="3" size="lg" :star-count="10" readonly />
</template>
```

```text
component 1 -> 5 stars, size class: size-4
component 2 -> 5 stars, size class: size-6
component 3 -> 10 stars, size class: size-8
```

- **`sm`, `md` and `lg`** use Tailwind's `size-4`, `size-6` and `size-8` (1, 1.5 and 2 rem).
- **`star-count`** is 5 by default. The component makes that many stars; ten stars with `lg` fills a wide row.
- **The rating is not scaled.** On ten stars, a rating of 5 is five full stars and five empty, not half the row.

## 6. Right-to-left pages

DataStar is built for the right-to-left pages of J2OS: the first star is at the right, and the rating grows toward the left. The partial star, however, is always filled from its **right** side, whether the page is right-to-left or left-to-right:

- **In a right-to-left page**, the fraction fills from the side where the stars start. This is the intended look.
- **In a left-to-right page**, the fraction still fills from the right. A 3.7 then looks as if the fourth star were filled "backwards".

If your page is left-to-right, round the rating for display or flip the star row with CSS.

## 7. Putting it together

A review box: the average of the reviews at the top, each review below, and a form to add one. The send button is disabled until the visitor chooses a rating, and after sending the form is cleared:

```vue
<script setup>
const reviews = ref([
  { who: 'Sara', stars: 5 },
  { who: 'Ali', stars: 4 }
])
const mine = ref(0)

const average = computed(() =>
  reviews.value.reduce((sum, r) => sum + r.stars, 0) / reviews.value.length
)

function send() {
  reviews.value.push({ who: 'You', stars: mine.value })
  mine.value = 0
}
</script>

<template>
  <h3>Average <DataStar :model-value="average" readonly /> <span id="avg">{{ average.toFixed(2) }}</span></h3>

  <p v-for="r in reviews" :key="r.who">
    {{ r.who }} <DataStar :model-value="r.stars" size="sm" readonly />
  </p>

  <DataStar v-model="mine" />
  <button id="send" :disabled="!mine" @click="send">Send</button>
</template>
```

```text
average: 4.50 | send disabled: True
after a 3-star review -> 4.00 | reviews: 3 | my rating reset, send disabled: True
```

- **`average`** is a `computed` over the reviews, so adding a review updates the stars at once.
- **Read-only and interactive together.** The average and the reviews use `readonly`; the form uses a plain `v-model`.
- **Clearing** is just `mine.value = 0`.
- **Saving** would be a `fetch` inside `send`. DataStar has no part in it.

## Appendix A. Quick reference

### A.1 Properties and model

| Name | Type | Default | Meaning |
|------|------|---------|---------|
| `v-model` | Number | none | The rating; empty counts as `0` |
| `starCount` | Number | `5` | Number of stars |
| `readonly` | Boolean | `false` | Show only |
| `size` | String | `'md'` | `sm`, `md` or `lg` |

### A.2 Behaviour of the stars

| Situation | Result |
|-----------|--------|
| Click star *n* | Rating becomes *n* |
| Click the star that equals the rating | Rating becomes `0` |
| Pointer over star *n* | Stars 1 to *n* show as gold outlines |
| Pointer leaves | The committed rating is shown |
| `readonly` | No buttons; role `img` with a label |

### A.3 Sizes

| `size` | Class | Width |
|--------|-------|-------|
| `sm` | `size-4` | 1 rem |
| `md` | `size-6` | 1.5 rem |
| `lg` | `size-8` | 2 rem |

## Appendix B. Failure handling

DataStar has no events for errors; what can go wrong is the value it is given.

| Value | What is drawn |
|-------|---------------|
| `null` or missing | All empty; label "0 از 5" |
| Negative | All empty; the label shows the number as given |
| Larger than `starCount` | All full; the label shows the number as given |
| `starCount` of `0` | No stars |
| A fractional `starCount` such as 2.5 | The whole part: 2 stars; the label shows 2.5 |
| A `size` that is not `sm`, `md` or `lg` | Stars without a size class, and a warning in development |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Expecting a half-star click.** A click always gives a whole number.
- **Left-to-right pages.** The partial star fills from its right side (chapter 6).
- **Rating above the star count.** It is drawn full, but the value is not clamped, and the label says the large number. Clamp it in the page.
- **A `starCount` computed from the rating.** `Math.ceil(10.7)` gives 11 stars. Use a fixed count.
- **The preview after a click.** Right after a click the pointer is still on the star, so the preview (outlines) shows until it leaves.
- **Missing icons.** The full and empty stars come from the heroicons set of Nuxt UI; without it no stars appear.

### C.2 Behaviour worth knowing

- **The value belongs to the page.** The component draws what the `v-model` says and writes back only on a click.
- **Both modes are separate.** If `readonly` changes at runtime, the picture turns into buttons, or the other way round.
- **The label text is Persian** and cannot be changed by a property.

### C.3 Limits

| What | Limit |
|------|-------|
| Click values | Whole numbers from 1 to `starCount` |
| Sizes | Three |
| Texts | Persian, fixed |

### C.4 Tuning

- **Averages.** Round to one decimal in the text next to the stars, as in chapter 2.
- **Large counts.** More than ten stars gets wide; use the `sm` size.
- **Saving.** Save from the model event, and decide on the server what `0` means, because a second click on the same star clears the rating.

## Appendix D. Message reference

### D.1 Texts

| Text | Meaning | Where |
|------|---------|-------|
| `<rating> از <count> ستاره` | `<rating>` of `<count>` stars | The `aria-label` of a readonly component |
| `<n> از <count> ستاره` | `<n>` of `<count>` stars | The `aria-label` of each button |

### D.2 Warnings

| Message | Cause |
|---------|-------|
| `Invalid prop: custom validator check failed for prop "size"` | `size` is not `sm`, `md` or `lg` (development builds) |
