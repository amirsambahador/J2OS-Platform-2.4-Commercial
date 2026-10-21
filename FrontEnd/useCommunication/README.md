# useCommunication

*J2OS Platform 2.4*

A shop page needs a "Contact the seller" box. Some customers prefer WhatsApp, some Telegram or Bale, some want to call, and some write an email. Every one of these channels is only a link with a particular shape: `https://wa.me/...`, `tel:...`, `mailto:...`. Writing those links by hand on every page means remembering the shape of each one, and encoding the message so that spaces, question marks and line breaks survive.

**useCommunication** is a composable of the Nuxt front end of J2OS that does this. It has three main features:

- **Six functions for six channels.** WhatsApp, Telegram, Bale, SMS, email and a phone call.
- **No server and no account.** The page only opens a link; the visitor's own apps do the sending.
- **The text is encoded for you.** Persian text, spaces, `&`, `#` and line breaks reach the other side as written.

The core of it, in code:

```vue
<script setup>
const talk = useCommunication()
</script>

<template>
  <button @click="talk.sendWhatsApp('15551230100', 'Is the teapot in stock?')">WhatsApp</button>
  <button @click="talk.callPhone('+15551230100')">Call</button>
  <button @click="talk.sendEmail('hello@shop.example', 'Teapot', 'Is it in stock?')">Email</button>
</template>
```

Start with how the composable works (chapter 1), then the messengers (2) and the phone channels (3). Chapter 4 is about writing numbers and usernames in the form each channel expects, chapter 5 about the text, and chapter 6 about what the composable cannot know. Chapter 7 builds a contact card for a product page.

## Contents

1. [How useCommunication works](#1-how-usecommunication-works)
2. [Messengers](#2-messengers)
3. [Calling, texting and emailing](#3-calling-texting-and-emailing)
4. [Numbers and usernames](#4-numbers-and-usernames)
5. [The text](#5-the-text)
6. [What the composable cannot know](#6-what-the-composable-cannot-know)
7. [A contact card](#7-a-contact-card)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The examples are full pages from a Nuxt project, and each is a single `.vue` file. In Nuxt `ref`, `computed` and the composables need no `import`, so none is written. The output under a page shows where the visitor is sent when a button is pressed: a new tab with an address, or the phone's own app with a `tel:`, `sms:` or `mailto:` address. The numbers and names in the examples are made up.

---

## 1. How useCommunication works

First the vocabulary, which comes back all the time:

- **Messenger link.** An ordinary `https` address of a messenger (WhatsApp, Telegram, Bale) that opens a chat with a person and, if asked, a ready text.
- **Scheme link.** An address that starts with `tel:`, `sms:` or `mailto:`. The browser hands it to the phone or computer, which opens the dialer, the messages app or the mail app.
- **Encoding.** Writing a text so that it can sit inside an address: a space becomes `%20`, a line break `%0A`, and so on. The composable does it for the text, the subject and the body.

The functions, one by one:

| Call | Opens | How |
|------|-------|-----|
| `sendWhatsApp(phone, message)` | `https://wa.me/phone?text=...` | A new tab |
| `sendTelegram(username, message)` | `https://t.me/username?text=...` | A new tab |
| `sendBale(username, message)` | `https://ble.ir/username?text=...` | A new tab |
| `sendSMS(phone, message)` | `sms:phone?body=...` | The current page asks the phone for its messages app |
| `sendEmail(email, subject, message)` | `mailto:email?subject=...&body=...` | The current page asks for the mail app |
| `callPhone(phone)` | `tel:phone` | The current page asks for the phone app |

```js
talk.sendWhatsApp('15551230100', 'text')      // new tab: wa.me
talk.sendTelegram('teashop', 'text')          // new tab: t.me
talk.sendBale('teashop', 'text')              // new tab: ble.ir

talk.callPhone('+15551230100')                // the phone app: tel:
talk.sendSMS('+15551230100', 'text')          // the messages app: sms:
talk.sendEmail('hello@shop.example', 'Subject', 'text')   // the mail app: mailto:
```

Four points are worth remembering:

- **Nothing is sent by the page.** The composable only opens an address. The visitor sees a chat or a draft and presses Send themselves.
- **There is no result.** Every function returns nothing; the page cannot tell whether the visitor sent the message or whether an app opened at all.
- **The numbers and names are not changed.** What the page passes is put into the address as it is (chapter 4).
- **Only the email has defaults.** The subject and body of `sendEmail` default to empty; the other texts do not (chapter 5).

## 2. Messengers

A product page has a text box with a suggested question, and three buttons that take the visitor to a chat with the shop, with the question already typed.

```vue
<script setup>
const talk = useCommunication()
const question = ref('Hello! Is the blue teapot still in stock?')
</script>

<template>
  <h3>Ask us about a product</h3>
  <textarea id="question" v-model="question" rows="3" />
  <button id="whatsapp" @click="talk.sendWhatsApp('15551230100', question)">WhatsApp</button>
  <button id="telegram" @click="talk.sendTelegram('teashop', question)">Telegram</button>
  <button id="bale" @click="talk.sendBale('teashop', question)">Bale</button>
</template>
```

```text
Visitor presses WhatsApp with the suggested text:   a new tab opens https://wa.me/15551230100?text=Hello!%20Is%20the%20blue%20teapot%20still%20in%20stock%3F
Visitor rewrites the question and presses Telegram: a new tab opens https://t.me/teashop?text=Two%20questions%3A%0AIs%20it%20in%20stock%3F%20Price%20%26%20shipping%20to%2012%20Main%20St%20%234%3F
Visitor presses Bale:                               a new tab opens https://ble.ir/teashop?text=Two%20questions%3A%0AIs%20it%20in%20stock%3F%20Price%20%26%20shipping%20to%2012%20Main%20St%20%234%3F
```

Each button is one call: the contact (a phone number for WhatsApp, a username for Telegram and Bale) and the text. The three differ only in the address they open; all three open it in a **new tab**, so the shop page stays where it was.

Look at the output as the visitor's last view before the chat opens. The text has been encoded: spaces became `%20`, the question mark `%3F`, the line break `%0A` and the `&` of "Price & shipping" became `%26`, so the messenger receives the text exactly as typed. Without that `&` the rest of the text would be taken as a new parameter.

## 3. Calling, texting and emailing

Some customers would rather talk. A help box for an order has three buttons: call, text and email. They do not open a new tab: the page asks the device for the right app.

```vue
<script setup>
const talk = useCommunication()
const order = ref('A-1042')

const email = () => talk.sendEmail(
  'support@teashop.example',
  `Question about order ${order.value}`,
  `Hello,\nI have a question about order ${order.value}.\nThank you`
)
</script>

<template>
  <h3>Need help with an order?</h3>
  <input id="order" v-model="order">
  <button id="call" @click="talk.callPhone('+15551230100')">Call us</button>
  <button id="sms" @click="talk.sendSMS('+15551230100', `Order ${order}: where is it?`)">Text us</button>
  <button id="email" @click="email">Email us</button>
</template>
```

```text
Visitor presses Call us:                                         the phone app is asked for tel:+15551230100
Visitor presses Text us:                                         the phone app is asked for sms:+15551230100?body=Order%20A-1042%3A%20where%20is%20it%3F
Visitor presses Email us:                                        the phone app is asked for mailto:support@teashop.example?subject=Question%20about%20order%20A-1042&body=Hello%2C%0D%0AI%20have%20a%20question%20about%20order%20A-1042.%0D%0AThank%20you
Visitor changes the order number to B&C #7 and presses Email us: the phone app is asked for mailto:support@teashop.example?subject=Question%20about%20order%20B%26C%20%237&body=Hello%2C%0D%0AI%20have%20a%20question%20about%20order%20B%26C%20%237.%0D%0AThank%20you
```

`callPhone` is the simplest: it only needs a number. `sendSMS` adds the text as the `body` of the message. `sendEmail` is the richest: the address, a subject and a body, here built from the order number the visitor typed.

The last line of the output shows the order number `B&C #7` inside the subject and the body as `B%26C%20%237`. Because the composable encodes every part, the `&` and the `#` stay part of the order number instead of cutting the address.

The composable does the encoding, so the page passes plain text: write `Hello, I have a question`, not an already-encoded string, or the text would be encoded twice.

## 4. Numbers and usernames

The composable puts a phone number or a username into the address exactly as it receives it. Messengers are strict about their form, so the page has to prepare it. A profile stores a number the way the customer typed it, `0912 345 6789`:

```vue
<script setup>
const talk = useCommunication()
const saved = ref('0912 345 6789')          // how the customer typed it in the profile

// WhatsApp wants the number as digits only, with the country code and without a leading 0
function forWhatsApp(number, country = '98') {
  const digits = number.replace(/\D/g, '')
  return digits.startsWith('0') ? country + digits.slice(1) : digits
}
</script>

<template>
  <h3>Message the customer</h3>
  <input id="phone" v-model="saved">
  <button id="raw" @click="talk.sendWhatsApp(saved, 'Your order has shipped.')">As typed</button>
  <button id="clean" @click="talk.sendWhatsApp(forWhatsApp(saved), 'Your order has shipped.')">Cleaned</button>
</template>
```

```text
Customer phone is saved as 0912 345 6789. Visitor presses As typed: a new tab opens https://wa.me/0912 345 6789?text=Your%20order%20has%20shipped.
Visitor presses Cleaned:                                            a new tab opens https://wa.me/989123456789?text=Your%20order%20has%20shipped.
```

Pressing "As typed" opens `wa.me` with spaces and a leading zero, which is not a number WhatsApp can find. After cleaning, the address holds only digits with the country code. Each channel has its own rules:

| Channel | Write the contact as | Example |
|---------|----------------------|---------|
| WhatsApp | Digits only, country code first, no leading 0, no `+` or spaces | `15551230100` |
| Telegram | The username without `@` | `teashop` |
| Bale | The username without `@` | `teashop` |
| Call and SMS | Digits, with a leading `+` if you like | `+15551230100` |

Keeping three tiny helpers next to the page solves it once:

```js
// WhatsApp: digits only, with the country code, no leading 0
const forWhatsApp = (number, country = '98') => {
  const digits = number.replace(/\D/g, '')
  return digits.startsWith('0') ? country + digits.slice(1) : digits
}

// Telegram and Bale: the username without the @
const forMessenger = name => name.replace(/^@/, '')

// calls and texts: digits and the plus sign
const forDialing = number => number.replace(/[^\d+]/g, '')
```

A username written with `@` is not rejected; it simply becomes `t.me/@teashop`, which is not the profile you meant. The composable does not clean anything because the right form depends on the country and the channel.

## 5. The text

The message, subject and body go through the browser's `encodeURIComponent`, so any character is safe: Persian letters, quotes, `&`, `#`, `%` and `+`. Nothing needs to be escaped by the page. Three details are worth knowing.

| Part | What the composable does |
|------|--------------------------|
| `message` of the messengers and SMS | Encoded; a line break becomes `%0A` |
| `subject` of the email | Encoded |
| `message` of the email | Encoded; each `%0A` is then written as `%0D%0A`, the line break mail apps expect |
| The email address, the phone, the username | Not encoded |

First, **write line breaks as `\n`.** The email function turns each `\n` into `\r\n` by itself, so a text that already uses `\r\n` ends up with an extra, broken line break.

```js
// good: \n becomes a line break in the mail
talk.sendEmail(to, 'Subject', 'Line one\nLine two')

// avoid: \r\n comes out as an extra, broken line break
talk.sendEmail(to, 'Subject', 'Line one\r\nLine two')
```

Second, **only `sendEmail` has default values.** Calling `sendEmail(address)` alone opens a draft with an empty subject and body. But `sendSMS(phone)` or `sendWhatsApp(phone)` without a text writes the word `undefined` as the message, so always pass one. The defaults apply only to a missing value (`undefined`): a `null` body becomes the word `null`.

```js
// Only sendEmail has defaults. For the others, always give the text:
talk.sendWhatsApp(number, message || 'Hello!')
talk.sendSMS(number, message || 'Hello!')
// without it the text of the message would be the word "undefined"
```

Third, the address in a very long text is long too. Messengers and mail apps accept long texts, but there is no promise at the very long end; keep the pre-filled text short and let the visitor add the rest.

## 6. What the composable cannot know

The composable opens an address and its job ends there. A few consequences:

- **Open it from a click.** The three messenger functions open a new tab, and browsers allow this when it follows a visitor's action. Call them inside a click handler, not from a timer or after a long wait.
- **There is no confirmation.** Not whether the app exists, not whether the visitor sent the text. If you must know, ask the visitor or use a server-side channel.
- **A computer may have no app.** `tel:` and `sms:` make sense on a phone; on a computer the browser may offer an app, do nothing, or ask the visitor what to use. Show these buttons where they help, for example next to the number so it can also be read and dialed by hand.
- **The way a phone reads `sms:` varies.** The composable always writes `?body=`; how a particular phone fills the message from it is the phone's own business.
- **The text is visible in the address.** Anything put in the text can be seen in the visitor's history and in the messenger's link. Do not put secrets in it.

Because the visitor is the one who sends, a pre-filled text is a suggestion that they can change. Write it as something they would be glad to send.

## 7. A contact card

All of the above on one page: the card on a product page. The seller may have only some of the channels, so the page shows a button for each channel that exists and builds the pre-filled text from the product.

```vue
<script setup>
const talk = useCommunication()

const sellers = {
  teashop: { name: 'Tea Shop', phone: '+1 555 123 0100', whatsapp: '15551230100', telegram: 'teashop', email: 'hello@teashop.example' },
  corner: { name: 'Corner Stall', phone: '+1 555 987 0200' }
}
const product = 'Blue teapot'
const who = ref('teashop')
const seller = computed(() => sellers[who.value])
const text = computed(() => `Hello, is "${product}" still available?`)
const dial = computed(() => seller.value.phone.replace(/[^\d+]/g, ''))   // keep digits and the plus sign
</script>

<template>
  <select id="seller" v-model="who">
    <option value="teashop">Tea Shop</option>
    <option value="corner">Corner Stall</option>
  </select>
  <h3>Contact {{ seller.name }}</h3>
  <div id="buttons">
    <button v-if="seller.whatsapp" id="whatsapp" @click="talk.sendWhatsApp(seller.whatsapp, text)">WhatsApp</button>
    <button v-if="seller.telegram" id="telegram" @click="talk.sendTelegram(seller.telegram, text)">Telegram</button>
    <button id="call" @click="talk.callPhone(dial)">Call</button>
    <button id="sms" @click="talk.sendSMS(dial, text)">Text</button>
    <button v-if="seller.email" id="email" @click="talk.sendEmail(seller.email, `About: ${product}`, text)">Email</button>
  </div>
</template>
```

```text
Page opens with Tea Shop:         buttons WhatsApp, Telegram, Call, Text, Email.
Visitor presses WhatsApp:         a new tab opens https://wa.me/15551230100?text=Hello%2C%20is%20%22Blue%20teapot%22%20still%20available%3F
Visitor presses Call:             the phone app is asked for tel:+15551230100
Visitor presses Email:            the phone app is asked for mailto:hello@teashop.example?subject=About%3A%20Blue%20teapot&body=Hello%2C%20is%20%22Blue%20teapot%22%20still%20available%3F
Visitor switches to Corner Stall: buttons Call, Text.
Visitor presses Text:             the phone app is asked for sms:+15559870200?body=Hello%2C%20is%20%22Blue%20teapot%22%20still%20available%3F
```

The card follows the same recipe as the earlier pages: the contact data sits in one object, the text is a computed value built from the product, and every button is a single call. The page cleans the number once (`dial`) and passes it to the call and the text. `v-if` hides the channels that a seller does not have, as the switch to the second seller shows: only Call and Text remain.

The recipe carries over to your own pages: keep each contact in the form its channel needs, build the text from what the visitor is looking at, always pass a text, write line breaks as `\n`, and call the function from a click.

---

## Appendix A. Quick reference

### A.1 Calls

| Call | Opens | New tab |
|------|-------|---------|
| `sendWhatsApp(phone, message)` | `https://wa.me/phone?text=message` | Yes |
| `sendTelegram(username, message)` | `https://t.me/username?text=message` | Yes |
| `sendBale(username, message)` | `https://ble.ir/username?text=message` | Yes |
| `sendSMS(phone, message)` | `sms:phone?body=message` | No |
| `sendEmail(email, subject, message)` | `mailto:email?subject=subject&body=message` | No |
| `callPhone(phone)` | `tel:phone` | No |

### A.2 Defaults

| Call | Defaults |
|------|----------|
| `sendEmail` | `subject` and `message` are empty |
| All the others | None; a missing text becomes `undefined` |

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| No app for the scheme on the device | Browser dependent; the composable cannot tell |
| New tab blocked by the browser | Nothing opens; no error is raised |
| `sendSMS` or a messenger called without a text | The text is the word `undefined` |
| Number written with spaces or a leading 0 for WhatsApp | The chat is not found |
| Username written with `@` | The address contains `@`; the profile is not found |
| `\r\n` in an email text | An extra, broken line break |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Passing an already-encoded text.** It is encoded again.
- **Forgetting the text.** The message becomes `undefined`.
- **A leading 0, spaces or a `+` in the WhatsApp number.** Clean it first.
- **An `@` in the Telegram or Bale username.** Remove it.
- **`\r\n` in the email.** Use `\n`.
- **Calling a messenger function after a long wait.** The browser may block the new tab.
- **Showing `tel:` and `sms:` buttons as the only way to reach you on a computer.** Print the number too.
- **Secrets in the text.** It is visible in the address.

### C.2 Behavior worth knowing

- Messenger functions open a new tab; the other three act on the current page.
- Only the text, subject and body are encoded; contacts are used as they are.
- The email body turns `%0A` into `%0D%0A` by itself.
- Nothing is returned, and nothing is sent by the page.

### C.3 Limits

| What | Limit |
|------|-------|
| Channels | WhatsApp, Telegram, Bale, SMS, email, phone call |
| Several recipients | Not supported by the functions |
| Attachments | Not supported |
| Result or confirmation | None |
| Cleaning of numbers and names | None |

### C.4 Tuning

- Keep the contact data in the form each channel needs, and clean it once.
- Build the pre-filled text from the product or order the visitor is viewing.
- Always pass a text to the messenger and SMS functions.
- Show only the channels that exist for the seller.
- Keep the pre-filled text short.

## Appendix D. Message reference

### D.1 Errors

The composable raises no errors of its own and has no messages.

### D.2 Values worth recognizing

| Value | Meaning |
|-------|---------|
| `%20`, `%3F`, `%26`, `%23` | An encoded space, `?`, `&`, `#` |
| `%0A` | An encoded line break in messengers and SMS |
| `%0D%0A` | The line break of an email |
| `undefined` | The text when none was passed to a messenger or SMS call |
