# jmeet

*J2OS Platform 2.4*

A video meeting in the browser looks simple: people appear in a window, talk, and share their screens. Behind it, the browsers have to find each other, agree on how to send sound and pictures, and keep doing that while people come and go. The media itself, the actual sound and video, does not need a server at all. Browsers can send it directly to each other with *WebRTC*, a technology built into every modern browser. What they cannot do alone is *introduce themselves*. Before two browsers can talk, they exchange a few small text messages through some go-between. This introduction is called *signaling*.

**jmeet** is the go-between and the browser code that uses it. It has three parts:

- `jmeet` (`org.j2os.platform.jmeet`) is a Java library for the server. It keeps the list of meetings and members, decides who may enter, and relays signaling messages over a *WebSocket* (a long-lived two-way connection between a page and the server). No audio or video passes through it.
- `useJMeet` is a Nuxt composable for the page. It joins a meeting, opens the microphone, camera or screen, shows the presenter, and can record the meeting.
- `useJMeetTranslator` is a second composable that listens to the speakers and shows what they say, translated into the language of the local user.

The whole server side, in a few lines:

```java
JMeet.configure()
        .issuer("j2os")
        .secretKey(secret)                      // the key that also signs your login tokens
        .tokenSource(TokenSource.COOKIE)
        .start();

JMeet.createMeeting("math-101", "sara");        // sara owns the meeting
JMeet.addMember("math-101", "sara", "ali");     // sara lets ali in
```

and the page, in a few more:

```js
const meet = useJMeet()
const session = meet.join(wsUrl, 'math-101', {
  videoElementId: 'main-video',                 // the <video> that shows the presenter
  onJoined: ({ username, isOwner }) => { /* the page may enable its buttons now */ }
})

await session.enableMicrophone()
await session.enableWebcam()
```

This guide goes step by step. The server chapters (2 to 12) come first and use small Java programs. The browser chapters (13 to 19) use short scripts. The last chapter puts the pieces together, and the appendices work as a reference.

## Contents

1. [How a meeting works](#1-how-a-meeting-works)
2. [Configuring the server](#2-configuring-the-server)
3. [Meetings and members](#3-meetings-and-members)
4. [Putting the endpoint on the web](#4-putting-the-endpoint-on-the-web)
5. [Trying the endpoint without a browser](#5-trying-the-endpoint-without-a-browser)
6. [Who may join](#6-who-may-join)
7. [Relaying negotiation messages](#7-relaying-negotiation-messages)
8. [One presenter at a time](#8-one-presenter-at-a-time)
9. [Reconnecting and second tabs](#9-reconnecting-and-second-tabs)
10. [Removing people and ending the meeting](#10-removing-people-and-ending-the-meeting)
11. [Hooks: gates and notifications](#11-hooks-gates-and-notifications)
12. [Slow connections](#12-slow-connections)
13. [The browser: joining a meeting](#13-the-browser-joining-a-meeting)
14. [Camera, screen and microphone](#14-camera-screen-and-microphone)
15. [Negotiation inside the browser](#15-negotiation-inside-the-browser)
16. [Leaving, ending and losing the connection](#16-leaving-ending-and-losing-the-connection)
17. [Recording](#17-recording)
18. [Live translation](#18-live-translation)
19. [Order, retries and recovery](#19-order-retries-and-recovery)
20. [Putting it together: a meeting in a web application](#20-putting-it-together-a-meeting-in-a-web-application)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The Java programs of the guide are complete: each has one `main` method and uses the `Peer` class of chapter 5. The browser scripts show the part that matters; the lines that prepare the stand-ins for browser features are left out (chapter 13 explains them).

---

## 1. How a meeting works

A meeting has an **id** chosen by the application, an **owner** (the member who created it), and **members** (the people allowed in). Being a member is not the same as being present: a member is *online* only while connected to the signaling endpoint.

Media travels in a **mesh**: every browser opens one direct connection to every other browser.

```text
      sara ─────── ali
        \         /
         \       /
           reza
```

Three people need three connections, four need six, ten need forty-five. A mesh is simple and needs no media server, but it is only suitable for small meetings, about a handful of people. For large conferences a server that forwards media (an *SFU*) is needed, and jmeet does not offer one.

Two rules shape the rest of the guide:

- **Audio is for everyone.** Every member can open the microphone at any time, and everybody hears everybody.
- **Video is for one.** At most one member at a time is the **presenter**, who shows either the *webcam* or the *desktop* (a screen, window or tab). Everyone else watches the presenter in a single video area.

Here is who does what when a person joins:

| Step | Browser | Server |
|------|---------|--------|
| 1 | Opens a WebSocket to `/jmeet?meeting_id=...` with its token | Checks the token, the meeting and the membership |
| 2 | Receives `existing-members`: who is online, who presents | Sends it to the newcomer only |
| 3 | Prepares one peer connection per online member | Tells the others `member-joined` |
| 4 | When someone starts sending sound or video, the two browsers exchange an *offer*, an *answer* and *ICE candidates* through the server | Relays each message to the member it is addressed to |
| 5 | Media flows directly between browsers | Not involved |

An *offer* and an *answer* are small descriptions ("I can send this kind of video, how about you?") of what each side wants to send and receive. *ICE candidates* are possible network routes between the two computers. The browsers create all of these themselves; the server only passes them on.

All meeting state lives in the memory of the server process. A restart ends every meeting, and two server instances do not share meetings (see Appendix C).

## 2. Configuring the server

Everything starts with one configuration call, usually at startup:

```java
JMeet.configure()
        .issuer("j2os")
        .secretKey(secret)
        .start();
```

`issuer` and `secretKey` are required. They create the object that reads login tokens (`TokenProcessor` of `jsecurity`), so they must be the **same issuer and key that sign the tokens of your users**. The key must be at least 64 bytes long. A meeting never creates tokens; it only reads them to learn who is connecting.

The other settings are optional:

| Setting | Default | Meaning |
|---------|---------|---------|
| `tokenSource(...)` | `AUTHORIZATION_HEADER_BEARER` | Where the endpoint looks for the token (chapter 6) |
| `cookieName(...)` | `token` | The cookie to read when the source is `COOKIE` |
| `onBeforeCreateMeeting(owner -> ...)` | allow | Can refuse the creation of a meeting (chapter 11) |
| `onBeforeMemberJoined((meeting, user) -> ...)` | allow | Can refuse a join (chapter 11) |
| `onMemberJoined`, `onMemberLeft`, `onMeetingEnded` | nothing | Notifications (chapter 11) |
| `maxQueuedCharsPerConnection(n)` | 4194304 | How much unsent text one connection may hold (chapter 12) |

`start()` applies all the changes together, so another thread never sees a half-finished configuration. Calling `configure()` again later changes only what it names; everything else stays:

```java
import org.j2os.platform.jmeet.JMeet;

public class Setup {

    public static void main(String[] args) {
        try {
            JMeet.createMeeting("m1", "sara");
        } catch (IllegalStateException e) {
            System.out.println("1. " + e.getMessage());
        }

        try {
            JMeet.configure().issuer("demo").start();
        } catch (IllegalStateException e) {
            System.out.println("2. " + e.getMessage());
        }

        try {
            JMeet.configure().issuer("demo").secretKey("too short").start();
        } catch (IllegalArgumentException e) {
            System.out.println("3. " + e.getMessage());
        }

        try {
            JMeet.configure().cookieName(" ");
        } catch (IllegalArgumentException e) {
            System.out.println("4. " + e.getMessage());
        }

        try {
            JMeet.configure().maxQueuedCharsPerConnection(0);
        } catch (IllegalArgumentException e) {
            System.out.println("5. " + e.getMessage());
        }

        JMeet.configure().issuer("demo").secretKey(Peer.KEY).start();
        JMeet.createMeeting("m1", "sara");
        System.out.println("6. configured, meeting created");

        // A later configure() changes only what it names.
        JMeet.configure().onMeetingEnded(id -> System.out.println("   ended: " + id)).start();
        JMeet.endMeeting("m1", "sara");
    }
}
```

```text
1. JMeet is not configured yet. Call JMeet.configure().issuer(...).secretKey(...)....start() before use.
2. issuer and secretKey must be set before start()
3. HMAC512 secret key must contain at least 64 UTF-8 bytes
4. cookieName must not be null or blank
5. maxQueuedCharsPerConnection must be positive
6. configured, meeting created
   ended: m1
```

- Line 1: before the first `start()`, every other call fails.
- Line 2: `start()` without a key fails, and a failed attempt changes nothing.
- Line 3: a key shorter than 64 bytes is refused by `TokenProcessor`.
- Lines 4 and 5: a blank cookie name and a non-positive limit are refused as soon as they are given, before `start()`.
- Line 6 and the last line: the second `configure()` adds a hook and keeps the issuer and the key.

A `Builder` is used by one thread, like a `StringBuilder`. The configuration is shared by the whole process, so one JVM cannot run two differently configured meeting systems.

In a Spring application the call lives in a component. The key comes from the environment, not from the source:

```java
@Component
public class JMeetConfig {

    @Value("${jmeet.secret}")
    private String secret;                         // at least 64 bytes, kept out of the source

    @PostConstruct
    void start() {
        JMeet.configure()
                .issuer("j2os")
                .secretKey(secret)
                .tokenSource(TokenSource.COOKIE)
                .cookieName("token")
                .start();
    }
}
```

## 3. Meetings and members

The application manages meetings with a few static methods of `JMeet`. This is usually done from REST endpoints (chapter 4), and every method that changes something takes **who is asking** as its second argument:

| Method | Does |
|--------|------|
| `createMeeting(id, owner)` | Creates a meeting with a chosen id. The owner is its first member |
| `addMember(id, owner, user)` | Allows `user` to enter. Does not connect anyone |
| `kickMember(id, owner, user)` | Removes a member and closes their connection |
| `endMeeting(id, owner)` | Ends the meeting for everybody |
| `owner(id)`, `members(id)`, `onlineMembers(id)` | Read the state |
| `meetingIds()` | Every active meeting id on this server |

The id is chosen by the caller, not generated. It can be tied to something the application already has, such as a class or an appointment number. Creating a meeting is atomic: if two requests race for one id, exactly one wins and the other gets `MeetingAlreadyExistsException`.

```java
import org.j2os.platform.jmeet.JMeet;
import org.j2os.platform.jmeet.exception.*;

public class Lifecycle {

    public static void main(String[] args) {
        JMeet.configure().issuer("demo").secretKey(Peer.KEY).start();

        JMeet.createMeeting("math-101", "sara");
        JMeet.addMember("math-101", "sara", "ali");
        System.out.println("members: " + JMeet.members("math-101").stream().sorted().toList());
        System.out.println("owner:   " + JMeet.owner("math-101"));

        try {
            JMeet.createMeeting("math-101", "ali");
        } catch (MeetingAlreadyExistsException e) {
            System.out.println("again:   " + e.getMessage());
        }
        try {
            JMeet.addMember("math-101", "ali", "reza");
        } catch (NotMeetingOwnerException e) {
            System.out.println("add:     " + e.getMessage());
        }
        try {
            JMeet.endMeeting("no-such-meeting", "sara");
        } catch (IllegalArgumentException e) {
            System.out.println("end:     " + e.getMessage());
        }

        JMeet.endMeeting("math-101", "sara");
        System.out.println("after end: " + JMeet.meetingIds());
        JMeet.createMeeting("math-101", "ali");
        System.out.println("id reused, new owner: " + JMeet.owner("math-101"));
    }
}
```

```text
members: [ali, sara]
owner:   sara
again:   A meeting with id math-101 already exists
add:     ali is not the owner of meeting math-101
end:     No meeting found with id no-such-meeting
after end: []
id reused, new owner: ali
```

- The failures are exceptions without a checked type: `MeetingAlreadyExistsException` for a repeated id, `NotMeetingOwnerException` when the caller is not the owner, and `IllegalArgumentException` when the meeting does not exist.
- After `endMeeting` the id is free again. The program creates a new meeting with the same id and a different owner. The library keeps no history of past meetings; an application that needs unique ids over time must track them itself.

Two more methods help when the token lives in an `httpOnly` cookie, which JavaScript cannot read. The server can look at it for the page:

```java
import org.j2os.platform.jmeet.JMeet;

public class Checks {

    public static void main(String[] args) {
        JMeet.configure().issuer("demo").secretKey(Peer.KEY).start();
        JMeet.createMeeting("m1", "sara");

        String sara = Peer.token("sara");
        String ali = Peer.token("ali");

        System.out.println("currentUsername(sara token)   = " + JMeet.currentUsername(sara));
        System.out.println("currentUsername(garbage)      = " + JMeet.currentUsername("garbage"));
        System.out.println("currentUsername(null)         = " + JMeet.currentUsername(null));
        System.out.println("isOwner(m1, sara token)       = " + JMeet.isOwner("m1", sara));
        System.out.println("isOwner(m1, ali token)        = " + JMeet.isOwner("m1", ali));
        System.out.println("isOwner(m1, garbage)          = " + JMeet.isOwner("m1", "garbage"));
        System.out.println("isOwner(no-such, sara token)  = " + JMeet.isOwner("no-such", sara));
    }
}
```

```text
currentUsername(sara token)   = sara
currentUsername(garbage)      = null
currentUsername(null)         = null
isOwner(m1, sara token)       = true
isOwner(m1, ali token)        = false
isOwner(m1, garbage)          = false
isOwner(no-such, sara token)  = false
```

Both return a plain answer and never throw for a bad token. They are information for the page, not a security gate; the real checks happen inside `JMeet` and the endpoint.

## 4. Putting the endpoint on the web

Two pieces are needed on the web side: a REST controller for the actions of chapter 3, and the WebSocket endpoint that members connect to.

**The endpoint.** `JMeetEndpoint` is an abstract class with no annotation of its own. The application extends it and decides the path:

```java
@Component
@ServerEndpoint(value = "/jmeet", configurator = JMeetEndpoint.HandshakeCapture.class)
public class MeetEndpoint extends JMeetEndpoint {
}
```

`HandshakeCapture` is required. A WebSocket session does not expose the headers, the cookies or the query string of the request that opened it, and the endpoint needs all three, so this small class copies them aside during the handshake.

Spring Boot with an embedded server does not scan for `@ServerEndpoint` classes by itself. A bean registers them:

```java
@Configuration
public class WebSocketConfig {

    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
}
```

**The controller.** Each action calls `JMeet` with the **verified identity of the caller**. In a project that uses `AuthorizationFilter` from `jsecurity`, that identity is the request attribute `authorization.subject`:

```java
@RestController
@RequestMapping("/meetings")
public class MeetingApi {

    @PostMapping("/{id}")
    public ResponseEntity<Void> create(@PathVariable String id, HttpServletRequest request) {
        try {
            JMeet.createMeeting(id, caller(request));
            return ResponseEntity.status(201).build();
        } catch (MeetingCreationRejectedException e) {
            return ResponseEntity.status(403).build();
        } catch (MeetingAlreadyExistsException e) {
            return ResponseEntity.status(409).build();
        }
    }

    @PutMapping("/{id}/members/{user}")
    public ResponseEntity<Void> add(@PathVariable String id, @PathVariable String user,
                                    HttpServletRequest request) {
        try {
            JMeet.addMember(id, caller(request), user);
            return ResponseEntity.noContent().build();
        } catch (NotMeetingOwnerException e) {
            return ResponseEntity.status(403).build();
        } catch (IllegalArgumentException e) {            // no such meeting
            return ResponseEntity.notFound().build();
        }
    }

    @DeleteMapping("/{id}/members/{user}")
    public ResponseEntity<Void> kick(@PathVariable String id, @PathVariable String user,
                                     HttpServletRequest request) {
        try {
            JMeet.kickMember(id, caller(request), user);
            return ResponseEntity.noContent().build();
        } catch (NotMeetingOwnerException e) {
            return ResponseEntity.status(403).build();
        } catch (OwnerCannotBeKickedException e) {
            return ResponseEntity.badRequest().build();
        } catch (IllegalArgumentException e) {
            return ResponseEntity.notFound().build();
        }
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> end(@PathVariable String id, HttpServletRequest request) {
        try {
            JMeet.endMeeting(id, caller(request));
            return ResponseEntity.noContent().build();
        } catch (NotMeetingOwnerException e) {
            return ResponseEntity.status(403).build();
        } catch (IllegalArgumentException e) {
            return ResponseEntity.notFound().build();
        }
    }

    /** The verified identity of the caller, never a name taken from the request body. */
    private String caller(HttpServletRequest request) {
        return (String) request.getAttribute("authorization.subject");
    }
}
```

The identity must come from the verified token and not from a request parameter or a body. `JMeet` compares the name it receives with the owner of the meeting, so a name chosen by the client would let anyone act as the owner.

The handshake of the WebSocket is an ordinary HTTP request, so it passes through the same filters as every other request. The endpoint checks the token itself, so decide whether the path `/jmeet` should be protected by the filter as well or left to the endpoint.

## 5. Trying the endpoint without a browser

A browser is not needed to see what the endpoint does. A **Peer** below plays one connection: it opens the endpoint the way a handshake would (with the real `HandshakeCapture`), sends text frames, and collects what the server sends back.

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import jakarta.websocket.CloseReason;
import jakarta.websocket.RemoteEndpoint;
import jakarta.websocket.Session;
import jakarta.websocket.server.HandshakeRequest;
import jakarta.websocket.server.ServerEndpointConfig;
import org.j2os.platform.jmeet.JMeetEndpoint;
import org.j2os.platform.jsecurity.token.TokenProcessor;

import java.net.URI;
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;

/** A browser stand-in: one object = one WebSocket connection to the meeting endpoint. */
public class Peer {

    public static class Endpoint extends JMeetEndpoint { }

    private static final ObjectMapper SORTED = new ObjectMapper()
            .configure(SerializationFeature.ORDER_MAP_ENTRIES_BY_KEYS, true);

    final String name;
    final List<String> inbox = new CopyOnWriteArrayList<>();
    volatile String closedWith;
    volatile CountDownLatch writeGate;          // when set, sendText blocks until it opens
    private final Endpoint endpoint = new Endpoint();

    private final Session session = new Session() {
        public void setMaxTextMessageBufferSize(int size) { }
        public void close() { closedWith = "closed"; }
        public void close(CloseReason r) {
            closedWith = r.getCloseCode().getCode() + " " + r.getReasonPhrase();
        }
        public RemoteEndpoint.Basic getBasicRemote() {
            return text -> {
                CountDownLatch gate = writeGate;
                if (gate != null) {
                    try { gate.await(); } catch (InterruptedException e) { return; }
                }
                inbox.add(sorted(text));
            };
        }
    };

    Peer(String name) { this.name = name; }

    /** Opens the connection the way a browser handshake would: query string plus headers. */
    Peer connect(String query, String... headerPairs) throws Exception {
        Map<String, List<String>> headers = new HashMap<>();
        for (int i = 0; i < headerPairs.length; i += 2) {
            headers.put(headerPairs[i], List.of(headerPairs[i + 1]));
        }
        HandshakeRequest request = new HandshakeRequest() {
            public Map<String, List<String>> getHeaders() { return headers; }
            public URI getRequestURI() { return URI.create("/jmeet" + (query.isEmpty() ? "" : "?" + query)); }
        };
        ServerEndpointConfig config = new ServerEndpointConfig() {
            private final Map<String, Object> props = new HashMap<>();
            public Map<String, Object> getUserProperties() { return props; }
        };
        new JMeetEndpoint.HandshakeCapture().modifyHandshake(config, request, null);
        endpoint.onOpen(session, config);
        return this;
    }

    /** Joins as {@code user} with a token in the access_token query parameter. */
    Peer join(String meetingId, String user) throws Exception {
        return connect("meeting_id=" + meetingId + "&access_token=" + token(user));
    }

    Peer say(String json) { endpoint.onMessage(json); return this; }
    Peer hangUp() { endpoint.onClose(); return this; }

    static final String KEY = "k".repeat(64);
    private static final TokenProcessor TOKENS = new TokenProcessor("demo", KEY);

    static String token(String user) {
        return TOKENS.createToken(user, Map.of(), 60L);
    }

    static void settle() throws InterruptedException { Thread.sleep(150); }

    /** Drops everything received so far. */
    Peer forget() throws InterruptedException {
        settle();
        inbox.clear();
        closedWith = null;
        return this;
    }

    /** Prints what this peer received since the last call, then forgets it. */
    void show() throws InterruptedException {
        settle();
        for (String frame : inbox) System.out.println(name + " <- " + frame);
        if (closedWith != null) System.out.println(name + " closed: " + closedWith);
        inbox.clear();
        closedWith = null;
    }

    private static String sorted(String json) {
        try { return SORTED.writeValueAsString(SORTED.readValue(json, Map.class)); }
        catch (Exception e) { return json; }
    }
}
```

- `join(meeting, user)` connects with a token in the `access_token` parameter, as a browser without cookies would. `connect(query, headers...)` gives full control.
- `say(json)` is a client sending a frame. `hangUp()` is the connection closing.
- `show()` prints what arrived since the last call, with the field names sorted so that the output is the same on every run. The order of fields in a JSON message carries no meaning.
- `KEY` is a demo key of 64 characters, and `token(user)` makes a token for a user with the issuer `demo`, which is what the programs configure.
- `settle()` waits briefly. The server writes to each connection from a thread of its own (chapter 12), so frames arrive a moment after the call that caused them.

The first meeting, with two people:

```java
import org.j2os.platform.jmeet.JMeet;

public class Joining {

    public static void main(String[] args) throws Exception {
        JMeet.configure().issuer("demo").secretKey(Peer.KEY).start();
        JMeet.createMeeting("math-101", "sara");
        JMeet.addMember("math-101", "sara", "ali");

        Peer sara = new Peer("sara").join("math-101", "sara");
        sara.show();

        Peer ali = new Peer("ali").join("math-101", "ali");
        ali.show();
        sara.show();

        System.out.println("online: " + JMeet.onlineMembers("math-101").stream().sorted().toList());
    }
}
```

```text
sara <- {"connectionIds":{},"isOwner":true,"type":"existing-members","usernames":[],"yourUsername":"sara"}
ali <- {"connectionIds":{"sara":1},"isOwner":false,"type":"existing-members","usernames":["sara"],"yourUsername":"ali"}
sara <- {"connectionId":2,"type":"member-joined","username":"ali"}
online: [ali, sara]
```

- Sara is the first to arrive. `existing-members` tells her she is the owner and that nobody else is online.
- Ali gets the same message, with sara in `usernames`. `connectionIds` gives the id of each online connection; chapter 9 explains what it is for.
- Sara hears `member-joined` about ali, with the id of ali's new connection.

## 6. Who may join

The endpoint closes a connection at once, with a reason, when the join is not allowed. The reason is visible to the page (the `reason` of the WebSocket close event). The code is always 1003 ("cannot accept"):

```java
import org.j2os.platform.jmeet.JMeet;

public class Rejected {

    public static void main(String[] args) throws Exception {
        JMeet.configure().issuer("demo").secretKey(Peer.KEY).start();
        JMeet.createMeeting("math-101", "sara");

        String good = Peer.token("sara");
        new Peer("no meeting id").connect("access_token=" + good).show();
        new Peer("no token").connect("meeting_id=math-101").show();
        new Peer("bad token").connect("meeting_id=math-101&access_token=abc.def").show();
        new Peer("unknown meeting").connect("meeting_id=zzz&access_token=" + good).show();
        new Peer("not a member").join("math-101", "reza").show();
        new Peer("bearer header").connect("meeting_id=math-101", "Authorization", "Bearer " + good).show();
        System.out.println("online: " + JMeet.onlineMembers("math-101"));
    }
}
```

```text
no meeting id closed: 1003 Missing meeting_id
no token closed: 1003 Missing token
bad token closed: 1003 Invalid token
unknown meeting closed: 1003 No such meeting
not a member closed: 1003 Not a member of this meeting
bearer header <- {"connectionIds":{},"isOwner":true,"type":"existing-members","usernames":[],"yourUsername":"sara"}
online: [sara]
```

The checks run in this order: `meeting_id` present, token present, token valid, meeting exists, meeting still running (`Meeting has ended` if it ended in the meantime), person is a member, and finally the `onBeforeMemberJoined` hook (chapter 11), which answers `Join rejected`.

Being a member is the real gate. A valid token for someone who was never added with `addMember` is turned away, and the id of a meeting alone is useless.

**Where the token comes from.** The `tokenSource` setting decides:

| Source | The endpoint reads | Typical client |
|--------|--------------------|----------------|
| `AUTHORIZATION_HEADER_BEARER` (default) | `Authorization: Bearer <token>`, and if there is no such header, the `access_token` query parameter | A native client, or a page that passes `token` to `join` |
| `COOKIE` | The cookie named by `cookieName` | A page whose login cookie is already set |

A browser cannot add headers to a WebSocket handshake. That is why bearer mode also accepts the query parameter. A token in a URL can end up in logs, so a browser application should prefer the cookie.

```java
import org.j2os.platform.jmeet.JMeet;
import org.j2os.platform.jmeet.TokenSource;

public class TokenSources {

    public static void main(String[] args) throws Exception {
        JMeet.configure().issuer("demo").secretKey(Peer.KEY).start();
        JMeet.createMeeting("m1", "sara");
        String token = Peer.token("sara");

        System.out.println("-- default: Authorization header or access_token --");
        new Peer("query").connect("meeting_id=m1&access_token=" + token).show();
        new Peer("cookie").connect("meeting_id=m1", "Cookie", "token=" + token).show();

        JMeet.configure().tokenSource(TokenSource.COOKIE).cookieName("session").start();

        System.out.println("-- cookie mode, cookie named session --");
        new Peer("query").connect("meeting_id=m1&access_token=" + token).show();
        new Peer("wrong name").connect("meeting_id=m1", "Cookie", "token=" + token).show();
        new Peer("right name").connect("meeting_id=m1", "Cookie", "theme=dark; session=" + token).show();
    }
}
```

```text
-- default: Authorization header or access_token --
query <- {"connectionIds":{},"isOwner":true,"type":"existing-members","usernames":[],"yourUsername":"sara"}
cookie closed: 1003 Missing token
-- cookie mode, cookie named session --
query closed: 1003 Missing token
wrong name closed: 1003 Missing token
right name <- {"connectionIds":{},"isOwner":true,"type":"existing-members","usernames":[],"yourUsername":"sara"}
```

In cookie mode the query parameter is ignored, and a cookie with another name does not count. The cookie header is split into its parts, so other cookies next to it do no harm.

For the cookie to be sent, the page and the signaling server must share a site (the same host name or a parent domain) and the cookie's `SameSite` setting must allow it.

## 7. Relaying negotiation messages

Three message types carry the introduction between two browsers: `offer`, `answer` and `ice-candidate`. The sender addresses a member with `to`. The server replaces `to` with `from`, adds the `connectionId` of the sending connection, and delivers the result to that one member:

```java
import org.j2os.platform.jmeet.JMeet;

public class Relay {

    public static void main(String[] args) throws Exception {
        JMeet.configure().issuer("demo").secretKey(Peer.KEY).start();
        JMeet.createMeeting("m1", "sara");
        JMeet.addMember("m1", "sara", "ali");
        JMeet.addMember("m1", "sara", "reza");

        Peer sara = new Peer("sara").join("m1", "sara");
        Peer ali = new Peer("ali").join("m1", "ali");
        sara.show(); ali.show();

        sara.say("{\"type\":\"offer\",\"to\":\"ali\",\"sdp\":{\"type\":\"offer\",\"sdp\":\"v=0...\"}}");
        ali.show();

        ali.say("{\"type\":\"answer\",\"to\":\"sara\",\"sdp\":\"v=0...\"}");
        sara.show();

        System.out.println("-- target offline, target unknown, garbage --");
        sara.say("{\"type\":\"offer\",\"to\":\"reza\",\"sdp\":\"x\"}");
        sara.say("{\"type\":\"ice-candidate\",\"candidate\":\"x\"}");
        sara.say("not json");
        sara.say("null");
        sara.say("{\"type\":\"dance\"}");
        sara.show(); ali.show();
        System.out.println("(nothing was delivered or answered)");
    }
}
```

```text
sara <- {"connectionIds":{},"isOwner":true,"type":"existing-members","usernames":[],"yourUsername":"sara"}
sara <- {"connectionId":2,"type":"member-joined","username":"ali"}
ali <- {"connectionIds":{"sara":1},"isOwner":false,"type":"existing-members","usernames":["sara"],"yourUsername":"ali"}
ali <- {"connectionId":1,"from":"sara","sdp":{"sdp":"v=0...","type":"offer"},"type":"offer"}
sara <- {"connectionId":2,"from":"ali","sdp":"v=0...","type":"answer"}
-- target offline, target unknown, garbage --
(nothing was delivered or answered)
```

- The content (`sdp`, `candidate`) is passed through untouched. Only browsers understand it.
- A message to someone who is not online is dropped, and so is everything the server cannot understand: text that is not JSON, the JSON `null`, a missing `to`, an unknown `type`. The sender gets no error for any of these.
- Messages that arrive before the join was accepted are ignored too.

## 8. One presenter at a time

A member asks to present with `start-presenting` and one of two types, `webcam` or `desktop`. The server keeps who presents and tells the others:

```java
import org.j2os.platform.jmeet.JMeet;

public class Presenter {

    public static void main(String[] args) throws Exception {
        JMeet.configure().issuer("demo").secretKey(Peer.KEY).start();
        JMeet.createMeeting("m1", "sara");
        JMeet.addMember("m1", "sara", "ali");
        JMeet.addMember("m1", "sara", "reza");

        Peer sara = new Peer("sara").join("m1", "sara");
        Peer ali = new Peer("ali").join("m1", "ali");
        sara.forget(); ali.forget();

        System.out.println("-- sara presents her webcam --");
        sara.say("{\"type\":\"start-presenting\",\"presentationType\":\"webcam\"}");
        sara.show(); ali.show();

        System.out.println("-- ali tries too, and sends a bad type --");
        ali.say("{\"type\":\"start-presenting\",\"presentationType\":\"desktop\"}");
        ali.say("{\"type\":\"start-presenting\",\"presentationType\":\"hologram\"}");
        sara.show(); ali.show();

        System.out.println("-- reza joins late and is told who presents --");
        Peer reza = new Peer("reza").join("m1", "reza");
        reza.show(); ali.show();

        System.out.println("-- sara drops while presenting --");
        sara.hangUp();
        ali.show(); reza.show();

        System.out.println("-- ali can now present --");
        ali.say("{\"type\":\"start-presenting\",\"presentationType\":\"desktop\"}");
        reza.show();
    }
}
```

```text
-- sara presents her webcam --
ali <- {"presentationType":"webcam","type":"presenter-changed","username":"sara"}
-- ali tries too, and sends a bad type --
ali <- {"code":"presenting-unavailable","message":"Another member is already presenting","type":"error"}
ali <- {"code":"invalid-presentation-type","message":"presentationType must be \"webcam\" or \"desktop\"","type":"error"}
-- reza joins late and is told who presents --
reza <- {"connectionIds":{"ali":2,"sara":1},"isOwner":false,"presentationType":"webcam","presenter":"sara","type":"existing-members","usernames":["sara","ali"],"yourUsername":"reza"}
ali <- {"connectionId":3,"type":"member-joined","username":"reza"}
-- sara drops while presenting --
ali <- {"connectionId":1,"type":"member-left","username":"sara"}
ali <- {"presentationType":"","type":"presenter-changed","username":"sara"}
reza <- {"connectionId":1,"type":"member-left","username":"sara"}
reza <- {"presentationType":"","type":"presenter-changed","username":"sara"}
-- ali can now present --
reza <- {"presentationType":"desktop","type":"presenter-changed","username":"ali"}
```

- The presenter does not receive a copy of its own `presenter-changed`.
- A second member who tries to present gets an `error` message with the code `presenting-unavailable`. The error goes to that member only. A type other than `webcam` or `desktop` gets `invalid-presentation-type`.
- A member who joins later is told who presents in `existing-members` (`presenter` and `presentationType`).
- When the presenter disconnects, the server sends `member-left` and then `presenter-changed` with an empty `presentationType`, which means "nobody presents". The place is free again.
- `stop-presenting` frees the place too. From someone who is not the presenter it does nothing.

The same member may ask again with the other type, for example to switch from the webcam to a shared screen. The server accepts it and tells the others the new type.

## 9. Reconnecting and second tabs

A username has **at most one connection per meeting**. When a connection arrives for a username that is already connected (a page refresh, a second tab, a reconnect after a network problem), the older connection is closed:

```java
import org.j2os.platform.jmeet.JMeet;

public class Reconnect {

    public static void main(String[] args) throws Exception {
        JMeet.configure().issuer("demo")
                .secretKey(Peer.KEY)
                .onMemberLeft((meeting, user) -> System.out.println("[hook] left: " + user))
                .start();
        JMeet.createMeeting("m1", "sara");
        JMeet.addMember("m1", "sara", "ali");

        Peer sara = new Peer("sara").join("m1", "sara");
        Peer ali = new Peer("ali").join("m1", "ali");
        ali.say("{\"type\":\"start-presenting\",\"presentationType\":\"webcam\"}");
        sara.forget(); ali.forget();

        System.out.println("-- ali opens a second tab (same user) --");
        Peer aliTab2 = new Peer("ali tab 2").join("m1", "ali");
        aliTab2.show(); sara.show(); ali.show();

        System.out.println("-- the old tab sends a message and then closes --");
        ali.say("{\"type\":\"offer\",\"to\":\"sara\",\"sdp\":\"stale\"}");
        ali.hangUp();
        sara.show();
        System.out.println("online: " + JMeet.onlineMembers("m1").stream().sorted().toList());
    }
}
```

```text
-- ali opens a second tab (same user) --
ali tab 2 <- {"connectionIds":{"sara":1},"isOwner":false,"type":"existing-members","usernames":["sara"],"yourUsername":"ali"}
sara <- {"presentationType":"","type":"presenter-changed","username":"ali"}
sara <- {"connectionId":3,"type":"member-joined","username":"ali"}
ali closed: closed
-- the old tab sends a message and then closes --
online: [ali, sara]
```

- The others hear `member-joined` with a **new** `connectionId`, and no `member-left` in between. A reconnect looks like a join.
- If the old connection was presenting, the place is released first (`presenter-changed` with an empty type), because the new connection has no media yet.
- The old tab is closed, and its last message (the stale offer) is ignored: the server checks that a message really comes from the *current* connection of that user.
- Nothing is reported to the `onMemberLeft` hook for the old connection, because the member never left. The final line shows ali still online. The `onMemberJoined` hook, on the other hand, runs again for the new connection.

The `connectionId` is a number that grows with every connection on the server. Browsers use it to tell a message from the current connection of a person apart from a late message of the previous one (chapter 15).

## 10. Removing people and ending the meeting

The owner can remove a member with `kickMember`, and end everything with `endMeeting`. The same two actions are possible from the page of the owner: a frame `{"type":"end-meeting"}` is the same as `endMeeting` and is ignored for anyone who is not the owner.

```java
import org.j2os.platform.jmeet.JMeet;
import org.j2os.platform.jmeet.exception.OwnerCannotBeKickedException;

public class KickEnd {

    public static void main(String[] args) throws Exception {
        JMeet.configure().issuer("demo").secretKey(Peer.KEY).start();
        JMeet.createMeeting("m1", "sara");
        JMeet.addMember("m1", "sara", "ali");
        JMeet.addMember("m1", "sara", "reza");

        Peer sara = new Peer("sara").join("m1", "sara");
        Peer ali = new Peer("ali").join("m1", "ali");
        Peer reza = new Peer("reza").join("m1", "reza");
        reza.say("{\"type\":\"start-presenting\",\"presentationType\":\"desktop\"}");
        sara.forget(); ali.forget(); reza.forget();

        System.out.println("-- the owner kicks reza, who is presenting --");
        JMeet.kickMember("m1", "sara", "reza");
        sara.show(); ali.show(); reza.show();
        System.out.println("members: " + JMeet.members("m1").stream().sorted().toList());
        new Peer("reza again").join("m1", "reza").show();

        System.out.println("-- the owner cannot be kicked --");
        try {
            JMeet.kickMember("m1", "sara", "sara");
        } catch (OwnerCannotBeKickedException e) {
            System.out.println(e.getMessage());
        }

        System.out.println("-- a non-owner sends end-meeting, then the owner does --");
        ali.say("{\"type\":\"end-meeting\"}");
        System.out.println("meetings: " + JMeet.meetingIds());
        sara.say("{\"type\":\"end-meeting\"}");
        sara.show(); ali.show();
        System.out.println("meetings: " + JMeet.meetingIds());
    }
}
```

```text
-- the owner kicks reza, who is presenting --
sara <- {"connectionId":3,"type":"member-left","username":"reza"}
sara <- {"presentationType":"","type":"presenter-changed","username":"reza"}
ali <- {"connectionId":3,"type":"member-left","username":"reza"}
ali <- {"presentationType":"","type":"presenter-changed","username":"reza"}
reza closed: closed
members: [ali, sara]
reza again closed: 1003 Not a member of this meeting
-- the owner cannot be kicked --
The meeting owner cannot be kicked — call endMeeting instead
-- a non-owner sends end-meeting, then the owner does --
meetings: [m1]
sara <- {"type":"meeting-ended"}
sara closed: closed
ali <- {"type":"meeting-ended"}
ali closed: closed
meetings: []
```

- A kick removes the membership and closes the connection. The kicked page learns it only through `onDisconnected`, because the close carries no reason. The others hear `member-left`, and a `presenter-changed` with an empty type if the kicked member was presenting, so the place is not stuck.
- The kicked person can no longer come back: the next join closes with `Not a member of this meeting`. The owner can add the name again.
- The owner cannot be kicked (`OwnerCannotBeKickedException`); `endMeeting` is the only way to remove them.
- `end-meeting` from ali had no effect. From sara, every connected member received `{"type":"meeting-ended"}` and the meeting was removed. The close happens only after that last message has been written, so it is not lost.
- Ending twice is harmless: only the call that actually removed the meeting fires the hooks.

## 11. Hooks: gates and notifications

Five hooks connect the library to the rest of the application. Two are **gates** that return `boolean`, three are **notifications**.

| Hook | Kind | When |
|------|------|------|
| `onBeforeCreateMeeting(owner)` | gate | In `createMeeting`. `false` throws `MeetingCreationRejectedException` |
| `onBeforeMemberJoined(meeting, user)` | gate | When a member connects. `false` closes the connection with `Join rejected` |
| `onMemberJoined(meeting, user)` | notification | After a successful join |
| `onMemberLeft(meeting, user)` | notification | After a connection of a member ends for any reason, a kick and the end of the meeting included |
| `onMeetingEnded(meeting)` | notification | After a meeting has ended and was removed |

```java
import org.j2os.platform.jmeet.JMeet;
import org.j2os.platform.jmeet.exception.MeetingCreationRejectedException;

public class Hooks {

    public static void main(String[] args) throws Exception {
        JMeet.configure().issuer("demo").secretKey(Peer.KEY)
                .onBeforeCreateMeeting(owner -> !owner.equals("guest"))
                .onBeforeMemberJoined((meeting, user) -> !user.equals("reza"))
                .onMemberJoined((meeting, user) -> System.out.println("[joined]  " + user))
                .onMemberLeft((meeting, user) -> System.out.println("[left]    " + user))
                .onMeetingEnded(meeting -> System.out.println("[ended]   " + meeting))
                .start();

        try {
            JMeet.createMeeting("m1", "guest");
        } catch (MeetingCreationRejectedException e) {
            System.out.println(e.getMessage());
        }

        JMeet.createMeeting("m1", "sara");
        JMeet.addMember("m1", "sara", "ali");
        JMeet.addMember("m1", "sara", "reza");

        Peer sara = new Peer("sara").join("m1", "sara");
        Peer ali = new Peer("ali").join("m1", "ali");
        new Peer("reza").join("m1", "reza").show();
        ali.hangUp();
        System.out.println("-- the owner ends the meeting --");
        JMeet.endMeeting("m1", "sara");
    }
}
```

```text
Meeting creation was rejected for owner guest
[joined]  sara
[joined]  ali
reza closed: 1003 Join rejected
[left]    ali
-- the owner ends the meeting --
[left]    sara
[ended]   m1
```

- Reza is a member, but the gate says no: his connection closes with `Join rejected`, and `onMemberJoined` never fires for him.
- When the meeting ends, `onMemberLeft` runs for each member who was online, and then `onMeetingEnded` runs once.
- A superseded connection (chapter 9) does not fire `onMemberLeft`.

Gates are the place for rules of the application: a limit on how many meetings one user may own, a check that a student paid for the course, a limit on the number of people online. Keep the hooks quick. The join gate runs while the meeting is locked, so a slow hook delays every other action on that meeting, and the other hooks block the thread that triggers them.

## 12. Slow connections

Sending to one slow browser must never slow the others. For this, each connection has a **queue of its own** and a **writer thread of its own**. The code that handles a meeting only puts the text in the queue and goes on; the writer sends it to the network.

The queue is limited by the number of characters waiting. A member whose connection cannot keep up and passes the limit is disconnected (WebSocket code 1013, reason `slow consumer`) and the rest of their queue is dropped. The default limit is 4194304 characters, about four million. A message is always accepted when the queue is empty, however large it is.

```java
import org.j2os.platform.jmeet.JMeet;

import java.util.concurrent.CountDownLatch;

public class SlowClient {

    public static void main(String[] args) throws Exception {
        JMeet.configure().issuer("demo").secretKey(Peer.KEY)
                .maxQueuedCharsPerConnection(300)
                .start();
        JMeet.createMeeting("m1", "sara");
        JMeet.addMember("m1", "sara", "ali");
        JMeet.addMember("m1", "sara", "reza");

        Peer sara = new Peer("sara").join("m1", "sara");
        Peer reza = new Peer("reza").join("m1", "reza");

        Peer ali = new Peer("ali");
        ali.writeGate = new CountDownLatch(1);          // ali's network stalls
        ali.join("m1", "ali");

        String offer = "{\"type\":\"offer\",\"to\":\"ali\",\"sdp\":\"" + "x".repeat(100) + "\"}";
        for (int i = 1; i <= 3; i++) {
            sara.say(offer);
            Peer.settle();
            System.out.println("after offer " + i + ": ali " + (ali.closedWith == null ? "still connected" : "closed with " + ali.closedWith));
        }

        System.out.println("online before the container reports the close: " + JMeet.onlineMembers("m1").stream().sorted().toList());
        ali.hangUp();                                    // the container's onClose
        System.out.println("online after:  " + JMeet.onlineMembers("m1").stream().sorted().toList());

        System.out.println("-- one huge message to an idle member is still accepted --");
        String huge = "{\"type\":\"offer\",\"to\":\"reza\",\"sdp\":\"" + "y".repeat(2000) + "\"}";
        Peer.settle();                                   // reza's queue is empty now
        sara.say(huge);
        Peer.settle();
        System.out.println("reza received " + reza.inbox.get(reza.inbox.size() - 1).length() + " characters");
        System.out.println("reza closed: " + reza.closedWith);
    }
}
```

```text
after offer 1: ali still connected
after offer 2: ali closed with 1013 slow consumer
after offer 3: ali closed with 1013 slow consumer
online before the container reports the close: [ali, reza, sara]
online after:  [reza, sara]
-- one huge message to an idle member is still accepted --
reza received 2056 characters
reza closed: null
```

Ali's network stalls, and the program lowers the limit to 300 characters so that this happens quickly. The second offer is too much for ali's queue, and he is closed. The container then reports the close, and the library takes ali off the online list. The huge message to reza, whose queue is empty, is accepted.

Warnings of this kind go to the Java logger:

```text
WARNING: Closing member connection 3: more than 300 characters are waiting to be written to it
```

The default is generous. Lower it only if memory is tight. A connection that closes with `slow consumer` needs a client that reconnects (chapter 16).

## 13. The browser: joining a meeting

On the page, everything goes through `useJMeet()`. The server side of the code is not needed to understand it; the scripts below play the part of the server by feeding messages to the composable.

**Playing the server.** The composable needs browser features that Node does not have. For the scripts, small stand-ins provide them: `FakeWS` is a WebSocket whose `receive(message)` plays a message that arrives from the server and whose `sent` lists what the page sent; a peer connection that accepts tracks; and cameras and microphones that return empty tracks. The listings leave out these preparation lines (`import` lines for the stand-ins), and call `sleep(ms)` to wait.

`join` is the main call:

```js
const session = meet.join(url, meetingId, options)
```

It returns a **session** object right away, and the connection opens in the background. It returns `null` on the server during server-side rendering, so call it from `onMounted` or from a click handler.

The address is the WebSocket URL of the endpoint, normally kept in the Nuxt runtime configuration, with `wss://` in production:

```js
// nuxt.config.js
export default defineNuxtConfig({
  runtimeConfig: {
    public: { jmeetWsUrl: 'wss://meet.example.com/jmeet' }
  }
})
```

`join` adds `meeting_id=` (encoded) to the address, and `access_token=` when `token` is given. Leave `token` out in cookie mode.

| Option | Meaning |
|--------|---------|
| `token` | A token to send as `access_token`. Must not be blank when given |
| `videoElementId` | The `id` of a `<video>` element. The composable shows the presenter there |
| `iceServers` | Network helpers for WebRTC (chapter 15). The default is one public STUN server |
| `onJoined({ username, isOwner })` | The server accepted this page. Time to enable the buttons |
| `onMemberJoined(username)` | Someone is present. Called once for each existing member and again for each newcomer |
| `onMemberLeft(username)` | Someone left |
| `onPresenterChanged(username, type)` | `type` is `webcam`, `desktop`, or `null` when the presentation ended |
| `onOwnPresentingEnded()` | This page's own presentation ended without the page asking (chapter 14) |
| `onMeetingEnded()` | The owner ended the meeting |
| `onDisconnected()` | The connection ended unexpectedly (chapter 16) |
| `onError(error)` | Something went wrong; the meeting may well continue |

Every option except `token`, `videoElementId` and `iceServers` is a callback, and every callback is optional.

```js
const meet = useJMeet()
const session = meet.join('ws://localhost:8082/jmeet', 'math-101', {
  token: 'abc 123',
  videoElementId: 'main-video',
  onJoined: me => console.log('onJoined', JSON.stringify(me)),
  onMemberJoined: u => console.log('onMemberJoined', u),
  onMemberLeft: u => console.log('onMemberLeft', u),
  onPresenterChanged: (u, t) => console.log('onPresenterChanged', u, t),
  onMeetingEnded: () => console.log('onMeetingEnded'),
  onDisconnected: () => console.log('onDisconnected'),
  onError: e => console.log('onError', e.message)
})

const ws = FakeWS.all[0]
console.log('url:', ws.url)
ws.open()

await ws.receive({ type: 'existing-members', yourUsername: 'ali', isOwner: false,
                   usernames: ['sara'], connectionIds: { sara: 1 },
                   presenter: 'sara', presentationType: 'webcam' })
await ws.receive({ type: 'member-joined', username: 'reza', connectionId: 3 })
await ws.receive({ type: 'presenter-changed', username: 'sara', presentationType: '' })
await ws.receive({ type: 'member-left', username: 'reza', connectionId: 3 })
console.log('isJoined:', meet.isJoined())
await ws.receive({ type: 'meeting-ended' })
console.log('isJoined:', meet.isJoined())
```

```text
url: ws://localhost:8082/jmeet?meeting_id=math-101&access_token=abc%20123
onMemberJoined sara
onJoined {"username":"ali","isOwner":false}
onPresenterChanged sara webcam
onMemberJoined reza
onPresenterChanged sara null
onMemberLeft reza
isJoined: true
onMeetingEnded
isJoined: false
```

- The address has the meeting id and the token, encoded.
- `onMemberJoined` for the people already present comes **before** `onJoined`. The page learns about its own identity last.
- If somebody is already presenting, `onPresenterChanged(username, type)` fires right after `onJoined`.
- The list of participants that the callbacks give does **not** contain the local user. The page adds its own name in `onJoined`.
- When the presenter leaves, `onPresenterChanged(username, null)` fires without any help from the server's `presenter-changed`.
- `meeting-ended` leads to `onMeetingEnded` only. There is no `onDisconnected` after it, because that is not a failure.

**Joining twice.** A second `join` for the same meeting returns the session that already exists, and its callbacks are ignored. A `join` for another meeting throws until `leave()` is called.

**Permissions first.** The browser asks for the microphone and the camera at the first use. `requestPermissionsUpfront()` asks for both once, then releases them, so that the prompt does not appear in the middle of the meeting. It ignores a refusal, and if the computer has no camera the combined request fails and is ignored.

A minimal state for a page, to be filled in by the later chapters:

```js
const participants = ref([])          // usernames, including me
const presenter = ref(null)           // username of the presenter, or null
const iAmPresenting = ref(false)
const isReady = ref(false)

const meet = useJMeet()
let session = null

async function enter(meetingId) {
  await meet.requestPermissionsUpfront()
  session = meet.join(useRuntimeConfig().public.jmeetWsUrl, meetingId, {
    videoElementId: 'main-video',
    onJoined: ({ username }) => {
      participants.value = [username, ...participants.value.filter(u => u !== username)]
      isReady.value = true
    },
    onMemberJoined: (u) => { if (!participants.value.includes(u)) participants.value.push(u) },
    onMemberLeft: (u) => { participants.value = participants.value.filter(x => x !== u) },
    onPresenterChanged: (u, type) => { presenter.value = type ? u : null; iAmPresenting.value = false },
    onOwnPresentingEnded: () => { iAmPresenting.value = false },
    onMeetingEnded: () => navigateTo('/'),
    onDisconnected: () => { isReady.value = false; session = null; navigateTo('/') },
    onError: (e) => console.warn(e.message)
  })
}

async function toggleWebcam() {
  if (iAmPresenting.value) {
    session.stopPresenting()
    iAmPresenting.value = false
    return
  }
  try {
    await session.enableWebcam()
    iAmPresenting.value = true
    presenter.value = null
  } catch (e) {
    alert(e.message)
  }
}

onBeforeUnmount(() => meet.leave())
```

```html
<video id="main-video" autoplay playsinline />
```

The `<video>` element needs `autoplay` and `playsinline`. The composable assigns the stream to it; the page never sets `srcObject`.

## 14. Camera, screen and microphone

The session object holds the actions.

| Call | Does |
|------|------|
| `enableWebcam()` | Asks for the camera, then becomes the presenter with `webcam`. Returns the stream |
| `enableDesktopSharing()` | The same for a screen, window or tab, chosen by the user in the browser's own dialog |
| `stopPresenting()` | Stops the camera or screen and frees the presenter's place |
| `enableMicrophone()` | First call: opens the microphone and returns `true`. Next calls: **mute / unmute**, returning the new state |
| `isMicrophoneEnabled()` | The current state |
| `endMeeting()` | Sends `end-meeting`. Works for the owner only |
| `muteVideoPlayer(boolean)` | Mutes or unmutes the `<video>` element |

```js
const meet = useJMeet()
const session = meet.join('ws://x/jmeet', 'm1', {
  videoElementId: 'main-video',
  onPresenterChanged: (u, t) => console.log('  onPresenterChanged', u, t),
  onOwnPresentingEnded: () => console.log('  onOwnPresentingEnded'),
  onError: e => console.log('  onError:', e.message)
})
const ws = FakeWS.all[0]
ws.open()
await ws.receive({ type: 'existing-members', yourUsername: 'ali', isOwner: false, usernames: ['sara'], connectionIds: { sara: 1 } })
const sent = () => ws.sent.splice(0).map(m => m.type + (m.presentationType ? ':' + m.presentationType : ''))

console.log('1. ali shares his webcam')
await session.enableWebcam()
console.log('  sent', sent(), '| preview shown:', videoEl.srcObject !== null)

console.log('2. ali stops')
session.stopPresenting()
console.log('  sent', sent(), '| preview shown:', videoEl.srcObject !== null)

console.log('3. sara starts presenting, then ali tries the webcam')
await ws.receive({ type: 'presenter-changed', username: 'sara', presentationType: 'desktop' })
try { await session.enableWebcam() } catch (e) { console.log('  enableWebcam threw:', e.message) }
console.log('  sent', sent())

console.log('4. sara stops; ali presents, the server says no (a race was lost)')
await ws.receive({ type: 'presenter-changed', username: 'sara', presentationType: '' })
await session.enableWebcam()
console.log('  sent', sent(), '| preview shown:', videoEl.srcObject !== null)
await ws.receive({ type: 'error', code: 'presenting-unavailable', message: 'Another member is already presenting' })
console.log('  preview shown:', videoEl.srcObject !== null)

console.log('5. ali presents again, then sara takes over')
await session.enableWebcam()
sent()
await ws.receive({ type: 'presenter-changed', username: 'sara', presentationType: 'webcam' })
console.log('  preview shown:', videoEl.srcObject !== null)
```

```text
1. ali shares his webcam
  sent [ 'start-presenting:webcam' ] | preview shown: true
2. ali stops
  sent [ 'stop-presenting' ] | preview shown: false
3. sara starts presenting, then ali tries the webcam
  onPresenterChanged sara desktop
  enableWebcam threw: Someone else is already presenting in this meeting
  sent []
4. sara stops; ali presents, the server says no (a race was lost)
  onPresenterChanged sara null
  sent [ 'start-presenting:webcam' ] | preview shown: true
  onOwnPresentingEnded
  onError: Another member is already presenting
  preview shown: false
5. ali presents again, then sara takes over
  onPresenterChanged sara webcam
  preview shown: false
```

1. A presentation starts: the stream is shown in the video element as a preview (muted, to avoid an echo) and `start-presenting` goes to the server.
2. `stopPresenting()` ends it and clears the preview.
3. When another member presents, `enableWebcam()` **throws** at once, with no frame sent. The page should disable the button when somebody else presents (`presenter !== null` in the state of chapter 13).
4. A race is possible: two members press the button at the same moment. The server accepts one and sends `presenting-unavailable` to the other. The composable undoes its preview and reports it through `onOwnPresentingEnded` and `onError`. The page must reset its own button state in `onOwnPresentingEnded`.
5. When the server announces another presenter, the local preview stops and the page is told through `onPresenterChanged`; there is no `onOwnPresentingEnded` for this. The page resets its state in `onPresenterChanged`, which in chapter 13 sets `iAmPresenting` to false.

`onOwnPresentingEnded` also fires when the person stops sharing with the browser's own "Stop sharing" bar: the camera or screen track ends and the composable stops presenting on its own.

Calling `enableDesktopSharing()` while the camera is on switches the source. The old track is released after the new one has replaced it for everybody.

**The microphone** is a toggle:

```js
const meet = useJMeet()
const session = meet.join('ws://x/jmeet', 'm1', { onError: e => console.log('onError:', e.message) })
const ws = FakeWS.all[0]
ws.open()
await ws.receive({ type: 'existing-members', yourUsername: 'ali', usernames: ['sara'], connectionIds: { sara: 1 } })

console.log('microphone on?', session.isMicrophoneEnabled())
console.log('first call  ->', await session.enableMicrophone(), '| tracks sent to sara:', FakePC.all[0].senders.length)
console.log('second call ->', await session.enableMicrophone(), '(muted, same track)')
console.log('third call  ->', await session.enableMicrophone(), '(unmuted)')
console.log('isMicrophoneEnabled:', session.isMicrophoneEnabled())

console.log('--- permission denied ---')
media.denied = true
try { await session.enableWebcam() } catch (e) { console.log('enableWebcam threw:', e.message) }
media.denied = false

console.log('--- audio of a remote member ---')
const heard = []
session.onAudioSourceAdded((key) => heard.push('added ' + key))
session.onAudioSourceRemoved((key) => heard.push('removed ' + key))
FakePC.all[0].ontrack({ track: { kind: 'audio' }, streams: [{ id: 's' }] })
await sleep()
console.log(heard, '| members with audio:', [...session.getMemberAudioStreams().keys()])
await ws.receive({ type: 'member-left', username: 'sara', connectionId: 1 })
console.log(heard)
```

```text
microphone on? false
first call  -> true | tracks sent to sara: 1
second call -> false (muted, same track)
third call  -> true (unmuted)
isMicrophoneEnabled: true
--- permission denied ---
enableWebcam threw: Permission denied
--- audio of a remote member ---
[ 'added sara' ] | members with audio: [ 'sara' ]
[ 'added sara', 'removed sara' ]
```

- The first call opens the microphone and adds a track to every peer connection. After that, a call only turns the track on or off; it stays open, so unmuting is instant.
- A refused permission is an exception from `enableWebcam()` or `enableMicrophone()` with the message of the browser. Catch it and tell the user.
- Voices of the other members play through audio elements that the composable creates. `onAudioSourceAdded` and `onAudioSourceRemoved` tell when a voice starts and ends, and `getMemberAudioStreams()` lists the streams by username. These are what the recording (chapter 17) and the translator (chapter 18) use. The local microphone is announced under the key `__self__`, not under a user name.

`muteVideoPlayer(true)` mutes only the `<video>` element. The voices of the members do not go through it, so it does not silence them. To silence a voice locally, disable its tracks: `stream.getAudioTracks().forEach(t => t.enabled = false)`.

A real page connects the state and the buttons like this:

```html
<video id="main-video" autoplay playsinline />

<UButton :label="iAmPresenting ? 'Stop camera' : 'Camera'"
         :disabled="presenter !== null"
         @click="toggleWebcam" />
```

## 15. Negotiation inside the browser

Chapter 7 showed the server's half. This chapter shows what the composable does with the messages, because it explains several behaviors of the page.

**One peer connection per member.** On `existing-members` and on each `member-joined`, the composable creates an `RTCPeerConnection` for that member. Nothing is sent yet. A connection is *negotiated* when somebody adds a track to it: the browser then asks for an offer, the offer travels through the server, the other side answers, and ICE candidates follow. A member who sends no media yet therefore causes no traffic, and as soon as one side turns on the microphone, the two sides connect.

**Two offers at once.** Both sides may add tracks at the same moment and send offers that cross each other. The composable uses the *polite peer* rule: for each pair, the member whose name is smaller in alphabetical order is *polite*. A polite peer drops its own offer and accepts the incoming one; the other one ignores the incoming offer. The pair settles without help from the server, and without anybody having to decide.

**Candidates that arrive early.** ICE candidates can arrive before the offer they belong to. They are kept and applied once the connection has its remote description. For each member, the incoming messages are processed one after another, never in parallel.

**Old messages.** Chapter 9 explained that a reconnect gives a person a new `connectionId`. The composable remembers the id of each member's current connection (from `existing-members` or `member-joined`), and ignores offers, answers and candidates that carry a different id:

```js
useJMeet().join('ws://x/jmeet', 'm1', { onError: e => console.log('onError:', e.message) })
const ws = FakeWS.all[0]
ws.open()
await ws.receive({ type: 'existing-members', yourUsername: 'ali', usernames: ['sara'], connectionIds: { sara: 5 } })
console.log('peer connections:', FakePC.all.length)

const answers = () => ws.sent.splice(0).filter(m => m.type === 'answer').length
const offer = (id) => ws.receive({ type: 'offer', from: 'sara', connectionId: id, sdp: { type: 'offer', sdp: 'x' } })

await offer(5);  console.log('offer from connection 5 -> answers sent:', answers())
await offer(4);  console.log('offer from connection 4 -> answers sent:', answers(), '(older connection of sara)')

console.log('sara reconnects as connection 7')
await ws.receive({ type: 'member-joined', username: 'sara', connectionId: 7 })
console.log('peer connections:', FakePC.all.length, '| first one closed:', FakePC.all[0].closed)
await offer(5);  console.log('offer from connection 5 -> answers sent:', answers())
await offer(7);  console.log('offer from connection 7 -> answers sent:', answers())
```

```text
peer connections: 1
offer from connection 5 -> answers sent: 1
offer from connection 4 -> answers sent: 0 (older connection of sara)
sara reconnects as connection 7
peer connections: 2 | first one closed: true
offer from connection 5 -> answers sent: 0
offer from connection 7 -> answers sent: 1
```

After the reconnect the first peer connection is closed and a new one is created, and the late offer from connection 5 is ignored.

**Failed connections.** When the browser reports that a connection has definitively *failed* (not merely "disconnected", which often heals itself), the composable asks for an *ICE restart*, which triggers a new offer.

**ICE servers.** Two browsers behind home routers often cannot reach each other directly. A *STUN* server helps a browser learn its public address. When even that is not enough, for example behind a strict company firewall, a *TURN* server relays the media. The default is one public STUN server. A list given in `iceServers` **replaces** the default, so include a STUN entry again if you want one:

```js
meet.join(wsUrl, meetingId, {
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'turn:turn.example.com:3478', username: 'meet', credential: secretFromYourServer }
  ]
})
```

TURN is an extra server that you run or rent. It carries the media itself, so it needs bandwidth. The credential belongs to the TURN server, not to the meeting; it is best created by your own server for each user.

## 16. Leaving, ending and losing the connection

There are five ways for a session to end, and the page learns about each in a different way:

| How | Callbacks |
|-----|-----------|
| The page calls `leave()` | None. The state is cleared at once |
| The owner ends the meeting | `onMeetingEnded` |
| The server closes the connection with a reason (rejection, `slow consumer`) | `onError` with the reason, then `onDisconnected` |
| The server closes the connection normally, without a reason (a kick, for example) | `onDisconnected` only |
| The network drops | `onDisconnected`, usually after an `onError` |

```js
const handlers = {
  onDisconnected: () => console.log('  onDisconnected'),
  onMeetingEnded: () => console.log('  onMeetingEnded'),
  onError: e => console.log('  onError:', e.message)
}

console.log('1. joining twice')
const meet = useJMeet()
const first = meet.join('ws://x/jmeet', 'math-101', handlers)
console.log('  same meeting returns the same session:', meet.join('ws://x/jmeet', 'math-101', handlers) === first)
try { meet.join('ws://x/jmeet', 'physics-7', handlers) } catch (e) { console.log('  other meeting threw:', e.message) }

console.log('2. leave() is silent')
meet.leave()
await sleep()
console.log('  isJoined:', meet.isJoined(), '| socket state:', FakeWS.all[0].readyState)

console.log('3. the server closes the connection')
meet.join('ws://x/jmeet', 'math-101', handlers)
FakeWS.all[1].open()
FakeWS.all[1].drop(1003, 'Not a member of this meeting')
console.log('  isJoined:', meet.isJoined())

console.log('4. normal close (code 1000) and a bad token')
meet.join('ws://x/jmeet', 'math-101', handlers)
FakeWS.all[2].drop(1000, '')
try { meet.join('ws://x/jmeet', 'math-101', { token: '  ' }) } catch (e) { console.log('  empty token threw:', e.message) }
try { meet.join('', 'math-101') } catch (e) { console.log('  empty url threw:', e.message) }
```

```text
1. joining twice
  same meeting returns the same session: true
  other meeting threw: Already joined meeting "math-101" — call leave() before joining a different meeting
2. leave() is silent
  isJoined: false | socket state: 3
3. the server closes the connection
  onError: JMeet connection closed: Not a member of this meeting
  onDisconnected
  isJoined: false
4. normal close (code 1000) and a bad token
  onDisconnected
  empty token threw: token must not be empty
  empty url threw: useJMeet: url must not be empty (check runtimeConfig.public.jmeetWsUrl)
```

1. A second `join` for the same meeting returns the same session. `join` for another meeting throws, with a message that tells to call `leave()` first.
2. `leave()` closes the socket, the peer connections and the microphone or camera. It is safe to call at any time; the composable also calls it when the component that created it is removed.
3. When the server closes the connection with a reason, the reason arrives in `onError` and `onDisconnected` follows.
4. A normal close (code 1000) calls `onDisconnected` without an error. A blank token and an empty address are refused with an exception before any connection is made.

`onDisconnected` means "this session is gone for good". The composable does **not** reconnect by itself. The page decides: go back to the home page, or call `join` again after a short pause (with growing pauses, to avoid a storm of reconnections after a server restart). A reconnect is like a first join, so the page must clear its list of participants first. It also loses the presenter's place and the microphone, which must be switched on again.

In the callback, reset everything that depends on the session: the flag that enables the buttons, the reference to the session, a running translation, a running recording.

## 17. Recording

The composable can record the meeting **on the user's computer**, and nothing is uploaded. `startRecording()` asks the user to pick a screen, a window or a tab (the browser's own dialog), records it, and mixes into the sound everything that is being said in the meeting: the voices of all members and the local microphone. Members who join after the recording starts are added to the mix. Sound that the browser attaches to the shared screen is removed, so the meeting is not heard twice.

| Call | Does |
|------|------|
| `startRecording({ onError })` | Starts. Throws if not joined, or if a recording is already running. A cancelled dialog also throws |
| `stopRecording(fileName)` | Stops and saves the file |
| `isRecording()` | Whether a recording is running |

```js
const meet = useJMeet()
try { await meet.startRecording() } catch (e) { console.log('before join:', e.message) }

meet.join('ws://x/jmeet', 'm1', {})
const ws = FakeWS.all[0]
ws.open()
await ws.receive({ type: 'existing-members', yourUsername: 'ali', usernames: [], connectionIds: {} })

try { meet.stopRecording() } catch (e) { console.log('nothing to stop:', e.message) }
await meet.startRecording({ onError: e => console.log('recording error:', e.message) })
console.log('isRecording:', meet.isRecording())
try { await meet.startRecording() } catch (e) { console.log('second start:', e.message) }
meet.stopRecording('lesson-1.mp4')
await sleep()
console.log('isRecording:', meet.isRecording())

console.log('--- leaving while recording stops it ---')
await meet.startRecording()
meet.leave()
await sleep()
console.log('isRecording:', meet.isRecording())
```

```text
before join: Cannot start recording before joining a meeting
nothing to stop: There is no recording to stop
isRecording: true
second start: A recording is already in progress
   download: lesson-1.webm
isRecording: false
--- leaving while recording stops it ---
   download: meeting-record.webm
isRecording: false
```

- The saved file name is the given name with the **extension replaced** by the one that fits what the browser really recorded (`lesson-1.mp4` became `lesson-1.webm`). Without a name the file is `meeting-record.webm`.
- The recording also ends, and **the file is saved**, when the user presses the browser's "Stop sharing" bar, and when the session ends in any way: `leave()`, the end of the meeting, or a lost connection. In both of these cases the file gets the default name.
- A recording that fails reports through the `onError` that was given to `startRecording`, and nothing is saved.
- There is no event for "the recording ended by itself", and `isRecording()` is a plain function, not a reactive value. Ask it before acting, as the example above does, or check it from time to time.

## 18. Live translation

`useJMeetTranslator` listens to the speakers of a meeting, shows what they said, and translates it into the language of the local user. It works only on the page of the one who asks for it; nothing is sent to the other members.

Recognition is done by the browser's speech recognition, with **one recognizer for each remote member**, tied to that member's voice. Who spoke is therefore known for certain by which recognizer produced the text, and not guessed. The local user's own microphone gets a recognizer of its own. Translation is done by the free MyMemory service, over HTTPS.

Requirements and limits:

- A recognizer bound to a voice needs **Chrome 139 or newer**. In Firefox and Safari it does not work, and there is no reliable way to test for this in advance: an unsupported browser shows up as a runtime error.
- The recognition is usually carried out by an online service of the browser vendor, and the recognized text goes to MyMemory. Tell the users that what they say leaves the page.
- MyMemory's free quota is 5000 characters a day for an anonymous user and 50000 with a contact email. The email is sent as the `de` parameter, so ask for the user's own address and do not put a fixed one into the source.

The page wires it to the session:

```js
const meet = useJMeet()
const translator = useJMeetTranslator()

const session = meet.join(wsUrl, meetingId, {
  onMemberLeft: (username) => translator.onMemberLeft(username),   // stop listening to people who left
  onDisconnected: () => translator.stop()
})

translator.start('fr-FR', 'en-US', session, { /* callbacks */ })
```

`start(otherLanguage, ownLanguage, session, callbacks)`:

- `otherLanguage` is the language the **others** speak. Their voices are recognized in it.
- `ownLanguage` is the language of the **local user**. Their microphone is recognized in it, and the others' speech is translated into it.
- `session` is the object returned by `join`. Without it, only the local speech is recognized.
- Language codes are the ones used by speech recognition: `fa-IR`, `en-US`, `fr-FR`, `ar-SA` and so on. Only `fa-IR` and `ar-SA` are given the direction `rtl`; every other code gets `ltr`.

The callbacks are all optional:

| Callback | Arguments | When |
|----------|-----------|------|
| `onInterimResult` | time, text, speaker | A member is speaking, text not final yet |
| `onInterimTranslatedResult` | time, text, translated, sourceDirection, targetDirection, speaker | A live preview of the translation |
| `onTranslatedResult` | time, text, translated, sourceDirection, targetDirection, speaker, ordinal, isLatest | A sentence is complete and translated |
| `onOwnSpeechRecognized` | time, text, ordinal, direction | The local user said a sentence (nothing to translate) |
| `onTranslationFailed` | time, text, speaker, ordinal | A sentence could not be translated |
| `onReconnecting` | count | Recognition is restarting after a problem |
| `onLocalRecognitionEnded` | error | The local microphone's recognition ended for good |
| `onError` | error | A problem, often temporary |
| `onEnd` | | The translation session is over |

`contactEmail` is also passed in this object.

```js
const translator = useJMeetTranslator()
translator.start('fr-FR', 'en-US', fakeSession(['ali']), {
  contactEmail: 'me@example.com',
  onInterimResult: (_t, text, who) => console.log('interim            ', who, '|', text),
  onInterimTranslatedResult: (_t, text, translated, _d1, _d2, who) => console.log('interim translated ', who, '|', translated),
  onTranslatedResult: (_t, text, translated, srcDir, dstDir, who, ordinal, isLatest) =>
    console.log('final              ', who, '|', text, '->', translated, '| ordinal', ordinal, 'latest', isLatest),
  onOwnSpeechRecognized: (_t, text, ordinal) => console.log('own speech         ', ordinal, '|', text),
  onEnd: () => console.log('onEnd')
})
await sleep(20)

const ali = recognizer('fr-FR')      // listens to ali, in French
const me = recognizer('en-US')       // listens to the local microphone, in English

ali.say('bonjour', false)
ali.say('bonjour tout le monde', false)
await sleep(1100)                     // a pause long enough for the live preview
ali.say('bonjour tout le monde', true)
await sleep(100)
me.say('hello ali', true)
await sleep(100)

translator.stop()
await sleep(1500)
const { q, pair, de } = mm.requests[0]
console.log('sent to the translation service:', { q, pair, de })
```

```text
interim             ali | bonjour
interim             ali | bonjour tout le monde
interim translated  ali | hello everyone
final               ali | bonjour tout le monde -> hello everyone | ordinal 2 latest true
own speech          3 | hello ali
onEnd
sent to the translation service: {
  q: 'bonjour tout le monde',
  pair: 'fr-FR|en-US',
  de: 'me@example.com'
}
```

- While a member speaks, `onInterimResult` follows the unfinished sentence. After 0.9 seconds without change, and only when it already has three words or more, a **preview** of the translation is requested and delivered through `onInterimTranslatedResult`. Shorter pieces are not translated, because every update would be translated from scratch and the text would flip back and forth.
- The final sentence arrives through `onTranslatedResult`. The `ordinal` numbers every recognized piece in the order it was heard, across all speakers. The numbers are not consecutive for finals, because interim updates take numbers as well.
- The local user's speech is delivered at once, untranslated, through `onOwnSpeechRecognized`.
- The request that leaves the page contains the text, the language pair and the contact email, which is shown on the last lines.
- `stop()` is graceful: recognizers are stopped, sentences still being translated are completed, and `onEnd` follows. The composable also calls `stop()` when its component goes away.

Keeping a transcript is a matter of collecting the callbacks and sorting by `ordinal`, because the lines do not arrive in the order they were spoken (chapter 19):

```js
const lines = []                                    // everything said in this translation session

meetingTranslator.start('fr-FR', 'en-US', session, {
  onTranslatedResult: (time, text, translated, _d1, _d2, speaker, ordinal) =>
    lines.push({ ordinal, text: `${speaker}: ${text} -> ${translated}` }),
  onOwnSpeechRecognized: (time, text, ordinal) =>
    lines.push({ ordinal, text: `me: ${text}` }),
  onTranslationFailed: (time, text, speaker, ordinal) =>
    lines.push({ ordinal, text: `${speaker}: ${text} (not translated)` }),
  onEnd: () => {
    lines.sort((a, b) => a.ordinal - b.ordinal)
    saveAsFile(lines.map(l => l.text).join('\n'))      // saveAsFile is the page's own helper
  }
})
```

## 19. Order, retries and recovery

**Order.** Sentences are translated in parallel, so a short sentence can finish before a long one that was spoken earlier. `onTranslatedResult` is nevertheless called **in the order the sentences were heard**; a finished translation waits until the earlier ones are ready. Sentences from the local user are shown at once, so a translation of something said earlier may come later than a line the user already saw. For this reason `isLatest` is `false` for such a late arrival: record it (in the transcript, for example) but do not put it over the newer line on the screen.

At most **two requests for final sentences run at once**, for all the speakers together.

```js
mm.latency = q => (q === 'ouvrez le livre' ? 600 : 50)     // the first sentence is slow

const translator = useJMeetTranslator()
translator.start('fr-FR', 'en-US', fakeSession(['ali']), {
  onTranslatedResult: (_t, text, translated, _a, _b, who, ordinal, isLatest) =>
    console.log('final    ', ordinal, translated, '| latest', isLatest),
  onOwnSpeechRecognized: (_t, text, ordinal) => console.log('own speech', ordinal, text),
  onEnd: () => console.log('onEnd')
})
await sleep(20)

const ali = recognizer('fr-FR'), me = recognizer('en-US')
ali.say('ouvrez le livre', true)        // ordinal 0, slow translation
ali.say('à la page douze', true)        // ordinal 1, fast translation
ali.say('bonjour tout le monde', true)  // ordinal 2, fast, but must wait for a free slot
me.say('which page?', true)             // ordinal 3, shown at once
await sleep(900)
console.log('requests in flight at most:', mm.maxActive)
translator.stop()
await sleep(1500)
```

```text
own speech 3 which page?
final     0 open the book | latest false
final     1 on page twelve | latest false
final     2 hello everyone | latest false
requests in flight at most: 2
onEnd
```

Three remote sentences and one local sentence. The first translation is slow. The local line comes first (ordinal 3, it needs no translation), the translations follow in order, all with `isLatest` false. Never more than two requests were in flight.

**Failures.** A request that fails or takes too long is tried again, up to two more times, with a pause of half a second. The waiting time for one request is 6 seconds plus half a second for each word, with a ceiling of 16 seconds. A sentence that still fails is reported through `onError` and `onTranslationFailed`, with the original text, so that the page can keep it.

MyMemory accepts at most 500 bytes of text per request. A longer sentence is cut at word boundaries into pieces under that limit, the pieces are translated in parallel and joined again. A single word longer than the limit is cut by byte without splitting a character. When one piece fails for good, the others are cancelled.

```js
const translator = useJMeetTranslator()
translator.start('fr-FR', 'en-US', fakeSession(['ali']), {
  onTranslatedResult: (_t, text, translated) => console.log('final:', translated.length > 60 ? translated.slice(0, 30) + '...' : translated),
  onTranslationFailed: (_t, text, who, ordinal) => console.log('failed:', who, ordinal, text),
  onError: e => console.log('onError:', e.message)
})
await sleep(20)
const ali = recognizer('fr-FR')

console.log('1. a flaky network: the first attempt fails, the retry works')
mm.fail = (q, attempt) => q === 'ouvrez le livre' && attempt === 1
ali.say('ouvrez le livre', true)
await sleep(1200)
console.log('   requests made:', mm.requests.length)

console.log('2. a dead network: three attempts, then the sentence is reported as failed')
mm.requests.length = 0
mm.fail = () => true
ali.say('à la page douze', true)
await sleep(3000)
console.log('   requests made:', mm.requests.length)

console.log('3. a long sentence is cut into pieces under 500 bytes')
mm.requests.length = 0
mm.fail = () => false
ali.say(('bonjour tout le monde ').repeat(40).trim(), true)
await sleep(500)
console.log('   requests:', mm.requests.map(r => r.bytes))
translator.stop()
await sleep(1500)
```

```text
1. a flaky network: the first attempt fails, the retry works
final: open the book
   requests made: 2
2. a dead network: three attempts, then the sentence is reported as failed
onError: Translation failed: network down
failed: ali 1 à la page douze
   requests made: 3
3. a long sentence is cut into pieces under 500 bytes
final: BONJOUR TOUT LE MONDE BONJOUR ...
   requests: [ 499, 379 ]
```

**Recovery.** Speech recognition stops now and then for ordinary reasons. The composable restarts it:

- After an ordinary end the restart waits 0.3 seconds. After a *network* error the delay is longer and doubles with each failure in a row (0.6 s, 1.2 s, 2.4 s, and so on up to 15 seconds). A real result resets it. `onReconnecting(count)` tells the page, so that it can show "reconnecting".
- A **fatal** error (permission denied, no audio capture, unsupported language) stops that recognizer for good. For a remote member this only affects that member. For the local microphone, the page is told through `onLocalRecognitionEnded`, because there is no automatic recovery: the translation must be stopped and started again. Call `stop()` and start again from the `onEnd` callback, because `start()` throws while the old session is still winding down.
- A recognizer that produces no result for 45 seconds is rebuilt from scratch (a known problem of Chrome's continuous mode). Silence counts as well, so in a quiet meeting `onReconnecting` can fire from time to time without anything being wrong. The same callback also fires after `no-speech` (nobody said anything), with `count` 0.
- The local recognizer pauses while the tab is hidden, since a hidden tab may lose the microphone, and starts again when the tab is visible. The recognizers of remote members do not pause, because their audio keeps flowing.
- A member who leaves must be told to the translator with `onMemberLeft`, which stops the recognizer. When the member comes back, a new recognizer replaces the old one.

```js
const session = fakeSession(['ali'])
const translator = useJMeetTranslator()
translator.start('fr-FR', 'en-US', session, {
  onReconnecting: n => console.log('onReconnecting', n),
  onLocalRecognitionEnded: e => console.log('onLocalRecognitionEnded:', e.message),
  onTranslatedResult: (_t, _o, translated) => console.log('   translated:', translated),
  onError: e => console.log('onError:', e.message),
  onEnd: () => console.log('onEnd')
})
await sleep(20)
const ali = recognizer('fr-FR'), me = recognizer('en-US')

console.log('1. the speech service drops three times in a row')
for (let i = 1; i <= 3; i++) {
  const before = Date.now()
  ali.fail('network')
  while (ali.starts.length <= i) await sleep(20)
  console.log('   restarted after about', Math.round((Date.now() - before) / 100) * 100, 'ms')
}
ali.say('ouvrez le livre', true)         // a real result: the streak is over
ali.fail('network')
const before = Date.now()
while (ali.starts.length <= 4) await sleep(20)
console.log('   after a good result: about', Math.round((Date.now() - before) / 100) * 100, 'ms')

console.log('2. the local microphone is blocked: fatal for the local recognizer only')
me.fail('not-allowed')
await sleep(50)
ali.say('ouvrez le livre', true)
await sleep(100)

console.log('3. ali leaves, and comes back with a new connection')
translator.onMemberLeft('ali')
await sleep(20)
console.log('   old recognizer stopped:', ali.stopped)
session.ready('ali', { id: 'captured-ali-2' })
await sleep(20)
console.log('   recognizers for fr-FR:', FakeRec.all.filter(r => r.lang === 'fr-FR').length)
translator.stop()
await sleep(1500)
```

```text
1. the speech service drops three times in a row
onError: Speech recognition error: network
onReconnecting 1
   restarted after about 600 ms
onError: Speech recognition error: network
onReconnecting 2
   restarted after about 1200 ms
onError: Speech recognition error: network
onReconnecting 3
   restarted after about 2400 ms
onError: Speech recognition error: network
onReconnecting 1
   translated: open the book
   after a good result: about 600 ms
2. the local microphone is blocked: fatal for the local recognizer only
onError: Speech recognition error: not-allowed
onError: Speech recognition ended fatally: not-allowed
onLocalRecognitionEnded: Speech recognition ended fatally: not-allowed
   translated: open the book
3. ali leaves, and comes back with a new connection
   old recognizer stopped: true
   recognizers for fr-FR: 2
onEnd
```

Starting the translator can also fail at once:

```js
const translator = useJMeetTranslator()
const attempt = (label, fn) => { try { fn(); console.log(label, '-> ok') } catch (e) { console.log(label, '->', e.message) } }

attempt('no languages        ', () => translator.start('fr-FR', null, fakeSession(), {}))
attempt('start               ', () => translator.start('fr-FR', 'en-US', fakeSession(), {}))
attempt('start again         ', () => translator.start('fr-FR', 'en-US', fakeSession(), {}))
translator.stop()
await sleep(1500)

const saved = globalThis.SpeechRecognition
delete globalThis.SpeechRecognition
attempt('browser without it  ', () => translator.start('fr-FR', 'en-US', fakeSession(), {}))
globalThis.SpeechRecognition = saved

navigator.mediaDevices.getUserMedia = async () => { throw new Error('Permission denied') }
translator.start('fr-FR', 'en-US', fakeSession(), {
  onError: e => console.log('onError:', e.message),
  onEnd: () => console.log('onEnd')
})
await sleep(50)
attempt('start after failure ', () => translator.start('fr-FR', 'en-US', fakeSession(), {}))
```

```text
no languages         -> Both the other party language and your own language must be set
start                -> ok
start again          -> Translation is already running
browser without it   -> Please use Chrome.
onError: Could not access the microphone for translation: Permission denied
onEnd
start after failure  -> ok
```

`start()` throws for a missing language, for a second call while the translation runs, and for a browser without speech recognition. A microphone that cannot be opened is reported through `onError` and `onEnd`, after which `start()` can be called again (the last line).

## 20. Putting it together: a meeting in a web application

The pieces, in the order the application uses them:

1. **At startup**: `JMeet.configure()...start()` (chapter 2), the endpoint and the exporter (chapter 4).
2. **When a teacher opens a class**: `POST /meetings/{id}` creates the meeting; the teacher is the owner.
3. **When a student enrolls**: `PUT /meetings/{id}/members/{user}` adds the student.
4. **When someone opens the meeting page**: the page calls `join`, with the address from the runtime configuration. The cookie carries the identity.
5. **During the meeting**: the page uses `session` for the buttons and the callbacks for the state (chapters 13 and 14).
6. **At the end**: the owner presses the button, which calls `session.endMeeting()` or the `DELETE` endpoint. Everyone receives `onMeetingEnded`.

The page is the glue. The state of chapter 13 is the core of it. The buttons look like this:

```html
<video id="main-video" autoplay playsinline />

<UButton :label="iAmPresenting ? 'Stop camera' : 'Camera'"
         :disabled="presenter !== null"
         @click="toggleWebcam" />
```

A short list of things to check before going live:

- The same issuer and key sign and read the tokens.
- The cookie is sent to the signaling server, or the page passes `token`.
- The address is `wss://` and the site uses HTTPS. Browsers allow the camera, the microphone and screen sharing only on secure pages (and on `localhost`).
- A TURN server is available for users behind strict networks.
- The REST endpoints take the caller from the verified token.
- The page handles `onDisconnected`, `onError` and `onMeetingEnded`.

---

## Appendix A. Quick reference

### A.1 Server: JMeet

| Method | Purpose |
|--------|---------|
| `configure()` | Starts a configuration builder |
| `createMeeting(id, owner)` | Creates a meeting. Returns the id |
| `addMember(id, owner, user)` | Allows a user to enter |
| `kickMember(id, owner, user)` | Removes a member and closes their connection |
| `endMeeting(id, owner)` | Ends the meeting for everyone |
| `owner(id)`, `members(id)`, `onlineMembers(id)` | Read a meeting |
| `meetingIds()` | All active meeting ids on this server |
| `isOwner(id, token)` | Whether the token belongs to the owner. `false` for anything invalid |
| `currentUsername(token)` | The subject of a token, or `null` |

### A.2 Server: the builder

| Method | Purpose |
|--------|---------|
| `tokenSource(TokenSource)` | `AUTHORIZATION_HEADER_BEARER` or `COOKIE` |
| `cookieName(name)` | Cookie to read (default `token`) |
| `issuer(...)`, `secretKey(...)` | Required. Same as the login tokens. Key of 64 bytes or more |
| `onBeforeCreateMeeting(owner -> boolean)` | Gate for creation |
| `onBeforeMemberJoined((meeting, user) -> boolean)` | Gate for joining |
| `onMemberJoined`, `onMemberLeft` | Notifications `(meeting, user)` |
| `onMeetingEnded(meeting -> ...)` | Notification |
| `maxQueuedCharsPerConnection(n)` | Limit of unsent characters (default 4194304) |
| `start()` | Applies everything at once |

### A.3 Server: exceptions

| Exception | Meaning |
|-----------|---------|
| `MeetingCreationRejectedException` | The `onBeforeCreateMeeting` gate said no |
| `MeetingAlreadyExistsException` | The id is in use |
| `NotMeetingOwnerException` | The caller is not the owner |
| `OwnerCannotBeKickedException` | An attempt to kick the owner |
| `IllegalArgumentException` | No such meeting, or an empty argument |
| `IllegalStateException` | `JMeet` is not configured |

All are unchecked.

### A.4 The wire protocol

Client to server:

| Message | Fields | Effect |
|---------|--------|--------|
| `offer`, `answer`, `ice-candidate` | `to`, content | Relayed to `to` |
| `start-presenting` | `presentationType`: `webcam` or `desktop` | Becomes presenter if free |
| `stop-presenting` | | Frees the place if presenter |
| `end-meeting` | | Ends the meeting if owner |

Server to client:

| Message | Fields | Sent to |
|---------|--------|---------|
| `existing-members` | `usernames`, `connectionIds`, `yourUsername`, `isOwner`, optional `presenter` and `presentationType` | The newcomer |
| `member-joined`, `member-left` | `username`, `connectionId` | The others |
| `presenter-changed` | `username`, `presentationType` (empty: nobody) | The others |
| `offer`, `answer`, `ice-candidate` | content, `from`, `connectionId` | The addressed member |
| `meeting-ended` | | Everyone connected |
| `error` | `code`, `message` | The sender of the rejected request |

### A.5 Browser: useJMeet

| Call | Purpose |
|------|---------|
| `join(url, meetingId, options)` | Joins. Returns the session (`null` during server rendering) |
| `leave()` | Leaves silently |
| `isJoined()` | Whether a session exists |
| `startRecording(options)`, `stopRecording(fileName)`, `isRecording()` | Local recording |
| `requestPermissionsUpfront()` | Asks for the camera and microphone once |

### A.6 Browser: the session

| Call | Purpose |
|------|---------|
| `enableWebcam()`, `enableDesktopSharing()` | Start presenting. Return the stream |
| `stopPresenting()` | Stop |
| `enableMicrophone()` | Open, then mute or unmute. Returns the state |
| `isMicrophoneEnabled()`, `getLocalAudioStream()` | The local microphone |
| `getMemberAudioStreams()` | Map of username to the voice stream |
| `onAudioSourceAdded(cb)`, `onAudioSourceRemoved(cb)` | Voice events. Return a function that removes the listener |
| `getCapturedAudioTrack(user)`, `onCapturedAudioReady(cb)` | A track that speech recognition accepts |
| `muteVideoPlayer(boolean)` | Mute the `<video>` element |
| `endMeeting()` | Ask the server to end the meeting |
| `close()` | Same as `leave()` |

### A.7 Browser: useJMeetTranslator

| Call | Purpose |
|------|---------|
| `start(otherLanguage, ownLanguage, session, callbacks)` | Starts. Throws when it cannot |
| `stop()` | Ends gracefully, then `onEnd` |
| `onMemberLeft(username)` | Stops listening to a member who left |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| `JMeet` is used before `start()` | `IllegalStateException` |
| The key is shorter than 64 bytes | `start()` fails with `IllegalArgumentException`, configuration unchanged |
| The same meeting id is created twice | `MeetingAlreadyExistsException` |
| A non-owner changes a meeting | `NotMeetingOwnerException` |
| The token is missing, invalid or expired | The connection closes with 1003 and the reason |
| A valid user who is not a member connects | The connection closes with `Not a member of this meeting` |
| The meeting ended while a person was joining | `Meeting has ended` |
| A message is malformed, unknown, or comes too early | Ignored without an answer |
| An offer is sent to someone offline | Dropped |
| A second member asks to present | `error` frame `presenting-unavailable` to that member |
| A member opens a second connection | The older connection closes; the others see `member-joined` |
| A member's queue grows past the limit | Closed with 1013 `slow consumer` |
| A write to a member fails | Closed with 1013 `write failed` |
| The presenter disconnects, is kicked or reconnects | The place is freed with `presenter-changed` |
| The server restarts | All meetings are gone; pages receive `onDisconnected` |
| The page tries to present while someone else does | `enableWebcam` throws |
| The user denies the camera or microphone | The call throws the browser's error |
| The page loses the network | `onDisconnected`, no automatic reconnection |
| A peer connection fails | ICE restart |
| A voice cannot start playing (autoplay policy) | `onError` with `Could not play audio for <user>` |
| Translation fails three times | `onError` and `onTranslationFailed` |
| Speech recognition fails with a network error | Restart with growing delay and `onReconnecting` |
| Speech recognition fails fatally | That recognizer stops; for the local one `onLocalRecognitionEnded` |
| The browser has no speech recognition | `start()` throws `Please use Chrome.` |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Taking the caller from the request.** Pass the identity from the verified token to `JMeet`. A name from the body lets anyone act as the owner.
- **Forgetting `addMember`.** Without it, a valid user is turned away.
- **Tokens in the address.** `access_token` ends up in logs. Prefer the cookie in browsers.
- **Cookies and the origin.** The endpoint reads the `Cookie`, `Authorization` and query of the handshake and does not look at the `Origin` header. With cookie authentication, rely on `SameSite` or check the origin in your own filter.
- **Different keys.** If the key of `JMeet` differs from the one that signs the logins, every join closes with `Invalid token`.
- **Plain `ws://` on a secure site.** Browsers block it. Use `wss://`.
- **Joining twice with new callbacks.** The second `join` for the same meeting ignores them.
- **Not resetting the page after `onDisconnected`.** The session is dead, but the buttons are still there.
- **State of the preview.** After `onPresenterChanged` and `onOwnPresentingEnded` the page must reset its own presenting flag.
- **Leaving during a recording.** The file is saved with the default name.
- **`muteVideoPlayer` does not silence people.** It only mutes the `<video>` element.
- **Autoplay.** A browser may refuse to play voices until the user has clicked something. `onError` reports it.
- **Hidden tabs.** The local recognizer pauses while the tab is hidden, so translation of one's own speech stops.
- **A fixed contact email.** Do not ship somebody else's email address in the source.
- **A slow hook.** The join gate runs under the meeting's lock, and the other hooks run on the thread that triggers them.

### C.2 Behavior worth knowing

- The server never handles media; only small JSON messages.
- A username has one connection per meeting. Messages from a replaced connection are ignored.
- `onMemberLeft` on the server fires once for each connection that really ended, including when a meeting ends. It does not fire for a replaced connection.
- Everything sent to a meeting passes through its lock, so messages are delivered in the order the server processed them.
- The buffer for an incoming text message is 128 KiB; an offer is far smaller.
- An id of a finished meeting can be reused.
- Ownership never changes. If the owner is gone, nobody else can add members or end the meeting; the meeting disappears only when it is ended by the owner or the process restarts.
- `enableMicrophone()` mutes by disabling the track. The device stays open, so the browser usually keeps showing that the microphone is in use.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Presenters per meeting | 1 |
| Connections per username per meeting | 1 |
| Media topology | Full mesh, `n × (n − 1) / 2` connections for `n` people |
| Queue per connection | 4194304 characters by default (`maxQueuedCharsPerConnection`) |
| Incoming text message buffer | 131072 (128 KiB) |
| Configuration | One per process |
| Meetings | In memory, one server process, lost on restart |
| Token key | 64 bytes or more |
| Translation request | 500 bytes of text, 2 in parallel, 3 attempts |
| Translation timeout per request | 6 s + 0.5 s per word, at most 16 s |
| Live preview | After 0.9 s of silence, from 3 words, texts up to 500 bytes |
| Recognition restart delay | 0.3 s doubling to 15 s |
| Silence before a recognizer is rebuilt | 45 s |
| Speech recognition from a voice track | Chrome 139 or newer |
| MyMemory free quota | 5000 characters a day, 50000 with a contact email |

### C.4 Tuning and practical advice

- Keep meetings small. For a class of thirty, a mesh is the wrong tool; use it for a handful of people or one presenter and a few speakers.
- Limit who may present video if bandwidth matters. The presenter uploads one copy of the video for every other member.
- Run a TURN server. A part of the users will not connect without it.
- Create unguessable meeting ids when the id appears in a link, and still add only the members who may enter.
- Ask for the camera and microphone before the meeting with `requestPermissionsUpfront()`.
- Reconnect after `onDisconnected` with growing pauses, and rebuild the page state from the new `existing-members`.
- Show `onReconnecting` to the user, and `onLocalRecognitionEnded` as a clear message with a restart button.
- Lower `maxQueuedCharsPerConnection` only when memory is tight; the default is generous.
- Several servers need sticky routing by meeting id, or an external coordinator, which jmeet does not provide.

## Appendix D. Message reference

### D.1 Server: exceptions

| Message | Exception | Cause |
|---------|-----------|-------|
| `JMeet is not configured yet. Call JMeet.configure().issuer(...).secretKey(...)....start() before use.` | `IllegalStateException` | A call before `start()` |
| `issuer and secretKey must be set before start()` | `IllegalStateException` | `start()` without an issuer or key |
| `Issuer cannot be null or blank` | `IllegalArgumentException` | Raised by `TokenProcessor` in `start()` |
| `Secret key cannot be null or blank` | `IllegalArgumentException` | Raised by `TokenProcessor` in `start()` |
| `HMAC512 secret key must contain at least 64 UTF-8 bytes` | `IllegalArgumentException` | A short key |
| `cookieName must not be null or blank` | `IllegalArgumentException` | A blank cookie name |
| `maxQueuedCharsPerConnection must be positive` | `IllegalArgumentException` | A limit of zero or less |
| `tokenSource must not be null`, `hook must not be null` | `NullPointerException` | A null argument |
| `meetingId must not be null or blank`, `owner must not be null or blank`, `username must not be null or blank` | `IllegalArgumentException` | An empty argument |
| `Meeting creation was rejected for owner <owner>` | `MeetingCreationRejectedException` | The creation gate returned `false` |
| `A meeting with id <id> already exists` | `MeetingAlreadyExistsException` | A repeated id |
| `<user> is not the owner of meeting <id>` | `NotMeetingOwnerException` | The caller is not the owner |
| `No meeting found with id <id>` | `IllegalArgumentException` | An unknown meeting |
| `The meeting owner cannot be kicked — call endMeeting instead` | `OwnerCannotBeKickedException` | An attempt to kick the owner |

### D.2 Server: connection close reasons

| Code | Reason | Cause |
|------|--------|-------|
| 1003 | `Missing meeting_id` | No `meeting_id` in the query |
| 1003 | `Missing token` | No token in the configured place |
| 1003 | `Invalid token` | The token cannot be read, or it expired |
| 1003 | `No such meeting` | The meeting does not exist |
| 1003 | `Meeting has ended` | The meeting ended while joining |
| 1003 | `Not a member of this meeting` | The user was not added, or was kicked |
| 1003 | `Join rejected` | The `onBeforeMemberJoined` gate said no |
| 1013 | `slow consumer` | More unsent characters than the limit |
| 1013 | `write failed` | A write to the connection failed |

### D.3 Server: error frames and logs

| Message | Meaning |
|---------|---------|
| `{"type":"error","code":"invalid-presentation-type", ...}` | `presentationType` is neither `webcam` nor `desktop` |
| `{"type":"error","code":"presenting-unavailable", ...}` | Another member already presents |
| `Closing member connection <id>: more than <n> characters are waiting to be written to it` | Logged at `WARNING` before a `slow consumer` close |

### D.4 Browser: useJMeet

| Message | Cause |
|---------|-------|
| `useJMeet: url must not be empty (check runtimeConfig.public.jmeetWsUrl)` | An empty address |
| `Already joined meeting "<id>" — call leave() before joining a different meeting` | `join` for another meeting |
| `token must not be empty` | A blank token |
| `Someone else is already presenting in this meeting` | `enableWebcam` or `enableDesktopSharing` while another member presents |
| `Already waiting on a previous enableWebcam()/enableDesktopSharing() call` | A second call while the first waits for permission |
| `Already waiting on a previous enableMicrophone() call` | The same for the microphone |
| `Session was closed while waiting for media permission` | The page left while the prompt was open |
| `Session was closed while waiting for microphone permission` | The same for the microphone |
| `Failed to switch audio for <user>: ...`, `Failed to switch video for <user>: ...`, `Failed to stop video for <user>: ...` | A track could not be replaced |
| `Negotiation with <user> failed: ...` | The browser could not create an offer |
| `Failed to handle offer from <user>: ...`, `Failed to handle answer from <user>: ...` | An offer or answer could not be applied |
| `Failed to add ICE candidate from <user>: ...` | A candidate could not be applied |
| `ICE restart failed for <user>: ...` | The restart after a failed connection did not work |
| `Could not play audio for <user>: ...` | The browser refused to play a voice |
| `JMeet connection error` | The socket reported an error |
| `JMeet connection closed: <reason>` | The server closed the connection with a reason |
| `JMeet server error: <code>` | An `error` frame without a message. With a message, the text of the server (such as `Another member is already presenting`) is used instead |
| `Cannot start recording before joining a meeting` | `startRecording` without a session |
| `A recording is already in progress` | A second `startRecording` |
| `This recorder has been disposed and can no longer start a recording` | The component that owns it was removed |
| `Recording was reset while waiting for screen-share permission` | The meeting ended while the dialog was open |
| `Recording stopped: <detail>` | The recorder failed |
| `There is no recording to stop` | `stopRecording` without a recording |

### D.5 Browser: useJMeetTranslator

| Message | Cause |
|---------|-------|
| `Translation is already running` | `start` while running |
| `Both the other party language and your own language must be set` | A missing language |
| `Please use Chrome.` | No speech recognition in the browser |
| `Could not access the microphone for translation: ...` | The local microphone was refused |
| `Could not start recognition for <user>: ...` | A recognizer for a member could not start |
| `Speech recognition error: <code>` | The browser reported an error (`network`, `not-allowed`, ...). `no-speech` is not reported |
| `Speech recognition ended fatally: <code>` | A recognizer stopped for good |
| `Failed to restart speech recognition: ...` | The restart after an error failed |
| `Watchdog restart failed: ...` | The rebuild of a silent recognizer failed |
| `Translation failed: <detail>` | A sentence could not be translated after the attempts |
| `Failed to fetch translation` | The service answered with an HTTP error |
| `Translation failed` | The service refused without details |
| `Translation cancelled` | A piece of a long sentence was cancelled |
