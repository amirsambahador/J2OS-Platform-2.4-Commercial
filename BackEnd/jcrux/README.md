# jcrux

*J2OS Platform 2.4*

Sooner or later two programs need to look at the same thing: a shopping cart that a web service and a background job both change, a counter of stock, a table of settings. The usual answer is a database, but sometimes a database is more than the job needs. All that is wanted is a plain Java object that lives in one place, that every program can call, and that is still there after a restart.

**jcrux** (`org.j2os.platform.jcrux`) is exactly that: a **container of live objects**. A server keeps the objects; clients create them, call their methods and read or write their fields over RMI (Java's own remote call mechanism). Clients never hold the object itself, only a short text **id**. Three things set it apart from the rest:

- **Real objects, no mapping.** Any serializable class works. There are no tables, no annotations and no interface to implement.
- **The code runs where the object is.** `invoke(id, "add", 5)` runs `add(5)` inside the server, so every client sees the same state at once.
- **Persistent on demand.** The container can be written to one file, either by a call or after every change, and is read back when the server starts.

Everything in one small example:

```java
// on the server: one container, published under a name
JCruxServer container = new JCruxServer("my-token", "data/shop.dat", true);
LocateRegistry.createRegistry(1099);
Naming.rebind("//localhost/shop", container);

// in any client: create an object once, then call it from anywhere
JCruxClient client = new JCruxClient();
client.connect("localhost", "shop", "my-token");
String cart = client.create("shop.Cart", "sara");        // the object lives in the server
client.invoke(cart, "add", "pen", 2);                      // the call runs in the server
System.out.println(client.invoke(cart, "total"));
```

In order, the chapters cover: starting a server (chapter 2), connecting (3), creating objects (4), calling methods (5), fields (6), getting, listing and removing (7), saving (8), adding classes while the server runs (9), watching the server (10), several clients at once (11), security (12) and a complete small workflow (13). The appendices are meant for quick lookups.

## Contents

1. [How jcrux works](#1-how-jcrux-works)
2. [Starting a server](#2-starting-a-server)
3. [Connecting a client](#3-connecting-a-client)
4. [Creating objects](#4-creating-objects)
5. [Calling methods](#5-calling-methods)
6. [Reading and writing fields](#6-reading-and-writing-fields)
7. [Getting, listing and removing](#7-getting-listing-and-removing)
8. [Saving and loading](#8-saving-and-loading)
9. [Adding classes while the server runs](#9-adding-classes-while-the-server-runs)
10. [Looking at the server](#10-looking-at-the-server)
11. [Several clients and threads](#11-several-clients-and-threads)
12. [Security](#12-security)
13. [A realistic scenario](#13-a-realistic-scenario)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The programs are complete, not fragments: each has one `main` method. All except `Setup`, `Security` and `Restart` are clients of the small server of chapter 2, which is started first, fresh and empty, for each program. They use a class `Counter` (an object with a label and a number) that both the server and the clients have on their classpath; its source is in chapter 4.

---

## 1. How jcrux works

Let us settle the vocabulary first:

- **Container.** The server object that holds all stored objects. It lives in one JVM and, if wanted, in one file.
- **Id.** Every stored object has a text id (a random UUID). The id is returned when an object is created, and it is all a client needs to use the object later.
- **RMI.** Remote Method Invocation: Java's way of calling an object in another JVM. The server is *bound* to a name in an *RMI registry* (a small directory service, by default on port 1099); a client *looks it up* by host and name.
- **Serialization.** Turning an object into bytes and back. Everything stored in jcrux must be `Serializable`, because objects are copied to bytes for saving, for `get()` and for results.
- **Token.** A shared secret the client sends with every call. A call with the wrong token is refused.

The classes a program meets:

| Class | Role |
|-------|------|
| `JCruxServer` | The container. Created once per container file and bound to an RMI name |
| `JCruxClient` | The client. Connects to one server and offers every operation |
| `JCruxObject` | Information about a stored object: id, type, creation and last access |

The three points to carry forward:

- **Clients pass ids, the server runs code.** `create`, `invoke`, `getField` and `setField` all happen on the server's copy of the object.
- **Everything that comes back is a copy.** The result of `invoke`, the value of `getField` and the object from `get` are serialized copies. Changing them changes nothing in the server.
- **Types come from the values.** jcrux finds a constructor or a method from the *runtime types* of the arguments, so an argument can never be `null` (chapters 4 and 5).

All failures reach the caller as exceptions. Appendix B lists what each situation produces.

## 2. Starting a server

A server needs a token and a file name. It is then bound to a name in an RMI registry:

| Constructor argument | Meaning |
|----------------------|---------|
| `token` | The shared secret. `null` is refused (`NullPointerException`) |
| `containerFile` | Where the container is written and read. The folder must already exist |
| `autosave` | `true` writes the file after every change; `false` only when asked |
| `serialFilter` | Optional. A list of classes allowed to arrive as arguments (chapter 12) |

The server of the examples is this small:

```java
import org.j2os.platform.jcrux.server.JCruxServer;

import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;

public class Server {
    public static void main(String[] args) throws Exception {
        JCruxServer container = new JCruxServer("demo-token", "demo.dat", false);

        LocateRegistry.createRegistry(1099);
        Naming.rebind("//localhost/demo", container);
        System.out.println("jcrux server is ready");
    }
}
```

What the constructor does, in order:

1. Applies the serial filter, if one was given.
2. Takes an exclusive lock on a file next to the container (`<file>.lock`), so that **only one server can use a container file at a time**.
3. Recompiles the classes added in earlier runs (chapter 9).
4. Reads the container file, if it exists, and restores the objects.

A failure at any step ends the constructor with an exception, and the lock is released. Binding to the registry is up to the program: `LocateRegistry.createRegistry(1099)` starts a registry in the same JVM, and `Naming.rebind` publishes the server under a name.

The next program starts several servers on purpose and shows what is refused:

```java
import org.j2os.platform.jcrux.server.JCruxServer;

import java.nio.file.Files;
import java.nio.file.Path;

public class Setup {
    public static void main(String[] args) throws Exception {
        JCruxServer first = new JCruxServer("t", "work/a.dat", false);
        System.out.println("1 first server on work/a.dat is up");

        try {
            new JCruxServer("t", "work/a.dat", false);
        } catch (Exception e) {
            System.out.println("2 a second server on the same file -> " + e.getClass().getSimpleName());
            System.out.println("  " + e.getMessage().split(";")[0]);
        }
        try {
            new JCruxServer(null, "work/b.dat", false);
        } catch (Exception e) {
            System.out.println("3 no token -> " + e.getClass().getSimpleName());
        }
        try {
            new JCruxServer("t", "work/missing/c.dat", false);
        } catch (Exception e) {
            System.out.println("4 the folder must exist -> " + e.getClass().getSimpleName());
        }
        try {
            new JCruxServer("t", "work/d.dat", false, " ");
        } catch (Exception e) {
            System.out.println("5 blank filter -> " + e.getMessage());
        }
        Files.writeString(Path.of("work/e.dat"), "not a container");
        try {
            new JCruxServer("t", "work/e.dat", false);
        } catch (Exception e) {
            System.out.println("6 a file that is not a container -> " + e.getClass().getSimpleName());
            System.out.println("  " + e.getMessage().split(";")[0]);
        }
        first.close();
        new JCruxServer("t", "work/a.dat", false).close();
        System.out.println("7 after close() the file can be used again");
        System.exit(0);
    }
}
```

```text
1 first server on work/a.dat is up
2 a second server on the same file -> RemoteException
  Another JCruxServer in this same JVM already holds the lock on work/a.dat
3 no token -> NullPointerException
4 the folder must exist -> RemoteException
5 blank filter -> serialFilter must not be blank
6 a file that is not a container -> RemoteException
  Failed to auto-restore container from work/e.dat
7 after close() the file can be used again
```

- **Case 2.** A second server on the same file is refused, as long as the first one is open. The same holds across JVMs and across programs: the lock is an operating system lock on the `.lock` file.
- **Case 3.** The token is mandatory.
- **Case 4.** jcrux creates the files but not the folder.
- **Case 6.** A file that exists but is not a container is never overwritten; the server refuses to start. A file that does not exist yet is fine: it is created at the first save.
- **Case 7.** `close()` releases the lock. It does **not** save. It is safe to call twice.

For a real server, save and close when the JVM ends:

```java
JCruxServer container = new JCruxServer(token, "data/shop.dat", true);
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    try {
        container.save();          // close() does not save by itself
        container.close();         // releases the lock file
    } catch (Exception ignored) { }
}));
```

After `close()`, the calls `save`, `load` and `addClass` on that object throw `IllegalStateException`.

## 3. Connecting a client

A client connects with the host (optionally with a port), the name the server was bound to and the token:

```java
import org.j2os.platform.jcrux.client.JCruxClient;

public class Connect {
    public static void main(String[] args) throws Exception {
        JCruxClient client = new JCruxClient();
        client.connect("localhost", "demo", "demo-token");
        System.out.println("1 container file: " + client.containerFilePath());
        System.out.println("2 objects in it: " + client.list().size());

        JCruxClient withPort = new JCruxClient();
        withPort.connect("localhost:1099", "demo", "demo-token");
        System.out.println("3 host and port: " + withPort.list().size());

        try {
            new JCruxClient().connect("localhost", "other", "demo-token");
        } catch (Exception e) {
            System.out.println("4 unknown service -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            new JCruxClient().connect("localhost:1098", "demo", "demo-token");
        } catch (Exception e) {
            System.out.println("5 nobody listens there -> " + e.getClass().getSimpleName());
        }

        JCruxClient wrong = new JCruxClient();
        wrong.connect("localhost", "demo", "nope");
        System.out.println("6 connect() accepted the wrong token");
        try {
            wrong.list();
        } catch (Exception e) {
            System.out.println("7 the first call fails -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
    }
}
```

```text
1 container file: demo.dat
2 objects in it: 0
3 host and port: 0
4 unknown service -> NotBoundException: other
5 nobody listens there -> ConnectException
6 connect() accepted the wrong token
7 the first call fails -> Exception: Invalid token
```

- **Cases 1 to 3.** `connect(ip, service, token)` looks the server up with `Naming.lookup("//" + ip + "/" + service)`. The host may carry a port: `"localhost:1099"`.
- **Cases 4 and 5.** An unknown name is a `NotBoundException`; a host where nothing listens is a `ConnectException`.
- **Cases 6 and 7.** **A wrong token is not noticed by `connect()`.** The token is checked on every call, so the first operation fails with `Exception: Invalid token`. A program that wants to know early can make a harmless call such as `list()` right after connecting.

One client object can be shared by many threads. A program that connects often should keep the client and reuse it, with a way to start over after a lost connection:

```java
public class Containers {
    private static JCruxClient client;

    public static synchronized JCruxClient get() throws Exception {
        if (client == null) {
            client = new JCruxClient();
            client.connect(System.getenv("JCRUX_HOST"), "shop", System.getenv("JCRUX_TOKEN"));
        }
        return client;
    }

    public static synchronized void forget() { client = null; }    // call after a ConnectException
}
```

```java
try {
    return Containers.get().invoke(id, "total");
} catch (java.rmi.ConnectException lost) {          // the server restarted or the network dropped
    Containers.forget();                             // connect again on the next call
    return Containers.get().invoke(id, "total");
}
```

## 4. Creating objects

`create(className, args...)` builds an object **inside the server** and returns its id. The class must be on the server's classpath. `put(object)` is the other way: the client builds the object and hands it over (the server needs the class too).

The class used in this guide:

```java
package shared;

public class Counter extends Entity {
    private static int created = 0;        // static state is not part of an object
    private String label;
    private int count;                     // primitive fields are fine

    public Counter(String label) {
        this.label = label;
        created++;
    }

    public Counter(String label, int start) {
        this(label);
        this.count = start;
    }

    public int add(int amount) {
        count += amount;
        return count;
    }

    public String describe() {
        return label + "=" + count;
    }

    public void fail() {
        count += 100;                      // changes the state, then fails
        throw new IllegalStateException("boom");
    }

    private String secret() {
        return "private " + label;
    }

    public static int twice(int n) {
        return n * 2;
    }
}
```

(`Entity` is a small superclass with a field `owner`.) And the program:

```java
import org.j2os.platform.jcrux.client.JCruxClient;
import shared.Counter;

public class Create {
    public static void main(String[] args) throws Exception {
        JCruxClient client = new JCruxClient();
        client.connect("localhost", "demo", "demo-token");

        String first = client.create("shared.Counter", "visits");
        String second = client.create("shared.Counter", "orders", 10);
        System.out.println("1 two objects, two ids: " + !first.equals(second));
        System.out.println("2 the constructor arguments were used: " + client.invoke(second, "describe"));

        String made = client.put(new Counter("built here", 5));
        System.out.println("3 put() stores an object made by the client: " + client.invoke(made, "describe"));

        try {
            client.create("shared.Counter", "x", null);
        } catch (Exception e) {
            System.out.println("4 null argument -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            client.create("shared.Counter", 42);
        } catch (Exception e) {
            System.out.println("5 no such constructor -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            client.create("shared.Broken");
        } catch (Exception e) {
            System.out.println("6 not serializable -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            client.create("shared.Nothing");
        } catch (Exception e) {
            System.out.println("7 unknown class -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        System.out.println("8 objects stored: " + client.list().size());
    }
}
```

```text
1 two objects, two ids: true
2 the constructor arguments were used: orders=10
3 put() stores an object made by the client: built here=5
4 null argument -> IllegalArgumentException: Parameter at index 1 is null — JCrux can't infer its type from a null value
5 no such constructor -> NoSuchMethodException: shared.Counter.<init>(int)
6 not serializable -> IllegalArgumentException: Object of type shared.Broken is not actually serializable — JCrux requires every stored object to serialize successfully, since save()/get()/invoke()/getField() all depend on it. Check for fields whose current value isn't Serializable.
7 unknown class -> ClassNotFoundException: shared.Nothing
8 objects stored: 3
```

How the constructor is chosen:

- By the **runtime types** of the arguments, in order. `"orders", 10` becomes `(String, Integer)`; if there is no such constructor, jcrux tries the primitive equivalents, so `Counter(String, int)` matches (case 2).
- There is **no matching by supertype and no overload choice**. An argument of type `ArrayList` does not match a parameter of type `List`. A constructor that takes an interface cannot be reached with `create`; use `put` instead.
- **Arguments can never be `null`** (case 4), because a `null` has no type to look at.
- Constructors that are not public work too.
- The object must **really serialize** (case 6). A class that implements `Serializable` but holds a field with a value that does not (a plain `Object` lock, a thread, a stream) is refused when created. jcrux checks at once, because saving and every later call would fail otherwise.

A class written for jcrux looks like any serializable class:

```java
package shop;

import java.io.Serializable;
import java.util.ArrayList;
import java.util.List;

public class Cart implements Serializable {
    private static final long serialVersionUID = 1L;

    private final String owner;
    private final List<String> items = new ArrayList<>();      // every field must serialize

    public Cart(String owner) { this.owner = owner; }

    public int add(String item, int quantity) {                 // runs in the server, one caller per object at a time
        for (int i = 0; i < quantity; i++) items.add(item);
        return items.size();
    }

    public int total() { return items.size(); }
}
```

Prefer `private final` fields and simple types, give the class a `serialVersionUID`, and keep the state in fields rather than in static variables: statics are not stored.

## 5. Calling methods

`invoke(id, method, args...)` runs a method of the stored object and returns its result:

```java
import org.j2os.platform.jcrux.client.JCruxClient;

public class Invoke {
    public static void main(String[] args) throws Exception {
        JCruxClient client = new JCruxClient();
        client.connect("localhost", "demo", "demo-token");
        String id = client.create("shared.Counter", "visits");

        System.out.println("1 add(5) returns " + client.invoke(id, "add", 5));
        System.out.println("2 add(2) returns " + client.invoke(id, "add", 2) + ": the object keeps its state");
        System.out.println("3 describe() returns " + client.invoke(id, "describe"));
        System.out.println("4 owner() is declared in the superclass: " + client.invoke(id, "owner"));
        System.out.println("5 a private method works too: " + client.invoke(id, "secret"));

        try {
            client.invoke(id, "fail");
        } catch (Exception e) {
            System.out.println("6 the method threw -> " + e.getClass().getSimpleName() + ", cause " + e.getCause());
        }
        System.out.println("7 but its change stayed: " + client.invoke(id, "describe"));

        try {
            client.invoke(id, "twice", 4);
        } catch (Exception e) {
            System.out.println("8 static method -> " + e.getClass().getSimpleName());
        }
        try {
            client.invoke(id, "missing");
        } catch (Exception e) {
            System.out.println("9 unknown method -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            client.invoke("no-such-id", "describe");
        } catch (Exception e) {
            System.out.println("10 unknown id -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
    }
}
```

```text
1 add(5) returns 5
2 add(2) returns 7: the object keeps its state
3 describe() returns visits=7
4 owner() is declared in the superclass: nobody
5 a private method works too: private visits
6 the method threw -> InvocationTargetException, cause java.lang.IllegalStateException: boom
7 but its change stayed: visits=107
8 static method -> IllegalArgumentException
9 unknown method -> NoSuchMethodException: missing not found in shared.Counter, its superclasses, or any interface it implements
10 unknown id -> NoSuchElementException: No instance for id no-such-id
```

- **Which method.** By name and the runtime types of the arguments, with the same primitive fallback as for constructors. Methods of the class, of its superclasses (case 4) and `default` methods of interfaces are found. Private methods are found too (case 5). Static methods are refused (case 8).
- **The result** is a copy. A method that returns a list or a map hands the client a separate copy; `void` and `null` come back as `null`.
- **If the method throws**, the client receives an `InvocationTargetException` whose cause is the original exception (case 6). **Changes made before the exception stay**, and with autosave on the container is still written (case 7). Write methods so that they check first and change afterwards.
- **Calls on one object run one at a time.** Calls on different objects run in parallel (chapter 11).
- A method that exists only in `Object` (such as `getClass`) is not reachable; a class that overrides `toString()` can be called as `toString`.

## 6. Reading and writing fields

`getField(id, name)` and `setField(id, name, value)` reach a field directly, without a getter or setter:

```java
import org.j2os.platform.jcrux.client.JCruxClient;

public class Fields {
    public static void main(String[] args) throws Exception {
        JCruxClient client = new JCruxClient();
        client.connect("localhost", "demo", "demo-token");
        String id = client.create("shared.Counter", "visits");

        System.out.println("1 count (a private int): " + client.getField(id, "count"));
        client.setField(id, "count", 7);
        System.out.println("2 after setField: " + client.getField(id, "count") + ", describe() says " + client.invoke(id, "describe"));
        client.setField(id, "owner", "sara");
        System.out.println("3 a field of the superclass: " + client.getField(id, "owner"));
        client.setField(id, "owner", null);
        System.out.println("4 null is allowed as a value: " + client.getField(id, "owner"));

        try {
            client.setField(id, "count", "seven");
        } catch (Exception e) {
            System.out.println("5 wrong type -> " + e.getClass().getSimpleName());
        }
        try {
            client.setField(id, "count", null);
        } catch (Exception e) {
            System.out.println("6 null into an int -> " + e.getClass().getSimpleName());
        }
        try {
            client.getField(id, "created");
        } catch (Exception e) {
            System.out.println("7 static field -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            client.getField(id, "nothing");
        } catch (Exception e) {
            System.out.println("8 unknown field -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
    }
}
```

```text
1 count (a private int): 0
2 after setField: 7, describe() says visits=7
3 a field of the superclass: sara
4 null is allowed as a value: null
5 wrong type -> IllegalArgumentException
6 null into an int -> IllegalArgumentException
7 static field -> IllegalArgumentException: Field 'created' on shared.Counter is static — static fields aren't supported by getField/setField, since they aren't captured by per-object serialization or per-object synchronization
8 unknown field -> NoSuchFieldException: nothing not found in shared.Counter or its superclasses
```

- Fields are found by name in the class and its superclasses (case 3). Private fields work (case 1), and primitive fields too: the value is boxed and unboxed on the way.
- `null` is a valid value for a reference field (case 4). It is **not** valid for a primitive (case 6), and a value of the wrong type is refused (case 5).
- Static fields are refused (case 7): they belong to the class, not to a stored object.
- `getField` returns a copy, like `invoke`.

Fields are handy for simple values and for tests. For anything that is read and then changed, use a method (chapter 11).

## 7. Getting, listing and removing

`get(id)` returns a **copy of the whole object** to the client, `list()` describes every stored object and `remove(id)` deletes one:

```java
import org.j2os.platform.jcrux.client.JCruxClient;
import org.j2os.platform.jcrux.share.JCruxObject;
import shared.Counter;

public class Reading {
    public static void main(String[] args) throws Exception {
        JCruxClient client = new JCruxClient();
        client.connect("localhost", "demo", "demo-token");
        String id = client.create("shared.Counter", "visits", 3);

        Counter copy = (Counter) client.get(id);
        copy.add(10);
        System.out.println("1 the copy changed here: " + copy.describe());
        System.out.println("2 the stored object did not: " + client.invoke(id, "describe"));

        JCruxObject info = client.list().get(0);
        System.out.println("3 id matches: " + info.getObjectId().equals(id) + ", type: " + info.getObjectType());
        System.out.println("4 created from: " + info.getCreationIP() + ", last used from: " + info.getLastAccessIP());

        Thread.sleep(50);
        client.invoke(id, "describe");
        JCruxObject later = client.list().get(0);
        System.out.println("5 creation time unchanged: " + (later.getCreationTime() == info.getCreationTime()));
        System.out.println("6 last access moved forward: " + (later.getLastAccessTime() > info.getLastAccessTime()));

        client.remove(id);
        client.remove(id);
        System.out.println("7 removed twice without trouble, objects left: " + client.list().size());
        try {
            client.get(id);
        } catch (Exception e) {
            System.out.println("8 get after remove -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
    }
}
```

```text
1 the copy changed here: visits=13
2 the stored object did not: visits=3
3 id matches: true, type: shared.Counter
4 created from: 127.0.0.1, last used from: 127.0.0.1
5 creation time unchanged: true
6 last access moved forward: true
7 removed twice without trouble, objects left: 0
8 get after remove -> NoSuchElementException: No instance for id <id>
```

- **`get`** needs the object's class on the **client** classpath (the server's own classes are not downloaded). Calling methods on the copy changes only the copy (cases 1 and 2); to change the stored object, use `invoke`.
- **`list`** returns `JCruxObject` entries with `getObjectId()`, `getObjectType()`, `getCreationTime()`, `getCreationIP()`, `getLastAccessIP()` and `getLastAccessTime()`. The order is not defined. Reading and changing an object (`get`, `invoke`, `getField`, `setField`) updates its last access; `list` itself does not. The addresses are those the server saw, or `unknown`.
- **`remove`** is silent when the id does not exist (case 7). Any other call with an unknown id fails with `NoSuchElementException` (case 8).

## 8. Saving and loading

The container lives in the server's memory. `save()` writes all objects to the container file, `load()` replaces everything in memory by the content of the file:

```java
import org.j2os.platform.jcrux.client.JCruxClient;

public class Saving {
    public static void main(String[] args) throws Exception {
        JCruxClient client = new JCruxClient();
        client.connect("localhost", "demo", "demo-token");

        try {
            client.load();
        } catch (Exception e) {
            System.out.println("1 load() before anything was saved -> " + e.getClass().getSimpleName());
        }
        System.out.println("2 never saved: count " + client.saveCount() + ", time " + client.lastSaveTime());

        String id = client.create("shared.Counter", "visits", 3);
        client.save();
        System.out.println("3 after save(): count " + client.saveCount() + ", time set: " + (client.lastSaveTime() > 0));

        client.invoke(id, "add", 4);
        System.out.println("4 changed in memory only: " + client.invoke(id, "describe"));
        client.load();
        System.out.println("5 load() brings back the saved state: " + client.invoke(id, "describe"));

        client.setAutoSave(true);
        client.invoke(id, "add", 1);
        client.invoke(id, "add", 1);
        System.out.println("6 with autosave every change is written: count " + client.saveCount());
        client.remove("no-such-id");
        System.out.println("7 removing nothing writes nothing: count " + client.saveCount());
        client.setAutoSave(false);
    }
}
```

```text
1 load() before anything was saved -> FileNotFoundException
2 never saved: count 0, time 0
3 after save(): count 1, time set: true
4 changed in memory only: visits=7
5 load() brings back the saved state: visits=3
6 with autosave every change is written: count 3
7 removing nothing writes nothing: count 3
```

- **Case 1.** `load()` needs a file. If nothing was ever saved, it is a `FileNotFoundException`.
- **Cases 2 and 3.** `saveCount()` and `lastSaveTime()` tell whether a save really happened.
- **Cases 4 and 5.** `load()` brings back the saved state and discards changes made since.
- **Cases 6 and 7.** With `setAutoSave(true)` every change writes the file: `create`, `put`, `setField`, a `remove` that removed something, and `invoke`, even when the method threw. Reads never write.

Things worth knowing about saving:

- **The whole container is rewritten each time.** Autosave is simple and safe, and fine for hundreds or thousands of small objects; for a very large container, turn it off and call `save()` at chosen moments.
- **The write is atomic**: jcrux writes `<file>.tmp` and then renames it over the container, so a crash never leaves half a file. A filesystem that cannot rename atomically (many network drives) is refused with an `IOException`.
- **A save can be skipped.** If an object changes while the snapshot is being built, the save does nothing, writes a line to the log and returns normally. Compare `saveCount()` before and after when it matters:

```java
long before = client.saveCount();
client.save();
if (client.saveCount() == before) {
    // the container changed while the snapshot was being built, so nothing was written: try again
    client.save();
}
```

- **If autosave fails after a change**, the change stays in memory but the caller gets the exception. For `create` this means the id is lost; `list()` shows it.
- **`load()` while others work** can leave their running calls on objects that no longer exist. Use it only when the container is idle, for example right after starting.

## 9. Adding classes while the server runs

A client may bring a class the server does not know. `addClass(name, sourceCode)` sends Java **source**, the server compiles it in memory and the class is usable at once:

```java
import org.j2os.platform.jcrux.client.JCruxClient;

public class Classes {
    static final String NOTE = """
            package shop;
            public class Note implements java.io.Serializable {
                private String text = "hello";
                public String text() { return text; }
            }
            """;

    public static void main(String[] args) throws Exception {
        JCruxClient client = new JCruxClient();
        client.connect("localhost", "demo", "demo-token");

        client.addClass("shop.Note", NOTE);
        String id = client.create("shop.Note");
        System.out.println("1 a class added at run time works like any other: " + client.invoke(id, "text"));
        System.out.println("2 its field too: " + client.getField(id, "text"));

        try {
            client.addClass("shop.Note", NOTE);
        } catch (Exception e) {
            System.out.println("3 the same name twice -> " + e.getClass().getSimpleName());
        }
        try {
            client.addClass("java.util.ArrayList", "package java.util; public class ArrayList {}");
        } catch (Exception e) {
            System.out.println("4 a name that exists already -> " + e.getClass().getSimpleName());
        }
        try {
            client.addClass("shop.Plain", "package shop; public class Plain {}");
        } catch (Exception e) {
            System.out.println("5 not Serializable -> " + e.getMessage());
        }
        try {
            client.addClass("shop.Bad", "package shop; public class Bad implements java.io.Serializable { int x = ; }");
        } catch (Exception e) {
            System.out.println("6 does not compile -> " + e.getMessage().replace("\n", " | "));
        }
        try {
            client.addClass("shop.Other", "package shop; public class Other implements java.io.Serializable { Note n; }");
        } catch (Exception e) {
            System.out.println("7 uses another added class -> " + e.getClass().getSimpleName());
        }
        try {
            client.get(id);
        } catch (Exception e) {
            System.out.println("8 get() needs the class on the client -> " + e.getClass().getSimpleName() + ", cause " + e.getCause().getClass().getSimpleName());
        }
    }
}
```

```text
1 a class added at run time works like any other: hello
2 its field too: hello
3 the same name twice -> IllegalStateException
4 a name that exists already -> IllegalArgumentException
5 not Serializable -> Class shop.Plain does not implement java.io.Serializable — every class used with JCrux must, since create()'d instances are serialized by save(), get(), invoke(), and getField()
6 does not compile -> Failed to compile class shop.Bad: | ERROR at line 1, column 74: illegal start of expression
7 uses another added class -> IllegalArgumentException
8 get() needs the class on the client -> UnmarshalException, cause ClassNotFoundException
```

- **It needs a JDK.** A server that runs on a plain JRE refuses with an `IllegalStateException`.
- **A name can be added once** (case 3), even after restarts, and **cannot be a name that already exists** on the classpath (case 4). There is no way to replace a class; use a new name.
- **The class must be `Serializable`** (case 5) and **may not use other classes added this way** (case 7). It can use the JDK and the server's own classpath.
- **A compile error** comes back as an `IllegalArgumentException` that lists each problem with its line and column (case 6).
- **A client that calls `get()`** needs the class too, and a class that came from `addClass` is not on the client (case 8). `invoke` and `getField` work, since the class is only needed on the server.

The source of every added class is saved next to the container (`<file>.classes`) and compiled again when the server starts, so added classes survive a restart together with their objects:

```java
import org.j2os.platform.jcrux.client.JCruxClient;
import org.j2os.platform.jcrux.server.JCruxServer;

import java.io.File;
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
import java.util.Arrays;

public class Restart {
    public static void main(String[] args) throws Exception {
        LocateRegistry.createRegistry(1099);

        JCruxServer first = new JCruxServer("t", "work/cart.dat", true);
        Naming.rebind("//localhost/cart", first);

        JCruxClient client = new JCruxClient();
        client.connect("localhost", "cart", "t");
        String counter = client.create("shared.Counter", "orders", 10);
        client.invoke(counter, "add", 5);
        client.addClass("shop.Note", """
                package shop;
                public class Note implements java.io.Serializable {
                    public String text() { return "added at run time"; }
                }
                """);
        String note = client.create("shop.Note");

        Naming.unbind("//localhost/cart");
        first.close();
        String[] files = new File("work").list();
        Arrays.sort(files);
        System.out.println("1 files on disk: " + Arrays.toString(files));

        JCruxServer second = new JCruxServer("t", "work/cart.dat", true);
        Naming.rebind("//localhost/cart", second);
        client.connect("localhost", "cart", "t");
        System.out.println("2 objects after the restart: " + client.list().size());
        System.out.println("3 the counter kept its state: " + client.invoke(counter, "describe"));
        System.out.println("4 the added class came back too: " + client.invoke(note, "text"));
        System.out.println("5 two invoke() calls, two writes: " + client.saveCount());

        Naming.unbind("//localhost/cart");
        second.close();
        System.exit(0);
    }
}
```

```text
1 files on disk: [cart.dat, cart.dat.classes, cart.dat.lock]
2 objects after the restart: 2
3 the counter kept its state: orders=15
4 the added class came back too: added at run time
5 two invoke() calls, two writes: 2
```

The container and the sources are two files, and both must be kept together (case 1 shows them with the lock file). Case 5 counts two writes for two calls; the creation of the objects wrote earlier.

## 10. Looking at the server

A few calls report on the server itself, for a health page or a log line:

```java
import org.j2os.platform.jcrux.client.JCruxClient;

public class Monitor {
    public static void main(String[] args) throws Exception {
        JCruxClient client = new JCruxClient();
        client.connect("localhost", "demo", "demo-token");

        long free = client.freeMemory(), total = client.totalMemory();
        System.out.println("1 memory: 0 < free <= total is " + (free > 0 && free <= total));
        System.out.println("2 uptime in milliseconds, small right after start: " + (client.uptime() < 60_000));
        System.out.println("3 JVM threads of the server: more than zero is " + (client.jvmThreadCount() > 0));
        System.out.println("4 container file: " + client.containerFilePath());
        System.out.println("5 saves so far: " + client.saveCount());
    }
}
```

```text
1 memory: 0 < free <= total is true
2 uptime in milliseconds, small right after start: true
3 JVM threads of the server: more than zero is true
4 container file: demo.dat
5 saves so far: 0
```

| Call | Returns |
|------|---------|
| `freeMemory()`, `totalMemory()` | The server JVM's memory, in bytes |
| `uptime()` | Milliseconds since this server object was created |
| `jvmThreadCount()` | Live threads in the server JVM |
| `containerFilePath()` | The container file as it was given to the constructor |
| `saveCount()`, `lastSaveTime()` | Saves since this server started, and the time of the last one in milliseconds since 1970 (`0` if none) |

## 11. Several clients and threads

Many clients, and many threads in each, can use the container at the same time. The rules are simple:

- **One object at a time.** The server holds the object's own monitor during `invoke`, `getField` and `setField`, so two calls on the same object never overlap. A method like `add` that changes a number is therefore safe without any `synchronized`.
- **Different objects in parallel.** Calls on different objects do not wait for each other.
- **`save`, `load` and `list` share one lock** and wait for one another.

```java
import org.j2os.platform.jcrux.client.JCruxClient;

public class Sharing {
    static JCruxClient connect() throws Exception {
        JCruxClient client = new JCruxClient();
        client.connect("localhost", "demo", "demo-token");
        return client;
    }

    public static void main(String[] args) throws Exception {
        String id = connect().create("shared.Counter", "hits", 0);

        Thread[] workers = new Thread[4];
        for (int i = 0; i < workers.length; i++) {
            workers[i] = new Thread(() -> {
                try {
                    JCruxClient mine = connect();          // every thread has its own client
                    for (int n = 0; n < 50; n++) mine.invoke(id, "add", 1);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            });
            workers[i].start();
        }
        for (Thread t : workers) t.join();

        System.out.println("1 4 threads x 50 calls: " + connect().invoke(id, "describe"));
        System.out.println("2 another client sees the same object: " + connect().list().size() + " object");
    }
}
```

```text
1 4 threads x 50 calls: hits=200
2 another client sees the same object: 1 object
```

Four threads, each with its own client, made fifty calls each; the total is exact. The guarantee covers **one call**. Two calls in a row are not atomic:

```java
// not safe: another client may change "count" between the two calls
int now = (Integer) client.getField(id, "count");
client.setField(id, "count", now + 1);

// safe: the whole change happens inside one call
client.invoke(id, "add", 1);
```

Whenever a value is read and then changed, put both steps in one method of the object.

## 12. Security

jcrux gives clients a lot of power: `create`, `invoke`, `setField` and `addClass` run code and change state on the server. **Anyone who holds the token can run arbitrary Java on the server's machine.** Treat the token like a shell password, and treat the server's port like a shell. Concretely:

- **Do not expose the registry or the server to an untrusted network.** Keep it on a private network or behind a firewall.
- **There is no encryption.** RMI is used without TLS here, so the token travels in clear text. Use a private network or a tunnel (a VPN or SSH tunnel).
- **Keep the token out of the source.** Read it from the environment, as in the holder of chapter 3.
- **Limit what can arrive.** The fourth constructor argument is a *serial filter*: a list of class patterns, separated by `;`, in the syntax of Java's `ObjectInputFilter`. Only matching classes may arrive as arguments of remote calls; `!*` at the end refuses everything else:

```java
// only the JDK basics and your own model package may arrive as arguments; everything else is refused
String filter = "java.lang.*;java.util.*;shop.*;!*";
JCruxServer container = new JCruxServer(token, "data/shop.dat", true, filter);
```

```java
import org.j2os.platform.jcrux.client.JCruxClient;
import org.j2os.platform.jcrux.server.JCruxServer;

import java.io.File;
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;

public class Security {
    public static void main(String[] args) throws Exception {
        JCruxServer container = new JCruxServer("s3cret", "work/safe.dat", false,
                "java.lang.*;java.util.*;shared.*;!*");
        LocateRegistry.createRegistry(1099);
        Naming.rebind("//localhost/safe", container);

        JCruxClient client = new JCruxClient();
        client.connect("localhost", "safe", "s3cret");
        String id = client.create("shared.Counter", "ok", 1);
        System.out.println("1 allowed types pass: " + client.invoke(id, "add", 2));

        try {
            client.invoke(id, "add", new File("/etc/passwd"));
        } catch (Exception e) {
            System.out.println("2 a type outside the list -> " + e.getClass().getSimpleName() + ", cause " + e.getCause().getClass().getSimpleName());
        }

        JCruxClient stranger = new JCruxClient();
        stranger.connect("localhost", "safe", "wrong");
        try {
            stranger.invoke(id, "add", new File("/etc/passwd"));
        } catch (Exception e) {
            System.out.println("3 the filter runs before the token is checked -> " + e.getClass().getSimpleName());
        }
        try {
            stranger.list();
        } catch (Exception e) {
            System.out.println("4 wrong token -> " + e.getMessage());
        }
        System.exit(0);
    }
}
```

```text
1 allowed types pass: 3
2 a type outside the list -> ServerException, cause UnmarshalException
3 the filter runs before the token is checked -> ServerException
4 wrong token -> Invalid token
```

- The filter is applied to the **arguments of remote calls, before the token is checked** (case 3), so a stranger cannot even make the server read a dangerous class.
- It does **not** apply to the container file, to `load()` or to objects the server returns.
- A blank or invalid filter is refused when the server starts.
- Authentication failures and every `create`, `invoke` and `addClass` are written to the server's log (through `slf4j`), with the class and method names. Keep that log.

## 13. A realistic scenario

Two services share the stock of a shop. The warehouse creates the counter, the shop sells from it, and every change is written to the file:

```java
import org.j2os.platform.jcrux.client.JCruxClient;

public class Workflow {
    static JCruxClient connect() throws Exception {
        JCruxClient client = new JCruxClient();
        client.connect("localhost", "demo", "demo-token");
        return client;
    }

    public static void main(String[] args) throws Exception {
        JCruxClient warehouse = connect();   // the service that owns the stock
        JCruxClient shop = connect();        // a second service that sells it

        warehouse.setAutoSave(true);
        String stock = warehouse.create("shared.Counter", "pens", 20);
        System.out.println("1 warehouse stores " + warehouse.invoke(stock, "describe"));

        for (int i = 0; i < 3; i++) shop.invoke(stock, "add", -2);
        System.out.println("2 shop sold 3 orders of 2: " + shop.invoke(stock, "describe"));
        System.out.println("3 warehouse sees it at once: " + warehouse.getField(stock, "count"));

        try {
            shop.invoke(stock, "fail");
        } catch (Exception e) {
            System.out.println("4 a method that throws -> " + e.getClass().getSimpleName());
        }
        System.out.println("5 its partial change was kept: " + shop.invoke(stock, "describe"));
        System.out.println("6 every change was written: " + (warehouse.saveCount() >= 5));
        System.out.println("7 last access recorded: " + warehouse.list().get(0).getLastAccessIP());
    }
}
```

```text
1 warehouse stores pens=20
2 shop sold 3 orders of 2: pens=14
3 warehouse sees it at once: 14
4 a method that throws -> InvocationTargetException
5 its partial change was kept: pens=114
6 every change was written: true
7 last access recorded: 127.0.0.1
```

- **Cases 2 and 3.** Both services work on the same object; the warehouse sees every sale at once.
- **Cases 4 and 5.** `fail()` adds 100 and then throws, standing for any method that changes something before it fails. The change is kept: `pens=114`. That is the reason to check first and change afterwards.
- **Case 6.** With autosave on, every call that changed something has been written.
- **Case 7.** `list()` knows who touched the object last.

## Appendix A. Quick reference

### A.1 JCruxServer

| Call | Purpose |
|------|---------|
| `new JCruxServer(token, file, autosave)` | A container. Locks the file, restores objects and added classes |
| `new JCruxServer(token, file, autosave, serialFilter)` | The same, with a filter for incoming arguments |
| `close()` | Unexports the server and releases the lock. Does not save. Safe to repeat |

The server is published with `Naming.rebind("//host/name", server)` after a registry exists.

### A.2 JCruxClient

| Call | Purpose |
|------|---------|
| `connect(ip, service, token)` | Looks the server up. The token is checked later |
| `create(class, args...)` | Builds an object in the server. Returns its id |
| `put(object)` | Stores an object made by the client. Returns its id |
| `get(id)` | A copy of the object. Needs the class on the client |
| `invoke(id, method, args...)` | Calls a method. Returns a copy of the result |
| `getField(id, name)`, `setField(id, name, value)` | Reads or writes a field |
| `remove(id)` | Deletes an object. Silent if missing |
| `list()` | `JCruxObject` entries for every stored object |
| `save()`, `load()` | Write all objects to the file, or replace them from it |
| `setAutoSave(boolean)` | Write the file after every change |
| `addClass(name, source)` | Compiles a class in the server |
| `freeMemory()`, `totalMemory()`, `uptime()`, `jvmThreadCount()` | Server health |
| `saveCount()`, `lastSaveTime()`, `containerFilePath()` | Save history and file |

### A.3 JCruxObject

| Call | Meaning |
|------|---------|
| `getObjectId()`, `getObjectType()` | Id and class name |
| `getCreationTime()`, `getCreationIP()` | When and from where it was created |
| `getLastAccessTime()`, `getLastAccessIP()` | When and from where it was last used |

### A.4 Files next to a container

| File | Content |
|------|---------|
| `<file>` | The objects |
| `<file>.lock` | The lock that keeps a second server out |
| `<file>.classes` | The sources of the added classes |
| `<file>.tmp`, `<file>.classes.tmp` | Short-lived while writing |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| `null` token in the constructor | `NullPointerException` |
| A second server on the same file | `RemoteException` "Another JCruxServer ... already holds the lock" |
| The container folder does not exist | `RemoteException` at the constructor |
| The file is not a container | `RemoteException` "Failed to auto-restore ..." with the cause |
| A blank or invalid serial filter | `IllegalArgumentException` |
| An unknown name | `NotBoundException` at `connect` |
| No registry at that host and port | `ConnectException` at `connect` |
| A wrong token | `Exception` "Invalid token" at the first call |
| An argument that is `null` | `IllegalArgumentException` |
| No matching constructor or method | `NoSuchMethodException` |
| An unknown class | `ClassNotFoundException` |
| A class that does not serialize | `IllegalArgumentException` |
| An unknown id | `NoSuchElementException` (except `remove`) |
| A static method or field | `IllegalArgumentException` |
| An unknown field | `NoSuchFieldException` |
| A wrong value type, or `null` for a primitive | `IllegalArgumentException` |
| The invoked method throws | `InvocationTargetException`; changes made before remain |
| An argument class refused by the filter | `ServerException` with the cause |
| `load()` without a file | `FileNotFoundException` |
| `save()` while the container changes | Skipped without error; see `saveCount()` |
| Autosave fails after a change | The caller gets the exception; the change stays |
| `addClass` with a name already added | `IllegalStateException` |
| `addClass` with an existing name, no `Serializable`, other added classes or a compile error | `IllegalArgumentException` |
| `get` of an object whose class the client lacks | `UnmarshalException` with `ClassNotFoundException` |
| A closed server used for `save`, `load` or `addClass` | `IllegalStateException` |
| The server is restarted | Calls fail with `ConnectException`; connect again |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Exposing the port.** The token gives code execution on the server. Keep it on a private network.
- **A token in the source or the log.** Read it from the environment.
- **Expecting `connect()` to check the token.** It is checked at the first call.
- **A `null` argument.** `create` and `invoke` cannot infer the type. Pass a typed value or use a different method.
- **Parameters declared as interfaces.** `List`, `Map` or `Number` parameters do not match an `ArrayList`, `HashMap` or `Integer` argument. Declare concrete types in methods meant for remote use.
- **A `long` parameter and an `int` argument.** The types must match exactly (apart from boxing).
- **Fields that cannot serialize.** A lock, a thread, a connection or a logger as a non-transient field makes the object refuse to be created.
- **Static state.** Static fields are not stored and not reachable.
- **Read, then write.** `getField` followed by `setField` is not atomic. Use a method.
- **Methods that fail halfway.** Changes made before the exception remain.
- **Changing the copy.** The result of `get`, `invoke` and `getField` is a copy.
- **Two servers for one file.** The second one is refused; do not delete the `.lock` file to get around it.
- **Moving only one file.** Keep the container and its `.classes` file together.
- **Replacing a class.** Not possible with `addClass`; choose a new name.
- **Forgetting `save()`.** Without autosave, a restart loses everything since the last save.
- **Assuming `close()` saves.** It does not.
- **Skipped saves.** Under heavy writing a `save()` may return without writing; compare `saveCount()`.
- **Unbounded growth.** The container is kept in memory and rewritten as a whole.

### C.2 Behavior worth knowing

- Objects get a random UUID as id, so two objects never get the same one in practice.
- Constructors and methods are found by exact runtime types, with boxing as the only flexibility.
- Calls on the same object are serialized by the object's monitor; different objects run in parallel.
- Results, fields and `get()` are deep copies made by serialization.
- `list()` returns copies of the information, in no fixed order.
- The file format starts with the marker `JCRX` and a version number; a file with another marker or version is refused.
- A restart restores objects and recompiles added classes before the first call is accepted.
- Added classes are compiled in memory and kept as source; they need a JDK on the server.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Containers per file | One server at a time |
| Container size | Held in memory and rewritten at every save; keep it small (the bundled example advises staying under about 1 GB) |
| Arguments of `create` and `invoke` | Never `null` |
| Argument matching | Exact runtime type, plus the primitive equivalent |
| Static members | Not supported |
| Added classes | Once per name, never replaced; serializable; no references to one another |
| Encryption | None |
| Authentication | One shared token |

### C.4 Tuning and practical advice

- Prefer **a few larger methods** over many field reads and writes: each call is a network round trip.
- Use autosave for small containers and a timer calling `save()` for large ones.
- Keep stored classes small and simple. Put big data in files and keep only the path in the object.
- Reuse one client per JVM and rebuild it after a `ConnectException`.
- Start the registry and the server together in one program, and register a shutdown hook that saves and closes.
- Put the container on a local disk, not on a network share.
- Limit access to the port by firewall, and use a serial filter listing only the packages of your model.
- Take copies of the container file together with the `.classes` file for backup, while no save is running.

## Appendix D. Message reference

### D.1 Server and container

| Message | Exception | Cause |
|---------|-----------|-------|
| `Another JCruxServer holds the lock on <file> — only one JCruxServer may run against a given container file at a time` | `RemoteException` | A second server on the same file |
| `Another JCruxServer in this same JVM already holds the lock on <file>` | `RemoteException` | The same, inside one JVM |
| `Failed to resolve the canonical path for <file>` | `RemoteException` | The path cannot be resolved |
| `Failed to acquire the lock file for <file>` | `RemoteException` | The lock file cannot be created, for example a missing folder |
| `Failed to auto-restore container from <file>` | `RemoteException` | The file exists but cannot be read |
| `Not a JCrux container file (bad magic number) at <file>` | `IOException` (the cause of the above) | The file is not a container |
| `Unsupported JCrux container file version <n> at <file> (this server reads version 1)` | `IOException` (the cause) | A file from another version |
| `Failed to restore dynamically-added classes for <file>` | `RemoteException` | The `.classes` file cannot be used |
| `Not a JCrux class-sources file (bad magic number) at <file>` | `IOException` (the cause) | The `.classes` file is not valid |
| `serialFilter must not be blank` | `IllegalArgumentException` | A blank filter |
| `serialFilter is not a valid pattern: ...` | `IllegalArgumentException` | A filter with bad syntax |
| `This JCruxServer has been closed` | `IllegalStateException` | `save`, `load` or `addClass` after `close()` |

### D.2 Calls

| Message | Exception | Cause |
|---------|-----------|-------|
| `Invalid token` | `Exception` | A wrong or missing token |
| `Parameter at index <n> is null — JCrux can't infer its type from a null value` | `IllegalArgumentException` | A `null` argument |
| `<class>.<init>(<types>)` | `NoSuchMethodException` | No constructor for those argument types |
| `<method> not found in <class>, its superclasses, or any interface it implements` | `NoSuchMethodException` | No method for that name and types |
| `Object of type <class> is not actually serializable — ...` | `IllegalArgumentException` | A field value that cannot serialize |
| `Method '<m>' on <class> is static — ...` | `IllegalArgumentException` | A static method |
| `Field '<f>' on <class> is static — static fields aren't supported by getField/setField, ...` | `IllegalArgumentException` | A static field |
| `<field> not found in <class> or its superclasses` | `NoSuchFieldException` | An unknown field |
| `No instance for id <id>` | `NoSuchElementException` | An unknown id |
| `No instance for id <id> (removed concurrently)` | `NoSuchElementException` | Removed by another client during the call |

### D.3 Saving and classes

| Message | Exception | Cause |
|---------|-----------|-------|
| `This filesystem doesn't support atomically replacing an existing file (needed to save <file> safely) — use a local filesystem ...` | `IOException` | The rename is not atomic |
| `Runtime class compilation requires running this server on a JDK, not a JRE` | `IllegalStateException` | `addClass` on a JRE |
| `Class <name> was already added via addClass; replacing an already-loaded dynamic class isn't supported` | `IllegalStateException` | The name was added before |
| `Class <name> does not implement java.io.Serializable — ...` | `IllegalArgumentException` | A class that is not serializable |
| `Failed to compile class <name>:` followed by `ERROR at line <n>, column <m>: <text>` | `IllegalArgumentException` | A compile error |

### D.4 Server log

| Level | Line |
|-------|------|
| INFO | `create requested for class '<class>'`, `created instance of '<class>' with id <id>` |
| INFO | `invoke requested — id=<id>, class='<class>', method='<method>'` |
| INFO | `addClass requested for '<name>'`, `'<name>' compiled, loaded, and persisted` |
| INFO | `restored <n> object(s) from <file>`, `recompiled <n> dynamically-added class(es) from <file>` |
| INFO | `skipping save — the container changed while this snapshot was being built` |
| WARN | `authentication failed (invalid or missing token)` |
| WARN | `entry '<id>' in <file> has no metadata (old-format or corrupted file?); it will be restored but won't appear in list()` |
