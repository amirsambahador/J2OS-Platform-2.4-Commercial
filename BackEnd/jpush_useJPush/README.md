# jpush

*J2OS Platform 2.4*

A web page normally asks and the server answers. But some news starts on the server: an order has shipped, a payment has arrived, a colleague has sent a message. The page cannot know this unless it asks again and again, which wastes work and still arrives late. The better way is to keep one connection open and let the server *push* the news when it happens. The standard tool for this is the *WebSocket*, a connection between browser and server that stays open and carries messages in both directions.

**jpush** (`org.j2os.platform.jpush`) is a small library that does the server side of this. It accepts WebSocket connections, finds out who is on the other end from a JWT (the signed token that your login already issues), and lets any code in your application send a text message to one user or to everyone. It has no dependency on a web framework, so the same code runs in Spring and in plain Jakarta EE. Two clients are included: `JPushClient` for Java programs, and `useJPush` for Nuxt and Vue pages.

```java
// once, at startup
JPushServer.configure()
        .tokenSource(TokenSource.AUTHORIZATION_HEADER_BEARER)
        .issuer("shop")
        .secretKey(secret)
        .start();

// anywhere in the server, for example in a REST method
JPushServer.sendMessage("shop", "ali", "Order 17 has shipped");
```

```js
const push = useJPush()
push.connect('wss://shop.example.com/jpush', {
  token: () => auth.token,
  onMessage: (from, message) => toast.add({ title: message })
})
```

The Java module has these types:

- `JPushServer` is the center. You configure it once and then call `sendMessage`, `broadcast`, `onlineAccounts`, `isOnline` and `disconnect` from anywhere in the application.
- `JPushEndpoint` is the class that receives the connections. You write an empty subclass of it.
- `Account` describes one connected user: whether it is a guest, its IP address and its user name.
- `TokenSource` tells the server where to look for the token: in a cookie, or in a bearer header.
- `JPushClient` and `JPushConnection` are the Java client.

`useJPush` is the Nuxt composable (a function that gives a page some ready-made behavior). This guide goes step by step: each chapter has a short program and the output it prints. The parts that only make sense in a web application are shown as Spring or Vue snippets. The appendices are for quick reference.

## Contents

1. [How a push works](#1-how-a-push-works)
2. [Starting the server](#2-starting-the-server)
3. [The first push](#3-the-first-push)
4. [Sending to one user and to everyone](#4-sending-to-one-user-and-to-everyone)
5. [Who is online](#5-who-is-online)
6. [Who is connecting: tokens, cookies and guests](#6-who-is-connecting-tokens-cookies-and-guests)
7. [Watching and limiting delivery: hooks and slow clients](#7-watching-and-limiting-delivery-hooks-and-slow-clients)
8. [The Java client](#8-the-java-client)
9. [The Nuxt composable useJPush](#9-the-nuxt-composable-usejpush)
10. [Reconnecting](#10-reconnecting)
11. [Putting it together: telling a customer that the order has shipped](#11-putting-it-together-telling-a-customer-that-the-order-has-shipped)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The server side needs Java 21 or newer, the Jakarta WebSocket API (every servlet container has it, and Spring Boot brings it with `spring-boot-starter-websocket`), and the **jsecurity** module, because jpush reads tokens with its `TokenProcessor`. The Java client uses only the JDK.

The programs in this guide print what the clients receive. For convenience, each program runs the web server and the clients together. In your application the server runs in the web server and the clients are other programs or browsers.

---

## 1. How a push works

Every push goes through the same four steps:

1. A client (a browser page or a Java program) opens a WebSocket to the path `/jpush` of your application. It can add a token to prove who it is.
2. `JPushEndpoint` receives the connection. It reads the token, learns the user name and registers the connection with `JPushServer`. A connection without any token is accepted as a *guest*.
3. Somewhere else in your code, for example in a REST method, you call `JPushServer.sendMessage(from, to, text)`.
4. The server writes the message to the connection of that user. The client's `onMessage` function receives it.

A message has two parts: the **sender** and the **text**. The sender is any string you choose. It is often a user name or a word such as `system`. On the wire the two parts are joined with a null character (`\0`) between them, and both clients split them again for you.

Some facts that shape the rest of the guide:

- A signed-in user has **one** connection. If the same user connects again, for example in a second browser tab, the older connection is closed.
- Sending never waits. The call returns at once, and a small background writer sends the text. A client that cannot keep up is disconnected instead of slowing everyone else.
- The clients are *receive-only*. They cannot send messages to the server over this connection. Use ordinary HTTP requests for that.
- The path `/jpush` is fixed. There is one endpoint per application.

## 2. Starting the server

Two small pieces start the server: the endpoint class, and the configuration that runs once at startup.

The endpoint is an empty subclass of `JPushEndpoint` with an annotation. The annotation is the ordinary `@ServerEndpoint` of Jakarta WebSocket. Its `configurator` is required, because it is what lets the endpoint read the cookies and headers of the handshake (the first HTTP request that turns into a WebSocket).

```java
@Component   // Spring only; a plain Jakarta EE container finds the class by its annotation
@ServerEndpoint(value = "/jpush", configurator = JPushEndpoint.HandshakeCapture.class)
public class PushEndpoint extends JPushEndpoint {
}
```

Do not override anything in it. The base class already handles opening, closing and errors, and those methods are `final`. In Spring Boot add the WebSocket starter:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

The configuration runs once. In Spring it can be a `@PostConstruct` method:

```java
@Component
public class PushStartup {

    @PostConstruct
    public void start() {
        JPushServer.configure()
                .tokenSource(TokenSource.AUTHORIZATION_HEADER_BEARER)
                .issuer("shop")
                .secretKey(System.getenv("SHOP_TOKEN_SECRET"))
                .start();
    }
}
```

In Jakarta EE the same chain goes into a `ServletContextListener`. This example reads the token from a cookie:

```java
@WebListener
public class PushStartup implements ServletContextListener {

    @Override
    public void contextInitialized(ServletContextEvent event) {
        JPushServer.configure()
                .tokenSource(TokenSource.COOKIE)
                .cookieName("access")
                .issuer("shop")
                .secretKey(System.getenv("SHOP_TOKEN_SECRET"))
                .start();
    }
}
```

The builder has these options.

| Option | Meaning |
|--------|---------|
| `tokenSource(...)` | **Required.** `TokenSource.COOKIE` or `TokenSource.AUTHORIZATION_HEADER_BEARER` |
| `cookieName(...)` | The name of the cookie. Required with `COOKIE`, not used otherwise |
| `issuer(...)` and `secretKey(...)` | The issuer and the secret that your login used to sign the tokens. They must be the same. The secret must be at least 64 bytes long |
| `onConnect(...)` and `onDisconnect(...)` | Functions called with the `Account` when someone connects or leaves |
| `onBeforeSend(...)` and `onError(...)` | A filter before each delivery, and a report of each failed delivery (chapter 7) |
| `maxQueuedCharsPerClient(...)` | The size of the waiting line of each client (chapter 7) |

`start()` checks the configuration and installs it. Call it **once**, at startup. A wrong configuration stops there with a clear message:

```java
JPushServer.configure().start();
JPushServer.configure().tokenSource(TokenSource.COOKIE).issuer("shop").secretKey(secret).start();
JPushServer.configure().tokenSource(TokenSource.AUTHORIZATION_HEADER_BEARER).issuer("shop").secretKey("too-short").start();
JPushServer.configure().maxQueuedCharsPerClient(0);
```

```text
tokenSource is required
cookieName is required when tokenSource is COOKIE
HMAC512 secret key must contain at least 64 UTF-8 bytes
maxQueuedCharsPerClient must be positive
```

The last line comes from `maxQueuedCharsPerClient`, which checks its value at once, before `start()`. The third message comes from the `TokenProcessor` of jsecurity, which requires a secret of at least 64 bytes. Keep the secret out of the source code, as the examples do with an environment variable.

If a client connects before `start()` has run, the endpoint fails with `IllegalStateException`: `JPushServer.configure()...start() must be called before accepting connections`.

## 3. The first push

The smallest complete example has a client and a push. The client is `JPushClient`, described fully in chapter 8. For now it is enough to know that `.token(...)` proves the identity of the user, `.onMessage(...)` receives pushes, and `.connect()` opens the connection and waits until it is open.

The token is the JWT of the user. In your application your login issues it. Here a `TokenProcessor` of jsecurity makes one for the user `ali`, valid for 60 minutes.

```java
String token = tokens.createToken("ali", null, 60L);   // normally issued by your login

JPushClient.server("ws://localhost:8080/jpush")
        .token(token)
        .onMessage((from, message) -> System.out.println("ali received: " + from + " -> " + message))
        .connect();
```

When ali is connected, any code of the server can send a message to him:

```java
JPushServer.sendMessage("system", "ali", "Your order has shipped");
```

```text
ali received: system -> Your order has shipped
```

The call to `sendMessage` returns at once, and the text arrives a moment later on ali's side. A program that exits right after sending must wait a little (the examples sleep for 300 milliseconds), because delivery is asynchronous.

## 4. Sending to one user and to everyone

`sendMessage(from, toUsername, message)` sends to one user.

```java
// ali and sara are connected, omid is not
JPushServer.sendMessage("ali", "sara", "Hi Sara");
JPushServer.sendMessage("sara", "ali", "Hi Ali");
JPushServer.sendMessage("system", "omid", "Nobody is listening");
JPushServer.sendMessage(null, "ali", "Message without a sender");
```

```text
ali  <- sara: Hi Ali
ali  <- null: Message without a sender
sara <- ali: Hi Sara
```

- The sender is only a label. The server does not check that it is a real user. The first two calls look like a chat between ali and sara, but your code decides who the sender is.
- A message to a user who is **not online** is silently dropped. The call to `omid` neither fails nor stores anything. If the user must not miss the news, also save it in your database and show it when the user comes back.
- A `null` sender arrives as the text `null`, as the third line of the output shows. Pass a real label, such as `"system"`.
- The order of lines between *different* clients is not fixed, because each client has its own writer. The order of messages to *one* client is the order in which you sent them.

`broadcast(from, message, authenticatedOnly)` sends to everyone.

```java
// ali is signed in, the other connection is a guest
JPushServer.broadcast("system", "Open to everyone", false);
JPushServer.broadcast("system", "Members only", true);
```

```text
guest <- Open to everyone
ali   <- Open to everyone
ali   <- Members only
```

With `authenticatedOnly` set to `false`, every open connection receives the message, guests too. With `true`, guests are skipped. Use `true` for anything that is not meant for the public.

## 5. Who is online

`onlineAccounts()` returns an `Account` for every open connection. `isOnline(username)` asks about one user. `disconnect(username)` closes the connection of one user from the server side.

```java
// ali and sara are signed in, one more connection is a guest
System.out.println("online      : " + JPushServer.onlineAccounts());
System.out.println("ali online? : " + JPushServer.isOnline("ali"));
System.out.println("omid online?: " + JPushServer.isOnline("omid"));

JPushServer.disconnect("sara");
Thread.sleep(300);
System.out.println("sara online?: " + JPushServer.isOnline("sara"));
```

```text
online      : [Account[isGuest=true, ip=null, username=null], Account[isGuest=false, ip=null, username=ali], Account[isGuest=false, ip=null, username=sara]]
ali online? : true
omid online?: false
sara online?: false
```

An `Account` is a record with three components: `isGuest`, `ip` and `username`. For a guest the user name is `null`. The list includes guests, so filter on `isGuest()` when you only want users.

The `ip` comes from the first address of the `X-Forwarded-For` header. A reverse proxy such as Nginx adds this header. Without a proxy there is no such header and `ip` is `null`, as in the output above. If your application runs behind a proxy, ask it to set the header:

```nginx
location /jpush {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

Do not trust this address for security decisions. A client that connects without a proxy in front can write any value in the header.

`disconnect("sara")` closes her connection with the code 1000 and the reason `disconnected by server`. The record of the connection is removed when the close is complete, which is why the example waits a moment before asking again. A client made with `useJPush` does not try to reconnect after this close (chapter 10).

## 6. Who is connecting: tokens, cookies and guests

How the server finds the token depends on the `TokenSource`.

| Source | Where the token is | Typical user |
|--------|-------------------|--------------|
| `COOKIE` | in the cookie with the configured name | a browser page whose login set that cookie |
| `AUTHORIZATION_HEADER_BEARER` | in the header `Authorization: Bearer <token>`, or, if there is no such header, in the query parameter `access_token` | a Java client, or a page that keeps the token in memory |

A browser cannot set headers on a WebSocket, which is why the bearer mode also reads the query parameter. `useJPush` adds `?access_token=...` to the address for you. The word `Bearer` is not case-sensitive.

In cookie mode the server takes the first cookie whose name is exactly the configured one:

```java
JPushServer.configure()
        .tokenSource(TokenSource.COOKIE)
        .cookieName("access")
        .issuer("shop")
        .secretKey(secret)
        .start();

// the browser sends the cookie by itself; the Java client can send one too
JPushClient.server(url).cookie("access", token).connect();
System.out.println("right cookie name : " + JPushServer.onlineAccounts());

JPushClient.server(url).cookie("other", token).connect();   // a different cookie name
Thread.sleep(200);
System.out.println("wrong cookie name : " + JPushServer.onlineAccounts());
```

```text
right cookie name : [Account[isGuest=false, ip=null, username=ali]]
wrong cookie name : [Account[isGuest=false, ip=null, username=ali], Account[isGuest=true, ip=null, username=null]]
```

The second connection uses another cookie name. The server does not find its cookie and accepts the connection **as a guest**. A missing token is not an error. This is the most common surprise in cookie mode, so check the cookie name first when a signed-in user appears as a guest.

A token that is *present but wrong* is a different case. The server accepts the WebSocket and immediately closes it with the code 1008 (policy violation):

```java
JPushClient.server(url).token("not-a-real-token").connect();
Thread.sleep(300);
System.out.println("after a bad token : " + JPushServer.onlineAccounts());

// a well-formed token, but signed with another secret
JPushClient.server(url).token(forgedToken).connect();
Thread.sleep(300);
System.out.println("after a forged one: " + JPushServer.onlineAccounts());
```

```text
after a bad token : []
after a forged one: []
```

Neither connection stays. The closing reason tells the client why: `invalid or expired token` when the signature, the issuer or the date is wrong, and `token has no subject` when the token is valid but names nobody. A browser page sees these in its `onClose` function (chapter 9).

One user has one connection. When the same user connects again, the server closes the older connection with the code 1000 and the reason `replaced by a newer connection`, and keeps the new one:

```java
JPushClient.server(url).token(token)
        .onMessage((from, message) -> System.out.println("first tab  <- " + message)).connect();
JPushClient.server(url).token(token)
        .onMessage((from, message) -> System.out.println("second tab <- " + message)).connect();

JPushServer.sendMessage("system", "ali", "Only the newest connection gets this");
System.out.println("online: " + JPushServer.onlineAccounts().size());
```

```text
second tab <- Only the newest connection gets this
online: 1
```

Only the second tab receives the message, and only one account is online. The server also calls `onDisconnect` for the replaced connection when it has closed. Guests are not subject to this rule, because they have no user name.

## 7. Watching and limiting delivery: hooks and slow clients

Four hooks let your code watch the server.

```java
JPushServer.configure()
        .tokenSource(TokenSource.AUTHORIZATION_HEADER_BEARER)
        .issuer("shop")
        .secretKey(secret)
        .onConnect(account -> System.out.println("[connect]    " + account.username()))
        .onDisconnect(account -> System.out.println("[disconnect] " + account.username()))
        .onBeforeSend((from, target, message) -> !message.contains("spam"))
        .start();

// later, with ali connected
JPushServer.sendMessage("system", "ali", "Hello");
JPushServer.sendMessage("system", "ali", "Buy cheap spam");
```

```text
[connect]    ali
ali <- Hello
[disconnect] ali
```

- `onConnect` and `onDisconnect` receive the `Account`. Use them for logs, for an online counter, or to set a flag in the database.
- `onBeforeSend(from, target, message)` runs before each delivery to one user. Return `false` to **silently skip** that delivery. The second message is skipped, and nothing is reported. It is the place for filters, for example a rule that a user may not receive messages from a blocked sender. If the hook throws a runtime exception, the delivery counts as failed.
- `onError(from, target, message, exception)` is called once for each failed delivery. It can be called from another thread. If the hook itself throws, the exception is only logged.

`target` in the last two hooks is the `Account` of the recipient.

### Slow clients

The server never waits for a client. Each connection has a waiting line (a buffer) and one background writer that empties it. `maxQueuedCharsPerClient` sets the size of the line in characters. The default is 8 MiB of characters. A message is always accepted when the line is empty, even if it is larger than the limit.

When a client reads more slowly than you send, its line fills up. The server then reports the failure to `onError`, closes that client with the code 1013 (`slow consumer`) and goes on. Other clients are not affected.

```java
JPushServer.configure()
        .tokenSource(TokenSource.AUTHORIZATION_HEADER_BEARER)
        .issuer("shop")
        .secretKey(secret)
        .maxQueuedCharsPerClient(1_000)
        .onError((from, target, message, error) ->
                System.out.println("[error] to " + target.username() + ": " + error.getMessage()))
        .start();

String big = "x".repeat(400);
for (int i = 0; i < 100; i++) {
    JPushServer.sendMessage("system", "ali", big);
}
Thread.sleep(500);
System.out.println("ali online? " + JPushServer.isOnline("ali"));
```

```text
[error] to ali: Outbound buffer of this client is full (limit 1000 characters); the client is too slow and its connection is being closed
ali online? false
```

If a write to the client fails, the connection is closed with the code 1013 and the reason `write failed`, and the failure is reported to `onError` in the same way. Note that a client that has stopped reading may see only an abnormal close (1006) instead of a clear code.

## 8. The Java client

`JPushClient` connects a Java program, for example a monitoring tool or a test. It uses the WebSocket of the JDK.

| Method | Meaning |
|--------|---------|
| `JPushClient.server(url)` | Starts a builder for the given `ws://` or `wss://` address |
| `.token(token)` | Sends the token as `Authorization: Bearer` |
| `.cookie(name, token)` | Sends the token in a cookie |
| `.onMessage((from, message) -> ...)` | Receives each push |
| `.onError(error -> ...)` | Receives connection errors |
| `.connectTimeout(duration)` | How long to wait for the connection. The default is 30 seconds |
| `.connect()` | Opens the connection and waits until it is open. Returns a `JPushConnection` |

Use `.token` **or** `.cookie`, not both. With neither, the client connects as a guest. `connect()` waits until the handshake is complete. If it fails, the error goes to `onError` and `connect()` throws the same failure as a `CompletionException`. Text frames that arrive in pieces are put back together before `onMessage` is called.

```java
JPushClient.server(url).token(" ");
JPushClient.server(url).cookie("access", "t").token("t").connect();
JPushClient.server(url).connectTimeout(Duration.ZERO);

// nobody listens at this address
JPushClient.server("ws://localhost:1/jpush")
        .onError(error -> System.out.println("onError: " + error.getClass().getSimpleName()))
        .connect();
```

```text
token cannot be blank
Call only one of .cookie(...) or .token(...), not both
connectTimeout must be positive
onError: ConnectException
thrown : CompletionException
```

The `JPushConnection` that `connect()` returns has one method, `close()`. It closes the connection normally and returns when the close handshake is finished.

```java
JPushConnection connection = JPushClient.server(url).token(token).connect();
// ...
connection.close();   // returns when the close handshake is finished
```

The client cannot send messages. It also has no callback for a close and no reconnect. A program that must stay connected has to call `connect()` again by itself.

## 9. The Nuxt composable useJPush

`useJPush` lives in `composables/useJPush.js`. Nuxt makes it available in every page without an import. It returns three functions: `connect`, `disconnect` and `isConnected`.

```js
const push = useJPush()

push.connect('ws://localhost:18080/jpush', {
  token: ali,
  onOpen: ({ reconnected }) => console.log('open, reconnected =', reconnected),
  onMessage: (from, message) => console.log(`message from ${from}: ${message}`),
  onClose: ({ code, reason }) => console.log('closed', code, reason)
})

setTimeout(() => { console.log('connected?', push.isConnected()); push.disconnect() }, 800)
```

```text
open, reconnected = false
message from system: Welcome ali
connected? true
closed 1000 
```

`connect(url, options)` accepts these options:

| Option | Meaning |
|--------|---------|
| `token` | The token, as a string or as a function that returns one. A function is called at every connection and reconnection, so a renewed token is used. Without `token` the page connects as a guest or, in cookie mode, with its cookie |
| `onMessage(from, message)` | Called for each push. `from` is `null` when the message has no sender part |
| `onOpen({ reconnected })` | Called when the connection opens. `reconnected` is `true` from the second opening on |
| `onClose({ code, reason, wasClean, reconnecting, delayMs })` | Called when the connection closes. `reconnecting` says whether a new attempt follows, and `delayMs` how long it waits |
| `onError(error)` | Called on connection errors |
| `reconnect` | `false`, `true`, or an object with the reconnect settings (chapter 10) |

A few rules:

- During server-side rendering `connect` returns `null` and does nothing, because a WebSocket exists only in the browser. Call it in `onMounted`.
- Calling `connect` again while a connection exists returns the existing connection. It does not open a second one.
- When the component or page that called `useJPush` is destroyed, the connection is closed by itself.
- `disconnect()` closes the connection with the code 1000 and stops any pending retry.
- In cookie mode, do not pass `token`. The browser sends the cookie by itself, and the cookie must be one that the browser sends to the WebSocket address.

How the token reaches the server, and what a wrong option does:

```js
useJPush().connect('ws://h/jpush', { token: '' })
useJPush().connect('ws://h/jpush', { reconnect: 'yes' })
useJPush().connect('ws://h/jpush', { reconnect: { minDelayMs: 500, maxDelayMs: 100 } })
useJPush().connect('ws://h/jpush', { reconnect: { jitter: 2 } })

useJPush().connect('ws://h/jpush', { token: () => 'a b&c' })
useJPush().connect('ws://h/jpush?lang=fa', { token: 'abc' })
```

```text
token must not be empty
reconnect must be a boolean or an options object
reconnect.maxDelayMs must be >= reconnect.minDelayMs
reconnect.jitter must be between 0 and 1
ws://h/jpush?access_token=a%20b%26c
ws://h/jpush?lang=fa&access_token=abc
```

The token is encoded and added to the address as `access_token`, with `&` instead of `?` when the address already has a query. An empty token, or an option of the wrong kind, stops `connect` at once with an error, so a mistake appears on the first run. The check of the token happens on the server, so a bad token is not an exception here. It shows up in `onClose`:

```js
const bad = useJPush()
bad.connect(url, {
  token: 'not-a-real-token',
  onClose: ({ code, reason, reconnecting }) => console.log('bad token  ->', code, reason, '| reconnecting:', reconnecting)
})

const kicked = useJPush()
kicked.connect(url, {
  token: kickToken,   // the server calls JPushServer.disconnect("kick") a moment later
  onClose: ({ code, reason, reconnecting }) => console.log('kicked     ->', code, reason, '| reconnecting:', reconnecting)
})
```

```text
bad token  -> 1008 invalid or expired token | reconnecting: false
kicked     -> 1000 disconnected by server | reconnecting: false
```

Both closes are final: `reconnecting` is `false`. The code 1008 means that the token was rejected, and the code 1000 means that the server closed the connection on purpose, because a newer connection replaced it or because the server called `disconnect`. Trying again would give the same answer, so `useJPush` does not.

## 10. Reconnecting

Networks fail. A laptop sleeps, a server restarts, a Wi-Fi connection drops. `useJPush` reconnects by itself, waiting longer after each failure so that a restarting server is not hit by every page at once. This is called *exponential backoff*.

```js
push.connect(url, {
  token: () => auth.token,
  reconnect: {
    minDelayMs: 1000,     // first wait
    maxDelayMs: 30000,    // longest wait
    factor: 2,            // each wait is this many times the one before
    jitter: 0.2,          // random spread of +/- 20 percent
    stableAfterMs: 10000, // a connection that lasts this long resets the counting
    maxAttempts: Infinity // give up after this many tries in a row
  }
})
```

The wait before attempt number *n* (counting from 0) is `minDelayMs × factor^n`, never more than `maxDelayMs`, then multiplied by a random number between `1 − jitter` and `1 + jitter` and again limited to `maxDelayMs`. The random part, called *jitter*, keeps many pages from retrying at the same second. The counting starts again after a connection that lasted at least `stableAfterMs`. After `maxAttempts` failures in a row it gives up.

The example below sets `jitter` to 0 and the first wait to 100 ms so that the numbers are easy to read. The server is down for the first two attempts, accepts the third connection and drops it quickly, and then stays up.

```js
useJPush().connect('ws://localhost:8080/jpush', {
  token: 'abc',
  reconnect: { minDelayMs: 100, jitter: 0 },
  onOpen: ({ reconnected }) => console.log('open, reconnected =', reconnected),
  onClose: ({ code, reconnecting, delayMs }) => console.log('closed', code, '| reconnecting:', reconnecting, '| in', delayMs, 'ms')
})
```

```text
closed 1006 | reconnecting: true | in 100 ms
closed 1006 | reconnecting: true | in 200 ms
open, reconnected = false
closed 1006 | reconnecting: true | in 400 ms
open, reconnected = true
```

- The waits grow: 100, 200, 400 milliseconds. The third wait is 400, not 100, because the connection that drops lasts only a moment, much less than `stableAfterMs`. A connection that opens and fails at once does not reset the counting.
- `onOpen` reports `reconnected: false` for the first opening and `true` for the next ones. Use it to reload what the page may have missed.
- `onClose` is called before the wait begins, and it tells you the delay.
- The codes 1000 and 1008 never reconnect. The codes 1013 (slow client) and 1006 (abnormal close) do.
- **Messages sent while the page was disconnected are not replayed.** After `reconnected: true`, ask your API for what is new.

To turn reconnecting off:

```js
push.connect(url, { token, reconnect: false })
```

`reconnect: true` and leaving the option out both mean the defaults. An object is merged into the defaults. Wrong values are rejected at once (see Appendix D).

## 11. Putting it together: telling a customer that the order has shipped

A shop marks an order as shipped in its back office. The customer, who has the shop open in a browser tab, should see the news at once.

On the server, the endpoint and the startup are the ones of chapter 2. The REST method that ships the order sends the push:

```java
@RestController
public class OrderController {

    @PutMapping("/orders/{id}/ship")
    public void ship(@PathVariable long id, @RequestParam String customer) {
        // ... mark the order as shipped in the database ...
        JPushServer.sendMessage("shop", customer, "Order " + id + " has shipped");
    }
}
```

On the customer's page, `useJPush` receives it and the page shows it:

```vue
<script setup>
const push = useJPush()
const auth = useAuthStore()
const notes = ref([])

onMounted(() => {
  push.connect('wss://shop.example.com/jpush', {
    token: () => auth.token,
    onMessage: (from, message) => notes.value.push(message)
  })
})
</script>

<template>
  <ul>
    <li v-for="(note, i) in notes" :key="i">{{ note }}</li>
  </ul>
</template>
```

`auth.token` is the token that the shop's login gave to the page. When the back office calls `PUT /orders/17/ship?customer=ali`, the method runs `JPushServer.sendMessage("shop", "ali", "Order 17 has shipped")`. This is what ali's page does when the push arrives:

```js
const orders = []
useJPush().connect('ws://localhost:18081/jpush', {
  token: () => readFileSync('ship-token.txt', 'utf8'),
  onMessage: (from, message) => { orders.push(message); console.log('page shows:', orders) }
})
```

```text
page shows: [ 'Order 17 has shipped' ]
```

If ali has no tab open at that moment, nothing arrives, and the push is lost. The shop should also save a note in its database, and the page should read it when it opens. The push is for speed, and the database is for certainty.

---

## Appendix A. Quick reference

### A.1 JPushServer

| Member | Purpose |
|--------|---------|
| `configure()` | Returns the builder |
| `sendMessage(from, toUsername, message)` | Sends to one user. Does nothing if the user is offline |
| `broadcast(from, message, authenticatedOnly)` | Sends to all open connections. With `true`, guests are skipped |
| `onlineAccounts()` | A list of `Account`, guests included |
| `isOnline(username)` | `true` when the user has an open connection |
| `disconnect(username)` | Closes the connection of the user with the code 1000 |

### A.2 JPushServer builder

| Method | Purpose |
|--------|---------|
| `tokenSource(TokenSource)` | **Required** |
| `cookieName(String)` | Required with `COOKIE` |
| `issuer(String)`, `secretKey(String)` | Passed to the `TokenProcessor`. The secret is at least 64 bytes |
| `onConnect(Consumer<Account>)` | Called when a connection is registered |
| `onDisconnect(Consumer<Account>)` | Called when a connection is removed |
| `onBeforeSend((from, target, message) -> boolean)` | `false` skips the delivery to that recipient |
| `onError((from, target, message, exception) -> ...)` | Called for each failed delivery |
| `maxQueuedCharsPerClient(long)` | Size of each client's waiting line. Default 8 MiB of characters |
| `start()` | Installs the configuration. Call once |

### A.3 Account and TokenSource

| Type | Members |
|------|---------|
| `Account` | `isGuest()`, `ip()`, `username()` |
| `TokenSource` | `COOKIE`, `AUTHORIZATION_HEADER_BEARER` |

### A.4 JPushClient

| Method | Purpose |
|--------|---------|
| `server(url)` | Starts the builder |
| `token(String)` or `cookie(String, String)` | The identity. Not both |
| `onMessage(BiConsumer<String, String>)` | Receives `from` and `message` |
| `onError(Consumer<Throwable>)` | Receives errors |
| `connectTimeout(Duration)` | Default 30 seconds. Must be positive |
| `connect()` | Opens the connection, returns `JPushConnection` |
| `JPushConnection.close()` | Closes normally and waits for the close handshake |

### A.5 useJPush

| Member | Purpose |
|--------|---------|
| `connect(url, options)` | Opens and keeps a connection. Returns `null` outside the browser |
| `disconnect()` | Closes with the code 1000 and stops retrying |
| `isConnected()` | `true` while the connection is open |
| `token` | String or function. Sent as `access_token` in the query |
| `reconnect` | `false`, `true` or an object. Defaults: `minDelayMs` 1000, `maxDelayMs` 30000, `factor` 2, `jitter` 0.2, `stableAfterMs` 10000, `maxAttempts` unlimited |

### A.6 Close codes

| Code | Reason | Sent when | `useJPush` reconnects |
|------|--------|-----------|-----------------------|
| 1000 | `replaced by a newer connection` | the same user connects again | no |
| 1000 | `disconnected by server` | the server calls `disconnect` | no |
| 1008 | `invalid or expired token` | the token is wrong or expired, or signed with another secret | no |
| 1008 | `token has no subject` | the token is valid but has no user name | no |
| 1008 | `JPush endpoint must be mapped to "/jpush"` | the endpoint is mapped to another path | no |
| 1013 | `slow consumer` | the waiting line of the client is full | yes |
| 1013 | `write failed` | writing to the client fails | yes |
| 1006 | none | the connection is lost without a close | yes |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|-------------|
| `start()` has a missing or wrong option | `start()` throws and nothing is installed (see Appendix D) |
| A client connects before `start()` | The endpoint fails with `IllegalStateException` |
| `sendMessage` to an offline user | Nothing happens |
| `sendMessage` to a connection that has just closed | Nothing happens |
| A token is present but wrong | The connection is closed with 1008 |
| The same user connects twice | The older connection is closed with 1000 |
| A delivery is skipped by `onBeforeSend` | No error. Nothing is sent to that recipient |
| `onBeforeSend` throws a runtime exception | The delivery counts as failed and goes to `onError` |
| A delivery fails | `onError` is called once for it |
| `onError` throws | The exception is only logged |
| A client is too slow | `onError` is called, the client is closed with 1013 |
| `JPushClient.connect()` fails | `onError` is called, then `connect()` throws a `CompletionException` |
| `useJPush` loses the connection | `onClose`, then a retry, unless the code is 1000 or 1008 |
| `useJPush` has a bad option or an empty token | `connect` throws at once |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **A signed-in user appears as a guest.** In cookie mode the cookie name must be exactly the configured one. A missing token is accepted as a guest, not rejected.
- **The endpoint has no `configurator`.** Without `configurator = JPushEndpoint.HandshakeCapture.class` the endpoint cannot see the cookies and headers.
- **The endpoint is not on `/jpush`.** The path is fixed. If it is mapped elsewhere, every connection is closed with 1008. A context path in front, such as `/shop/jpush`, is fine.
- **`start()` is called twice.** It is meant to run once at startup.
- **Messages to offline users are lost.** There is no storage and no replay. Keep important news in your database.
- **A second tab replaces the first.** One user has one connection. If your users keep many tabs open, only the newest one receives pushes.
- **A secret in source code.** Read it from the environment or from a secret store, and use the one your login uses.
- **Waiting too little after sending.** A program that ends right after `sendMessage` may end before the text is written.
- **Behind a proxy.** The proxy must pass the WebSocket upgrade (`Upgrade` and `Connection` headers), or the connection never opens. See the Nginx block in chapter 5.

### C.2 Behaviour worth knowing

- The server is one registry for the whole JVM. All static methods of `JPushServer` work on it.
- Each connection has its own buffer and its own writer, so a slow client cannot slow another one.
- The IP in `Account` is the first address of `X-Forwarded-For`, or `null`.
- `onlineAccounts()` is a snapshot. A connection can open or close the next moment.
- `disconnect(username)` only asks the connection to close. The registry entry is removed when the close completes.
- The clients are receive-only by design.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Endpoint path | `/jpush`, fixed. One endpoint per application |
| Connections per signed-in user | 1 |
| Waiting line per client | 8 MiB of characters by default. A message is accepted when the line is empty, whatever its size |
| Secret key | At least 64 bytes |
| Java version | 21 or newer |
| Message content | Text. The sender name must not contain the null character, because the null character separates the two parts |
| Client to server | Not supported |

### C.4 Tuning

- Choose `maxQueuedCharsPerClient` from the size of your messages. Thousands of short notifications fit in the default. Lower it if you want slow clients dropped sooner.
- Keep messages short. Send an identifier or a one-line text, and let the page load the details from your API.
- Put the secret and the issuer in one place, used by both your login and `JPushServer`.
- In `useJPush`, pass `token` as a function, so a renewed token is used at the next reconnect.
- Raise `minDelayMs` and `jitter` for a large number of pages, so a restart is not followed by a wave of reconnects.
- Use `onOpen({ reconnected })` to refresh the data the page may have missed.
- Use `onBeforeSend` for rules and `onError` for monitoring.

## Appendix D. Message reference

### D.1 JPushServer

| Message | Cause |
|---------|-------|
| `tokenSource is required` | `start()` without `tokenSource` |
| `cookieName is required when tokenSource is COOKIE` | `COOKIE` without a cookie name, or a blank one |
| `HMAC512 secret key must contain at least 64 UTF-8 bytes` | The secret is too short (from jsecurity's `TokenProcessor`) |
| `maxQueuedCharsPerClient must be positive` | A limit of zero or less |
| `JPushServer.configure()...start() must be called before accepting connections` | A connection arrives before `start()` |
| `Outbound buffer of this client is full (limit N characters); the client is too slow and its connection is being closed` | Passed to `onError` when a client's waiting line is full |

An empty or invalid issuer is rejected by the `TokenProcessor` of jsecurity with its own message.

### D.2 JPushClient

| Message | Cause |
|---------|-------|
| `token cannot be blank` | A blank value in `token(...)`, or in `cookie(name, token)` |
| `cookieName cannot be blank` | A blank cookie name |
| `Call only one of .cookie(...) or .token(...), not both` | Both are set before `connect()` |
| `connectTimeout must be positive` | A null, zero or negative timeout |

### D.3 useJPush

| Message | Cause |
|---------|-------|
| `token must not be empty` | The token (or the value the function returns) is empty or not a string |
| `reconnect must be a boolean or an options object` | `reconnect` has another type |
| `reconnect.minDelayMs must be >= 0` | Negative first wait |
| `reconnect.maxDelayMs must be >= reconnect.minDelayMs` | The longest wait is shorter than the first |
| `reconnect.factor must be >= 1` | The factor is below 1 |
| `reconnect.jitter must be between 0 and 1` | Jitter outside the range |
| `reconnect.stableAfterMs must be >= 0` | Negative |
| `reconnect.maxAttempts must be >= 0` | Negative |
| `JPush connection error` | Passed to `onError` when the browser reports a socket error |

### D.4 Close reasons

| Reason | Code |
|--------|------|
| `replaced by a newer connection` | 1000 |
| `disconnected by server` | 1000 |
| `invalid or expired token` | 1008 |
| `token has no subject` | 1008 |
| `JPush endpoint must be mapped to "/jpush"` | 1008 |
| `slow consumer` | 1013 |
| `write failed` | 1013 |
