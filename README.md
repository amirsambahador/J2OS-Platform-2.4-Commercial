# J2OS Platform 2.4

J2OS Platform is a collection of small, independent building blocks for business applications. It has a Java back end (libraries under `org.j2os.platform`) and a Vue/Nuxt front end (composables and components). Each block does one job, does it once and does it well, so that an application does not have to write the same code again: lists and tables, validation, reports, security, messaging, and artificial intelligence from speech and image recognition to training a language model and making a talking-face video.

Every module comes with a tutorial book. This file is the map: what the platform is, which module to read for which job, and where everything is.

## Contents

- [What J2OS is, and what it is not](#what-j2os-is-and-what-it-is-not)
- [Which module do I need?](#which-module-do-i-need)
- [Modules that work together](#modules-that-work-together)
- [Back end (Java)](#back-end-java)
- [Front end (Vue / Nuxt)](#front-end-vue--nuxt)
- [What the AI modules need](#what-the-ai-modules-need)
- [How the books are organized](#how-the-books-are-organised)
- [Folder layout](#folder-layout)
- [What is new in 2.4](#what-is-new-in-24)
- [Notes](#notes)

## What J2OS is, and what it is not

**It does not depend on a framework.** You do not have to change the framework of your project to use a module. Each module can be taken alone and used in a program written with Spring, with Jakarta EE or with plain Java. Where a web layer is needed, the book of that module shows how to connect it to each of them. The front-end modules are plain composables and components for Nuxt.

**Principles shared by all modules:**

- **One job per module.** You add only what you need.
- **Plain inputs and outputs.** `String`, `Map`, files and small records go in and out. Classes of the libraries underneath (OpenNLP, Weka, Jakarta Mail and so on) do not appear in your code.
- **Small, readable APIs.** A first result takes a few lines, and the whole interface of a module usually fits on one page.
- **AI runs on your own machine.** The AI modules work offline and send nothing out. Most fetch or install something on the first run; the table below lists it.
- **Errors explain themselves.** A wrong input is caught early with a message that says what to change.

**What it is not:** it is not an application server, a framework or a single product to install. It is a shelf of parts.

## Which module do I need?

| I want to | Read |
|-----------|------|
| Show a list or table with search, sort and paging | [page2](BackEnd/page2_DataTable_DataCard/README.md), [DataTable](FrontEnd/DataTable/README.md), [DataCard](FrontEnd/DataCard/README.md) |
| Add repeating rows to a form, such as invoice lines | [useDynamicForm](FrontEnd/useDynamicForm/README.md) |
| Split a long form into steps | [DataWizard](FrontEnd/DataWizard/README.md) |
| Check form input | [useValidation](FrontEnd/useValidation_useValidationRefBased/README.md), [jvalidation](BackEnd/jvalidation/README.md) |
| Use Persian dates and write amounts in words | [useDate](FrontEnd/useDate_useDateRefBased/README.md), [useNumber](FrontEnd/useNumber_useNumberRefBased/README.md), [jutil](BackEnd/jutil/README.md) |
| Produce PDF, Excel and Word reports | [jreport](BackEnd/jreport/README.md) |
| Handle sign-in, tokens and application security | [jsecurity](BackEnd/jsecurity/README.md), [useTokenProcessor](FrontEnd/useTokenProcessor/README.md) |
| Clean text that users typed (XSS) | [useXssProtector](FrontEnd/useXssProtector/README.md), [jsecurity](BackEnd/jsecurity/README.md) |
| Send and read e-mail | [jmail](BackEnd/jmail/README.md) |
| Build a Telegram bot | [jtelegram](BackEnd/jtelegram/README.md) |
| Push news from the server to open pages | [jpush](BackEnd/jpush_useJPush/README.md) |
| Hold video meetings in the browser | [jmeet](BackEnd/jmeet_useJMeet_useJMeetTranslator/README.md) |
| Pass messages between programs through queues | [jartemis](BackEnd/jartemis/README.md) |
| Call another service from Java | [jclient](BackEnd/jclient/README.md), [jbalancer](BackEnd/jbalancer/README.md), [resicord](BackEnd/resicord/README.md) |
| Retry and protect calls to an unreliable dependency | [resicord](BackEnd/resicord/README.md) |
| Run a business process with waiting steps, such as a leave approval | [jflow](BackEnd/jflow/README.md) |
| Split a huge table over several databases | [jshard](BackEnd/jshard/README.md) |
| Work with files and folders | [jfile](BackEnd/jfile/README.md) |
| Run system commands locally or on another machine | [jterminal](BackEnd/jterminal/README.md) |
| Turn text into speech | [ai.dl.sound.speech](BackEnd/ai_dl_sound_speech/README.md) |
| Turn speech into text | [ai.dl.sound.recognition](BackEnd/ai_dl_sound_recognition_useMicrophone/README.md), [useMicrophone](FrontEnd/useMicrophone/README.md) |
| Read text in a particular person's voice | [ai.dl.sound.cloner](BackEnd/ai_dl_sound_cloner/README.md) |
| Make a video of a talking face | [ai.dl.vision.talkinghead](BackEnd/ai_dl_vision_talkinghead/README.md) |
| Train a language model on my own examples | [ai.dl.language.finetune](BackEnd/ai_dl_language_finetune/README.md) |
| Read text inside a picture | [ai.dl.vision.ocr](BackEnd/ai_dl_vision_ocr/README.md) |
| Find faces or objects in a photo, or compare pictures | [ai.dl.vision.face](BackEnd/ai_dl_vision_face/README.md), [ai.dl.vision.object](BackEnd/ai_dl_vision_object/README.md), [ai.dl.vision.similarity](BackEnd/ai_dl_vision_similarity/README.md) |
| Translate and analyse text, or learn from data | [ai.dl.language.translator](BackEnd/ai_dl_language_translator/README.md), [ai.dl.language.nlp](BackEnd/ai_dl_language_nlp/README.md), [ai.ml](BackEnd/ai_ml/README.md) |

## Modules that work together

Some modules have two halves, one in Java and one in the browser, or the output of one is the input of another.

- **page2 + DataTable + DataCard.** Java produces one page of rows with the total; the components show it.
- **useMicrophone + ai.dl.sound.recognition.** The browser records the voice; the server turns it into text.
- **useJMeet + jmeet.** The server keeps meetings and members; sound and video go straight between the browsers.
- **useJPush + jpush.** The server sends a message to an open page over WebSocket.
- **jclient + jbalancer + resicord.** Choose the address among the copies of a service, call it, and protect the call against failure.
- **ai.dl.sound.speech or ai.dl.sound.cloner + ai.dl.vision.talkinghead.** A text becomes a WAV, and the WAV becomes a talking-face video.
- **useBinder + tools that need an id.** A Nuxt UI component has no id; `useBinder` supplies one so that tools such as `useNumber` or `useDate` can attach to it.

## Back end (Java)

Thirty-two books.

### Artificial intelligence and machine learning

Language, sound and pictures. All of these run on your own machine and send no data out.

- **[ai.dl.language.finetune](BackEnd/ai_dl_language_finetune/README.md)**: Teaches an existing language model your own style and facts. You give a base model and a file of examples; the library prepares its own Python environment, checks the examples, looks at your graphics card, chooses safe settings, trains in the background and saves the result. The result can be exported to GGUF. The book assumes you have never fine-tuned anything.
- **[ai.dl.language.nlp](BackEnd/ai_dl_language_nlp/README.md)**: Everyday text analysis on Apache OpenNLP: sentences and words, parts of speech, word stems, keywords, names of people and places, language detection and answering a question from a text. Plain `String` in, small records out.
- **[ai.dl.language.translator](BackEnd/ai_dl_language_translator/README.md)**: Offline translation on LibreTranslate, so the text never leaves your machine. The pattern is short: load, translate, close.
- **[ai.dl.sound.cloner](BackEnd/ai_dl_sound_cloner/README.md)**: Voice cloning on Piper and OpenVoice. You enroll a voice from a clean recording of 10 to 30 seconds, then ask for any new text in that voice. Clone only a voice whose owner has agreed.
- **[ai.dl.sound.effect](BackEnd/ai_dl_sound_effect/README.md)**: Four sound effects made with plain digital signal processing, not with AI: a WAV goes in, a changed WAV comes out. Nothing to download or install.
- **[ai.dl.sound.recognition (with useMicrophone)](BackEnd/ai_dl_sound_recognition_useMicrophone/README.md)**: Offline speech recognition on Vosk. The server loads the model once and turns any number of recordings into text; the front-end composable `useMicrophone` records in the browser and sends the audio.
- **[ai.dl.sound.speech](BackEnd/ai_dl_sound_speech/README.md)**: Offline text-to-speech on Piper: give a text, get a WAV file or a `byte[]`. No online service, no key, no price per character.
- **[ai.dl.vision.face](BackEnd/ai_dl_vision_face/README.md)**: Finds faces in a photo and cuts them out, on OpenCV. Good for profile pictures and for splitting a group photo into one picture per person. The detector data is bundled.
- **[ai.dl.vision.object](BackEnd/ai_dl_vision_object/README.md)**: Object detection: says what is in a picture and where, using DJL and a pretrained neural network that is downloaded on first use.
- **[ai.dl.vision.ocr](BackEnd/ai_dl_vision_ocr/README.md)**: Reads the text inside pictures (OCR) on Tesseract, from invoices and scans to street signs. Give a picture, get text.
- **[ai.dl.vision.talkinghead](BackEnd/ai_dl_vision_talkinghead/README.md)**: Makes an MP4 video of a talking face from a voice recording and either a photo or a cartoon avatar that the library draws. The mouth follows the words and the eyes blink. Runs on the processor, with a queue, progress reports and cancel.
- **[ai.dl.vision.similarity](BackEnd/ai_dl_vision_similarity/README.md)**: Measures how much two pictures, or two faces, resemble each other by turning each picture into a list of numbers. For "similar products", rejecting a duplicate upload and face comparison. You supply the neural network file.
- **[ai.ml](BackEnd/ai_ml/README.md)**: Classic machine learning on Weka for modest amounts of data: classification, predicting a number and clustering. Trains from a file, keeps the model in a file and answers with plain `Map` inputs and small answer objects.

### Communication and messaging

Sending and receiving messages: e-mail, Telegram, message queues, live messages to the browser, video meetings and calls to other services.

- **[jartemis](BackEnd/jartemis/README.md)**: A message broker on Apache ActiveMQ Artemis with very little to configure. A queue gives each message to one reader; a topic gives it to every subscriber. The broker can run inside your own program.
- **[jclient](BackEnd/jclient/README.md)**: An HTTP client built from a Java interface: each method is tied to one request with one line, on OpenFeign. Calling a remote service feels like calling a normal method.
- **[jmail](BackEnd/jmail/README.md)**: Sends and reads e-mail over SMTP and IMAP with one small client on Jakarta Mail and a short, fluent API. One client per mailbox, no global configuration.
- **[jmeet (with useJMeet and useJMeetTranslator)](BackEnd/jmeet_useJMeet_useJMeetTranslator/README.md)**: Browser video meetings with WebRTC. The Java server only keeps meetings and members and relays signaling; sound and video go straight between the browsers. Front-end code and a live translation helper are included.
- **[jpush (with useJPush)](BackEnd/jpush_useJPush/README.md)**: Pushes messages from the server to open pages over WebSocket. It knows the user from a JWT and sends to one user or to everyone. Works with Spring and with plain Jakarta EE.
- **[jtelegram](BackEnd/jtelegram/README.md)**: A Telegram bot with four simple classes on telegrambots: receive text, answer commands, show buttons and send messages.

### Infrastructure and data

Parts that work underneath an application: paging, shared objects, files, business processes, pools, sharding and terminals.

- **[jbalancer](BackEnd/jbalancer/README.md)**: Round-robin choice among the addresses of the copies of a service: each name has a list of addresses and each call returns the next one. Tiny and predictable.
- **[jcrux](BackEnd/jcrux/README.md)**: A container of live Java objects over RMI. The server keeps the objects; clients create them, call their methods and read their fields. Any serializable class works, with no tables and no annotations.
- **[jfile](BackEnd/jfile/README.md)**: The `JFile` class for files, folders and web addresses: read and write, copy, safe zipping, multi-connection downloads and watching a directory for changes.
- **[jflow](BackEnd/jflow/README.md)**: The Flowable business process engine with BPMN definitions, made available over the network with RMI. Starts processes and moves forward the tasks that wait for a person.
- **[jpool](BackEnd/jpool/README.md)**: An object pool for objects that are expensive to create, such as a report engine or a connection to another system. Lends a fixed number of ready objects and never creates more than the limit.
- **[jshard](BackEnd/jshard/README.md)**: Splits tables over several databases on Apache ShardingSphere and HikariCP. The result is an ordinary `DataSource`, so your code keeps using JDBC or JPA.
- **[jterminal](BackEnd/jterminal/README.md)**: A real, persistent terminal for running commands from Java. The shell stays alive between commands, and interactive programs such as `psql` and `ssh` can be driven too.
- **[page2 (with DataTable and DataCard)](BackEnd/page2_DataTable_DataCard/README.md)**: Paging, sorting and searching for lists on both sides. In Java, page2 turns a JPA entity, a JPQL query, a SQL query or a `List` into "one page of rows plus the total"; DataTable and DataCard show it.

### General tools

Jobs that almost every application needs: reports, security, calendars and numbers, validation and protection against unreliable dependencies.

- **[jreport](BackEnd/jreport/README.md)**: Builds PDF, Excel and Word reports from data and a template, on JasperReports and DynamicReports. Sends the file for download or saves it on disk.
- **[jsecurity](BackEnd/jsecurity/README.md)**: Application security with small tools: hashing and encryption, signed tokens, a request filter, text sanitizing, safe file upload with a virus check, and sign-in with Google.
- **[jutil](BackEnd/jutil/README.md)**: Five shared classes: the Persian calendar, amounts and numbers in words, JSON, geographic distance and access to Spring.
- **[jvalidation](BackEnd/jvalidation/README.md)**: Validation with readable rule descriptions. Every rule is checked and every problem comes back together, with field name, code and message.
- **[resicord](BackEnd/resicord/README.md)**: Protects calls to unreliable dependencies with four shields: retry, time limit, bulkhead (sharing capacity) and fallback.

## Front end (Vue / Nuxt)

Thirty-three books.

### Components

Vue components that draw something on the page.

- **[DataCard](FrontEnd/DataCard/README.md)**: A Vue component for long lists, shown as cards in a responsive grid. More rows arrive as you scroll; search, sorting, slow networks and server errors are handled for you.
- **[DataMap](FrontEnd/DataMap/README.md)**: A map component on Leaflet and OpenStreetMap, loaded only when needed. Picks an address, shows one place or many branches, and can show routes and live tracking.
- **[DataStar](FrontEnd/DataStar/README.md)**: A small star component that shows an average rating or collects one from the visitor, with the same row of stars.
- **[DataTable](FrontEnd/DataTable/README.md)**: A ready table for business pages on Nuxt UI, with search, sorting, paging and row selection. Follows the page2 convention and handles requests and errors itself.
- **[DataTree](FrontEnd/DataTree/README.md)**: A tree for folders inside folders, such as company departments or shop categories. Each level is fetched from the server only when it is opened.
- **[DataWizard](FrontEnd/DataWizard/README.md)**: A multi-step dialog for long forms or forms that depend on earlier answers, built on Nuxt UI modal, stepper and alert. Each step is checked before the next.

### Composables

Functions that make one browser job simple for a page. Some have a RefBased version, which finds the element with a ref instead of an id.

- **[useApiClient](FrontEnd/useApiClient/README.md)**: Six functions for talking to the server: read, send a form, send a file and download. Adds the sign-in token, reads the answer correctly and gives server errors to the page in one shape.
- **[useBinder](FrontEnd/useBinder/README.md)**: A bridge between tools that need an element id and Nuxt UI components that usually have none. Waits for the field to appear, gives its input an id and then calls your function with it.
- **[useCodeEditor and useCodeEditorRefBased](FrontEnd/useCodeEditor_useCodeEditorRefBased/README.md)**: The Ace code editor in place of a textarea, with colors, indentation and suggestions. The RefBased version finds the element with a ref instead of an id.
- **[useCommunication](FrontEnd/useCommunication/README.md)**: Ready contact links for WhatsApp, Telegram, Bale, SMS, e-mail and phone. Encodes the message so spaces, question marks and new lines arrive intact.
- **[useCryptography](FrontEnd/useCryptography/README.md)**: Locks and unlocks text with a password in the browser, on its built-in cryptography. Chooses the algorithm and parameters once, safely, so the page needs two functions.
- **[useDate and useDateRefBased](FrontEnd/useDate_useDateRefBased/README.md)**: Turns a text field into a Jalaali (Persian) date box that refuses a date that does not exist while the user types. The RefBased version finds the field with a ref.
- **[useDigitalSignature and useDigitalSignatureRefBased](FrontEnd/useDigitalSignature_useDigitalSignatureRefBased/README.md)**: A signature box for mouse or finger, a restart button and saving the signature as an image, for delivery receipts, contracts and consent forms.
- **[useDynamicForm](FrontEnd/useDynamicForm/README.md)**: Repeating rows in a form, such as invoice lines or phone numbers. Keeps the rows, gives each a stable identity and prevents removing the last one.
- **[useFileLoader](FrontEnd/useFileLoader/README.md)**: Loads a script or stylesheet only for the page that needs it and removes it when the visitor leaves; for heavy libraries such as maps, editors and charts.
- **[useFullscreen](FrontEnd/useFullscreen/README.md)**: Wraps the browser fullscreen mode in four functions and hides its traps: it works only after a click, it does not report success, and it knows nothing about Vue.
- **[useJCookie](FrontEnd/useJCookie/README.md)**: Reads, writes and removes cookies with three functions, for remembering a language, a cookie banner choice or a short-lived choice such as an open cart.
- **[useKeyListener and useKeyListenerRefBased](FrontEnd/useKeyListener_useKeyListenerRefBased/README.md)**: Keyboard habits for forms: Enter to move to the next field, Ctrl+S to save, Escape to close. The RefBased version is for fields that come and go, such as modals and wizard steps.
- **[useLocalStorage](FrontEnd/useLocalStorage/README.md)**: The browser local storage in four functions, for keeping a dark theme, half-written text or a sign-in state until the next visit.
- **[useLocationAPI](FrontEnd/useLocationAPI/README.md)**: Does two steps behind five functions: ask the device for coordinates and ask a map service for the name of the place. For filling in an address and for "stores near me".
- **[useMicrophone](FrontEnd/useMicrophone/README.md)**: Records the microphone for dictation: asks permission, records, converts to the format a speech recognizer needs, uploads and reads the answer, behind a few refs and three functions.
- **[useNetworkAPI](FrontEnd/useNetworkAPI/README.md)**: Network information behind simple functions: online or offline, connection quality and approximate location from an outside service.
- **[useNumber and useNumberRefBased](FrontEnd/useNumber_useNumberRefBased/README.md)**: Keeps the display and the value of a number apart: the user sees 1,250,000 and the server gets 1250000. Writes an amount in words as well. The RefBased version is for inputs that come and go.
- **[useOperatingSystemNotification](FrontEnd/useOperatingSystemNotification/README.md)**: A notification from the operating system itself, outside the browser and over any open window, for when the visitor has moved on to other work.
- **[useQR and useQRRefBased](FrontEnd/useQR_useQRRefBased/README.md)**: Draws a QR code inside a page element and saves it as an image, for a product link or a ticket number. The RefBased version finds the element with a ref.
- **[useRichText and useRichTextRefBased](FrontEnd/useRichText_useRichTextRefBased/README.md)**: The SunEditor rich text editor in place of a textarea, so users can write headings, bold text and lists and the page keeps HTML. The RefBased version finds the textarea with a ref.
- **[useScreenRecorder](FrontEnd/useScreenRecorder/README.md)**: Records the screen with microphone sound and saves the video file on the user computer, behind four functions; for bug reports, lessons and support.
- **[useSpeechRecognitionAPI](FrontEnd/useSpeechRecognitionAPI/README.md)**: The speech recognition of the browser in a simple interface: start, collect results, stop and save the text to a file.
- **[useString](FrontEnd/useString/README.md)**: Shortens text with three dots at the end, for a table summary, a notification or a small hint.
- **[useTokenProcessor](FrontEnd/useTokenProcessor/README.md)**: Reads a JWT in the browser: user name, role and the end of the session, without asking the server again.
- **[useTranslatorAPI](FrontEnd/useTranslatorAPI/README.md)**: Quick translation of a text with the free public MyMemory service, for when a rough machine translation is enough.
- **[useValidation and useValidationRefBased](FrontEnd/useValidation_useValidationRefBased/README.md)**: Checks a form straight from its HTML: filled in, length, the shape of a mobile number or e-mail, with uniform messages. The RefBased version finds the form with a template ref.
- **[useXssProtector](FrontEnd/useXssProtector/README.md)**: Cleans text before it is shown as HTML with DOMPurify, so a script hidden in a comment or description does not run (an XSS attack).

## What the AI modules need

The AI modules run on your own machine. Most of them fetch or install something the first time. Each book explains the details and how to check them.

| Module | What is needed | First run |
|--------|----------------|-----------|
| ai.dl.sound.speech | Piper and a Piper voice | Nothing more |
| ai.dl.sound.cloner | Piper, Python 3 and a private environment that the library builds | About 900 MB of model and several gigabytes of disk; a few minutes |
| ai.dl.sound.recognition | A Vosk model | Loads the model once |
| ai.dl.sound.effect | Nothing | Nothing |
| ai.dl.vision.talkinghead | Python 3.9 or newer; no graphics card | Builds a Python environment once; photos also download a small model |
| ai.dl.vision.face | Nothing; the detector data is bundled | Nothing |
| ai.dl.vision.object | Internet on the first run | Downloads a pretrained model once |
| ai.dl.vision.ocr | Tesseract | Nothing more |
| ai.dl.vision.similarity | A neural network file that you supply | Nothing |
| ai.dl.language.finetune | Python 3.10 or newer; an NVIDIA graphics card is strongly recommended | PyTorch is several gigabytes; the base model is downloaded |
| ai.dl.language.translator | LibreTranslate | Depends on your installation |
| ai.dl.language.nlp and ai.ml | No separate tool | Nothing |

Voice cloning deserves one more word: clone only a voice whose owner has agreed.

## How the books are organized

Every module has a tutorial book. A book starts from a small, realistic problem, explains the idea in simple words, shows a short example with its real output, and ends with advice. Chapters build on each other, so read them in order. The books themselves are independent, so you can read any one of them alone.

At the end of every book there are four appendices:

- **A. Quick reference.** Calls, settings and defaults in tables.
- **B. Failure handling.** What goes wrong, and what happens when it does.
- **C. Pitfalls, limits and tuning.** The risky points and practical advice.
- **D. Message reference.** Every message the module can produce, with its cause.

Languages:

- English: `README.md` in each module folder.
- Persian: `README.fa.html` in each module folder, with `index.html` as the start page. It introduces the platform, lists every book with a paragraph and has a search box.

If you have never used a module, a short book such as [useString](FrontEnd/useString/README.md) shows the style of the books quickly.

## Folder layout

```text
README/
├── README.md             (this file)
├── index.html            (Persian start page)
├── BackEnd/
│   └── <module>/
│       ├── README.md
│       └── README.fa.html
└── FrontEnd/
    └── <module>/
        ├── README.md
        └── README.fa.html
```

## What is new in 2.4

Three AI modules were added to the back end. They are written for readers who have never touched machine learning:

- [ai.dl.sound.cloner](BackEnd/ai_dl_sound_cloner/README.md) (`VoiceCloner`) speaks text in a voice taken from a short recording.
- [ai.dl.vision.talkinghead](BackEnd/ai_dl_vision_talkinghead/README.md) (`JTalkingHead`) makes a video of a talking face from a voice and a photo or avatar.
- [ai.dl.language.finetune](BackEnd/ai_dl_language_finetune/README.md) (`JFineTune`) trains a language model on your own examples.

## Notes

- Some modules call tools outside Java, such as a speech engine, ffmpeg or Python. Each book says what is needed, what is downloaded the first time, and how to check it.
- Examples in the books are short on purpose. Outputs shown in the books come from running the examples. For the modules that need external tools (voice, video, training), the examples were run against small stand-in tools, so real timings and file sizes on your machine will differ.
