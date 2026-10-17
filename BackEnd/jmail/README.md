# jmail

*J2OS Platform 2.4*

Almost every application sends e-mail sooner or later: an order confirmation, a password reset, a monthly report. Some also read it: a support desk that answers customers, or a tool that files incoming invoices. Both jobs mean talking to a mail provider in its own languages. *SMTP* is the one used to **send** a message. *IMAP* is the one used to **read and organize** a mailbox that stays on the provider's server, with its folders, its unread marks and its attachments.

**jmail** (`org.j2os.platform.jmail`) wraps both in one small client. It is built on Jakarta Mail, the standard Java library for e-mail, but hides its many classes and settings behind a short, fluent API. Two properties set it apart:

- **One client per account.** There is no global configuration. Every caller supplies the address and password of its own mailbox, so an application can send as the shop in one place and read the mailbox of a support agent in another.
- **Small and safe by default.** Connections are encrypted, every wait has a time limit, attachments are read completely before a message is handed over, and a closed client refuses further work instead of misbehaving.

The whole library in a few lines:

```java
try (JMailClient client = JMail.connect()
        .smtpHost("smtp.example.com")
        .imapHost("imap.example.com")
        .username("shop@example.com")
        .password(password)                       // from the environment, never from the source
        .connect()) {

    client.compose()                              // sending
            .to("customer@example.com")
            .subject("Your order")
            .content("<p>Thank you!</p>")
            .send();

    for (MailSummary mail : client.folder("INBOX").list()) {   // reading
        System.out.println(mail.from() + " | " + mail.subject());
    }
}
```

This guide goes step by step: connecting an account (chapter 2), sending (3 to 7), reading and organizing a mailbox (8 to 12), the life of a client (13) and a complete small workflow (14). The appendices work as a reference.

## Contents

1. [How jmail works](#1-how-jmail-works)
2. [Connecting an account](#2-connecting-an-account)
3. [Sending a message](#3-sending-a-message)
4. [Templates and variables](#4-templates-and-variables)
5. [Attachments](#5-attachments)
6. [Sending in the background](#6-sending-in-the-background)
7. [When sending fails](#7-when-sending-fails)
8. [Folders and the list of messages](#8-folders-and-the-list-of-messages)
9. [Reading a message](#9-reading-a-message)
10. [Saving attachments](#10-saving-attachments)
11. [Read and unread](#11-read-and-unread)
12. [Deleting, copying and moving](#12-deleting-copying-and-moving)
13. [Lifetime, connections and threads](#13-lifetime-connections-and-threads)
14. [Putting it together: a small support desk](#14-putting-it-together-a-small-support-desk)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The programs of the guide are complete: each has one `main` method. They use four accounts, `sara`, `ali`, `reza` and `omid`, at `example.com` (mail servers `smtp.example.com` and `imap.example.com`), all with the password `secret`; replace the hosts and credentials by those of a real account. Most programs end with a small method `open(user)` that builds a client for one of the four. A program that reads mail says in a comment which messages the inbox must already hold.

---

## 1. How jmail works

First, a few terms, because they come up throughout the guide:

- **SMTP** is the protocol for sending. The client connects to the provider's SMTP server, logs in, hands over one message and disconnects.
- **IMAP** is the protocol for reading. The client logs in once and keeps the connection open while it lists folders, reads messages and changes them. Nothing is downloaded for good: the messages stay on the server.
- A **folder** is a named box inside a mailbox: `INBOX` for new mail, and others such as `Sent`, `Archive` or any the user creates. The names beyond `INBOX` depend on the provider.
- A message in a folder has a number, its **id** (its *UID* in IMAP terms). The id identifies the message inside that one folder. The same message has another id in another folder.
- **TLS** is the encryption of a connection. *Implicit TLS* means the connection is encrypted from the first byte (port 465 for SMTP, 993 for IMAP). *STARTTLS* means it begins in the clear and switches to encryption before anything secret is sent (port 587 for SMTP).

The classes a program meets:

| Class | Role |
|-------|------|
| `JMail` | The only door in: `JMail.connect()` starts a builder |
| `JMailClient.Builder` | Collects the host names, ports, login and time limits, then `connect()` |
| `JMailClient` | One account. Sends mail (`compose()`), lists and manages folders |
| `JMailClient.Compose` | One outgoing message under construction |
| `JMailClient.MailboxFolder` | One folder: list, read, mark, copy, move, delete |
| `MailSummary` | A line of a folder listing: id, subject, sender |
| `MailMessage` | A message read in full: the text and the attachments |
| `MailAttachment` | One attachment with its bytes |

Keep these two points about connections in mind:

- **SMTP is opened per message.** Building a client opens nothing. Each `send()` connects, sends and disconnects by itself.
- **IMAP is opened on first use.** The first call that touches the mailbox (`folder(...)`, `folders()`, `createFolder(...)`, `removeFolder(...)`) logs in; after that the connection is kept. If the server hangs up later, the next call logs in again (chapter 13).

All failures are unchecked exceptions (`RuntimeException` and its subclasses). Appendix B lists what each situation produces.

## 2. Connecting an account

A client is built from a few settings. Only the SMTP host, the user name and the password are required:

| Setting | Default | Meaning |
|---------|---------|---------|
| `smtpHost(host)` | none, required | The SMTP server, for example `smtp.example.com` |
| `smtpPort(port)` | 587 | 465 means implicit TLS, any other port means STARTTLS |
| `imapHost(host)` | none, optional | The IMAP server. Needed only for mailbox work |
| `imapPort(port)` | 993 | IMAP is always encrypted from the first byte |
| `username(name)` | none, required | The login **and** the sender address of every message |
| `password(secret)` | none, required | The password, used for SMTP and IMAP |
| `connectTimeout(ms)` | 30000 | How long to wait for a connection to open |
| `readTimeout(ms)` | 120000 | How long to wait for the server during a read or a write |

Some of these deserve a closer look:

- **The user name is also the `From` address.** Most providers log in with the full address, and jmail puts the same text in the `From` line. A user name that is not an e-mail address makes every send fail (chapter 7).
- **Encryption is not optional.** Port 465 gives implicit TLS; every other SMTP port requires STARTTLS, and a server that does not offer it is refused. IMAP always uses implicit TLS. A provider that only speaks unencrypted mail cannot be used.
- **Login is always on.** There is no anonymous SMTP.
- **The timeouts protect the caller.** A server that never answers ends the call with an exception after the time limit. A value of `0` means "wait forever", which lets one dead server block a thread for good, so it is best left alone.

The password should come from the environment or a secrets store, and never from a string in the source:

```java
String password = System.getenv("MAIL_PASSWORD");   // or a secrets store; never a string in the source
JMailClient client = JMail.connect()
        .smtpHost("smtp.example.com")
        .username("shop@example.com")
        .password(password)
        .connect();
```

Providers with two-step verification, Gmail among them, often refuse the normal account password for programs and ask for an *app password* created in the account settings. Use that as the `password`.

`connect()` checks the settings and builds the client. It **opens no connection**, so a wrong host or password is discovered at the first `send()` or the first mailbox call, not here.

**The accounts used in the examples.** Every program builds its client with the builder above and the same values: `smtp.example.com`, `imap.example.com`, the user `sara@example.com` (or `ali`, `reza`, `omid`) and the password `secret`. Programs that need several users end with a method `open(user)` that holds the builder, so the main code stays short.

```java
import org.j2os.platform.jmail.JMail;
import org.j2os.platform.jmail.JMailClient;

public class Setup {
    public static void main(String[] args) {
        System.out.println("defaults: connect " + JMailClient.DEFAULT_CONNECT_TIMEOUT_MILLIS
                + " ms, read " + JMailClient.DEFAULT_READ_TIMEOUT_MILLIS + " ms");

        try {
            JMail.connect().username("sara@example.com").password("x").connect();
        } catch (Exception e) {
            System.out.println("1 no smtpHost -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            JMail.connect().smtpHost("smtp.example.com").username("sara@example.com").connect();
        } catch (Exception e) {
            System.out.println("2 no password -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            JMail.connect().smtpHost(" ");
        } catch (Exception e) {
            System.out.println("3 blank host -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            JMail.connect().smtpPort(0);
        } catch (Exception e) {
            System.out.println("4 port 0 -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            JMail.connect().imapPort(70000);
        } catch (Exception e) {
            System.out.println("5 port 70000 -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            JMail.connect().readTimeout(-1);
        } catch (Exception e) {
            System.out.println("6 negative timeout -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            JMail.connect().connectTimeout(5_000_000_000L);
        } catch (Exception e) {
            System.out.println("7 huge timeout -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }

        // connect() opens nothing: a host that does not exist is accepted
        try {
            JMail.connect().smtpHost("nowhere.invalid").username("sara@example.com").password("x").connect().close();
            System.out.println("8 unknown host -> ok");
        } catch (Exception e) {
            System.out.println("8 unknown host -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }

        // the IMAP host is only needed for mailbox work
        try (JMailClient client = JMail.connect().smtpHost("smtp.example.com")
                .username("sara@example.com").password("secret").connect()) {
            try {
                client.compose();
                System.out.println("9 compose without imapHost -> ok");
            } catch (Exception e) {
                System.out.println("9 compose without imapHost -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
            }
            try {
                client.folder("INBOX");
            } catch (Exception e) {
                System.out.println("10 folder without imapHost -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
            }
        }
    }
}
```

```text
defaults: connect 30000 ms, read 120000 ms
1 no smtpHost -> IllegalStateException: smtpHost is required
2 no password -> IllegalStateException: username and password are required
3 blank host -> IllegalArgumentException: smtpHost cannot be blank
4 port 0 -> IllegalArgumentException: smtpPort must be between 1 and 65535: 0
5 port 70000 -> IllegalArgumentException: imapPort must be between 1 and 65535: 70000
6 negative timeout -> IllegalArgumentException: readTimeout must be between 0 and 2147483647 milliseconds: -1
7 huge timeout -> IllegalArgumentException: connectTimeout must be between 0 and 2147483647 milliseconds: 5000000000
8 unknown host -> ok
9 compose without imapHost -> ok
10 folder without imapHost -> IllegalStateException: imapHost is not set — required for mailbox operations
```

- Cases 1 and 2: a missing SMTP host, or a missing user name or password, is `IllegalStateException`.
- Cases 3 to 7: values that cannot be right are refused the moment they are given: a blank host, a port outside 1 to 65535, a negative timeout, a timeout too large for the underlying library.
- Case 8: a host that does not exist is accepted, because nothing is opened.
- Cases 9 and 10: without an IMAP host a client can send, but the first mailbox call fails with `IllegalStateException`.

Close a client when it is no longer needed. It is `AutoCloseable`, so `try-with-resources` is the natural way. For a client that only sends, closing releases nothing important; for one that reads, it ends the IMAP connection (chapter 13).

## 3. Sending a message

`compose()` starts a message. The methods can be chained in any order and end with `send()`:

| Method | Does |
|--------|------|
| `to(address...)`, `cc(address...)`, `bcc(address...)` | Add recipients. Each call adds to the earlier ones |
| `subject(text)` | The subject line. Any language; `null` is allowed |
| `content(html)` | The body, as HTML (see below) |
| `templateContent(path)`, `variable(name, value)` | The body from a template file (chapter 4) |
| `attach(file...)` | Attach files (chapter 5) |
| `send()` | Send now and wait for the result |
| `send(callback)` | Send in the background (chapter 6) |

```java
import org.j2os.platform.jmail.JMail;
import org.j2os.platform.jmail.JMailClient;
import org.j2os.platform.jmail.MailSummary;

public class Send {
    public static void main(String[] args) throws Exception {
        try (JMailClient client = open("sara")) {
            client.compose()
                    .to("ali@example.com")
                    .cc("reza@example.com")
                    .bcc("omid@example.com")
                    .subject("Meeting at ten")
                    .content("<p>See you in <b>room 4</b>.</p>")
                    .send();                       // returns when the provider has taken it
        }

        for (String user : new String[]{"ali", "reza", "omid", "sara"}) {
            try (JMailClient client = open(user)) {
                System.out.print(user + "'s inbox:");
                for (MailSummary summary : client.folder("INBOX").list()) {
                    System.out.print("  " + summary.id() + "=" + summary.subject());
                }
                System.out.println();
            }
        }
    }

    static JMailClient open(String user) {
        return JMail.connect()
                .smtpHost("smtp.example.com")
                .imapHost("imap.example.com")
                .username(user + "@example.com")
                .password("secret")
                .connect();
    }
}
```

```text
ali's inbox:  1=Meeting at ten
reza's inbox:  1=Meeting at ten
omid's inbox:  1=Meeting at ten
sara's inbox:
```

- Ali (To), Reza (Cc) and Omid (Bcc) each received the message. Nobody is told about Omid, because a blind copy appears in no recipient's headers.
- Sara's own inbox stays empty. **jmail does not file a copy in a "Sent" folder.** Some providers do it themselves for mail sent over SMTP, others do not, so an application that needs a record of what it sent must keep one.
- `send()` returns when the provider has **accepted** the message. It does not promise delivery to the reader: a wrong but well-formed address may come back later as a bounce message in the inbox.

**The body is always HTML.** Whatever the text, jmail sends it as `text/html` in UTF-8. This has consequences:

- Persian, Arabic and any other language work in the subject and the body without extra steps.
- A line break in the text is only a space for the reader's program. Use `<br>` or `<p>` where a break is wanted.
- A `<` or an `&` in text meant to be shown literally must be written as `&lt;` and `&amp;`.
- There is no plain-text alternative. A message sent with `content("Hello")` is still an HTML message.

Each recipient is checked for syntax when the message is built: it must look like an address (`name@domain`, optionally `Name <name@domain>`). Whether the mailbox exists is for the provider to say. A blank address is refused at once with `IllegalArgumentException`.

One `Compose` can be sent again. Each `send()` builds a new message from what the object holds at that moment, so a program can change the subject and send to the next person.

## 4. Templates and variables

A message that repeats with different details (an order confirmation, a reminder) is better kept as a file than assembled from strings. `templateContent(path)` reads an HTML file from the **classpath** (in a Maven or Gradle project, the files under `src/main/resources`) as UTF-8, and `variable(name, value)` fills the gaps. A gap is written `${name}`. The template used by the examples, `mail/order.html`, is:

```html
<h2>Order ${orderId}</h2>
<p>Dear ${customerName}, your total is ${total}.</p>
```

```java
import org.j2os.platform.jmail.JMail;
import org.j2os.platform.jmail.JMailClient;

public class Templates {
    public static void main(String[] args) throws Exception {
        try (JMailClient sara = open("sara"); JMailClient ali = open("ali")) {
            // mail/order.html says: Order ${orderId} ... Dear ${customerName} ... ${total}
            sara.compose().to("ali@example.com").subject("Order")
                    .templateContent("mail/order.html")
                    .variable("orderId", "12345")
                    .variable("customerName", "Ali")
                    .variable("total", "$45.00")        // a $ in a value is safe
                    .variable("unused", "never shown")   // extra variables are ignored
                    .send();
            System.out.println(ali.folder("INBOX").read(1).content());

            try {
                sara.compose().to("ali@example.com")
                        .templateContent("mail/order.html").variable("orderId", "7").send();
            } catch (Exception e) {
                System.out.println("missing variable -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
            }
            try {
                sara.compose().templateContent("mail/nope.html");
            } catch (Exception e) {
                System.out.println("missing template -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
            }
            try {
                sara.compose().variable("x", null);
            } catch (Exception e) {
                System.out.println("null value -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
            }

            // a value goes in exactly as given
            sara.compose().to("ali@example.com").subject("Order")
                    .templateContent("mail/order.html")
                    .variable("orderId", "8")
                    .variable("customerName", "<script>alert(1)</script>")
                    .variable("total", "$1")
                    .send();
            System.out.println(ali.folder("INBOX").read(2).content());

            // the last of content(...) and templateContent(...) wins, and variable(...) then does nothing
            sara.compose().to("ali@example.com").subject("Plain")
                    .templateContent("mail/order.html").variable("orderId", "9")
                    .content("Just a line, ${orderId} stays as it is")
                    .send();
            System.out.println(ali.folder("INBOX").read(3).content());

            // only letters, digits and _ are placeholder names
            sara.compose().to("ali@example.com").subject("Dots")
                    .content("Hello ${user.name}").send();
            System.out.println(ali.folder("INBOX").read(4).content());
        }
    }

    static JMailClient open(String user) {
        return JMail.connect()
                .smtpHost("smtp.example.com")
                .imapHost("imap.example.com")
                .username(user + "@example.com")
                .password("secret")
                .connect();
    }
}
```

```text
<h2>Order 12345</h2>
<p>Dear Ali, your total is $45.00.</p>

missing variable -> IllegalStateException: Template variable 'customerName' has not been set
missing template -> IllegalArgumentException: Template not found on classpath: mail/nope.html
null value -> NullPointerException: Value of variable 'x' cannot be null
<h2>Order 8</h2>
<p>Dear <script>alert(1)</script>, your total is $1.</p>

Just a line, ${orderId} stays as it is
Hello ${user.name}
```

- The first message shows the result. A `$` inside a value (`$45.00`) is safe; values are inserted as plain text.
- A variable the template asks for but the program never set is `IllegalStateException`, naming the variable. It surfaces when the message is built, which for `send()` means the call itself. An extra variable that no placeholder uses is ignored.
- A template that does not exist is `IllegalArgumentException`, raised by `templateContent` itself, so a mistyped path is found early. A `null` value is refused at once.
- **A value goes in exactly as given.** The `<script>` text in the second message reached the body untouched. When a value comes from a person (a name typed into a form), escape it first, or one customer can put markup into mail that carries the shop's name:

```java
import org.springframework.web.util.HtmlUtils;

client.compose()
        .to(order.customerEmail())
        .subject("Order " + order.id())
        .templateContent("mail/order.html")
        .variable("orderId", String.valueOf(order.id()))
        .variable("customerName", HtmlUtils.htmlEscape(order.customerName()))   // the name came from a form
        .variable("total", order.total())
        .send();
```

- `content(...)` and `templateContent(...)` replace each other: the one called last decides, and `variable(...)` has then no effect. With `content(...)` nothing is substituted, so `${orderId}` stays as written (third message).
- Only letters, digits and `_` make a placeholder name. A gap such as `${user.name}` is not recognized and stays in the text (fourth message). If a template includes CSS or script that contains `${...}`, rename those or keep them out of the file.

The template is read when `templateContent` is called, not when the message is sent.

## 5. Attachments

`attach(file...)` attaches one or more files. It can be called several times; the files add up.

```java
import org.j2os.platform.jmail.JMail;
import org.j2os.platform.jmail.JMailClient;
import org.j2os.platform.jmail.MailAttachment;

import java.io.File;
import java.nio.file.Files;

public class Attach {
    public static void main(String[] args) throws Exception {
        File dir = Files.createTempDirectory("jmail-demo").toFile();
        File report = new File(dir, "report.txt");
        File data = new File(dir, "data.bin");
        Files.writeString(report.toPath(), "Total: 42\n");
        Files.write(data.toPath(), new byte[]{0, 1, 2, (byte) 255});

        try (JMailClient sara = open("sara"); JMailClient ali = open("ali")) {
            sara.compose().to("ali@example.com").subject("Files")
                    .content("<p>Attached.</p>")
                    .attach(report, data)
                    .send();

            for (MailAttachment file : ali.folder("INBOX").read(1).attachments()) {
                System.out.println(file.name() + " | " + file.size() + " bytes | contentId=" + file.contentId());
            }

            try {
                sara.compose().attach(new File("nope.pdf"));
            } catch (Exception e) {
                System.out.println("missing file -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
            }
            try {
                sara.compose().attach(new File("."));
            } catch (Exception e) {
                System.out.println("a folder -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
            }

            // the file is checked when attach() is called, and read when the message is built
            var late = sara.compose().to("ali@example.com").subject("Late").content("x").attach(report);
            report.delete();
            try {
                late.send();
            } catch (Exception e) {
                System.out.println("deleted before send -> " + e.getClass().getSimpleName() + ": " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
            }
        }
    }

    static JMailClient open(String user) {
        return JMail.connect()
                .smtpHost("smtp.example.com")
                .imapHost("imap.example.com")
                .username(user + "@example.com")
                .password("secret")
                .connect();
    }
}
```

```text
report.txt | 10 bytes | contentId=null
data.bin | 4 bytes | contentId=null
missing file -> IllegalArgumentException: Invalid attachment: nope.pdf
a folder -> IllegalArgumentException: Invalid attachment: .
deleted before send -> RuntimeException: Failed to build the message  (cause: NoSuchFileException)
```

- The receiver sees each file by its name, with its exact size in bytes.
- A path that is not an existing regular file (a missing file, a directory) is refused by `attach` itself with `IllegalArgumentException`.
- The file is **read when the message is built**: in `send()` at the call, in `send(callback)` before the call returns (chapter 6). The file may therefore be changed or deleted right after an asynchronous `send`, but a file deleted between `attach` and a synchronous `send()` ends in `RuntimeException` ("Failed to build the message", caused by the missing file), as the last line shows.
- The name shown to the receiver is the file's own name. The type of the content is guessed from it by the operating system and falls back to a generic binary type. Prefer plain ASCII file names: a name in another script may not arrive intact in every mail program.
- The whole file is held in memory while the message is built, and providers refuse large messages, commonly somewhere around 20 to 25 MB for everything together. For big files, send a download link instead.
- Only real files can be attached. An image meant to appear **inside** the HTML cannot be embedded with `cid:` when sending; use an image hosted at an `https` address.

## 6. Sending in the background

`send()` blocks until the provider answers, which for a real server can take a noticeable moment and, in a bad case, up to the read timeout. A web request should not wait for that. `send(callback)` hands the message to a worker and returns at once:

```java
import org.j2os.platform.jmail.JMail;
import org.j2os.platform.jmail.JMailClient;

import java.io.File;
import java.nio.file.Files;
import java.util.concurrent.CountDownLatch;

public class Async {
    public static void main(String[] args) throws Exception {
        File note = File.createTempFile("note", ".txt");
        Files.writeString(note.toPath(), "hello");

        JMailClient sara = open("sara");

        CountDownLatch done = new CountDownLatch(1);
        sara.compose().to("ali@example.com").subject("Slow").content("x").attach(note)
                .send(error -> {
                    System.out.println("callback: error=" + error + ", thread=" + Thread.currentThread().getName());
                    done.countDown();
                });
        note.delete();                                     // too late to matter: the file was read before send() returned
        done.await();

        // a build failure is also reported on a worker thread, not thrown
        CountDownLatch failed = new CountDownLatch(1);
        sara.compose().to("ali@example.com").templateContent("mail/order.html")
                .send(error -> {
                    System.out.println("callback: " + error.getClass().getSimpleName() + ": " + error.getMessage()
                            + ", thread=" + Thread.currentThread().getName());
                    failed.countDown();
                });
        failed.await();

        // closing the client does not cancel a send that was already submitted
        CountDownLatch last = new CountDownLatch(1);
        sara.compose().to("omid@example.com").subject("Closing").content("x")
                .send(error -> { System.out.println("after close: error=" + error); last.countDown(); });
        sara.close();
        last.await();
        try {
            sara.compose();
        } catch (Exception e) {
            System.out.println("compose on a closed client -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
    }

    static JMailClient open(String user) {
        return JMail.connect()
                .smtpHost("smtp.example.com")
                .imapHost("imap.example.com")
                .username(user + "@example.com")
                .password("secret")
                .connect();
    }
}
```

```text
callback: error=null, thread=jmail-send
callback: IllegalStateException: Template variable 'orderId' has not been set, thread=jmail-send
after close: error=null
compose on a closed client -> IllegalStateException: JMailClient has already been closed
```

- **Everything is prepared before the call returns.** The template is resolved and the attachments are read. That is why deleting the file right after `send(callback)` does no harm.
- **The callback runs exactly once, on a worker thread** named `jmail-send`, never on the calling thread. It receives `null` on success or the failure. A callback that throws is not called a second time, and its exception is not caught by jmail.
- **Preparation failures come to the callback too.** A missing template variable does not throw from `send(callback)`; it is passed to the callback, again on a worker thread. Only two problems are thrown at the call: a `null` callback (`NullPointerException`) and a closed client (`IllegalStateException`).
- **The workers are shared by every client of the JVM:** at most 16 sends run at the same time, and at most 1000 wait for a free worker. A send beyond that is refused by `RejectedExecutionException` and its callback is **not** called.

```java
try {
    client.compose().to(address).subject(subject).content(html).send(error -> {
        if (error != null) {
            log.warn("Mail to {} failed", address, error);      // runs on a "jmail-send" thread
        }
    });
} catch (RejectedExecutionException busy) {
    // 16 sends are running and 1000 are waiting: keep the mail somewhere and try again later
}
```

- **Closing the client does not cancel a submitted send.** In the last send, the result is still a success even though the client is closed right after the call.
- The workers are daemon threads. A short program whose `main` returns right after `send(callback)` may end before the message leaves, so wait for the callback (as the programs do with a `CountDownLatch`). In a long-running server this does not matter.
- Several asynchronous sends run in parallel, so they may reach the provider in a different order than they were submitted. There is no automatic retry; a callback that receives an error decides what to do.

## 7. When sending fails

Failures fall into three groups, each recognizable by the exception:

| Group | Exception | Examples |
|-------|-----------|----------|
| The program passed something wrong | `IllegalArgumentException`, `NullPointerException` | A blank address, a missing file, an unknown template |
| The client cannot do it now | `IllegalStateException` | Closed client, missing template variable, missing IMAP host |
| The message or the network went wrong | `RuntimeException` with a message of jmail and the original Jakarta Mail exception as `getCause()` | Bad address syntax, refused recipient, wrong password, unreachable server |

For the third group the message says where it went wrong: `Failed to build the message` (the message could not be put together: an address with bad syntax, an unreadable attachment) or `Failed to send email` (the provider or the network refused).

```java
import org.j2os.platform.jmail.JMail;
import org.j2os.platform.jmail.JMailClient;

public class SendFailures {
    public static void main(String[] args) {
        try (JMailClient sara = open("sara", "secret")) {
            try {
                sara.compose().to(" ");
            } catch (Exception e) {
                System.out.println("1 blank address -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
            }
            try {
                sara.compose().to("not-an-address").subject("x").content("x").send();
            } catch (Exception e) {
                System.out.println("2 bad address -> " + e.getClass().getSimpleName() + ": " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
            }
            try {
                sara.compose().subject("x").content("x").send();
            } catch (Exception e) {
                System.out.println("3 no recipient -> " + e.getClass().getSimpleName() + ": " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
            }
            try {
                sara.compose().to("nobody@example.com").subject("x").content("x").send();
            } catch (Exception e) {
                System.out.println("4 unknown user -> " + e.getClass().getSimpleName() + ": " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
            }
            try {
                sara.compose().to("ali@example.com", "nobody@example.com").subject("x").content("x").send();
            } catch (Exception e) {
                System.out.println("5 one good, one unknown -> " + e.getClass().getSimpleName() + ": " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
            }
        }

        try (JMailClient client = open("sara", "wrong")) {
            client.compose().to("ali@example.com").subject("x").content("x").send();
        } catch (Exception e) {
            System.out.println("6 wrong password -> " + e.getClass().getSimpleName() + ": " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }
        try (JMailClient client = JMail.connect().smtpHost("smtp.example.com").smtpPort(2525)
                .username("sara@example.com").password("secret").connect()) {
            client.compose().to("ali@example.com").subject("x").content("x").send();
        } catch (Exception e) {
            System.out.println("7 nothing listens there -> " + e.getClass().getSimpleName() + ": " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }
        try (JMailClient client = JMail.connect().smtpHost("smtp.example.com")
                .username("sara").password("secret").connect()) {
            client.compose().to("ali@example.com").subject("x").content("x").send();
        } catch (Exception e) {
            System.out.println("8 username is not an address -> " + e.getClass().getSimpleName() + ": " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }
    }

    static JMailClient open(String user, String password) {
        return JMail.connect()
                .smtpHost("smtp.example.com")
                .username(user + "@example.com")
                .password(password)
                .connect();
    }
}
```

```text
1 blank address -> IllegalArgumentException: Email address cannot be null or blank
2 bad address -> RuntimeException: Failed to build the message  (cause: AddressException)
3 no recipient -> RuntimeException: Failed to send email  (cause: SendFailedException)
4 unknown user -> RuntimeException: Failed to send email  (cause: SendFailedException)
5 one good, one unknown -> RuntimeException: Failed to send email  (cause: SendFailedException)
6 wrong password -> RuntimeException: Failed to send email  (cause: AuthenticationFailedException)
7 nothing listens there -> RuntimeException: Failed to send email  (cause: MessagingException)
8 username is not an address -> RuntimeException: Failed to build the message  (cause: AddressException)
```

- Case 1 is refused at once; case 2 is only found when the message is built, because the syntax check belongs there.
- Cases 3 and 4: no recipient at all, or one the provider does not know, come back as `SendFailedException`.
- Case 5: **one refused recipient fails the whole send.** Ali is valid, yet nothing reached him (the final listing is empty). Check addresses before sending to a list, or send one message per person.
- Case 6: a wrong password is `AuthenticationFailedException`. The cause of a real provider often says more, for example that an app password is needed.
- Case 7: a host or port where nothing listens is a plain `MessagingException` as the cause.
- Case 8: a user name that is not a full address cannot be used as the sender, so even valid recipients fail ("Failed to build the message").

The cause gives the details when a program has to react:

```java
try {
    client.compose().to("ali@example.com", "typo@example.cmo").subject("Hi").content("<p>Hi</p>").send();
} catch (RuntimeException e) {
    if (e.getCause() instanceof AuthenticationFailedException) {
        // the account or password is wrong
    } else if (e.getCause() instanceof SendFailedException refused) {
        for (Address address : refused.getInvalidAddresses()) {   // may be null when the provider names none
            System.out.println("refused: " + address);
        }
    }
}
```

For an asynchronous send the same exception arrives in the callback, wrapped the same way.

## 8. Folders and the list of messages

Reading starts with the folders. `folders()` returns the full names of all of them, in the order the server gives:

```java
import org.j2os.platform.jmail.JMail;
import org.j2os.platform.jmail.JMailClient;
import org.j2os.platform.jmail.MailSummary;

public class Folders {
    public static void main(String[] args) throws Exception {
        // sara's inbox already holds three messages: "Lunch?" from Ali <ali@example.com>,
        // "Report" from reza@example.com and "سلام" from omid@example.com
        try (JMailClient client = open("sara")) {
            System.out.println("folders: " + client.folders());

            for (MailSummary summary : client.folder("inbox").list()) {   // any spelling of INBOX
                System.out.println(summary.id() + " | " + summary.from() + " | " + summary.subject());
            }

            client.createFolder("Projects");
            client.createFolder("Projects");                              // already there: nothing happens
            System.out.println("folders: " + client.folders());
            System.out.println("Projects holds " + client.folder("Projects").list().size());

            var ghost = client.folder("Nope");                            // no check yet
            try {
                ghost.list();
            } catch (Exception e) {
                System.out.println("list a missing folder -> " + e.getClass().getSimpleName() + ": " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
            }
            try {
                client.folder(" ");
            } catch (Exception e) {
                System.out.println("blank folder name -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
            }

            client.removeFolder("Projects");
            client.removeFolder("Projects");                              // gone already: nothing happens
            System.out.println("folders: " + client.folders());
        }
    }

    static JMailClient open(String user) {
        return JMail.connect()
                .smtpHost("smtp.example.com")
                .imapHost("imap.example.com")
                .username(user + "@example.com")
                .password("secret")
                .connect();
    }
}
```

```text
folders: [INBOX]
1 | Ali <ali@example.com> | Lunch?
2 | reza@example.com | Report
3 | omid@example.com | سلام
folders: [INBOX, Projects]
Projects holds 0
list a missing folder -> RuntimeException: Failed to open folder 'Nope'  (cause: MessagingException)
blank folder name -> IllegalArgumentException: Folder name cannot be null or blank
folders: [INBOX]
```

- `folders()` is how names beyond `INBOX` are discovered. They depend on the provider and often on the language of the account (a Gmail account has `[Gmail]/Sent Mail`, for example), so never assume them.
- `folder(name)` returns a handle. It logs in to IMAP if that has not happened yet, but it does not check the folder. Any spelling of `INBOX` is accepted (`inbox`, `Inbox`); every other name is used exactly as given.
- The handle is cheap, so there is no need to keep it. A name that does not exist fails at the first operation with `RuntimeException` ("Failed to open folder ..."). A blank name is `IllegalArgumentException`.
- `list()` returns one `MailSummary` per message: the `id`, the `subject` (possibly `null`) and the `from`, which is the first sender as text, display name included (`Ali <ali@example.com>`). Only these three are fetched, in one round trip, never the bodies.
- **`list()` has no paging.** It returns every message of the folder, in the order of the server, which is usually oldest first. For a folder with many thousands of messages that is heavy; a mailbox that is read often should be kept small, for example by moving processed mail to an archive folder (chapter 12). For a newest-first screen, reverse the list.
- `createFolder` does nothing when the folder already exists. `removeFolder` deletes the folder **with all the messages in it**, and does nothing when the folder is already gone.

## 9. Reading a message

`read(id)` returns the whole message as a `MailMessage`: its text in `content()` and its attachments in `attachments()`.

The program expects sara's inbox to hold the seven messages below, as other people's mail programs would have left them:

| Id | Subject | Shape |
|----|---------|-------|
| 1 | plain text only | one `text/plain` part |
| 2 | plain and html | a plain version and an HTML version of the same text |
| 3 | body, file and inline image | plain and HTML text (the HTML shows `<img src="cid:logo">`), the file `report.pdf`, and an inline image with the content id `logo` |
| 4 | part with no name at all | a text part and an image part that has neither a name nor a disposition |
| 5 | two text parts | two `text/plain` parts, the second one a list footer |
| 6 | only an attachment | no text, only the file `scan.pdf` |
| 7 | a text file attached | a text part and a second `text/plain` part named `notes.txt` |

```java
import org.j2os.platform.jmail.JMail;
import org.j2os.platform.jmail.JMailClient;
import org.j2os.platform.jmail.MailAttachment;
import org.j2os.platform.jmail.MailMessage;
import org.j2os.platform.jmail.MailSummary;

public class Reading {
    public static void main(String[] args) throws Exception {
        // sara's inbox already holds seven messages of different shapes (see the list in the text)
        try (JMailClient client = JMail.connect()
                .smtpHost("smtp.example.com").imapHost("imap.example.com")
                .username("sara@example.com").password("secret").connect()) {
            for (MailSummary summary : client.folder("INBOX").list()) {
                MailMessage message = client.folder("INBOX").read(summary.id());
                String body = message.content() == null ? "null"
                        : "\"" + message.content().replace("\n", "\\n") + "\"";
                System.out.println(summary.id() + " " + summary.subject());
                System.out.println("   content: " + body);
                for (MailAttachment file : message.attachments()) {
                    System.out.println("   attachment: name=" + file.name() + ", contentId=" + file.contentId()
                            + ", " + file.size() + " bytes");
                }
            }
            try {
                client.folder("INBOX").read(99);
            } catch (Exception e) {
                System.out.println("unknown id -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
            }
        }
    }
}
```

```text
1 plain text only
   content: "Just words."
2 plain and html
   content: "<p>Hello</p>"
3 body, file and inline image
   content: "<img src="cid:logo">"
   attachment: name=report.pdf, contentId=null, 7 bytes
   attachment: name=null, contentId=logo, 3 bytes
4 part with no name at all
   content: "See below"
   attachment: name=null, contentId=null, 3 bytes
5 two text parts
   content: "The message.\n-- list footer"
6 only an attachment
   content: null
   attachment: name=scan.pdf, contentId=null, 7 bytes
7 a text file attached
   content: "Notes are attached."
   attachment: name=notes.txt, contentId=null, 8 bytes
unknown id -> IllegalArgumentException: No message found with id=99
```

The rules, as the seven messages show them:

1. **A single text part** is the content (message 1).
2. **HTML wins over plain text.** When a message carries both versions, `content()` is the HTML one (messages 2 and 3). A program that wants the plain text of an HTML message has to strip the markup itself.
3. **Several text parts of one kind are joined** with a line break, so a footer added by a mailing list is not lost (message 5).
4. **Anything that is not text is an attachment**, even when the sender gave it no name or no disposition (message 4). Its `name()` is then `null`.
5. **A text part with a file name is an attachment**, not the body (message 7).
6. **An inline image** (the HTML says `<img src="cid:logo">`) is an attachment whose `contentId()` is `logo`, the text after `cid:` (message 3). To show it, replace the `cid:` address in the HTML by something the browser can load.
7. **`content()` is `null` when the message has no text at all** (message 6). `attachments()` is never `null`; it is an empty list when there is nothing attached.
8. An unknown id is `IllegalArgumentException`.

Two warnings for programs that show the content:

- **The HTML comes from outside.** Never put it into a page unfiltered; a message can carry scripts. Use an HTML sanitizer, or show it in a sandboxed frame.
- **Attachments are read completely.** A message with a large attachment takes time and memory, and holds the client busy meanwhile (chapter 13). The attachments stay usable after the folder or the client is closed.

## 10. Saving attachments

`MailAttachment` has four methods: `name()`, `contentId()`, `size()` (the exact number of bytes) and `download()` (the bytes, already decoded). The array is the object's own, not a copy.

The **name is chosen by the sender**. It can be `null`, it can repeat inside one message, and it can contain folder parts such as `../../x`. A program that writes attachments to disk should never use it as a path unchecked:

```java
String name = attachment.name() == null ? "attachment" : attachment.name();
String safe = Path.of(name).getFileName().toString();          // drops any folders the sender put in the name
Files.write(targetFolder.resolve(safe), attachment.download());
```

`Path.of(name).getFileName()` keeps only the last part. Add a counter or the message id when two attachments share a name, and look at `size()` before writing if disk space matters.

## 11. Read and unread

IMAP keeps a *seen* mark on every message. jmail exposes it with three methods of the folder:

```java
import org.j2os.platform.jmail.JMail;
import org.j2os.platform.jmail.JMailClient;

public class Flags {
    public static void main(String[] args) throws Exception {
        // sara's inbox already holds three unread messages from ali: One, Two, Three (ids 1 to 3)
        try (JMailClient client = open("sara")) {
            var inbox = client.folder("INBOX");
            System.out.println("unread: " + inbox.unreadCount());

            inbox.markAsRead(2);
            System.out.println("after markAsRead(2): " + inbox.unreadCount());

            inbox.markAsUnread(2);
            System.out.println("after markAsUnread(2): " + inbox.unreadCount());

            inbox.markAsRead(1);
            inbox.markAsRead(1);                       // doing it twice changes nothing
            System.out.println("after markAsRead(1) twice: " + inbox.unreadCount());

            try {
                inbox.markAsRead(99);
            } catch (Exception e) {
                System.out.println("markAsRead(99) -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
            }
        }
    }

    static JMailClient open(String user) {
        return JMail.connect()
                .smtpHost("smtp.example.com")
                .imapHost("imap.example.com")
                .username(user + "@example.com")
                .password("secret")
                .connect();
    }
}
```

```text
unread: 3
after markAsRead(2): 2
after markAsUnread(2): 3
after markAsRead(1) twice: 2
markAsRead(99) -> IllegalArgumentException: No message found with id=99
```

- `unreadCount()` is the number of messages without the mark.
- `markAsRead(id)` and `markAsUnread(id)` set and clear it. Repeating a call changes nothing. An unknown id is `IllegalArgumentException`.
- A server may mark a message as read by itself when its body is fetched. When the state matters (a support desk that works through unread mail), set it explicitly after `read`, as chapter 14 does.

## 12. Deleting, copying and moving

Deleting a message takes two steps in IMAP, and so it does in jmail: `delete(id)` puts a *deleted* mark on the message, and `expunge()` removes the marked messages for good. Between the two, the message is still in the folder.

```java
import org.j2os.platform.jmail.JMail;
import org.j2os.platform.jmail.JMailClient;
import org.j2os.platform.jmail.MailSummary;

import java.util.function.Consumer;

public class Moving {
    public static void main(String[] args) throws Exception {
        // sara's inbox already holds four messages from ali, with ids 1 to 4: One, Two, Three, Four
        try (JMailClient client = open("sara")) {
            var inbox = client.folder("INBOX");
            Consumer<String> show = folder -> {
                StringBuilder line = new StringBuilder(folder + ":");
                for (MailSummary s : client.folder(folder).list()) line.append("  ").append(s.id()).append("=").append(s.subject());
                System.out.println(line);
            };
            show.accept("INBOX");

            System.out.println("-- delete(1) only sets a flag");
            inbox.delete(1);
            show.accept("INBOX");

            System.out.println("-- copyTo(2, Work) creates Work and keeps the original");
            inbox.copyTo(2, "Work");
            show.accept("INBOX");
            show.accept("Work");

            System.out.println("-- moveTo(3, Archive) also purges what was flagged");
            inbox.moveTo(3, "Archive");
            show.accept("INBOX");
            show.accept("Archive");

            System.out.println("-- refused requests leave no folder behind");
            try {
                inbox.moveTo(2, "inbox");
            } catch (Exception e) {
                System.out.println("moveTo same folder -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
            }
            try {
                inbox.moveTo(99, "Elsewhere");
            } catch (Exception e) {
                System.out.println("moveTo unknown id -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
            }
            System.out.println(client.folders());

            System.out.println("-- expunge purges every flagged message");
            inbox.delete(2);
            inbox.expunge();
            show.accept("INBOX");
        }
    }

    static JMailClient open(String user) {
        return JMail.connect()
                .smtpHost("smtp.example.com")
                .imapHost("imap.example.com")
                .username(user + "@example.com")
                .password("secret")
                .connect();
    }
}
```

```text
INBOX:  1=One  2=Two  3=Three  4=Four
-- delete(1) only sets a flag
INBOX:  1=One  2=Two  3=Three  4=Four
-- copyTo(2, Work) creates Work and keeps the original
INBOX:  1=One  2=Two  3=Three  4=Four
Work:  1=Two
-- moveTo(3, Archive) also purges what was flagged
INBOX:  2=Two  4=Four
Archive:  1=Three
-- refused requests leave no folder behind
moveTo same folder -> IllegalArgumentException: Destination cannot be the same as the source folder: INBOX
moveTo unknown id -> IllegalArgumentException: No message found with id=99
[INBOX, Work, Archive]
-- expunge purges every flagged message
INBOX:  4=Four
```

- **`delete(id)` only marks.** The listing still shows message 1: it is marked as deleted, not gone.
- **`copyTo(id, folder)`** copies the message and leaves the original in place. The target folder is created when it does not exist. In the new folder the message has **its own id** (`1` in `Work`), which is not the id it had in `INBOX`.
- **`moveTo(id, folder)`** copies, marks the original as deleted and expunges. IMAP has no real "move", and jmail builds it from these three steps. Two things follow. First, `expunge` is folder-wide, so **every message marked as deleted in the source folder disappears too**: message 1, marked earlier, is gone after moving message 3. Second, the three steps are not one transaction: if the network fails in the middle, the message can end up in both folders.
- **Refused requests change nothing.** Moving to the folder the message is already in (any spelling of `INBOX` counts as the same folder) or moving an id that does not exist is `IllegalArgumentException`, and no new folder is left behind.
- **`expunge()`** removes every message marked as deleted in the folder, not only those marked through this handle.

Some providers treat folders as labels rather than boxes (Gmail is the usual example), and then "delete" and "move" can mean something different from the plain IMAP meaning. Try a destructive operation on a test account first.

## 13. Lifetime, connections and threads

```java
import org.j2os.platform.jmail.JMail;
import org.j2os.platform.jmail.JMailClient;
import org.j2os.platform.jmail.MailMessage;

public class Lifecycle {
    public static void main(String[] args) throws Exception {
        // sara's inbox already holds one message from ali: "Hello" with the body "Body"
        JMailClient client = open("sara");              // nothing connects yet

        var inbox = client.folder("INBOX");
        System.out.println("unread: " + inbox.unreadCount());
        System.out.println("folders: " + client.folders());
        MailMessage message = inbox.read(1);

        client.close();
        client.close();                                 // a second close is harmless

        try {
            client.folder("INBOX");
        } catch (Exception e) {
            System.out.println("folder after close -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            inbox.list();
        } catch (Exception e) {
            System.out.println("old handle after close -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            client.compose();
        } catch (Exception e) {
            System.out.println("compose after close -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            client.folders();
        } catch (Exception e) {
            System.out.println("folders after close -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        System.out.println("a message read earlier is still usable: " + message.content());
    }

    static JMailClient open(String user) {
        return JMail.connect()
                .smtpHost("smtp.example.com")
                .imapHost("imap.example.com")
                .username(user + "@example.com")
                .password("secret")
                .connect();
    }
}
```

```text
unread: 1
folders: [INBOX]
folder after close -> IllegalStateException: JMailClient has already been closed
old handle after close -> IllegalStateException: JMailClient has already been closed
compose after close -> IllegalStateException: JMailClient has already been closed
folders after close -> IllegalStateException: JMailClient has already been closed
a message read earlier is still usable: Body
```

- **Building opens nothing**, and the first call that needs the mailbox opens one IMAP connection. Later calls reuse it.
- **A lost connection is replaced.** After the provider has hung up, the next call logs in again by itself. Folders cached for the old connection are dropped. An operation that was running **at the moment** of the loss fails with `RuntimeException`; repeat it.
- **`close()` ends everything.** Open folders and the IMAP connection are closed, and from then on every operation of the client throws `IllegalStateException`, including those of folder handles obtained earlier. There is no automatic reconnect after `close()`. A second `close()` is harmless.
- **Results stay usable.** The `MailMessage` read before the close still works, because its content is already in memory. Asynchronous sends already submitted still finish.

**Threads.** A client can be shared. Every mailbox operation takes the client's lock, so operations on one client run **one at a time**; a long `read` of a big attachment makes the others wait. Sending does not take that lock, so a mailbox in the middle of a slow read does not hold up `send`. Two programs that need to read in parallel use two clients. A `Compose` object, on the other hand, belongs to one thread.

**How long to keep a client.** Logging in to IMAP costs a round trip with encryption, so a program that reads often keeps its client for a while and closes it when idle. A program that only sends can build a client per message, since building is free and SMTP is connected per send anyway. A server that keeps many users' clients should close the ones that have been idle for a while: servers close idle IMAP connections themselves, and the client would just have to reconnect.

## 14. Putting it together: a small support desk

A shop confirms an order by mail, the customer asks a question, and the shop's mailbox is worked through: read, mark as read, move to an archive folder.

```java
import org.j2os.platform.jmail.JMail;
import org.j2os.platform.jmail.JMailClient;
import org.j2os.platform.jmail.MailAttachment;
import org.j2os.platform.jmail.MailMessage;
import org.j2os.platform.jmail.MailSummary;

import java.io.File;
import java.nio.file.Files;
import java.util.concurrent.CountDownLatch;

public class Workflow {
    public static void main(String[] args) throws Exception {
        File receipt = new File(Files.createTempDirectory("shop").toFile(), "receipt.txt");
        Files.writeString(receipt.toPath(), "Order 12345: 1 x Notebook");

        try (JMailClient shop = open("sara"); JMailClient customer = open("ali")) {

            // 1. the shop confirms an order, in the background
            CountDownLatch sent = new CountDownLatch(1);
            shop.compose()
                    .to("ali@example.com")
                    .subject("Your order 12345")
                    .templateContent("mail/order.html")
                    .variable("orderId", "12345")
                    .variable("customerName", "Ali")
                    .variable("total", "$45.00")
                    .attach(receipt)
                    .send(error -> {
                        System.out.println(error == null ? "confirmation sent" : "failed: " + error.getMessage());
                        sent.countDown();
                    });
            sent.await();

            // 2. the customer reads it and asks a question
            MailSummary first = customer.folder("INBOX").list().get(0);
            MailMessage confirmation = customer.folder("INBOX").read(first.id());
            MailAttachment file = confirmation.attachments().get(0);
            System.out.println("customer got: " + first.subject() + " with " + file.name());
            customer.folder("INBOX").markAsRead(first.id());
            customer.compose().to("sara@example.com").subject("Question about 12345")
                    .content("<p>Can I change the address?</p>").send();

            // 3. the shop works through its unread mail
            var inbox = shop.folder("INBOX");
            System.out.println("shop unread: " + inbox.unreadCount());
            for (MailSummary summary : inbox.list()) {
                MailMessage message = inbox.read(summary.id());
                System.out.println("shop reads: " + summary.from() + " | " + summary.subject() + " | " + message.content());
                inbox.markAsRead(summary.id());
                inbox.moveTo(summary.id(), "Answered");
            }
            System.out.println("shop unread: " + inbox.unreadCount());
            System.out.println("folders: " + shop.folders());
            System.out.println("Answered: " + shop.folder("Answered").list().size());
        }
    }

    static JMailClient open(String user) {
        return JMail.connect()
                .smtpHost("smtp.example.com")
                .imapHost("imap.example.com")
                .username(user + "@example.com")
                .password("secret")
                .connect();
    }
}
```

```text
confirmation sent
customer got: Your order 12345 with receipt.txt
shop unread: 1
shop reads: ali@example.com | Question about 12345 | <p>Can I change the address?</p>
shop unread: 0
folders: [INBOX, Answered]
Answered: 1
```

The steps and the chapters they come from:

1. The confirmation is built from a template with a receipt attached and sent in the background, and the program waits for the callback (chapters 4 to 6).
2. The customer's client lists the inbox and reads the message and its attachment (chapters 8 and 9), then marks it as read (chapter 11) and answers with `compose()`.
3. The shop counts its unread mail, reads each message, marks it as read and moves it to `Answered`, which `moveTo` creates (chapters 11 and 12). The inbox is empty of unread mail afterwards, and the archive holds the finished work.

In a Spring application the sending side is usually a service. The shop's settings come from the application's own store, the client is built per message, and the callback closes it:

```java
@Service
public class OrderMailer {

    private static final Logger log = LoggerFactory.getLogger(OrderMailer.class);

    private final MailAccounts accounts;          // your own store of the shop's mail settings

    public OrderMailer(MailAccounts accounts) {
        this.accounts = accounts;
    }

    public void confirm(Order order) {
        MailAccount account = accounts.shop();
        JMailClient client = JMail.connect()      // nothing is opened here
                .smtpHost(account.smtpHost())
                .username(account.address())
                .password(account.password())
                .connect();

        client.compose()
                .to(order.customerEmail())
                .subject("Your order " + order.id())
                .templateContent("mail/order.html")
                .variable("orderId", String.valueOf(order.id()))
                .variable("customerName", HtmlUtils.htmlEscape(order.customerName()))
                .variable("total", order.total())
                .send(error -> {
                    if (error != null) {
                        log.warn("Confirmation of order {} failed", order.id(), error);
                    }
                    client.close();
                });
    }
}
```

Before going live:

- The password comes from the environment or a secrets store.
- The user name is the full address, and the port and host match the provider (587 or 465 for SMTP, 993 for IMAP).
- Every value from a person is escaped before it goes into a template.
- A callback logs failures; a failed confirmation must not vanish.
- Reading code expects `null` content, `null` attachment names and large mailboxes.
- The code that handles the mailbox of many users closes clients that are no longer used.

---

## Appendix A. Quick reference

### A.1 Entry and builder

| Call | Purpose |
|------|---------|
| `JMail.connect()` | Starts a `JMailClient.Builder` |
| `smtpHost(host)` | Required. Not blank |
| `smtpPort(port)` | 1 to 65535. Default 587. 465 means implicit TLS, others STARTTLS |
| `imapHost(host)` | Optional. Not blank |
| `imapPort(port)` | 1 to 65535. Default 993 |
| `username(name)`, `password(secret)` | Required. Not blank. The user name is also the sender |
| `connectTimeout(ms)` | 0 to 2147483647. Default 30000. 0 means no limit |
| `readTimeout(ms)` | 0 to 2147483647. Default 120000. Also the write limit. 0 means no limit |
| `connect()` | Builds the client. Opens nothing |

### A.2 Compose

| Call | Purpose |
|------|---------|
| `to(...)`, `cc(...)`, `bcc(...)` | Add recipients. Not blank |
| `subject(text)` | Subject. UTF-8. May be `null` |
| `content(html)` | Literal HTML body. Not `null` |
| `templateContent(path)` | Body from a classpath HTML file. `${name}` gaps |
| `variable(name, value)` | Value for a gap. Neither may be blank or `null` |
| `attach(file...)` | Attach existing regular files |
| `send()` | Send and wait |
| `send(callback)` | Send in the background. Callback gets `null` or the failure |

### A.3 JMailClient

| Call | Purpose |
|------|---------|
| `compose()` | Starts a message |
| `folder(name)` | A handle to a folder. Connects IMAP on first use |
| `folders()` | The full names of all folders |
| `createFolder(name)` | Creates a folder unless it exists |
| `removeFolder(name)` | Deletes a folder with its messages. Silent if missing |
| `close()` | Ends the client for good. Safe to repeat |
| `INBOX` | The constant `"INBOX"` |
| `DEFAULT_CONNECT_TIMEOUT_MILLIS`, `DEFAULT_READ_TIMEOUT_MILLIS` | 30000 and 120000 |

### A.4 MailboxFolder

| Call | Purpose |
|------|---------|
| `list()` | Every message as `MailSummary` |
| `read(id)` | The `MailMessage` with its attachments |
| `unreadCount()` | Messages without the read mark |
| `markAsRead(id)`, `markAsUnread(id)` | Set or clear the read mark |
| `delete(id)` | Mark as deleted. Nothing is removed yet |
| `expunge()` | Remove every message marked as deleted in the folder |
| `copyTo(id, folder)` | Copy. Creates the target if needed |
| `moveTo(id, folder)` | Copy, mark deleted, expunge. Creates the target if needed |

### A.5 Result types

| Type | Members |
|------|---------|
| `MailSummary` | `id()`, `subject()` (may be `null`), `from()` (may be `null`) |
| `MailMessage` | `content()` (may be `null`), `attachments()` (never `null`) |
| `MailAttachment` | `name()` (may be `null`), `contentId()` (may be `null`), `size()`, `download()` |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| A required builder setting is missing | `IllegalStateException` at `connect()` |
| A blank host, port or timeout out of range | `IllegalArgumentException` at the builder call |
| Wrong host or password | Not found at `connect()`; the first send or mailbox call fails |
| No IMAP host and a mailbox call | `IllegalStateException` |
| A blank address, folder name or variable name | `IllegalArgumentException` |
| A missing file, a directory as attachment | `IllegalArgumentException` at `attach` |
| A missing template | `IllegalArgumentException` at `templateContent` |
| A template variable never set | `IllegalStateException` when the message is built; in the callback for `send(callback)` |
| An address with bad syntax | `RuntimeException` "Failed to build the message" |
| Attachment deleted before a synchronous send | `RuntimeException` "Failed to build the message" |
| No recipient, refused recipient, wrong password, unreachable server | `RuntimeException` "Failed to send email" with the cause |
| One refused recipient among several | The whole message is not sent |
| The pool is full for `send(callback)` | `RejectedExecutionException`; no callback |
| A closed client is used | `IllegalStateException` |
| A folder that does not exist | `RuntimeException` "Failed to open folder ..." at the first operation |
| An unknown message id | `IllegalArgumentException` |
| The IMAP login fails | `RuntimeException` "IMAP connection failed" |
| The server hangs up between calls | The next call reconnects |
| The server hangs up during a call | `RuntimeException`; repeat the call |
| `moveTo` or `copyTo` to the same folder | `IllegalArgumentException`; nothing is created |
| `moveTo` interrupted by the network | The message may be in both folders |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Passwords in the source.** Read them from the environment or a secrets store, and never log them.
- **A user name that is not an address.** It is also the sender, so every message fails.
- **Expecting a copy in "Sent".** jmail files nothing; the provider may or may not.
- **Plain text in `content`.** The body is HTML. Line breaks vanish and `<` and `&` need escaping.
- **Unescaped template values.** Values from people go in raw. Escape them.
- **`${...}` in a template that is not a placeholder.** Names with dots or dashes stay as they are.
- **A list of recipients with one bad address.** The whole message is refused. Check first or send one by one.
- **"Sent" is not "delivered".** `send()` succeeded when the provider accepted the message; bounces arrive as mail.
- **Short programs and `send(callback)`.** The workers are daemon threads, so wait for the callback before `main` returns.
- **Ignoring `RejectedExecutionException`.** A full pool drops the send without calling the callback.
- **Non-ASCII file names.** They may not survive in every mail program.
- **Inline images when sending.** Not possible with `cid:`. Use hosted images.
- **`list()` on a huge folder.** No paging; keep mailboxes small.
- **Showing HTML from a message unfiltered.** Sanitize it.
- **Trusting `attachment.name()`.** It may be `null`, repeated or contain folders.
- **Assuming `delete` removes.** It only marks; and `moveTo` expunges every marked message in the folder.
- **Using a handle after `close()`.** Every operation throws.
- **Ids across folders.** An id means something in one folder only.

### C.2 Behavior worth knowing

- Building a client and `compose()` open no connection. SMTP connects per send, IMAP on first mailbox use.
- Subjects and bodies are sent in UTF-8; the body is always `text/html`.
- `bcc` recipients are served but never named in the delivered message.
- All mailbox operations of one client are serialized. Sending is not blocked by them.
- After a lost IMAP connection, folders cached for it are discarded and the next call logs in again.
- Read messages are fully in memory and independent of the connection.
- The send workers belong to the JVM, not to a client.
- `INBOX` is matched without regard to case; other folder names are not changed.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Asynchronous sends running at once (per JVM) | 16 |
| Asynchronous sends waiting (per JVM) | 1000 |
| Connection timeout | 30000 ms by default |
| Read and write timeout | 120000 ms by default |
| SMTP port | 587 (STARTTLS) by default; 465 gives implicit TLS |
| IMAP port | 993 (implicit TLS) by default |
| Authentication | Always required |
| Listing | Whole folder, no paging |
| Message size | Held in memory; the provider's own limit applies |
| Placeholder names | Letters, digits and `_` |
| Idle async workers | End after 30 seconds |

### C.4 Tuning and practical advice

- Prefer `send(callback)` in web requests, with a callback that logs failures.
- Build a client per message for sending only; keep one per user for a while when reading often.
- Lower `readTimeout` for interactive screens (a few seconds) and keep the default for large attachments.
- Keep mailboxes small by moving processed mail to an archive folder, and read only what is new by remembering the last id handled.
- Handle `RuntimeException` from mailbox calls by repeating the call once; a lost connection is replaced on the next call.
- For tests, use a test mail server (such as GreenMail) or a provider's test account, never customers' addresses.
- Do not run bulk newsletters through this client; providers limit the number of messages per day and per minute.

## Appendix D. Message reference

### D.1 Builder and client

| Message | Exception | Cause |
|---------|-----------|-------|
| `smtpHost is required` | `IllegalStateException` | `connect()` without an SMTP host |
| `username and password are required` | `IllegalStateException` | `connect()` without a login |
| `smtpHost cannot be blank`, `imapHost cannot be blank`, `username cannot be blank`, `password cannot be blank` | `IllegalArgumentException` | A null or blank value |
| `smtpPort must be between 1 and 65535: <n>`, `imapPort must be between 1 and 65535: <n>` | `IllegalArgumentException` | A port out of range |
| `connectTimeout must be between 0 and 2147483647 milliseconds: <n>`, `readTimeout ...` | `IllegalArgumentException` | A negative or too large timeout |
| `JMailClient has already been closed` | `IllegalStateException` | Any use after `close()` |
| `imapHost is not set — required for mailbox operations` | `IllegalStateException` | A mailbox call without an IMAP host |
| `IMAP connection failed` | `RuntimeException` | Login or connection to IMAP failed |

### D.2 Compose

| Message | Exception | Cause |
|---------|-----------|-------|
| `addresses cannot be null` | `NullPointerException` | `to(...)` given a null array |
| `Email address cannot be null or blank` | `IllegalArgumentException` | A blank recipient |
| `Content cannot be null` | `NullPointerException` | `content(null)` |
| `classpathPath cannot be null` | `NullPointerException` | `templateContent(null)` |
| `Template not found on classpath: <path>` | `IllegalArgumentException` | A missing template file |
| `Failed to read template '<path>'` | `RuntimeException` | The template could not be read |
| `Variable name cannot be blank` | `IllegalArgumentException` | `variable("", ...)` |
| `Value of variable '<name>' cannot be null` | `NullPointerException` | A null value |
| `Template variable '<name>' has not been set` | `IllegalStateException` | A placeholder without a value |
| `files cannot be null`, `Attachment file cannot be null` | `NullPointerException` | A null file or array |
| `Invalid attachment: <file>` | `IllegalArgumentException` | Not an existing regular file |
| `callback cannot be null` | `NullPointerException` | `send(null)` |
| `Failed to build the message` | `RuntimeException` | Bad address syntax, unreadable attachment |
| `Failed to send email` | `RuntimeException` | The provider or network refused |

### D.3 Folders and messages

| Message | Exception | Cause |
|---------|-----------|-------|
| `Folder name cannot be null or blank` | `IllegalArgumentException` | A blank folder name |
| `Failed to read the folder list` | `RuntimeException` | `folders()` failed |
| `Failed to create folder '<name>'` | `RuntimeException` | `createFolder` failed |
| `Folder creation did not succeed: <name>` | the cause of the above | The server declined to create it |
| `Failed to remove folder '<name>'` | `RuntimeException` | `removeFolder` failed |
| `Folder deletion did not succeed: <name>` | the cause of the above | The server declined to delete it |
| `Failed to open folder '<name>'` | `RuntimeException` | The folder is missing or cannot be opened |
| `Failed to read folder '<name>'` | `RuntimeException` | `list()` failed |
| `Failed to read message id=<id>` | `RuntimeException` | `read` failed |
| `Failed to read the message` | `RuntimeException` | The content could not be read |
| `No message found with id=<id>` | `IllegalArgumentException` | An unknown id |
| `Failed to change flag on message id=<id>` | `RuntimeException` | A mark could not be set |
| `Failed to count unread messages` | `RuntimeException` | `unreadCount` failed |
| `Failed to expunge folder '<name>'` | `RuntimeException` | `expunge` failed |
| `Destination cannot be the same as the source folder: <name>` | `IllegalArgumentException` | `copyTo` or `moveTo` into the same folder |
| `Failed to look up message id=<id>` | `RuntimeException` | The check before a copy or move failed |
| `Failed to copy message id=<id>` | `RuntimeException` | `copyTo` failed |
| `Failed to move message id=<id>` | `RuntimeException` | `moveTo` failed |
