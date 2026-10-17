# ai.dl.language.translator

*J2OS Platform 2.4*

Sooner or later an application meets text in a language its users do not read: a product description, a support message, a news item. Sending it to an online translation service is easy, and it also means that the text leaves the building. **Offline translation** keeps the text on your own machine: a translation engine runs next to the application and answers requests locally.

**ai.dl.language.translator** (`org.j2os.platform.ai.dl.language.translator`) brings this to J2OS. It is built on LibreTranslate, an open-source translation server, and has one class, `Translation`, with one habit: *load, translate, close*. Three properties set it apart:

- **One call to translate.** `translate(text, "en", "fa")` returns the translated `String`. No HTTP code, no JSON in the application.
- **The server is looked after.** If no server runs on the port, `load` starts LibreTranslate, waits until it is ready and stops it again when the last user closes. A server that is already running is used as it is.
- **Safe to share.** An instance can be used by many threads at once, and several instances can share one server.

The whole library in a few lines:

```java
try (Translation translation = Translation.load(5000)) {            // 1. start (or find) the server
    String fa = translation.translate("Good morning", "en", "fa");  // 2. translate
    System.out.println(fa);
}                                                                   // 3. close: the server stops with the last user
```

This guide goes from the first translation (chapter 2) through sharing a server (3), using a server that already runs (4) and start-up trouble (5), to a complete small workflow (6). The appendices work as a reference.

## Contents

1. [How ai.dl.language.translator works](#1-how-aidllanguagetranslator-works)
2. [The first translation](#2-the-first-translation)
3. [Sharing a server](#3-sharing-a-server)
4. [A server that already runs](#4-a-server-that-already-runs)
5. [When the start fails](#5-when-the-start-fails)
6. [Putting it together](#6-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The programs of the guide are complete: each has one `main` method. They use ports 5000 to 5005 on the local machine and the language codes `en` (English) and `fa` (Persian).

---

## 1. How ai.dl.language.translator works

First, a few terms, because they come up throughout the guide:

- **LibreTranslate.** An open-source translation server written in Python. It answers HTTP requests on a port of the local machine. It is installed with `pip`:

```text
pip install libretranslate
```

- **Language code.** A short code that names a language: `en`, `fa`, `de`. The same codes are used in `translate` and in the language list.
- **Language list.** The comma-separated codes that a server loads, such as `"en,fa"`. A language that is not loaded cannot be translated from or to. The default list is `en,fa`.
- **Server ownership.** A `Translation` that started its server owns it and stops it at the end. A `Translation` that found a server already running only uses it.

The classes a program meets:

| Class | Role |
|-------|------|
| `Translation` | Starts or finds a server, translates, and releases the server on `close` |
| `TranslationFailedException` | The one checked exception of the module |

Keep these three points in mind:

- **The engine is a separate process.** Translation runs in Python, outside the JVM. The first `load` on a port can take a while, because the models must be read; after that, calls are fast.
- **A `Translation` is a lease on a server.** Several instances on the same port share one server, and the server stops when the last of them is closed.
- **A server speaks only the languages it loaded.** Asking for another language fails; so does loading a `Translation` with languages the running server does not have.

## 2. The first translation

`Translation.load(port)` gets a server ready and returns an object that translates:

```java
import org.j2os.platform.ai.dl.language.translator.Translation;

public class First {
    public static void main(String[] args) throws Exception {
        try (Translation translation = Translation.load(5000)) {
            String fa = translation.translate("Hello, how are you?", "en", "fa");
            System.out.println(fa);
            System.out.println(translation.translate(fa, "fa", "en"));
        }
    }
}
```

```text
سلام، حال شما چطور است؟
Hello, how are you?
```

- **`load(5000)`.** Looks for a server on port `5000`. There is none, so it starts `libretranslate`, loading English and Persian, and waits until the server answers.
- **`translate`.** Takes the text, the code of its language and the code of the language wanted. The result is plain text.
- **Both directions.** One server loaded with `en,fa` translates English to Persian and back.
- **`try`-with-resources.** `Translation` is `AutoCloseable`. Closing the last user of a server stops it, so a program that forgets to close can leave the server running.
- **The command.** `load(port)` runs the command `libretranslate` and so expects it on the `PATH`. Chapter 5 shows how to name a Python folder instead.

## 3. Sharing a server

Starting a server is the slow part, so the module shares it. Instances on one port use one server:

```java
import java.io.IOException;
import java.net.Socket;
import org.j2os.platform.ai.dl.language.translator.Translation;

public class Shared {
    static boolean listening(int port) {
        try (Socket s = new Socket("localhost", port)) { return true; } catch (IOException e) { return false; }
    }

    public static void main(String[] args) throws Exception {
        Translation a = Translation.load(5000);          // starts the server
        Translation b = Translation.load(5000);          // same port: shares it
        System.out.println("server up: " + listening(5000));

        a.close();
        System.out.println("after first close: " + listening(5000)
                + ", b still works: " + b.translate("Thank you", "en", "fa"));

        b.close();
        System.out.println("after last close: " + listening(5000));

        // a Python folder instead of the PATH, and another port: a different server with its own languages
        try (Translation fa = Translation.load("../python", 5001, "fa,en")) {
            System.out.println(fa.translate("Good morning", "en", "fa"));
        }
    }
}
```

```text
server up: true
after first close: true, b still works: متشکرم
after last close: false
صبح بخیر
```

- **Same port, same server.** The second `load(5000)` finds the server that the first started and does not start another.
- **Counting.** The server runs until the last instance on the port is closed. After the first `close`, the second instance still translates.
- **After the last close.** The port is free again.
- **Another port, another server.** The last block runs a separate server on port `5001`. It is useful for different language lists or for keeping two uses apart.
- **A Python folder.** `load("../python", 5001, "fa,en")` starts the server from `bin/libretranslate` in that folder (on Windows `Scripts/libretranslate.exe`) instead of the `PATH`.
- **Closing twice.** `close` is safe to call again; it does nothing the second time.

## 4. A server that already runs

An administrator may run LibreTranslate as a service. If a server already answers on the port, `load` uses it and never stops it:

```java
import java.io.IOException;
import java.net.Socket;
import org.j2os.platform.ai.dl.language.translator.Translation;

public class Reuse {
    static boolean listening(int port) {
        try (Socket s = new Socket("localhost", port)) { return true; } catch (IOException e) { return false; }
    }

    public static void main(String[] args) throws Exception {
        // a server that somebody else started, with three languages
        Process external = new ProcessBuilder("libretranslate", "--port", "5002", "--load-only", "en,fa,de").start();
        while (!listening(5002)) Thread.sleep(200);

        try (Translation t = Translation.load(5002, "en,fa")) {       // subset: reused
            System.out.println(t.translate("Thank you", "en", "fa"));
        }
        System.out.println("server left running: " + listening(5002));

        try {
            Translation.load(5002, "en,fr");                           // fr is not loaded there
        } catch (Exception e) {
            System.out.println(e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        external.destroy();
    }
}
```

```text
متشکرم
server left running: true
TranslationFailedException: A libretranslate server is already running on port 5002 but was not loaded with: fr (it only has: de, en, fa). Stop that server, or use a different port, to start one with the languages you need.
```

- **Reuse.** The server on port `5002` was started by someone else. `load` sees it, checks its languages and returns.
- **A subset is fine.** The program asked for `en,fa`; the server has more.
- **Left running.** After `close`, the server is still up, because this program did not start it.
- **A missing language.** Asking for `fr` fails at `load`, not at the first `translate`. The message names the missing languages and the ones the server has (their order is not fixed).
- **The same check inside one JVM.** A second `load` on a port this program already started is checked in the same way: its languages must be among the ones the server loaded.

## 5. When the start fails

Starting a server can go wrong in three ways, and each has a clear message:

```java
import java.time.Duration;
import org.j2os.platform.ai.dl.language.translator.Translation;

public class Startup {
    interface Step { void run() throws Exception; }

    static void attempt(String label, Step step) {
        try {
            step.run();
            System.out.println(label + " -> ok");
        } catch (Exception e) {
            System.out.println(label + " -> " + e.getClass().getSimpleName() + ": "
                    + e.getMessage().replace(System.getProperty("user.dir"), "."));
        }
    }

    public static void main(String[] args) {
        attempt("no python folder", () -> Translation.load("no-such-python", 5000));
        attempt("server exits", () -> Translation.load(5004, "en,xx"));
        attempt("too slow", () -> Translation.load(5005, "de", Duration.ofSeconds(2), Duration.ofSeconds(5)));
    }
}
```

```text
no python folder -> TranslationFailedException: java.io.FileNotFoundException: libretranslate executable not found under: ./no-such-python (expected Scripts/libretranslate.exe or bin/libretranslate)
server exits -> TranslationFailedException: libretranslate exited before becoming ready (exit code 1):
error: unknown language code 'xx'
too slow -> TranslationFailedException: libretranslate did not become ready within 2 seconds:
```

- **No such Python folder.** The folder holds neither `Scripts/libretranslate.exe` nor `bin/libretranslate`.
- **The server exits.** An unknown language, a missing model or a broken installation may make `libretranslate` stop. The exception carries the exit code and the last lines (up to 50) that the process printed, which is usually the whole explanation.
- **Too slow.** If the server does not answer in time, it is stopped together with the processes it started. The default wait is two minutes, since the first start reads big models.
- **Slow machines.** The long overload sets the wait for the server and the time limit of each request:

```java
Translation translation = Translation.load(
        5000,                        // port
        "en,fa",                     // languages to load
        Duration.ofMinutes(5),       // how long to wait for a starting server
        Duration.ofSeconds(60));     // time limit of every translate call
```

## 6. Putting it together

A page shows labels in two languages. The translations are asked for from several threads, kept so that each text is translated once, and a failed translation falls back to the original text:

```java
import java.util.*;
import java.util.concurrent.*;
import org.j2os.platform.ai.dl.language.translator.Translation;
import org.j2os.platform.ai.dl.language.translator.exception.TranslationFailedException;

public class Workflow {
    static final Map<String, String> cache = new ConcurrentHashMap<>();

    // translate once per (text, direction); on failure keep the original so the page still renders
    static String tr(Translation t, String text, String from, String to) {
        return cache.computeIfAbsent(from + to + text, k -> {
            try {
                return t.translate(text, from, to);
            } catch (TranslationFailedException e) {
                return text + "  [untranslated]";
            }
        });
    }

    public static void main(String[] args) throws Exception {
        try (Translation t = Translation.load(5000, "en,fa")) {
            List<String> labels = List.of("Good morning", "Thank you", "Where is the station?",
                    "The weather is nice today.", "Thank you", "Order #42: \"gift\" card\nline two");
            ExecutorService pool = Executors.newFixedThreadPool(4);
            List<Future<String>> out = new ArrayList<>();
            for (String l : labels) out.add(pool.submit(() -> tr(t, l, "en", "fa")));
            for (int i = 0; i < labels.size(); i++) {
                System.out.println(labels.get(i).replace("\n", "\\n") + "  ->  " + out.get(i).get().replace("\n", "\\n"));
            }
            System.out.println(tr(t, "Thank you", "en", "de"));       // a language the server does not have
            System.out.println("distinct translations kept: " + cache.size());
            pool.shutdown();
        }
    }
}
```

```text
Good morning  ->  صبح بخیر
Thank you  ->  متشکرم
Where is the station?  ->  ایستگاه کجاست؟
The weather is nice today.  ->  امروز هوا خوب است.
Thank you  ->  متشکرم
Order #42: "gift" card\nline two  ->  Order #42: "gift" card\nline two
Thank you  [untranslated]
distinct translations kept: 6
```

- **Many threads, one object.** Four threads share one `Translation`. No lock is needed.
- **A cache.** `Thank you` appears twice in the labels but is translated once.
- **Special characters.** Quotes and line breaks travel to the server and back unharmed.
- **A failure.** The server has no German, so the translation fails with an exception; the program shows the original text marked as untranslated and goes on.
- **Close at the end.** The `try` block stops the server when the work is done. In a long-running application there is no such block; hold one instance and close it when the application stops:

```java
// one instance for the whole application; the server must not outlive it
Translation translation = Translation.load(5000, "en,fa");
Runtime.getRuntime().addShutdownHook(new Thread(translation::close));
```

- **Long texts.** One request carries the whole text. For a long document, translate it paragraph by paragraph so that each request is short and a failure costs only one paragraph:

```java
// translate a long document paragraph by paragraph: short requests, partial results survive a failure
for (String paragraph : document.split("\\n\\s*\\n")) {
    result.add(translation.translate(paragraph, "en", "fa"));
}
```

## Appendix A. Quick reference

### A.1 Loading

| Call | Meaning |
|------|---------|
| `load(port)` | Command `libretranslate` from the `PATH`; languages `en,fa`; defaults for the times |
| `load(port, languages)` | The same, with a language list |
| `load(port, languages, startupTimeout, requestTimeout)` | The same, with both times |
| `load(pythonDirectory, port)` | The server is started from the Python folder; languages `en,fa` |
| `load(pythonDirectory, port, languages)` | The same, with a language list |
| `load(pythonDirectory, port, languages, startupTimeout, requestTimeout)` | The same, with both times |

All of them throw `TranslationFailedException` when the server cannot be started or reached.

### A.2 Using

| Call | Meaning |
|------|---------|
| `translate(text, from, to)` | The translated text. Throws `TranslationFailedException` |
| `close()` | Releases the server. Safe to call twice. Does not throw |

### A.3 Defaults

| Setting | Default |
|---------|---------|
| Languages | `en,fa` |
| Wait for a starting server | 120 seconds |
| Time limit of a request | 30 seconds |
| Readiness check | Every 500 milliseconds |
| Server address | `localhost`, the given port |

## Appendix B. Failure handling

```java
import java.time.Duration;
import org.j2os.platform.ai.dl.language.translator.Translation;

public class Checks {
    interface Step { void run() throws Exception; }

    static void attempt(String label, Step step) {
        try {
            step.run();
            System.out.println(label + " -> ok");
        } catch (Exception e) {
            System.out.println(label + " -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
    }

    public static void main(String[] args) throws Exception {
        Translation t = Translation.load(5000);
        attempt("unknown language", () -> t.translate("Thank you", "en", "de"));
        attempt("null text", () -> t.translate(null, "en", "fa"));
        attempt("zero timeout", () -> Translation.load(5000, "en,fa", Duration.ZERO, Duration.ofSeconds(5)));
        t.close();
        attempt("after close", () -> t.translate("Thank you", "en", "fa"));
    }
}
```

```text
unknown language -> TranslationFailedException: libretranslate returned HTTP 400: {"error": "de is not supported"}
null text -> NullPointerException: text must not be null
zero timeout -> IllegalArgumentException: startupTimeout must be positive, was PT0S
after close -> TranslationFailedException: This Translation instance has been closed
```

| Situation | What happens |
|-----------|--------------|
| `libretranslate` is not installed or not on the `PATH` | `TranslationFailedException` caused by an `IOException` |
| A Python folder without the executable | `TranslationFailedException` caused by a `FileNotFoundException` |
| The server exits while starting | `TranslationFailedException` with the exit code and the last output |
| The server is not ready in time | `TranslationFailedException`; the process is stopped |
| The thread is interrupted while waiting | `TranslationFailedException`; the interrupt flag is kept; the process is stopped |
| A running server lacks a requested language | `TranslationFailedException` at `load` |
| The server answers anything but HTTP 200 | `TranslationFailedException` with the code and the body |
| The server cannot be reached, or the request times out | `TranslationFailedException` caused by the `IOException` |
| The answer has no `translatedText` | `TranslationFailedException` showing the answer |
| `translate` after `close` | `TranslationFailedException` |
| A `null` argument | `NullPointerException` naming the argument |
| A zero or negative time | `IllegalArgumentException` |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Forgetting to close.** The server is a separate process. If the program ends without `close`, the server can stay behind. Close in `finally` or in a shutdown hook.
- **Closing too early.** The last `close` on a port stops the server for every other user in the program. Close an instance only when its user is finished.
- **A language that is not loaded.** It fails at `translate`, or at `load` when the server already runs. Name every language in the list.
- **Two programs, one port.** The second program finds the first one's server and reuses it, with whatever languages it has.
- **Loading per request.** Starting a server takes seconds. Load once and keep the object.
- **A single huge text.** The whole text goes in one request. Split long documents.
- **Trusting the result.** Machine translation can be wrong, especially for names, numbers and short fragments. Show it as a help, not as a fact.

### C.2 Behavior worth knowing

- **Instances are thread-safe.** Calls from many threads work; so does closing from another thread.
- **Per-port control.** Starting and closing on the same port are serialized.
- **A dead server is replaced.** If a server this program started has died, the next `load` on that port starts a new one.
- **Plain text.** The text is sent as plain text, not as HTML; line breaks and quotes are kept.
- **Output is drained.** The server's console output is read continuously, so a chatty server cannot block, and only the last 50 lines are kept for error messages.
- **UTF-8.** The server is started with UTF-8 output, and texts are sent and read as UTF-8.
- **Closing is gentle.** The server and the processes it started are asked to stop.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Server | LibreTranslate on `localhost` |
| Instances per server | Any number; counted |
| Servers per JVM | One per port |
| Request format | Plain text, one text per request |
| Languages | Those the server loaded |
| Retries | None |
| Streaming | None |
| Detecting the source language | None; the caller names it |

### C.4 Tuning and practical advice

- **One server per application.** Load at start-up, share the object, close at shutdown.
- **Start it early.** Load while the application starts, not at the first user request.
- **List only the languages you need.** Each language costs memory and start-up time.
- **Raise the start-up wait on slow machines.** The first run may also download models.
- **Cache.** Translating the same text again gives the same answer; keep it.
- **Detect first.** If the source language is not known, find it before translating; see the `ai.dl.language.nlp` module.
- **Run it as a service for production.** An administrator-run server survives restarts of your application, and `load` will reuse it.

## Appendix D. Message reference

### D.1 Starting

| Message | Exception | Cause |
|---------|-----------|-------|
| `libretranslate executable not found under: <folder> (expected Scripts/libretranslate.exe or bin/libretranslate)` | `TranslationFailedException` | The Python folder has no executable |
| `libretranslate exited before becoming ready (exit code %d):` and the last output | `TranslationFailedException` | The server stopped while starting |
| `libretranslate did not become ready within %d seconds:` and the last output | `TranslationFailedException` | The wait ended |
| `A libretranslate server is already running on port %d but was not loaded with: <codes> (it only has: <codes>). Stop that server, or use a different port, to start one with the languages you need.` | `TranslationFailedException` | A requested language is missing on a running server |
| `Could not verify languages of the already-running server on port %d (HTTP %d)` | `TranslationFailedException` | The language list of the running server could not be read |
| the text of the cause | `TranslationFailedException` | An `IOException` or an interrupt while starting |

### D.2 Translating

| Message | Exception | Cause |
|---------|-----------|-------|
| `libretranslate returned HTTP %d: <body>` | `TranslationFailedException` | The server refused the request, for example an unknown language |
| `Unexpected response from libretranslate: <body>` | `TranslationFailedException` | The answer has no `translatedText` |
| `This Translation instance has been closed` | `TranslationFailedException` | `translate` after `close` |
| the text of the cause | `TranslationFailedException` | A network error, a time-out or an interrupt |

### D.3 Argument checks

| Message | Exception | Cause |
|---------|-----------|-------|
| `<name> must not be null` | `NullPointerException` | `pythonDirectory`, `loadOnlyLanguages`, `startupTimeout`, `requestTimeout`, `text`, `sourceLanguage` or `targetLanguage` is `null` |
| `<name> must be positive, was <duration>` | `IllegalArgumentException` | `startupTimeout` or `requestTimeout` is zero or negative |
