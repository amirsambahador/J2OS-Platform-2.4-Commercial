# jpool

*J2OS Platform 2.4*

Some objects are expensive to create. A report engine that loads fonts and compiles templates, a connection to another system, a parser that builds large tables when it starts: each of them can take a second or more to prepare, and a busy application needs many of them at once. Creating a new one for every request wastes time, and creating them without limit can exhaust memory or overload the system behind them.

The usual answer is an *object pool*: a fixed number of ready objects that callers *borrow* and give back. A caller that arrives when all objects are in use waits for one to be returned. Nothing is created more often than needed, and nothing is created beyond the limit.

**jpool** (`org.j2os.platform.jpool`) is this pool. It works inside one program, and it can also share a pool with other programs over the network through *RMI* (Remote Method Invocation, the mechanism of Java for calling an object that lives in another program). It needs nothing but the JDK.

```java
JPool jpool = new JPool();
jpool.pool("engines", InvoiceEngine::new, 5, true);     // declare: at most 5 engines

try (InvoiceEngine engine = jpool.lookup("engines")) {  // borrow one
    engine.render(invoiceNumber);
}                                                       // closing gives it back
```

The module has these types:

- `JPool` is the pool manager. You declare pools, borrow instances and read statistics through it.
- `Poolable` is the base class of every object that a pool manages.
- `PoolableRemote` is the base interface of objects that other programs may borrow.
- `PoolConfig` and `PoolStatistics` describe the settings and the state of a pool.
- `RemoteAddress` is the address of a server that offers pools.
- Three exceptions report problems: `PoolExhaustedException`, `PoolNotDeclaredException` and `PoolAccessDeniedException`.

The guide runs in this order: each chapter has a short program and the output it prints. The parts that only make sense inside a web application are shown as Spring snippets. The appendices are meant for quick lookups.

## Contents

1. [Borrowing and giving back](#1-borrowing-and-giving-back)
2. [Declaring a pool](#2-declaring-a-pool)
3. [When no instance is free](#3-when-no-instance-is-free)
4. [Cleaning an instance: reset](#4-cleaning-an-instance-reset)
5. [Watching a pool](#5-watching-a-pool)
6. [Shutting a pool down](#6-shutting-a-pool-down)
7. [Mistakes the pool reports](#7-mistakes-the-pool-reports)
8. [Sharing a pool with other programs](#8-sharing-a-pool-with-other-programs)
9. [Several servers and automatic failover](#9-several-servers-and-automatic-failover)
10. [When a remote lookup fails](#10-when-a-remote-lookup-fails)
11. [From idea to application: an invoice engine in a web application](#11-from-idea-to-application-an-invoice-engine-in-a-web-application)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The classes of `jpool` need only the JDK (Java 17 or newer is enough for the module itself). The programs of the guide are complete: each one has a single `main` method and the small classes it needs. Programs that use RMI end with `System.exit(0)`, because the RMI runtime keeps a program alive until it is told to stop.

---

## 1. Borrowing and giving back

A pooled class extends `Poolable` and implements one method, `reset()`. The pool calls it every time an instance is given back, so that the next borrower finds a clean object. The first program uses a `Printer` that remembers the last text it was given.

```java
import org.j2os.platform.jpool.JPool;
import org.j2os.platform.jpool.Poolable;

public class FirstPool {

    static class Printer extends Poolable {
        private static int created = 0;
        final int id = ++created;
        String text;

        @Override
        protected void reset() {
            text = null;
        }
    }

    public static void main(String[] args) {
        JPool jpool = new JPool();
        jpool.pool("printers", Printer::new, 2, false);

        try (Printer printer = jpool.lookup("printers")) {
            printer.text = "Invoice 17";
            System.out.println("printer " + printer.id + " prints: " + printer.text);
        }

        try (Printer printer = jpool.lookup("printers")) {
            System.out.println("printer " + printer.id + " again, text = " + printer.text);
        }
    }
}
```

```text
printer 1 prints: Invoice 17
printer 1 again, text = null
```

- `jpool.pool("printers", Printer::new, 2, false)` *declares* a pool called `printers`. It holds at most 2 instances, which the second argument creates through the constructor reference `Printer::new`. The last argument, `false`, means that instances are created only when needed.
- `jpool.lookup("printers")` *borrows* an instance. The type of the variable decides the type that you get back.
- `close()` gives the instance back. `Poolable` implements `AutoCloseable`, so a `try` block with a resource gives it back even when an exception is thrown. Always borrow this way.
- The second borrow receives **the same printer** (number 1), and `reset()` has cleared its text. This is the point of a pool: the object is reused, not created again.

The instance is a normal object of your own class. Its methods are called directly, with no wrapper.

## 2. Declaring a pool

A pool is declared once, at startup, with `pool(...)`. There are two forms:

```java
// short form: queue size = maxInstance, wait at most 30 seconds
jpool.pool(name, factory, maxInstance, eagerInitialization);

// long form
jpool.pool(name, factory, maxInstance, maxThreadQueueForGetInstance,
        maxThreadTimeoutForGetInstance, eagerInitialization);
```

| Parameter | Meaning | Default of the short form |
|-----------|---------|---------------------------|
| `poolName` | The name used by `lookup`. It must be unique in the `JPool` | none |
| `instanceFactory` | A `Supplier` that creates one instance | none |
| `maxInstance` | The most instances that exist at the same time, borrowed and idle together | none |
| `maxThreadQueueForGetInstance` | How many threads may wait at the same time for a free instance | `maxInstance` |
| `maxThreadTimeoutForGetInstance` | How long a waiting thread waits, in milliseconds | 30000 |
| `eagerInitialization` | `true` creates all instances now, `false` creates them when needed | none |

A class that needs arguments, for example a shared dependency, is created by a lambda instead of a constructor reference:

```java
TemplateStore store = new TemplateStore();

// a class that needs arguments is created by a lambda instead of a constructor reference
jpool.pool("engines", () -> new InvoiceEngine(store), 5, true);
```

Eager and lazy creation differ in *when* the work happens:

```java
import org.j2os.platform.jpool.*;

public class PoolSetup {

    static class Connection extends Poolable {
        static int created = 0;
        Connection() {
            System.out.println("  creating connection " + (++created));
        }

        @Override
        protected void reset() {
        }
    }

    public static void main(String[] args) {
        JPool jpool = new JPool();

        System.out.println("declaring the eager pool:");
        jpool.pool("eager", Connection::new, 3, true);
        System.out.println("declaring the lazy pool:");
        jpool.pool("lazy", Connection::new, 3, false);

        PoolStatistics before = jpool.getStatistics("lazy");
        System.out.println("lazy before use: created=" + before.getTotalCreatedCount()
                + " idle=" + before.getIdleCount() + " active=" + before.getActiveCount());

        System.out.println("first lookup in the lazy pool:");
        Connection connection = jpool.lookup("lazy");
        PoolStatistics during = jpool.getStatistics("lazy");
        System.out.println("lazy while borrowed: created=" + during.getTotalCreatedCount()
                + " idle=" + during.getIdleCount() + " active=" + during.getActiveCount());
        connection.close();
        PoolStatistics after = jpool.getStatistics("lazy");
        System.out.println("lazy after close: created=" + after.getTotalCreatedCount()
                + " idle=" + after.getIdleCount() + " active=" + after.getActiveCount());

        PoolConfig config = jpool.getPoolConfig("lazy");
        System.out.println("max=" + config.getMaxInstance() + " queue=" + config.getMaxThreadQueueForGetInstance()
                + " timeout=" + config.getMaxThreadTimeoutForGetInstance() + " ms eager=" + config.isEagerInitialization());
        System.out.println("pools: " + jpool.listPools().stream().sorted().toList());
    }
}
```

```text
declaring the eager pool:
  creating connection 1
  creating connection 2
  creating connection 3
declaring the lazy pool:
lazy before use: created=0 idle=0 active=0
first lookup in the lazy pool:
  creating connection 4
lazy while borrowed: created=1 idle=0 active=1
lazy after close: created=1 idle=1 active=0
max=3 queue=3 timeout=30000 ms eager=false
pools: [eager, lazy]
```

- The eager pool creates its three instances while it is being declared. The program is slower to start, but every borrower finds a ready instance. Use it when creation is slow and the instances will certainly be needed.
- The lazy pool created nothing at first. The first `lookup` created one instance, and the later ones create more only when none is idle, up to `maxInstance`. Use it when the instances are expensive to keep, or may never be needed.
- `getStatistics(name)` returns a `PoolStatistics` with three numbers: `getTotalCreatedCount()` (instances created so far), `getIdleCount()` (ready to borrow) and `getActiveCount()` (borrowed now). The active count is the created count minus the idle count. A closed instance moves from active to idle, and it is not destroyed.
- `getPoolConfig(name)` returns the settings of the pool. The queue size is 3, because the short form uses `maxInstance`. The timeout is 30 seconds.
- `listPools()` returns the names of the declared pools.

If the factory throws an exception while an eager pool is declared, the instances that were already created get a best-effort `reset()` (a call whose errors are ignored, so they can release what they hold), the exception reaches the caller and the pool is not declared. In a lazy pool, the same exception comes out of `lookup`, and the capacity that the failed creation reserved is given back.

## 3. When no instance is free

When all instances are borrowed, the next caller *waits*. Three settings control the wait: how many instances exist (`maxInstance`), how many threads may wait (`maxThreadQueueForGetInstance`) and how long each waits (`maxThreadTimeoutForGetInstance`). If the wait is not over in time, or the waiting room is full, `lookup` throws `PoolExhaustedException`.

```java
import org.j2os.platform.jpool.*;
import org.j2os.platform.jpool.exception.PoolExhaustedException;

public class Waiting {

    static class Worker extends Poolable {
        @Override
        protected void reset() {
        }
    }

    public static void main(String[] args) throws Exception {
        JPool jpool = new JPool();
        // one instance, room for 2 waiting threads, wait at most 500 ms
        jpool.pool("slow", Worker::new, 1, 2, 500, true);
        // one instance and no waiting at all
        jpool.pool("strict", Worker::new, 1, 0, 500, true);

        Worker held = jpool.lookup("slow");
        Thread.ofVirtual().start(() -> {
            try { Thread.sleep(200); } catch (InterruptedException e) { }
            held.close();
        });
        long start = System.currentTimeMillis();
        try (Worker second = jpool.lookup("slow")) {
            System.out.println("got an instance after waiting about "
                    + (System.currentTimeMillis() - start) / 100 * 100 + " ms");
        }

        Worker keep = jpool.lookup("slow");
        try {
            jpool.lookup("slow");
        } catch (PoolExhaustedException e) {
            System.out.println("timeout : " + e.getMessage());
        }
        keep.close();

        Worker taken = jpool.lookup("strict");
        try {
            jpool.lookup("strict");
        } catch (PoolExhaustedException e) {
            System.out.println("no queue: " + e.getMessage());
        }
        taken.close();
    }
}
```

```text
got an instance after waiting about 200 ms
timeout : Pool 'slow' did not respond within 500 ms
no queue: Pool 'strict' is full and its wait queue is full
```

- In the first case another thread gives the instance back after 200 milliseconds, and the waiting `lookup` receives it.
- In the second case nobody gives it back. After 500 milliseconds the caller receives `Pool 'slow' did not respond within 500 ms`.
- In the third case the pool has no wait queue at all (queue size 0), so the caller fails at once with `Pool 'strict' is full and its wait queue is full`.

Choose the numbers with the work in mind. The default queue size equals `maxInstance`, which is small for a busy server: with 2 instances, the third waiting thread and every one after it are refused at once. The practical example in chapter 11 sets a larger queue for this reason.

`PoolExhaustedException` is an unchecked exception. In a web application, answer it with status 503 (service unavailable) and ask the caller to try again.

## 4. Cleaning an instance: reset

`reset()` runs every time an instance is closed, before it goes back to the idle queue. It is the place to clear what one borrower leaves behind: collections, temporary values, open cursors.

If `reset()` throws a `RuntimeException` or an `Error`, the pool cannot know in what state the instance was left. It does the careful thing: the instance is **discarded**, not returned, and its place is freed so that a fresh instance can be created. The exception is not hidden: `close()` throws it to the caller.

```java
import org.j2os.platform.jpool.*;

public class ResetFailure {

    static class Session extends Poolable {
        static int created = 0;
        final int id = ++created;

        @Override
        protected void reset() {
            if (id == 1) {
                throw new IllegalStateException("cannot clean session " + id);
            }
        }
    }

    public static void main(String[] args) {
        JPool jpool = new JPool();
        jpool.pool("sessions", Session::new, 1, false);

        try {
            Session session = jpool.lookup("sessions");
            session.close();
        } catch (IllegalStateException e) {
            System.out.println("close threw: " + e.getMessage());
        }

        PoolStatistics stats = jpool.getStatistics("sessions");
        System.out.println("after the failure: created=" + stats.getTotalCreatedCount()
                + " idle=" + stats.getIdleCount() + " active=" + stats.getActiveCount());

        try (Session next = jpool.lookup("sessions")) {
            System.out.println("the next borrower gets session " + next.id);
        }
    }
}
```

```text
close threw: cannot clean session 1
after the failure: created=0 idle=0 active=0
the next borrower gets session 2
```

The broken session is thrown away: after the failure the pool owns 0 instances. The next borrower causes a new one to be created (session 2). The pool heals itself, and your code still learns about the problem.

Two more rules protect the pool:

- `close()` can be called twice in one borrow, and the second call does nothing. It is final, so a subclass cannot change it.
- **Do not keep a reference to an instance after closing it.** The pool hands out the object itself, not a wrapper. After `close()` the same object may already be in the hands of another borrower.

## 5. Watching a pool

Besides the statistics of chapter 2, `listLongRunningBorrows(poolName, thresholdMillis)` finds borrowers that forget to give back. It returns the list of durations, in milliseconds, of the borrows that are older than the threshold and not yet closed. It only observes. It changes nothing.

```java
import org.j2os.platform.jpool.*;

public class Monitoring {

    static class Worker extends Poolable {
        @Override
        protected void reset() {
        }
    }

    public static void main(String[] args) throws Exception {
        JPool jpool = new JPool();
        jpool.pool("workers", Worker::new, 3, false);

        Worker a = jpool.lookup("workers");
        Thread.sleep(300);
        Worker b = jpool.lookup("workers");

        System.out.println("borrowed longer than 200 ms: " + jpool.listLongRunningBorrows("workers", 200).size());
        System.out.println("borrowed longer than 0 ms  : " + jpool.listLongRunningBorrows("workers", 0).size());

        a.close();
        b.close();
        System.out.println("after closing both         : " + jpool.listLongRunningBorrows("workers", 0).size());
    }
}
```

```text
borrowed longer than 200 ms: 1
borrowed longer than 0 ms  : 2
after closing both         : 0
```

One borrow is 300 milliseconds old when the program asks for those older than 200, and both count with a threshold of 0. After the instances are closed the list is empty. A scheduled task that logs a warning when the list is not empty is a cheap way to find leaks, for example a missing `try` block.

## 6. Shutting a pool down

`shutdown(poolName)` stops one pool, and `shutdownAll()` stops all of them. Call it when the application stops.

```java
import org.j2os.platform.jpool.*;

public class Shutdown {

    static class Worker extends Poolable {
        @Override
        protected void reset() {
            System.out.println("  reset of an idle instance");
        }
    }

    public static void main(String[] args) {
        JPool jpool = new JPool();
        jpool.pool("a", Worker::new, 2, true);
        jpool.pool("b", Worker::new, 1, true);

        System.out.println("shutting down a:");
        jpool.shutdown("a");
        try {
            jpool.lookup("a");
        } catch (RuntimeException e) {
            System.out.println(e.getClass().getSimpleName() + ": " + e.getMessage());
        }

        System.out.println("shutting down everything:");
        jpool.shutdownAll();
        System.out.println("pools are still listed: " + jpool.listPools().stream().sorted().toList());
    }
}
```

```text
shutting down a:
  reset of an idle instance
  reset of an idle instance
PoolExhaustedException: Pool 'a' is shutting down
shutting down everything:
  reset of an idle instance
pools are still listed: [a, b]
```

- Idle instances get a best-effort `reset()` and are discarded. This is the last chance for them to release what they hold. Errors in these calls are ignored, so one broken instance does not stop the rest.
- Instances that are still borrowed are not touched. When their borrowers close them, they are reset and discarded instead of returning to the pool.
- A new `lookup` fails with `PoolExhaustedException`: `Pool 'a' is shutting down`. A thread that is waiting at that moment fails in the same way.
- The pool stays in `listPools()` and its name cannot be declared again in the same `JPool`.

## 7. Mistakes the pool reports

```java
import org.j2os.platform.jpool.*;

public class LocalErrors {

    static class Worker extends Poolable {
        @Override
        protected void reset() {
        }
    }

    public static void main(String[] args) {
        JPool jpool = new JPool();
        jpool.pool("workers", Worker::new, 1, false);

        try { jpool.pool("workers", Worker::new, 1, false); }
        catch (RuntimeException e) { System.out.println(e.getClass().getSimpleName() + ": " + e.getMessage()); }
        try { jpool.lookup("missing"); }
        catch (RuntimeException e) { System.out.println(e.getClass().getSimpleName() + ": " + e.getMessage()); }
        try { jpool.getStatistics("missing"); }
        catch (RuntimeException e) { System.out.println(e.getClass().getSimpleName() + ": " + e.getMessage()); }
        try { jpool.pool("empty", () -> null, 1, false); jpool.lookup("empty"); }
        catch (RuntimeException e) { System.out.println(e.getClass().getSimpleName() + ": " + e.getMessage()); }
        try { new RemoteAddress("", 1099); }
        catch (RuntimeException e) { System.out.println(e.getClass().getSimpleName() + ": " + e.getMessage()); }
        try { new RemoteAddress("localhost", 70000); }
        catch (RuntimeException e) { System.out.println(e.getClass().getSimpleName() + ": " + e.getMessage()); }
        try { jpool.lookup("workers", PoolableRemote.class); }
        catch (RuntimeException e) { System.out.println(e.getClass().getSimpleName() + ": " + e.getMessage()); }
    }
}
```

```text
IllegalStateException: Pool 'workers' has already been declared
IllegalArgumentException: Pool 'missing' is not declared
IllegalArgumentException: Pool 'missing' is not declared
IllegalStateException: Instance factory for pool 'empty' returned null
IllegalArgumentException: host must not be null or empty
IllegalArgumentException: port must be between 1 and 65535: 70000
IllegalArgumentException: At least one RemoteAddress must be specified
```

- Declaring a name twice is an `IllegalStateException`.
- Asking for a pool, or for the statistics of a pool, that was never declared is an `IllegalArgumentException`. This is a programming mistake in local code, so it is not a `PoolNotDeclaredException`. That exception belongs to remote lookups (chapter 10).
- A factory must never return `null`. The pool checks and refuses.
- `RemoteAddress` checks its parts at once: the host must not be empty, and the port must be between 1 and 65535.
- A remote lookup needs at least one address.

## 8. Sharing a pool with other programs

Sometimes the instances must live in one program and be used from others. A machine holds the licensed report engine, or the only connection to a legacy system, and other applications must use it. jpool can *serve* a pool over RMI. A client program borrows an instance, calls its methods, and the calls run **on the server**. The client holds only a *stub*, a stand-in object that forwards each call over the network.

Three things are needed.

1. The pooled class implements an interface that extends `PoolableRemote`. Its methods must declare `throws RemoteException`, and their arguments and results must be serializable (RMI sends them over the network).
2. The server calls `bind(host, port, token)` and declares its pools as usual.
3. The client calls `lookup(poolName, interfaceClass, addresses...)`. It needs the same interface and the same token.

```java
import org.j2os.platform.jpool.*;
import java.rmi.RemoteException;

public class RemoteFirst {

    public interface Greeter extends PoolableRemote {
        String greet(String name) throws RemoteException;
    }

    static class GreeterImpl extends Poolable implements Greeter {
        @Override
        protected void reset() {
        }

        @Override
        public String greet(String name) {
            return "Hello, " + name;
        }
    }

    public static void main(String[] args) throws Exception {
        // the server side
        JPool server = new JPool();
        server.bind("127.0.0.1", 21099, "s3cret-token");
        server.pool("greeters", GreeterImpl::new, 2, false);

        // the client side (here in the same program, normally another machine)
        JPool client = new JPool();
        client.setRemoteAccessToken("s3cret-token");

        try (Greeter greeter = client.lookup("greeters", Greeter.class, new RemoteAddress("127.0.0.1", 21099))) {
            System.out.println(greeter.greet("Sara"));
            System.out.println("on the server, active = " + server.getStatistics("greeters").getActiveCount());
        }
        System.out.println("after close, active = " + server.getStatistics("greeters").getActiveCount()
                + ", idle = " + server.getStatistics("greeters").getIdleCount());
        System.exit(0);
    }
}
```

```text
Hello, Sara
on the server, active = 1
after close, active = 0, idle = 1
```

- `bind("127.0.0.1", 21099, "s3cret-token")` starts the service. The host is the address the server listens on, and it is the address that clients are told to use. Give the real address of the machine, not a name that only resolves locally. The port is the RMI registry port. The token is a shared secret: a client that does not present it is refused.
- The client-only `JPool` calls `setRemoteAccessToken(...)`. A `JPool` that has called `bind` uses its own token for lookups as well, so it does not need this call.
- The result of `lookup` is typed as the interface, `Greeter`. Calling `greet` runs on the server. The server sees one active instance while the client holds it.
- `close()` on the stub runs the real `close()` on the server: `reset()` runs there, and the instance goes back to the server's pool. After the close, the server has 0 active and 1 idle instance. No separate release call exists.
- A client must close what it borrows, exactly as in the local case. Use `try` with a resource.

Notes for the server:

- `bind` can be called once for a `JPool`. It also accepts the three timeouts explicitly:

```java
jpool.bind("10.0.0.5", 1099, token);                       // defaults: 3 s, 10 s, 2 minutes

jpool.bind("10.0.0.5", 1099, token,
        5_000,        // connect timeout, milliseconds
        30_000,       // read timeout, milliseconds
        60_000L);     // lease value, milliseconds
```

  The connect and read timeouts apply to the sockets that clients use to talk to the exported instances. A single remote call that takes longer than the read timeout fails on the client with a `RemoteException` (an `UnmarshalException` whose cause is a `SocketTimeoutException`), so raise the timeout if your methods are slow. The lease value is explained in chapter 10.
- Pools may be declared before or after `bind`. The server sizes its internal channel by itself to fit the wait time of every pool.
- The registry listens on the port you gave. The service and each borrowed instance are exported on *other* ports, chosen by the system. If a firewall stands between the machines, it must allow these connections too. This is a general property of RMI.
- `java.rmi.server.hostname` and the lease value are JVM-wide settings. If several server-role `JPool` objects live in one program, they must be bound with the same host and the same lease value. Otherwise `bind` refuses the second one (see Appendix D).
- Do not put the token in source code. Read it from the environment or from a secret store.

One class can offer several interfaces, for example a read-only view and a full view. A client that takes the read-only interface cannot call the other methods, because the compiler does not know them:

```java
public interface PersonFullAccess extends PoolableRemote {
    void doSomething() throws RemoteException;
    void updateSensitiveData(String data) throws RemoteException;
}

public interface PersonReadOnly extends PoolableRemote {
    void doSomething() throws RemoteException;
}

// one class, two views of it
public class Person extends Poolable implements PersonFullAccess, PersonReadOnly {
    // ...
}
```

A pool that is used only inside its own program needs none of this. Its class does not have to implement a remote interface.

## 9. Several servers and automatic failover

A client can give a list of servers. `lookup` tries them in turn. If one cannot be reached, answers too slowly, has no pool of that name, or has the pool but no free instance in time, the next server is tried. The client sees an error only when all of them fail.

```java
JPool jpool = new JPool();
jpool.setRemoteAccessToken(token);   // a client that never calls bind must set the token itself

try (Greeter greeter = jpool.lookup("greeters", Greeter.class,
        new RemoteAddress("10.0.0.5", 1099),
        new RemoteAddress("10.0.0.6", 1099))) {
    greeter.greet("Sara");
}
```

The first server to try rotates from call to call (*round-robin*), so the load is spread over the servers. The next program gives a list in which the first server does not exist:

```java
import org.j2os.platform.jpool.*;
import java.rmi.RemoteException;

public class RemoteFailover {

    public interface Greeter extends PoolableRemote {
        String greet(String name) throws RemoteException;
    }

    static class GreeterImpl extends Poolable implements Greeter {
        @Override
        protected void reset() {
        }

        @Override
        public String greet(String name) {
            return "Hello, " + name;
        }
    }

    public static void main(String[] args) throws Exception {
        JPool server = new JPool();
        server.bind("127.0.0.1", 21098, "s3cret-token");
        server.pool("greeters", GreeterImpl::new, 2, false);

        JPool client = new JPool();
        client.setRemoteAccessToken("s3cret-token");

        RemoteAddress down = new RemoteAddress("127.0.0.1", 21097);   // nothing listens here
        RemoteAddress up = new RemoteAddress("127.0.0.1", 21098);

        for (int i = 1; i <= 3; i++) {
            try (Greeter greeter = client.lookup("greeters", Greeter.class, down, up)) {
                System.out.println("lookup " + i + ": " + greeter.greet("Ali"));
            }
        }

        try {
            client.lookup("greeters", Greeter.class, down);
        } catch (RuntimeException e) {
            System.out.println(e.getClass().getSimpleName() + ": " + e.getMessage());
            System.out.println("cause: " + e.getCause().getClass().getSimpleName());
        }
        System.exit(0);
    }
}
```

```text
lookup 1: Hello, Ali
lookup 2: Hello, Ali
lookup 3: Hello, Ali
PoolExhaustedException: None of the remote servers for pool 'greeters' responded
cause: ConnectException
```

- All three lookups worked, although the first address is dead. The dead server is skipped each time it comes up.
- When every server fails, `lookup` throws `PoolExhaustedException` with the message `None of the remote servers for pool 'greeters' responded`. The exception carries the **last** failure as its cause. Here it is a `ConnectException`, which tells you that nobody is listening.
- All servers of the list must offer a pool with the same name and the same interface. The servers are interchangeable copies of one service.

Failover does not continue in two cases, because trying another server would not help:

- The token is wrong. This is a mistake of the client, so `PoolAccessDeniedException` is thrown at once.
- The instance does not implement the requested interface. This is a mistake of the program, so `IllegalArgumentException` is thrown at once.

## 10. When a remote lookup fails

```java
import org.j2os.platform.jpool.*;
import java.rmi.RemoteException;

public class RemoteErrors {

    public interface Greeter extends PoolableRemote {
        String greet(String name) throws RemoteException;
    }

    public interface Other extends PoolableRemote {
    }

    static class GreeterImpl extends Poolable implements Greeter {
        @Override
        protected void reset() {
        }

        @Override
        public String greet(String name) {
            return "Hello, " + name;
        }
    }

    static class LocalOnly extends Poolable {
        @Override
        protected void reset() {
        }
    }

    public static void main(String[] args) {
        JPool server = new JPool();
        server.bind("127.0.0.1", 21096, "s3cret-token");
        server.pool("greeters", GreeterImpl::new, 1, false);
        server.pool("local", LocalOnly::new, 1, false);
        RemoteAddress address = new RemoteAddress("127.0.0.1", 21096);

        JPool client = new JPool();

        try { client.lookup("greeters", Greeter.class, address); }
        catch (RuntimeException e) { System.out.println("no token      : " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        client.setRemoteAccessToken("wrong-token");
        try { client.lookup("greeters", Greeter.class, address); }
        catch (RuntimeException e) { System.out.println("wrong token   : " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        client.setRemoteAccessToken("s3cret-token");
        try { client.lookup("missing", Greeter.class, address); }
        catch (RuntimeException e) { System.out.println("no such pool  : " + e.getClass().getSimpleName() + ": " + e.getMessage()
                + " / cause " + e.getCause().getClass().getSimpleName()); }

        try { client.lookup("local", Greeter.class, address); }
        catch (RuntimeException e) { System.out.println("not remote    : " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { client.lookup("greeters", Other.class, address); }
        catch (RuntimeException e) { System.out.println("wrong type    : " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { server.bind("127.0.0.1", 21095, "again"); }
        catch (RuntimeException e) { System.out.println("second bind   : " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        System.out.println("server active after all that: " + server.getStatistics("greeters").getActiveCount()
                + " / " + server.getStatistics("local").getActiveCount());
        System.exit(0);
    }
}
```

```text
no token      : PoolAccessDeniedException: Access token is invalid
wrong token   : PoolAccessDeniedException: Access token is invalid
no such pool  : PoolExhaustedException: None of the remote servers for pool 'missing' responded / cause PoolNotDeclaredException
not remote    : IllegalStateException: Pool 'local' does not implement any interface extending PoolableRemote and cannot be accessed remotely
wrong type    : IllegalArgumentException: The instance for pool 'greeters' on server 127.0.0.1:21096 does not implement interface RemoteErrors$Other
second bind   : IllegalStateException: bind has already been called once
server active after all that: 0 / 0
```

| Output line | What happened |
|-------------|---------------|
| `no token` and `wrong token` | The client has no token, or a different one, so the server refused with `PoolAccessDeniedException`. A client that never calls `bind` must call `setRemoteAccessToken`. A server whose token is `null` refuses everybody |
| `no such pool` | The server has no pool of that name. A single server is skipped like an unreachable one, and the failure ends as `PoolExhaustedException` whose cause is `PoolNotDeclaredException` |
| `not remote` | The pool exists, but its class implements no interface that extends `PoolableRemote`, so it cannot be exported. The server gives the instance back at once |
| `wrong type` | The instance does not implement the interface that the client asked for. The client closes the borrowed instance before throwing |
| `second bind` | `bind` is called twice on the same `JPool` |

The last line shows that the server has no active instance after all these failures. Failed lookups return what they borrowed.

**A client that disappears.** A client may crash or lose the network while it holds an instance. The server cannot wait forever, so it uses the *lease* of RMI. A client that holds a stub renews its lease regularly. When the lease is not renewed within the lease value (2 minutes by default), the server treats the instance as closed: `reset()` runs and the instance goes back to the pool. A shorter lease value frees the capacity of lost clients sooner, and costs a little more network traffic. The JDK reads this setting only once, when the first RMI object of the program is exported, so set it through the `bind` call before any other RMI work, or pass `-Djava.rmi.dgc.leaseValue=...` on the command line.

## 11. From idea to application: an invoice engine in a web application

A web application produces invoices as PDF files. Preparing the engine takes a long time, because it loads fonts and compiles the template. A request needs the engine for a very short time. The application therefore keeps a few engines and shares them among requests.

The engine is a `Poolable`. Its `reset()` clears whatever one request put into it:

```java
public class InvoiceEngine extends Poolable {

    private final Map<String, String> variables = new HashMap<>();

    public InvoiceEngine() {
        // slow work done once per instance: load fonts, compile templates
    }

    public byte[] renderPdf(int number) {
        variables.put("number", String.valueOf(number));
        // ... fill the template and produce the file ...
        return new byte[0];
    }

    @Override
    protected void reset() {
        variables.clear();   // the next borrower must not see this request's data
    }
}
```

The pool is declared once, when the application starts, and stopped when it ends. Four engines are created at once, up to 50 requests may wait for a free one, and each waits at most 5 seconds:

```java
@Configuration
public class InvoicePoolConfig {

    @Bean
    public JPool jpool() {
        JPool jpool = new JPool();
        jpool.pool("engines", InvoiceEngine::new, 4, 50, 5_000, true);
        return jpool;
    }

    @PreDestroy
    public void stop() {
        jpool().shutdownAll();
    }
}
```

The controller borrows an engine for the length of the request. When none is free in time, `PoolExhaustedException` becomes the answer 503:

```java
@RestController
public class InvoiceController {

    private final JPool jpool;

    public InvoiceController(JPool jpool) {
        this.jpool = jpool;
    }

    @GetMapping(value = "/invoices/{number}", produces = "application/pdf")
    public byte[] invoice(@PathVariable int number) {
        try (InvoiceEngine engine = jpool.lookup("engines")) {
            return engine.renderPdf(number);
        }
    }

    @ExceptionHandler(PoolExhaustedException.class)
    @ResponseStatus(HttpStatus.SERVICE_UNAVAILABLE)
    public String busy(PoolExhaustedException exception) {
        return "The server is busy, please try again";
    }
}
```

The program below does the same work with six requests at the same time and two engines. Preparing an engine takes 300 milliseconds and rendering takes 100:

```java
import org.j2os.platform.jpool.*;
import java.util.ArrayList;
import java.util.List;

public class Practical {

    static class InvoiceEngine extends Poolable {
        static int created = 0;

        InvoiceEngine() {
            created++;
            sleep(300);                      // loading fonts and templates is slow
        }

        String render(int invoiceNumber) {
            sleep(100);
            return "invoice " + invoiceNumber + " rendered";
        }

        @Override
        protected void reset() {
        }

        static void sleep(long millis) {
            try { Thread.sleep(millis); } catch (InterruptedException e) { }
        }
    }

    public static void main(String[] args) throws Exception {
        JPool jpool = new JPool();
        jpool.pool("engines", InvoiceEngine::new, 2, 10, 5_000, true);

        List<Thread> requests = new ArrayList<>();
        for (int number = 1; number <= 6; number++) {
            int invoiceNumber = number;
            requests.add(Thread.ofPlatform().start(() -> {
                try (InvoiceEngine engine = jpool.lookup("engines")) {
                    engine.render(invoiceNumber);
                }
            }));
        }
        for (Thread request : requests) {
            request.join();
        }

        PoolStatistics stats = jpool.getStatistics("engines");
        System.out.println("requests served : 6");
        System.out.println("engines created : " + stats.getTotalCreatedCount());
        System.out.println("in use now      : " + stats.getActiveCount());
    }
}
```

```text
requests served : 6
engines created : 2
in use now      : 0
```

Six requests are served by only two engines, and nothing is in use at the end. A queue of 10 lets the four extra requests wait for their turn. With the default queue size of 2, two of them would have been refused at once.

---

## Appendix A. Quick reference

### A.1 JPool: pools

| Method | Purpose |
|--------|---------|
| `pool(name, factory, maxInstance, eager)` | Declares a pool. Queue size is `maxInstance`, wait is 30 seconds |
| `pool(name, factory, maxInstance, maxQueue, timeoutMillis, eager)` | Declares a pool with explicit queue size and wait time |
| `lookup(name)` | Borrows a local instance. Waits if none is free |
| `getStatistics(name)` | Returns a `PoolStatistics` snapshot |
| `getPoolConfig(name)` | Returns the `PoolConfig` of the pool |
| `listPools()` | The names of the declared pools |
| `listLongRunningBorrows(name, thresholdMillis)` | Durations in milliseconds of open borrows older than the threshold |
| `shutdown(name)` and `shutdownAll()` | Stops one pool or all pools |

### A.2 JPool: remote

| Method | Purpose |
|--------|---------|
| `bind(host, port, token)` | Starts serving the pools. Timeouts 3 s and 10 s, lease value 120 s |
| `bind(host, port, token, connectTimeoutMillis, readTimeoutMillis, leaseValueMillis)` | Same, with explicit timeouts and lease value |
| `setRemoteAccessToken(token)` | The token a client presents. Set by `bind` automatically |
| `lookup(name, interface, addresses...)` | Borrows a remote instance, trying the servers round-robin |

### A.3 Poolable and PoolableRemote

| Member | Purpose |
|--------|---------|
| `reset()` | Abstract. Clears per-use state. Runs on every `close()` |
| `close()` | Final. Gives the instance back. Idempotent within one borrow. Rethrows a failure of `reset()` |
| `unreferenced()` | Final. Called by RMI when a client vanished. Same as `close()` |
| `PoolableRemote.close()` | On a stub, closes the server-side instance. Throws `RemoteException` |

### A.4 PoolConfig, PoolStatistics, RemoteAddress

| Type | Members |
|------|---------|
| `PoolConfig` | `getPoolName()`, `getMaxInstance()`, `getMaxThreadQueueForGetInstance()`, `getMaxThreadTimeoutForGetInstance()`, `isEagerInitialization()` |
| `PoolStatistics` | `getPoolName()`, `getActiveCount()`, `getIdleCount()`, `getTotalCreatedCount()` |
| `RemoteAddress` | `RemoteAddress(host, port)`, `getHost()`, `getPort()` |

### A.5 Exceptions

| Exception | Meaning |
|-----------|---------|
| `PoolExhaustedException` | No instance within the wait time, the waiting room is full, the pool is shutting down, or no remote server responded |
| `PoolNotDeclaredException` | A server has no pool of the requested name (remote lookups only) |
| `PoolAccessDeniedException` | The token does not match the server's token |

All three are unchecked.

## Appendix B. Failure handling

| Situation | What happens |
|-----------|-------------|
| All instances are borrowed | The caller waits, up to the timeout |
| The waiting room is full | `PoolExhaustedException` at once |
| The wait is too long | `PoolExhaustedException` after the timeout |
| `reset()` throws | The instance is discarded, its capacity is freed, and `close()` rethrows the exception |
| The factory throws while a lazy pool creates an instance | The exception reaches the caller of `lookup`, and the reserved capacity is released |
| The factory throws while an eager pool is declared | Created instances get a best-effort `reset()`, the exception reaches the caller, and the pool is not declared |
| The factory returns `null` | `IllegalStateException` |
| A borrowed instance is closed after `shutdown` | It is reset and discarded |
| A thread waits while the pool shuts down | It fails with `PoolExhaustedException` |
| A remote server cannot be reached | The next server is tried |
| A remote server has no such pool, or no free instance | The next server is tried |
| The token is wrong | `PoolAccessDeniedException` at once, no failover |
| The class does not implement the requested interface | `IllegalArgumentException` at once. The instance is given back |
| All remote servers fail | `PoolExhaustedException` with the last failure as its cause |
| A client vanishes | After the lease value, the server closes the instance for it |
| `bind` fails partway | The registry and service that were started are released, so a second attempt can succeed |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Not closing.** An instance that is never closed stays borrowed forever and slowly empties the pool. Always use `try` with a resource, and watch `listLongRunningBorrows`.
- **Using an instance after `close()`.** The object may belong to someone else already.
- **State that `reset()` forgets.** The next borrower sees it. Write `reset()` first, when you write the class.
- **A queue that is too small.** The default queue size equals `maxInstance`. A busy server needs a larger queue, or it refuses callers while instances are only a moment away from being free.
- **A timeout that is too long.** Waiting threads pile up behind a slow resource. Prefer a short wait and a clear 503.
- **A name that is used twice.** The second `pool` call fails, also after `shutdown`.
- **A remote interface without `throws RemoteException`.** Remote methods must declare it.
- **Slow remote methods.** A call that takes longer than the read timeout of `bind` fails on the client with a `RemoteException`.
- **Different token or interface on the client.** Both must match the server.
- **A firewall that only opens the registry port.** The instances use other ports.
- **Two server-role `JPool` objects with different hosts in one program.** The second `bind` is refused.

### C.2 Behavior worth knowing

- The pooled object is handed out directly, with no wrapper. This is fast, and it is why a reference must not be kept after `close()`.
- `getStatistics` is a snapshot. Under heavy use the numbers describe approximately one moment.
- Idle instances are kept in a queue, and borrowers receive them in turn.
- The time a thread waits is measured with a monotonic clock, so a change of the system time does not shorten or lengthen it.
- The wait time of a remote lookup is the one set on the **server's** pool. A client cannot change it.
- A lookup in a remote pool counts against the pool of the server. The client's own `JPool` keeps no local state for it.
- `shutdown` does not remove a pool from `listPools()`.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Instances per pool | `maxInstance`, idle and borrowed together |
| Waiting threads per pool | `maxThreadQueueForGetInstance` (default `maxInstance`) |
| Wait time | `maxThreadTimeoutForGetInstance` (default 30000 ms) |
| `bind` calls per `JPool` | 1 |
| Default timeouts of `bind` | connect 3000 ms, read 10000 ms, lease 120000 ms |
| Port of `RemoteAddress` | 1 to 65535 |
| Server-role `JPool` objects with different hosts or lease values in one JVM | Not allowed |
| Pooled class used remotely | Must implement an interface that extends `PoolableRemote` |

### C.4 Tuning

- Set `maxInstance` from the resource behind the objects: the licenses you own, the connections the other system accepts, or the memory you can spend. Do not set it from the number of threads.
- Use an eager pool when creation is slow and the objects will be needed, so the first users do not pay for it. Use a lazy pool for rarely used pools.
- Keep the borrow as short as possible. Borrow, work, close. Do not hold an instance while you wait for a user or for another slow call.
- Prefer a short timeout and handle `PoolExhaustedException`: show a message, try again, or use another server.
- Give the client several addresses of equivalent servers, so that one failed machine does not stop the work.
- Declare small, specific remote interfaces. A read-only interface limits what a caller can do.
- Lower the lease value (for example to 60 seconds) when clients may disappear and the capacity matters.
- Call `shutdownAll()` when the application stops.

## Appendix D. Message reference

### D.1 Pool declaration and lookup

| Message | Exception | Cause |
|---------|-----------|-------|
| `Pool '<name>' has already been declared` | `IllegalStateException` | `pool` with a name that exists |
| `Pool '<name>' is not declared` | `IllegalArgumentException` | A local `lookup`, `getStatistics`, `getPoolConfig`, `listLongRunningBorrows` or `shutdown` for an unknown name |
| `Instance factory for pool '<name>' returned null` | `IllegalStateException` | The factory returns `null` |
| `Pool '<name>' is full and its wait queue is full` | `PoolExhaustedException` | No free instance and no room to wait |
| `Pool '<name>' did not respond within <n> ms` | `PoolExhaustedException` | The wait exceeds the timeout |
| `Pool '<name>' is shutting down` | `PoolExhaustedException` | The pool has been shut down |
| `Interrupted while waiting for an instance from pool '<name>'` | `PoolExhaustedException` | The waiting thread is interrupted |
| `Pool '<name>' cannot accept the released instance: queue is full` | `IllegalStateException` | An internal consistency error on release |

### D.2 Remote

| Message | Exception | Cause |
|---------|-----------|-------|
| `bind has already been called once` | `IllegalStateException` | A second `bind` on one `JPool` |
| `Failed to start JPool on <host>:<port>` | `IllegalStateException` | The registry or service could not start, for example the port is taken. The cause is attached |
| `Another JPool in this JVM is already bound with hostname '<host>'. ...` | `IllegalStateException` | A second server-role `JPool` with another host |
| `Another JPool in this JVM has already set leaseValueMillis=<n>. ...` | `IllegalStateException` | A second server-role `JPool` with another lease value |
| `Failed to automatically increase the control channel timeout for pool '<name>'` | `IllegalStateException` | The server cannot resize its internal channel when a pool is declared |
| `At least one RemoteAddress must be specified` | `IllegalArgumentException` | A remote `lookup` without addresses |
| `None of the remote servers for pool '<name>' responded` | `PoolExhaustedException` | Every server failed. The last failure is the cause |
| `Access token is invalid` | `PoolAccessDeniedException` | The token does not match |
| `Pool '<name>' is not declared on this server` | `PoolNotDeclaredException` | The server has no such pool |
| `Pool '<name>' does not implement any interface extending PoolableRemote and cannot be accessed remotely` | `IllegalStateException` | The pooled class is not remote-capable |
| `The instance for pool '<name>' on server <address> does not implement interface <type>` | `IllegalArgumentException` | The wrong interface is requested |
| `JPool service not found on <address>` | `RemoteException` | The address answers, but no `JPool` is bound there. This is treated as a server failure, so the next server is tried |

### D.3 RemoteAddress

| Message | Cause |
|---------|-------|
| `host must not be null or empty` | An empty host |
| `port must be between 1 and 65535: <port>` | A port out of range |
