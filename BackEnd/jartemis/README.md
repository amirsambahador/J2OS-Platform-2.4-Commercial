# jartemis

*J2OS Platform 2.4*

Programs that talk to each other directly have a weakness: the receiver must be running at the moment the sender speaks. An order placed while the packing service restarts is lost, and a slow receiver slows the sender down. The usual cure is a **message broker**, a small server in the middle that keeps messages until someone takes them. The sender drops a message in and goes on; the receiver picks it up whenever it is ready.

**jartemis** (`org.j2os.platform.jartemis`) gives a Java program such a broker with very little ceremony. It wraps Apache ActiveMQ Artemis and offers two things: **queues**, where every message goes to exactly one reader, and **topics**, where every message goes to every subscriber. Three properties set it apart:

- **The broker can live inside the program.** One call starts it, with its files in a directory of your choice. The same code can instead connect to a broker that runs elsewhere.
- **Messages are plain Java objects.** A record or bean goes in and comes out; the library turns it into JSON on the way.
- **One kind of failure.** Every checked problem is a `JArtemisException` (or a subclass), and a transaction that fails is rolled back for you.

The whole library in a few lines:

```java
MessageManagerFactory factory = EmbeddedArtemisBroker
        .bind("127.0.0.1", 61616)                       // this program is the broker
        .storage(Path.of("broker-data"))                // where it keeps its files
        .createMessageManagerFactory();                 // once, at start-up

try (MessageManager m = factory.createMessageManager()) {   // one per thread
    m.queue("orders").send(new Order("A-1", 3));            // a JSON message in a queue
    Order next = m.queue("orders").timeout(1000).getMessage(Order.class);

    m.topic("news").fromCurrentMessage()                    // everybody listening gets a copy
            .onMessage(Event.class, e -> System.out.println(e));
}
```

This guide goes step by step: a first message (chapter 2), the builder (3), the factory and its managers (4), sending (5), reading (6), listening (7), sending without waiting (8), topics (9), durable subscriptions (10), history (11), transactions (12), a client of another broker (13), several machines (14), the log (15) and a complete small workflow (16). The appendices work as a reference.

## Contents

1. [How jartemis works](#1-how-jartemis-works)
2. [A first message](#2-a-first-message)
3. [The builder](#3-the-builder)
4. [The factory and its managers](#4-the-factory-and-its-managers)
5. [Sending](#5-sending)
6. [Reading from a queue](#6-reading-from-a-queue)
7. [Listening to a queue](#7-listening-to-a-queue)
8. [Sending without waiting](#8-sending-without-waiting)
9. [Topics](#9-topics)
10. [Durable subscriptions](#10-durable-subscriptions)
11. [Keeping history](#11-keeping-history)
12. [Transactions](#12-transactions)
13. [A client of another broker](#13-a-client-of-another-broker)
14. [Several machines](#14-several-machines)
15. [The log](#15-the-log)
16. [Putting it together](#16-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The programs of the guide are complete: each has one `main` method and starts its own broker on the local machine, with its files in a folder named `broker-data`. Ports and names are made up; change them to suit your system.

---

## 1. How jartemis works

First, a few terms, because they come up throughout the guide:

- **Broker.** The server that receives, keeps and hands out messages.
- **Embedded broker.** A broker that runs inside your own program. `bind(...)` starts one.
- **Client.** A program that uses a broker running somewhere else. `remote(...)` makes one.
- **Queue.** A line of messages. Each message is taken by one reader; if several readers wait, they share the work.
- **Topic.** A channel. Each message is copied to every subscriber that is listening.
- **Subscription.** The standing request "give me what arrives here". It is stopped with `cancel()`.
- **Durable subscription.** A subscription with a name that the broker remembers. Messages sent while the subscriber is away wait for it.
- **Transaction.** A group of sends or reads that all happen or none happens.

The classes a program meets:

| Class | Role |
|-------|------|
| `EmbeddedArtemisBroker` | The builder. Chooses embedded or client, and sets storage, credentials and the rest |
| `MessageManagerFactory` | One per program. Owns the broker (if embedded) and the connection to it |
| `MessageManager` | One per thread. The entry point to `queue(...)`, `topic(...)` and the transaction |
| `MessageQueue`, `MessageTopic` | Send and receive on one named queue or topic |
| `MessageTransaction` | `begin`, `commit` and `rollback` on a manager |
| `Subscription` | The handle returned when listening; `cancel()` stops it |
| `JArtemisException` | The checked exception of the library. Subclasses: `MissingStorageDirectoryException`, `ReplicationTopologyException`, `TransactionRolledBackException` |

Keep these three points in mind:

- **The factory is built once and kept.** It starts the broker and opens one connection that every manager shares. Closing it stops everything.
- **A manager belongs to one thread.** It is cheap to create and not safe to share.
- **A message body is JSON.** The library uses Jackson, so a body is any object Jackson can write and read back, such as a record, a bean, a `Map` or a `String`.

## 2. A first message

Starting an embedded broker, putting an order in a queue and taking it out again:

```java
import org.j2os.platform.jartemis.*;
import java.nio.file.Path;

public class First {
    record Order(String id, int qty) {}

    public static void main(String[] args) throws Exception {
        try (MessageManagerFactory factory = EmbeddedArtemisBroker
                .bind("127.0.0.1", 61616)
                .storage(Path.of("broker-data"))
                .createMessageManagerFactory();
             MessageManager m = factory.createMessageManager()) {

            m.queue("orders").send(new Order("A-1", 3));

            Order got = m.queue("orders").timeout(1000).getMessage(Order.class);
            System.out.println("got " + got);
        }
    }
}
```

```text
got Order[id=A-1, qty=3]
```

- `bind("127.0.0.1", 61616)` makes this program a broker listening on that address. `storage(...)` says where its files live; it is required for `bind`.
- `createMessageManagerFactory()` starts the broker and connects to it. This is done once.
- `send` puts the order in the queue named `orders`. The queue is created on first use.
- `getMessage(Order.class)` takes the next message and turns it back into an `Order`. `timeout(1000)` means: wait at most one second.
- The `try` block closes the manager and then the factory, which stops the broker. A shutdown hook also closes the factory when the JVM stops, but closing it yourself is cleaner.

## 3. The builder

`EmbeddedArtemisBroker` is a builder. It starts with `bind` or `remote`, takes optional settings, and ends with `createMessageManagerFactory()`.

| Call | Meaning |
|------|---------|
| `bind(host, port)` | This program is a broker and listens on `host:port` |
| `remote(host, port)` | This program is a client of a broker at `host:port` |
| `storage(path)` | The directory for the broker's files. **Required** with `bind`; not used with `remote` |
| `credentials(user, password)` | With `bind`: creates a user that clients must use. With `remote`: the user to log in as |
| `clientId(id)` | A fixed name for this program, needed for durable subscriptions (chapter 10) |
| `autoReconnect(ms)` | After a break, try again every `ms` milliseconds |
| `replicationPair(...)` | Declares a primary and its backup (chapter 14) |

Arguments are checked as soon as they are given:

```java
import org.j2os.platform.jartemis.*;

public class Checks {
    public static void main(String[] args) {
        try { EmbeddedArtemisBroker.bind(" ", 61616); }
        catch (Exception e) { System.out.println("blank host -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { EmbeddedArtemisBroker.bind("0.0.0.0", 61616); }
        catch (Exception e) { System.out.println("0.0.0.0 -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { EmbeddedArtemisBroker.remote("10.0.0.5", 70000); }
        catch (Exception e) { System.out.println("port -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { EmbeddedArtemisBroker.remote("10.0.0.5", 61616).autoReconnect(0); }
        catch (Exception e) { System.out.println("autoReconnect -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
    }
}
```

```text
blank host -> IllegalArgumentException: host must not be null or blank.
0.0.0.0 -> IllegalArgumentException: host must not be "0.0.0.0". "0.0.0.0" is only valid for an acceptor (listening on all interfaces), but this same address value is also used for the connector (which clients use to connect), where it is invalid. Provide a real, reachable IP address or hostname instead (for example, the actual IP address of this machine).
port -> IllegalArgumentException: port must be between 1 and 65535; given value: 70000
autoReconnect -> IllegalArgumentException: intervalMillis must be a positive number; given value: 0
```

- **Blank host.** A host must be text with something in it.
- **`0.0.0.0`.** It is refused. The same address is used for the broker to listen on and for clients to connect to, and `0.0.0.0` only makes sense for the first. Give the real address of the machine, or `127.0.0.1` for a local test.
- **Port.** It must be between 1 and 65535.
- **`autoReconnect`.** The interval must be positive.

The address for a real system comes from configuration, not from the code:

```java
// JARTEMIS_HOST=10.0.0.5   JARTEMIS_PORT=61616   JARTEMIS_DATA=/var/lib/shop/broker
MessageManagerFactory factory = EmbeddedArtemisBroker
        .bind(System.getenv("JARTEMIS_HOST"), Integer.parseInt(System.getenv("JARTEMIS_PORT")))
        .storage(Path.of(System.getenv("JARTEMIS_DATA")))
        .autoReconnect(1000)                              // optional: retry every second after a break
        .createMessageManagerFactory();
```

Without `credentials`, anyone who can reach the port can use the broker. With them, `bind` creates the user, and every client must give the same name and password.

## 4. The factory and its managers

`createMessageManagerFactory()` does the real work, and it can still refuse:

```java
import org.j2os.platform.jartemis.*;
import java.nio.file.Path;

public class Setup {
    public static void main(String[] args) throws Exception {
        try { EmbeddedArtemisBroker.bind("127.0.0.1", 61628).createMessageManagerFactory(); }
        catch (Exception e) { System.out.println("no storage -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { EmbeddedArtemisBroker.remote("127.0.0.1", 61628)
                .replicationPair("10.0.0.1", 61616, "10.0.0.2", 61616).createMessageManagerFactory(); }
        catch (Exception e) { System.out.println("remote + pair -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        EmbeddedArtemisBroker builder = EmbeddedArtemisBroker.bind("127.0.0.1", 61629).storage(Path.of("broker-data"));
        try (MessageManagerFactory first = builder.createMessageManagerFactory()) {
            System.out.println("first factory is up");
            try { builder.createMessageManagerFactory(); }
            catch (Exception e) { System.out.println("same builder again -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

            try { EmbeddedArtemisBroker.bind("127.0.0.1", 61629).storage(Path.of("broker-data")).createMessageManagerFactory(); }
            catch (Exception e) { System.out.println("port taken -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
        }
    }
}
```

```text
no storage -> MissingStorageDirectoryException: Node 127.0.0.1:61628 is an embedded broker (bound, or present in a replicationPair) but no storage(path) was configured.
remote + pair -> JArtemisException: Combining remote() with replicationPair() is invalid: a node that participates in the HA topology must itself call bind() (it must be an embedded broker itself).
first factory is up
same builder again -> JArtemisException: This EmbeddedArtemisBroker instance has already been used to create a MessageManagerFactory; each builder instance may only be used once (otherwise the broker would attempt to bind the same port again).
port taken -> JArtemisException: Starting the embedded broker(s) on 127.0.0.1:61629 failed.
```

- **No storage.** A bound node without `storage` fails with `MissingStorageDirectoryException`, a subclass of `JArtemisException`.
- **Client and pair.** A client cannot be part of a replication group; only a node that is itself a broker can.
- **Same builder again.** A builder works **once**. It is marked as used before anything is checked, so a builder whose first attempt failed cannot be tried again either; make a new one.
- **Port taken.** If the port is already in use, starting the broker fails and the cause tells why (`getCause()`).

A factory is built once, at start-up, and kept for the life of the program. Managers are taken from it as needed:

- `createMessageManager()` gives a manager in auto-commit mode: every call is final.
- `setAutoCommit(false).createMessageManager()` gives a manager that works in transactions (chapter 12).
- `execute(action)` takes a manager, runs the action and closes it. The action cannot throw a checked exception, so a call to `send` has to be wrapped:

```java
factory.execute(m -> {
    try {
        m.queue("orders").send(order);
    } catch (JArtemisException e) {
        throw new RuntimeException(e);                    // the lambda cannot throw a checked exception
    }
});
```

A manager must never be used by two threads at once. Each thread takes its own:

```java
executor.submit(() -> {
    try (MessageManager m = factory.createMessageManager()) {   // each thread takes its own
        m.queue("orders").send(new Order("B-1", 1));
    }
    return null;
});
```

With a framework the factory becomes a bean, and the container closes it:

```java
@Configuration
public class JartemisConfig {
    @Bean(destroyMethod = "close")                        // the container closes the broker at shutdown
    public MessageManagerFactory messageManagerFactory(
            @Value("${jartemis.host:localhost}") String host,
            @Value("${jartemis.port:61717}") int port,
            @Value("${jartemis.storage}") String storage) throws JArtemisException {
        return EmbeddedArtemisBroker.bind(host, port)
                .storage(Path.of(storage))
                .autoReconnect(1000)
                .createMessageManagerFactory();
    }
}
```

## 5. Sending

A queue or topic is reached through a manager by name. Sending looks the same for both:

```java
import org.j2os.platform.jartemis.*;
import java.nio.file.Path;

public class Send {
    record Order(String id, int qty) {}

    public static void main(String[] args) throws Exception {
        try (MessageManagerFactory factory = EmbeddedArtemisBroker
                .bind("127.0.0.1", 61617)
                .storage(Path.of("broker-data"))
                .createMessageManagerFactory();
             MessageManager m = factory.createMessageManager()) {

            m.queue("orders").send(new Order("A-1", 3));
            m.queue("orders").timeout(2000).send(new Order("A-2", 1));
            System.out.println("two orders sent");

            try { m.queue("orders").send(new Object()); }
            catch (Exception e) { System.out.println("empty bean -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

            try { m.queue(" ").send("x"); }
            catch (Exception e) { System.out.println("blank name -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

            try { m.queue("orders").timeout(0); }
            catch (Exception e) { System.out.println("timeout 0 -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
        }
    }
}
```

```text
two orders sent
empty bean -> JArtemisException: Serializing the message body with Jackson failed.
blank name -> IllegalArgumentException: queueName must not be null or blank.
timeout 0 -> IllegalArgumentException: millis must be a positive number; given value: 0
```

- **Two sends.** The first waits for the broker to accept the message, however long that takes. The second, with `timeout(2000)`, waits at most two seconds and then throws a `JArtemisException`. Such a timeout only means that no answer arrived yet; the message may still arrive later.
- **Empty bean.** `new Object()` has nothing to write as JSON, so the send fails before anything leaves the program: `Serializing the message body with Jackson failed.` The Jackson error is the cause.
- **Blank name.** A queue or topic name must not be blank.
- **Timeout 0.** A timeout must be positive. A `timeout` belongs to the one `queue(...)` or `topic(...)` object it was called on.

## 6. Reading from a queue

`getMessage` takes one message:

```java
import org.j2os.platform.jartemis.*;
import java.nio.file.Path;

public class Receive {
    record Order(String id, int qty) {}

    public static void main(String[] args) throws Exception {
        try (MessageManagerFactory factory = EmbeddedArtemisBroker
                .bind("127.0.0.1", 61618)
                .storage(Path.of("broker-data"))
                .createMessageManagerFactory();
             MessageManager m = factory.createMessageManager()) {

            m.queue("orders").send(new Order("A-1", 3));
            m.queue("orders").send(new Order("A-2", 1));

            System.out.println(m.queue("orders").timeout(500).getMessage(Order.class));
            System.out.println(m.queue("orders").timeout(500).getMessage(Order.class));
            System.out.println(m.queue("orders").timeout(500).getMessage(Order.class));

            m.queue("notes").send("just a text");
            for (int i = 1; i <= 2; i++) {
                try { m.queue("notes").timeout(500).getMessage(Order.class); }
                catch (Exception e) { System.out.println("try " + i + " -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
            }
            System.out.println(m.queue("notes").timeout(500).getMessage(String.class));
        }
    }
}
```

```text
Order[id=A-1, qty=3]
Order[id=A-2, qty=1]
null
try 1 -> JArtemisException: Deserializing the message body to type Receive$Order failed.
try 2 -> JArtemisException: Deserializing the message body to type Receive$Order failed.
just a text
```

- **First two lines.** Messages come out in the order they went in.
- **Third line.** The queue is empty. After the timeout `getMessage` returns `null`. It is not an error. Without a `timeout` it waits for ever.
- **`try 1` and `try 2`.** The text `just a text` cannot become an `Order`. The call fails with `Deserializing the message body to type ... failed.`, and the message is **not** removed. The second try meets the same message and fails the same way.
- **Last line.** The same message read as a `String` works, and now it is gone.

A message that always fails will block a reader that always asks for the same type. Read it as a `String`, log it and decide what to do with it:

```java
try {
    Order o = m.queue("orders").timeout(500).getMessage(Order.class);
} catch (JArtemisException bad) {
    // the same message is still in the queue and will fail the same way
    // read it as String, log it, then decide: park it in another queue, or drop it
    String raw = m.queue("orders").timeout(500).getMessage(String.class);
}
```

## 7. Listening to a queue

`getMessage` asks. `onMessage` instead gives the library a piece of code to run for each message:

```java
import org.j2os.platform.jartemis.*;
import java.nio.file.Path;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class Listen {
    public static void main(String[] args) throws Exception {
        try (MessageManagerFactory factory = EmbeddedArtemisBroker
                .bind("127.0.0.1", 61619)
                .storage(Path.of("broker-data"))
                .createMessageManagerFactory();
             MessageManager m = factory.createMessageManager()) {

            AtomicInteger a = new AtomicInteger(), b = new AtomicInteger();
            CountDownLatch six = new CountDownLatch(6);

            Subscription workerA = m.queue("jobs").onMessage(String.class, job -> { a.incrementAndGet(); six.countDown(); });
            Subscription workerB = m.queue("jobs").onMessage(String.class, job -> { b.incrementAndGet(); six.countDown(); });

            for (int i = 1; i <= 6; i++) m.queue("jobs").send("job-" + i);
            six.await(5, TimeUnit.SECONDS);
            System.out.println("handled: " + (a.get() + b.get()));
            System.out.println("both workers took part: " + (a.get() > 0 && b.get() > 0));

            workerA.cancel();
            workerB.cancel();
            m.queue("jobs").send("job-7");
            System.out.println("waiting in the queue: " + m.queue("jobs").timeout(500).getMessage(String.class));
        }
    }
}
```

```text
handled: 6
both workers took part: true
waiting in the queue: job-7
```

- **Two workers on one queue.** Each of the six jobs went to exactly one of them, so the total is six. The broker shares the jobs among the listeners; do not rely on an exact split.
- **After `cancel`.** Cancelling stops the delivery. A message sent later simply waits in the queue, and `getMessage` finds it.
- **The thread.** The handler runs on a thread of the library, one message after another per subscription. It must not use a manager that another thread is using; give it a manager of its own.
- **Closing a manager** cancels all its subscriptions.

If the handler throws, the message is not acknowledged and the broker may deliver it again, so a handler should be able to run twice for the same message:

```java
m.queue("orders").onMessage(Order.class, order -> {
    if (!stockHas(order)) {
        throw new IllegalStateException("out of stock");  // not acknowledged: the broker may deliver it again
    }
    ship(order);
});
```

In a framework, a listener is started once and stopped with the application:

```java
@Component
public class OrderListener {
    private final MessageManagerFactory factory;
    private MessageManager manager;
    private Subscription subscription;

    public OrderListener(MessageManagerFactory factory) { this.factory = factory; }

    @PostConstruct
    public void start() throws JArtemisException {
        manager = factory.createMessageManager();                      // its own manager
        subscription = manager.queue("orders").onMessage(Order.class, this::handle);
    }

    private void handle(Order order) { /* work */ }

    @PreDestroy
    public void stop() {
        subscription.cancel();
        manager.close();
    }
}
```

## 8. Sending without waiting

`sendAsync` returns at once and reports the outcome later:

```java
import org.j2os.platform.jartemis.*;
import java.nio.file.Path;
import java.util.concurrent.*;

public class Async {
    public static void main(String[] args) throws Exception {
        try (MessageManagerFactory factory = EmbeddedArtemisBroker
                .bind("127.0.0.1", 61620)
                .storage(Path.of("broker-data"))
                .createMessageManagerFactory();
             MessageManager m = factory.createMessageManager()) {

            CountDownLatch done = new CountDownLatch(1);
            m.queue("logs").sendAsync("first line", failure -> {
                System.out.println(failure == null ? "broker accepted it" : "failed: " + failure);
                done.countDown();
            });
            done.await(5, TimeUnit.SECONDS);

            try { m.queue("logs").sendAsync("x", null); }
            catch (Exception e) { System.out.println("no callback -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
        }
    }
}
```

```text
broker accepted it
no callback -> IllegalArgumentException: callback must not be null.
```

- **The callback.** It receives `null` when the broker accepted the message, or the exception when it did not. It is called at most once, on a thread of the library.
- **With `timeout(ms)`** on the queue or topic, a message that is not confirmed within the time makes the callback receive a `JArtemisException`. The message may still arrive later.
- **No callback.** A `null` callback is refused straight away.
- Inside a transaction, a failed `sendAsync` marks the transaction; the next call on it (`begin`, `commit`, `rollback`, `send`, `getMessage`) rolls it back and throws `TransactionRolledBackException` (chapter 12).

## 9. Topics

A topic delivers a copy to everyone who listens at that moment:

```java
import org.j2os.platform.jartemis.*;
import java.nio.file.Path;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class Live {
    record Event(String text) {}

    public static void main(String[] args) throws Exception {
        try (MessageManagerFactory factory = EmbeddedArtemisBroker
                .bind("127.0.0.1", 61621)
                .storage(Path.of("broker-data"))
                .createMessageManagerFactory();
             MessageManager m = factory.createMessageManager()) {

            AtomicInteger screen = new AtomicInteger(), logger = new AtomicInteger();
            Subscription s1 = m.topic("news").fromCurrentMessage().onMessage(Event.class, e -> screen.incrementAndGet());
            Subscription s2 = m.topic("news").fromCurrentMessage().onMessage(Event.class, e -> logger.incrementAndGet());

            m.topic("news").send(new Event("one"));
            m.topic("news").send(new Event("two"));
            Thread.sleep(300);
            System.out.println("screen=" + screen + " logger=" + logger);

            AtomicInteger late = new AtomicInteger();
            Subscription s3 = m.topic("news").fromCurrentMessage().onMessage(Event.class, e -> late.incrementAndGet());
            Thread.sleep(300);
            System.out.println("late subscriber saw " + late + " of the old events");

            try { m.topic(" "); }
            catch (Exception e) { System.out.println("blank name -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
        }
    }
}
```

```text
screen=2 logger=2
late subscriber saw 0 of the old events
blank name -> IllegalArgumentException: topicName must not be null or blank.
```

- **Two subscribers.** Both got both events: `screen=2 logger=2`. This is the difference from a queue.
- **Late subscriber.** `fromCurrentMessage()` means "from now on". It sees nothing of what was sent before, and nothing at all if it is away when something is sent.
- **Blank name.** A topic name must not be blank.
- `send` and `sendAsync` on a topic work like those on a queue, including `timeout`.

## 10. Durable subscriptions

A subscription that must not miss anything has a name:

```java
import org.j2os.platform.jartemis.*;
import java.nio.file.Path;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class Durable {
    record Event(String text) {}

    public static void main(String[] args) throws Exception {
        try (MessageManagerFactory factory = EmbeddedArtemisBroker
                .bind("127.0.0.1", 61622)
                .storage(Path.of("broker-data"))
                .createMessageManagerFactory();
             MessageManager m = factory.createMessageManager()) {

            Subscription audit = m.topic("news").fromLastReceivedMessage("audit")
                    .onMessage(Event.class, e -> System.out.println("audit (online):  " + e.text()));
            m.topic("news").send(new Event("one"));
            Thread.sleep(300);
            audit.cancel();

            m.topic("news").send(new Event("two"));
            m.topic("news").send(new Event("three"));

            Subscription back = m.topic("news").fromLastReceivedMessage("audit")
                    .onMessage(Event.class, e -> System.out.println("audit (back):    " + e.text()));
            Thread.sleep(300);
            back.cancel();

            try { m.topic("news").fromLastReceivedMessage(""); }
            catch (Exception e) { System.out.println("empty id -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
        }
    }
}
```

```text
audit (online):  one
audit (back):    two
audit (back):    three
empty id -> IllegalArgumentException: durableSubscriptionId must not be null or blank.
```

- **`audit (online)`.** The subscriber is connected and receives `one`.
- **Away.** After `cancel()` the program sends `two` and `three`. The broker keeps them for the name `audit`.
- **`audit (back)`.** A new subscription with the same name receives the two missed messages and then goes on.
- **Empty id.** The subscription name must not be blank.

The broker recognizes a durable subscription by the **program's id** together with the **name**. The id comes from the library: for an embedded broker it is made once and stored in the `storage` directory, so it survives restarts. A client has no storage, so it gets a new random id at every start, and its durable subscriptions would not resume. Give it a fixed one:

```java
MessageManagerFactory factory = EmbeddedArtemisBroker
        .remote("broker.example.org", 61616)
        .credentials("shop", "...")
        .clientId("shop-billing")                         // stable and unique for this program
        .createMessageManagerFactory();

factory.createMessageManager().topic("news")
        .fromLastReceivedMessage("billing")               // now it can resume after a restart
        .onMessage(Event.class, e -> {});
```

A cancelled durable subscription still exists on the broker and keeps collecting messages. If nobody will ever return to it, it is wasted space.

## 11. Keeping history

Normally a new subscriber sees only what comes after it. `keepHistory(n)` makes the broker remember the last `n` messages of a topic for subscribers that arrive later:

```java
import org.j2os.platform.jartemis.*;
import java.nio.file.Path;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class History {
    record Event(String text) {}

    public static void main(String[] args) throws Exception {
        try (MessageManagerFactory factory = EmbeddedArtemisBroker
                .bind("127.0.0.1", 61623)
                .storage(Path.of("broker-data"))
                .createMessageManagerFactory();
             MessageManager m = factory.createMessageManager()) {

            m.topic("prices").keepHistory(2);
            for (int i = 1; i <= 4; i++) m.topic("prices").send(new Event("price-" + i));

            Subscription durable = m.topic("prices").fromLastReceivedMessage("chart")
                    .onMessage(Event.class, e -> System.out.println("durable: " + e.text()));
            Subscription live = m.topic("prices").fromCurrentMessage()
                    .onMessage(Event.class, e -> System.out.println("live:    " + e.text()));
            Thread.sleep(300);
            m.topic("prices").send(new Event("price-5"));
            Thread.sleep(300);
            live.cancel(); durable.cancel();

            try { m.topic("prices").keepHistory(0); }
            catch (Exception e) { System.out.println("keepHistory(0) -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
        }
    }
}
```

```text
durable: price-3
durable: price-4
durable: price-5
live:    price-5
keepHistory(0) -> IllegalArgumentException: keepHistory only accepts a positive value; Artemis has no "unlimited" option for this setting (retroactiveMessageCount<=0 means no history at all, not unlimited).
```

- **Setting.** `keepHistory(2)` keeps the last two messages. Four prices were sent with nobody listening; `price-3` and `price-4` were kept.
- **Durable subscriber.** A new durable subscription starts with the history (`price-3`, `price-4`) and then receives `price-5`.
- **Live subscriber.** `fromCurrentMessage()` never gets history, even when `keepHistory` is on. It sees only `price-5`. To make this sure, the library sends a hidden message to the topic when the subscription is created; it is never shown to any handler, but it uses one place of the history like any other message.
- **`keepHistory(0)`.** Zero and negative numbers are refused. There is no "unlimited" setting, and the number is a count of messages.
- Asking for a different number for a topic that already has one is ignored, and a warning is logged.
- It works only on a program that is itself a broker, and it applies to that broker alone (chapter 13 and 14).

## 12. Transactions

A manager created with `setAutoCommit(false)` works in transactions. Nothing it sends is visible to others until `commit()`, and `rollback()` throws the work away:

```java
import org.j2os.platform.jartemis.*;
import java.nio.file.Path;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class Transactions {
    record Order(String id) {}

    public static void main(String[] args) throws Exception {
        try (MessageManagerFactory factory = EmbeddedArtemisBroker
                .bind("127.0.0.1", 61624)
                .storage(Path.of("broker-data"))
                .createMessageManagerFactory();
             MessageManager tx = factory.setAutoCommit(false).createMessageManager();
             MessageManager plain = factory.createMessageManager()) {

            tx.getTransaction().begin();
            tx.queue("orders").send(new Order("A-1"));
            tx.queue("orders").send(new Order("A-2"));
            tx.getTransaction().rollback();
            System.out.println("after rollback: " + plain.queue("orders").timeout(300).getMessage(Order.class));

            tx.getTransaction().begin();
            tx.queue("orders").send(new Order("A-1"));
            tx.queue("orders").send(new Order("A-2"));
            tx.getTransaction().commit();
            System.out.println("after commit:   " + plain.queue("orders").timeout(300).getMessage(Order.class));

            tx.getTransaction().begin();
            System.out.println("read inside tx: " + tx.queue("orders").timeout(300).getMessage(Order.class));
            tx.getTransaction().rollback();
            System.out.println("after rollback: " + plain.queue("orders").timeout(300).getMessage(Order.class));

            try { tx.queue("orders").send(new Order("A-3")); }
            catch (Exception e) { System.out.println("no begin -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

            try { plain.getTransaction().begin(); }
            catch (Exception e) { System.out.println("autocommit begin -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

            tx.getTransaction().begin();
            try { tx.getTransaction().begin(); }
            catch (Exception e) { System.out.println("begin twice -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
            tx.getTransaction().rollback();
        }
    }
}
```

```text
after rollback: null
after commit:   Order[id=A-1]
read inside tx: Order[id=A-2]
after rollback: Order[id=A-2]
no begin -> IllegalStateException: This MessageManager was created with setAutoCommit(false), but no transaction has been started via getTransaction().begin(). begin() must be called before send/getMessage.
autocommit begin -> IllegalStateException: begin() is only meaningful on a MessageManager created with setAutoCommit(false).
begin twice -> IllegalStateException: A transaction is already in progress; commit() or rollback() must be called before begin() again.
```

- **After rollback.** The two orders were sent and rolled back; a second manager finds nothing.
- **After commit.** The same two orders were committed and can be read; `A-1` was taken.
- **Reading inside a transaction.** `A-2` was read, then the transaction was rolled back, and the message was back in the queue for the next reader.
- **No begin.** Every send or read needs `begin()` first on such a manager.
- **Auto-commit manager.** `begin()` only exists for a manager created with `setAutoCommit(false)`.
- **Begin twice.** Finish a transaction (`commit` or `rollback`) before starting another.

When something goes wrong inside a transaction, the library rolls it back for you and throws `TransactionRolledBackException`, a subclass of `JArtemisException`:

```java
import org.j2os.platform.jartemis.*;
import java.nio.file.Path;

public class Poison {
    record Order(String id) {}

    public static void main(String[] args) throws Exception {
        try (MessageManagerFactory factory = EmbeddedArtemisBroker
                .bind("127.0.0.1", 61631)
                .storage(Path.of("broker-data"))
                .createMessageManagerFactory();
             MessageManager tx = factory.setAutoCommit(false).createMessageManager();
             MessageManager plain = factory.createMessageManager()) {

            plain.queue("orders").send("this is not an order");

            tx.getTransaction().begin();
            try { tx.queue("orders").timeout(500).getMessage(Order.class); }
            catch (Exception e) { System.out.println("read -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

            System.out.println("transaction still active: " + tx.getTransaction().isActive());
            System.out.println("message still queued: " + plain.queue("orders").timeout(500).getMessage(String.class));
        }
    }
}
```

```text
read -> TransactionRolledBackException: Reading/deserializing the message failed (it may not have been a TextMessage); the transaction was rolled back so the malformed message would not be committed.
transaction still active: false
message still queued: this is not an order
```

- **The read.** The body is not an `Order`, so the read fails and the transaction is rolled back.
- **After it.** The transaction is no longer active, and the message is still queued; reading it inside a new transaction would fail again.
- If a `commit` fails without a clear answer from the broker, the outcome is unknown: it may even have succeeded. The exception says so; do not simply send the same work again.
- If the rollback itself fails, the exception asks you to close the manager and take a new one.

## 13. A client of another broker

`remote(host, port)` connects to a broker that is not part of the program. To keep the example in one program, the broker here is a second factory with `bind`:

```java
import org.j2os.platform.jartemis.*;
import java.nio.file.Path;

public class Remote {
    record Order(String id) {}

    public static void main(String[] args) throws Exception {
        try (MessageManagerFactory server = EmbeddedArtemisBroker
                .bind("127.0.0.1", 61625)
                .credentials("app", "secret")
                .storage(Path.of("broker-data"))
                .createMessageManagerFactory();
             MessageManagerFactory client = EmbeddedArtemisBroker
                .remote("127.0.0.1", 61625)
                .credentials("app", "secret")
                .createMessageManagerFactory();
             MessageManager c = client.createMessageManager();
             MessageManager s = server.createMessageManager()) {

            c.queue("orders").send(new Order("R-1"));
            System.out.println("server read " + s.queue("orders").timeout(500).getMessage(Order.class));

            try { c.topic("prices").keepHistory(10); }
            catch (Exception e) { System.out.println("keepHistory on a client -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
        }

        try { EmbeddedArtemisBroker.remote("127.0.0.1", 61626).createMessageManagerFactory(); }
        catch (Exception e) { System.out.println("no broker there -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try (MessageManagerFactory server = EmbeddedArtemisBroker.bind("127.0.0.1", 61627)
                .credentials("app", "secret").storage(Path.of("broker-data")).createMessageManagerFactory()) {
            try { EmbeddedArtemisBroker.remote("127.0.0.1", 61627).credentials("app", "wrong").createMessageManagerFactory(); }
            catch (Exception e) { System.out.println("wrong password -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
        }
    }
}
```

```text
server read Order[id=R-1]
keepHistory on a client -> JArtemisException: keepHistory can only be applied on a node that is itself an embedded broker (not merely a remote client), since it must be applied directly to an ActiveMQServer.
no broker there -> JArtemisException: Opening the shared JMS Connection failed.
wrong password -> JArtemisException: Opening the shared JMS Connection failed.
```

- **Read.** An order sent by the client was read on the broker's side. After the connection, queues and topics work as in the embedded case.
- **`keepHistory`.** Not available to a client; the setting must be applied on the broker itself.
- **No broker there.** The connection cannot be opened: `Opening the shared JMS Connection failed.` The cause says why.
- **Wrong password.** The same message; the cause tells that the login was refused.

What differs from the embedded case:

| | `bind` | `remote` |
|---|--------|----------|
| `storage(...)` | Required | Not used |
| `credentials(...)` | Creates the user | The user must already exist on that broker |
| `keepHistory(...)` | Works | `JArtemisException` |
| `replicationPair(...)` | Allowed | Refused |
| Program id | Stored in `storage`, stable | Random at every start, unless `clientId(...)` is given |

With `autoReconnect(ms)`, the client keeps trying to connect also the first time, so `createMessageManagerFactory()` may wait for a broker that is not up yet instead of failing.

## 14. Several machines

One embedded broker is a single point of failure. A **replication pair** adds a second broker, the **backup**, that receives a copy of everything the **primary** gets. If the primary dies, the broker itself promotes the backup; when the primary returns, it takes its role back.

All machines run the same code. Each finds its role by comparing its own `bind` address with the pairs:

```java
// the same code runs on all six machines; each one finds its own role from bind(...)
MessageManagerFactory factory = EmbeddedArtemisBroker
        .bind(myAddress, 61616)                           // 10.0.0.1 ... 10.0.0.6
        .storage(Path.of("/data/jartemis"))
        .autoReconnect(1000)                              // needed so clients follow a failover
        .replicationPair("10.0.0.1", 61616, "10.0.0.2", 61616)   // primary, backup
        .replicationPair("10.0.0.3", 61616, "10.0.0.4", 61616)
        .replicationPair("10.0.0.5", 61616, "10.0.0.6", 61616)
        .createMessageManagerFactory();
```

- **`autoReconnect` is part of the setup.** The broker side of a failover is automatic, but a client connection that was not told to reconnect stays where it was. A warning is logged when pairs are declared without it.
- **Three pairs at least.** With fewer than three pairs, the vote that confirms a dead primary is switched off, and there is a risk of split-brain if only the network between the two is cut. The library logs a warning and goes on.
- **The messages are shared.** Pairs form one cluster, so a queue or topic is reached from every node.
- **`keepHistory` is per node.** It must be called on every node that should keep history.

The library checks the declaration before it starts anything:

```java
import org.j2os.platform.jartemis.*;
import java.nio.file.Path;

public class Topology {
    static EmbeddedArtemisBroker node() {
        return EmbeddedArtemisBroker.bind("10.0.0.1", 61616).storage(Path.of("broker-data")).autoReconnect(2000);
    }

    public static void main(String[] args) {
        try { node().replicationPair("10.0.0.1", 61616, "10.0.0.2", 61616)
                    .replicationPair("10.0.0.2", 61616, "10.0.0.1", 61616).createMessageManagerFactory(); }
        catch (Exception e) { System.out.println("mirror pair -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { node().replicationPair("10.0.0.1", 61616, "10.0.0.2", 61616)
                    .replicationPair("10.0.0.1", 61616, "10.0.0.3", 61616).createMessageManagerFactory(); }
        catch (Exception e) { System.out.println("two backups -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { node().replicationPair("10.0.0.1", 61616, "10.0.0.2", 61616)
                    .replicationPair("10.0.0.2", 61616, "10.0.0.3", 61616).createMessageManagerFactory(); }
        catch (Exception e) { System.out.println("chain -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { EmbeddedArtemisBroker.bind("10.0.0.1", 7777).storage(Path.of("broker-data")).autoReconnect(2000)
                    .replicationPair("10.0.0.1", 61616, "10.0.0.2", 61616).createMessageManagerFactory(); }
        catch (Exception e) { System.out.println("wrong port -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
    }
}
```

```text
mirror pair -> ReplicationTopologyException: Conflicting replicationPair definition: node 10.0.0.1:61616 is both the primary and the backup of node 10.0.0.2:61616.
two backups -> ReplicationTopologyException: Node 10.0.0.1:61616 has two different backups (10.0.0.2:61616 and 10.0.0.3:61616), but Artemis does not allow a primary to have more than one active backup at the same time (the second backup is rejected with ActiveMQAlreadyReplicatingException). If you need multiple standby copies, each primary must be its own separate, independent replicationPair, not shared.
chain -> ReplicationTopologyException: Address 10.0.0.2:61616 is used both as primary (paired with backup 10.0.0.3:61616) and as backup (paired with primary 10.0.0.1:61616) - a single broker instance cannot hold two HA roles at once. For multiple roles, use a separate port for each role (for example, 10.0.0.2:61616 for the backup role and a different port for the primary role).
wrong port -> JArtemisException: bind("10.0.0.1", 7777) does not match any of the ports declared for this host in replicationPair(...); this port would never have been bound. The bind(...) port must be exactly one of the ports under which this host appears as primary or backup in replicationPair(...).
```

- **Mirror pair.** `A` as primary of `B` and `B` as primary of `A` is a contradiction.
- **Two backups.** A primary has one backup. For more copies, make separate pairs.
- **Chain.** One address cannot be a backup in one pair and a primary in another. A node that needs both roles uses two ports, as the next snippet shows.
- **Wrong port.** The `bind` port must be one of the ports the host has in the pairs.

```java
.replicationPair("10.0.0.1", 61616, "10.0.0.2", 10)       // 10.0.0.2 is the backup of 10.0.0.1 on port 10 ...
.replicationPair("10.0.0.2", 20, "10.0.0.3", 61616)       // ... and the primary of 10.0.0.3 on port 20
```

For a node with two roles the library starts two brokers in the same JVM, each with its own files under `storage`.

## 15. The log

The library writes its events to the logger `org.j2os.platform.jartemis.connection` through `slf4j`, so they go wherever the program's logging goes. A run of chapter 2 produces:

```text
Node 127.0.0.1:61616 (embedded broker) connected.
Node 127.0.0.1:61616 (JMS connection) connected.
Producer connected to "orders".
Consumer connected to "orders".
Consumer on "orders" disconnected.
Node 127.0.0.1:61616 (embedded broker) disconnected.
```

- **INFO** lines tell about connections: a node, a producer, a consumer.
- **WARN** lines mark things that need a look: a role change in a cluster, fewer than three pairs, pairs without `autoReconnect`, a client with no stable program id, a `keepHistory` value that was ignored, and a handler that threw.
- A line for the producer is written at **every send**, and one for the consumer at every `getMessage`. A program that sends or polls in a tight loop writes a great many INFO lines; set the level of that logger to WARN for such programs.

## 16. Putting it together

A small shop. Orders go into a queue, a worker packs them and announces each one on a topic, and an audit subscriber with a durable name records the announcements. The shop sends in a transaction; one order is rolled back:

```java
import org.j2os.platform.jartemis.*;
import java.nio.file.Path;
import java.util.concurrent.*;

public class Workflow {
    record Order(String id, int qty) {}
    record Done(String id, String status) {}

    public static void main(String[] args) throws Exception {
        try (MessageManagerFactory factory = EmbeddedArtemisBroker
                .bind("127.0.0.1", 61630)
                .storage(Path.of("broker-data"))
                .createMessageManagerFactory();
             MessageManager shop = factory.setAutoCommit(false).createMessageManager();
             MessageManager worker = factory.createMessageManager();
             MessageManager board = factory.createMessageManager()) {

            CountDownLatch seen = new CountDownLatch(2);

            Subscription audit = board.topic("order-done").fromLastReceivedMessage("audit")
                    .onMessage(Done.class, d -> { System.out.println("audit: " + d); seen.countDown(); });

            Subscription packer = worker.queue("orders").onMessage(Order.class, o -> {
                try { worker.topic("order-done").send(new Done(o.id(), "packed")); }
                catch (Exception e) { throw new RuntimeException(e); }
            });

            shop.getTransaction().begin();
            shop.queue("orders").send(new Order("A-1", 2));
            shop.queue("orders").send(new Order("A-2", 5));
            shop.getTransaction().commit();

            shop.getTransaction().begin();
            shop.queue("orders").send(new Order("A-3", 9));
            shop.getTransaction().rollback();

            seen.await(5, TimeUnit.SECONDS);
            Thread.sleep(300);
            System.out.println("still waiting in the queue: " + board.queue("orders").timeout(300).getMessage(Order.class));
            packer.cancel();
            audit.cancel();
        }
    }
}
```

```text
audit: Done[id=A-1, status=packed]
audit: Done[id=A-2, status=packed]
still waiting in the queue: null
```

- The shop manager works in transactions: `A-1` and `A-2` are committed, `A-3` is rolled back and never exists.
- The worker listens on the queue and uses **its own manager** to publish on the topic, because the handler runs on another thread.
- The audit subscriber has a durable name, so it would receive the announcements even if it were started later.
- Nothing is left in the queue at the end, and the `try` block stops the broker.

---

## Appendix A. Quick reference

### A.1 EmbeddedArtemisBroker

| Call | Purpose |
|------|---------|
| `bind(host, port)` | Static. This program is an embedded broker. Host not blank and not `0.0.0.0`; port 1 to 65535 |
| `remote(host, port)` | Static. This program is a client of a broker elsewhere |
| `credentials(user, password)` | The user to create (`bind`) or to log in as (`remote`) |
| `storage(path)` | Directory for the broker's files. Required with `bind` |
| `clientId(id)` | A fixed program id for durable subscriptions |
| `autoReconnect(ms)` | Retry interval after a break, in milliseconds; must be positive |
| `replicationPair(primaryHost, primaryPort, backupHost, backupPort)` | Declares one primary and its backup. Only with `bind` |
| `createMessageManagerFactory()` | Validates, starts the broker, connects. Once per builder |

### A.2 MessageManagerFactory

| Call | Purpose |
|------|---------|
| `createMessageManager()` | A manager in auto-commit mode |
| `setAutoCommit(boolean).createMessageManager()` | A manager in auto-commit mode or in transaction mode |
| `execute(action)` | Runs the action with a manager and closes it. The action cannot throw a checked exception |
| `close()` | Closes the connection and stops the broker. Safe to call twice |

### A.3 MessageManager

| Call | Purpose |
|------|---------|
| `queue(name)` | A `MessageQueue`. The name must not be blank |
| `topic(name)` | A `MessageTopic`. The name must not be blank |
| `getTransaction()` | The `MessageTransaction` of this manager |
| `isAutoCommit()` | Whether it was created in auto-commit mode |
| `close()` | Cancels its subscriptions, closes its session. Not a thread-safe object |

### A.4 MessageQueue

| Call | Purpose |
|------|---------|
| `timeout(ms)` | Limit for this object; positive. Applies to `send`, `sendAsync` and `getMessage` |
| `send(body)` | Sends and waits for the broker (for ever, or for `timeout`) |
| `sendAsync(body, callback)` | Sends and returns; the callback gets `null` or the failure |
| `getMessage(type)` | Takes one message, or `null` after the timeout; waits for ever without one |
| `onMessage(type, handler)` | Listens; returns a `Subscription` |

### A.5 MessageTopic

| Call | Purpose |
|------|---------|
| `timeout(ms)` | Limit for `send` and `sendAsync` |
| `send(body)`, `sendAsync(body, callback)` | As on a queue |
| `keepHistory(n)` | Keep the last `n` messages for later subscribers. Only on a broker node |
| `fromCurrentMessage().onMessage(type, handler)` | A live subscription: only what is sent from now on |
| `fromLastReceivedMessage(id).onMessage(type, handler)` | A durable subscription named `id` |

### A.6 MessageTransaction

| Call | Purpose |
|------|---------|
| `begin()` | Starts a transaction. Only on a manager created with `setAutoCommit(false)` |
| `commit()` | Makes the work final |
| `rollback()` | Throws the work away |
| `isActive()` | Whether a transaction is running |

### A.7 Other types

| Type | Purpose |
|------|---------|
| `Subscription` | `cancel()` stops it; `close()` does the same |
| `SendCallback` | `onComplete(Throwable failureOrNull)` |
| `MessageHandler<T>` | `onMessage(T body)` |
| `JArtemisException` | Checked; the base of all library failures |
| `MissingStorageDirectoryException` | `bind` without `storage` |
| `ReplicationTopologyException` | A contradictory replication declaration; `getConflictingNodeAddresses()` |
| `TransactionRolledBackException` | A transaction was rolled back |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| Blank host, `0.0.0.0`, port out of range, non-positive interval or timeout | `IllegalArgumentException` at the call |
| `bind` without `storage` | `MissingStorageDirectoryException` from `createMessageManagerFactory()` |
| A builder used a second time | `JArtemisException` |
| `remote` with `replicationPair` | `JArtemisException` |
| A contradictory replication declaration | `ReplicationTopologyException` |
| `bind` port not among the pair ports | `JArtemisException` |
| The broker cannot start (port in use) | `JArtemisException`; the cause tells why |
| The connection cannot be opened (no broker, wrong login) | `JArtemisException` "Opening the shared JMS Connection failed." |
| A body that Jackson cannot write | `JArtemisException` "Serializing the message body with Jackson failed." |
| A body that does not fit the requested type | `JArtemisException` "Deserializing the message body ...", message stays queued |
| `getMessage` on an empty queue after the timeout | Returns `null` |
| A send that is not confirmed in time | `JArtemisException`; the message may still arrive |
| A send, read or subscribe that the broker or connection refuses | `JArtemisException` "Operation on queue/topic ... failed"; the cause tells why |
| The same inside a transaction | The transaction is rolled back; `TransactionRolledBackException` |
| A rollback that fails too | `JArtemisException`: close the manager and take a new one |
| A `commit` without a clear answer | `JArtemisException`: the outcome is unknown |
| A send or read on a transaction manager before `begin()` | `IllegalStateException` |
| `begin()` on an auto-commit manager, or twice | `IllegalStateException` |
| A handler that throws | Logged; the message is not acknowledged and may come again |
| `keepHistory` on a client | `JArtemisException` |
| `keepHistory` with zero or a negative number | `IllegalArgumentException` |
| `sendAsync` with a `null` callback | `IllegalArgumentException` |
| Subscription setup refused | `JArtemisException` "Setting up the subscription ... failed" |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Building a factory per request.** It starts a broker and opens a connection. Build it once.
- **Sharing a manager between threads.** Give each thread its own, including the threads of handlers.
- **Using in a handler a manager that other threads also use.** Give the handler one that only it touches.
- **Forgetting `close()`.** The broker keeps running until the factory is closed.
- **Reusing a builder.** Even after a failed attempt it refuses.
- **A host of `0.0.0.0`.** Refused; use the address clients will connect to.
- **No `timeout` on `getMessage`.** It waits for ever, which is right for a worker thread and wrong for a request thread.
- **Treating a send timeout as failure.** The message may still arrive; make the receiver able to see a repeat.
- **A handler that cannot run twice.** A message that was not acknowledged can come again.
- **A body that Jackson cannot read back.** The reader gets an error and the message stays; read it as `String` and decide. Typical causes: a class with no usable constructor, fields the reader's class does not have, and date types, since the library uses a plain Jackson setup without extra modules.
- **A generic body type.** `getMessage(List<Order>.class)` does not exist; wrap the list in a record.
- **A client with durable subscriptions and no `clientId`.** It starts a new subscription at every start.
- **Two independent programs with the same `clientId`.** The id must be unique.
- **A durable subscription nobody comes back to.** It collects messages for ever.
- **Counting on `keepHistory` for a client.** It does not exist there.
- **Calling `fromCurrentMessage()` in a loop.** Each call adds a hidden message that takes a place from the history.
- **Pairs without `autoReconnect`.** The brokers fail over, the clients do not follow.
- **Rolling back by throwing inside `execute`.** `execute` does not know transactions; use a manager with `setAutoCommit(false)`.

### C.2 Behavior worth knowing

- One factory has one connection, shared by all its managers; each manager has its own session.
- `getMessage` in auto-commit mode uses a separate short session and acknowledges the message only after it was read and converted. A failure leaves the message in the queue.
- `onMessage` works on a session of its own, so it is not tied to the manager's transaction.
- Producers are kept per manager and destination and reused.
- A failure in a transaction rolls it back at once, because the state of the session after an error is not certain.
- A bound node with `storage` has persistence switched on, with its files under `storage/port-<port>`; a file `.jartemis-instance-id` in `storage` holds the program id.
- With `credentials`, security is switched on and one role is created for the user.
- A cluster is configured with static connections to the other nodes; the primary checks for an already active server at start-up, and the backup hands the role back when the primary returns.
- The library also registers a shutdown hook that closes the connection and stops the brokers.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Factories per builder | One |
| Threads per manager | One |
| Message body | Anything Jackson can write and read; no extra Jackson modules |
| Backups per primary | One |
| Roles per port | One |
| `keepHistory` | A count only; no size limit and no "unlimited"; not on clients; per node |
| Client with `replicationPair` | Not allowed |
| Pairs for the vote | Three or more; fewer gives only a warning |
| Automatic recovery of a dead shared connection | None; a new factory is needed |

Limits that the example documentation of the module itself leaves open: whether `keepHistory` stays in force on a backup after a failover, how a durable subscription behaves after several reconnects in a row, and what happens to the factory when the shared connection is lost for good.

### C.4 Tuning and practical advice

- Keep the factory in one place (a field or a bean) and hand it to the code that needs it.
- Set `autoReconnect` in every program that should survive a restart of the broker.
- Put the host, port, storage and credentials in configuration.
- Use a stable `clientId` for every client that has durable subscriptions, and one `storage` directory per installation.
- Choose `timeout` values by the job: seconds for a request, none for a worker.
- Make handlers safe to run twice.
- Prefer small bodies: ids and a few fields, not documents.
- For several steps that must succeed together, use a transaction manager and keep the transaction short.
- Lower the level of the logger `org.j2os.platform.jartemis.connection` when sending in a tight loop.
- For high availability use three pairs or more, and give each node the same code.

## Appendix D. Message reference

### D.1 EmbeddedArtemisBroker

| Message | Exception | Cause |
|---------|-----------|-------|
| `host must not be null or blank.` | `IllegalArgumentException` | A blank host |
| `host must not be "0.0.0.0". ...` | `IllegalArgumentException` | `0.0.0.0` as host |
| `port must be between 1 and 65535; given value: %d` | `IllegalArgumentException` | Port out of range |
| `intervalMillis must be a positive number; given value: %d` | `IllegalArgumentException` | `autoReconnect` with zero or less |
| `This EmbeddedArtemisBroker instance has already been used ...` | `JArtemisException` | `createMessageManagerFactory()` twice on one builder |
| `Combining remote() with replicationPair() is invalid ...` | `JArtemisException` | A client with a pair |
| `bind("%s", %d) does not match any of the ports declared for this host in replicationPair(...) ...` | `JArtemisException` | The `bind` port is not one of the host's pair ports |
| `Node %s is an embedded broker ... but no storage(path) was configured.` | `MissingStorageDirectoryException` | `bind` without `storage` |

### D.2 ReplicationTopology

| Message | Exception | Cause |
|---------|-----------|-------|
| `Conflicting replicationPair definition: node %s is both the primary and the backup of node %s.` | `ReplicationTopologyException` | Two pairs that are the reverse of each other |
| `Address %s is used both as primary ... and as backup ...` | `ReplicationTopologyException` | One address in both roles |
| `A replicationPair was declared more than once with identical values. ...` | `ReplicationTopologyException` | A pair declared twice |
| `Node %s has two different backups (%s and %s) ...` | `ReplicationTopologyException` | Two backups for one primary |
| `Node %s is declared as the backup for two different primaries (%s and %s) ...` | `ReplicationTopologyException` | One backup for two primaries |

### D.3 MessageManagerFactory and MessageManager

| Message | Exception | Cause |
|---------|-----------|-------|
| `Starting the embedded broker(s) on %s failed.` | `JArtemisException` | The broker could not start; see the cause |
| `Opening the shared JMS Connection failed.` | `JArtemisException` | No broker, wrong login, or the connection was refused |
| `Starting the shared JMS Connection failed after creation.` | `JArtemisException` | The connection opened but could not be set up |
| `Reading/creating the persisted instance identifier at %s failed.` | `JArtemisException` | The file `.jartemis-instance-id` could not be read or written |
| `Creating the JMS Session failed.` | `JArtemisException` | A manager could not get its session |
| `This MessageManager was created with setAutoCommit(false), but no transaction has been started ...` | `IllegalStateException` | A send or read before `begin()` |

### D.4 MessageQueue and MessageTopic

| Message | Exception | Cause |
|---------|-----------|-------|
| `queueName must not be null or blank.` / `topicName must not be null or blank.` | `IllegalArgumentException` | A blank name |
| `millis must be a positive number; given value: %d` | `IllegalArgumentException` | `timeout` with zero or less |
| `callback must not be null.` | `IllegalArgumentException` | `sendAsync` with `null` |
| `durableSubscriptionId must not be null or blank.` | `IllegalArgumentException` | A blank subscription name |
| `keepHistory only accepts a positive value; ...` | `IllegalArgumentException` | `keepHistory` with zero or less |
| `keepHistory can only be applied on a node that is itself an embedded broker ...` | `JArtemisException` | `keepHistory` on a client |
| `Applying keepHistory on the broker for topic "%s" failed.` | `JArtemisException` | The broker refused the setting |
| `Serializing the message body with Jackson failed.` | `JArtemisException` | The body cannot be written as JSON |
| `Deserializing the message body to type %s failed.` | `JArtemisException` | The body does not fit the requested type |
| `Send on queue "%s" was not acknowledged within %d ms ...` | `JArtemisException` | `send` with `timeout` and no answer in time. In a transaction it is rolled back and the manager should be replaced |
| `Asynchronous send on queue "%s" was not acknowledged within %d ms ...` | `JArtemisException` (to the callback) | `sendAsync` with `timeout` and no answer in time |
| `Send on topic "%s" ...` / `Asynchronous send on topic "%s" ...` | `JArtemisException` | The same for topics |
| `Send on queue "%s" was interrupted while waiting (thread interrupt).` | `JArtemisException` | The waiting thread was interrupted |
| `Operation on queue "%s" failed (see this exception's cause ...)` | `JArtemisException` | A send or read that the broker or connection refused |
| `Operation on topic "%s" failed (see this exception's cause ...)` | `JArtemisException` | The same for a topic |
| `An operation inside the transaction failed ...; the transaction was rolled back.` | `TransactionRolledBackException` | Any such failure inside a transaction |
| `An operation inside the transaction failed, and the rollback attempt also failed ...` | `JArtemisException` | The rollback failed too; replace the manager |
| `Reading/deserializing the message failed ...; the transaction was rolled back ...` | `TransactionRolledBackException` | A bad body read inside a transaction |
| `Reading the message failed (it may not have been a TextMessage).` | `JArtemisException` | A message that is not text |
| `Setting up the subscription on queue "%s" failed ...` | `JArtemisException` | The broker refused the subscription |
| `Setting up the subscription on topic "%s" failed ...` | `JArtemisException` | The same for a topic |
| `Setting up fromCurrentMessage() on topic "%s" did not complete: ...` | `JArtemisException` | The hidden message did not return within 30 seconds; no subscription was made |

### D.5 MessageTransaction

| Message | Exception | Cause |
|---------|-----------|-------|
| `begin() is only meaningful on a MessageManager created with setAutoCommit(false).` | `IllegalStateException` | `begin()` on an auto-commit manager |
| `A transaction is already in progress; commit() or rollback() must be called before begin() again.` | `IllegalStateException` | `begin()` twice |
| `No transaction is in progress; begin() must be called first.` | `IllegalStateException` | `commit()` or `rollback()` without `begin()` |
| `The broker explicitly confirmed the transaction was rolled back.` | `TransactionRolledBackException` | `commit()` was refused |
| `commit() failed but the broker did not confirm a definitive rollback ...` | `JArtemisException` | The outcome is unknown; do not retry blindly |
| `rollback() failed on the broker ...` | `JArtemisException` | Replace the manager |
| `A sendAsync inside this transaction had failed; the transaction was rolled back ...` | `TransactionRolledBackException` | An asynchronous send failed earlier in the transaction |
| `A sendAsync inside this transaction did not receive acknowledgement within its configured deadline ...` | `JArtemisException` | An asynchronous send was not confirmed in time; replace the manager |

### D.6 Log messages

| Level | Message | Meaning |
|-------|---------|---------|
| INFO | `Node {} ({}) connected.` / `disconnected.` | An embedded broker or a connection started or stopped |
| INFO | `Producer connected to "{}".` | A message was sent to a queue or topic |
| INFO | `Consumer{} connected to "{}".` / `Consumer{} on "{}" disconnected.` | A reader or listener started or stopped; `(durable)` is added for durable subscriptions |
| INFO | `keepHistory applied for topic "{}" ...` | The history setting was applied |
| WARN | `keepHistory for topic "{}" ... was ignored.` | A different number for a topic that already had one |
| WARN | `No explicit clientId was configured ...` | A random program id was made; durable subscriptions will not resume |
| WARN | `The HA topology has fewer than 3 HA pairs; ...` | Voting is off, with a risk of split-brain |
| WARN | `replicationPair(...) is configured but autoReconnect(...) was not called ...` | Clients will not follow a failover |
| WARN | `Role change on node {}: from {} to {} (reason: {})` | A failover or failback happened |
| WARN | `Unexpected Connection-level error: {}` | The shared connection reported an error |
| WARN | `Reading the message body on ... failed ...` / `Deserializing the message on ... failed ...` | A listener could not read a message; it was not acknowledged |
