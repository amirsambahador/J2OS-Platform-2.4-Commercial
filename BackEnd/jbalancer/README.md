# jbalancer

*J2OS Platform 2.4*

A service that matters rarely runs on a single machine. The orders service has three copies, the search service two, and a new copy appears whenever the load grows. A program that calls such a service must decide **which copy** gets each request. If every request goes to the first copy, the other two do nothing and the first one drowns. The simplest fair answer is to take turns: first copy, second copy, third copy, first copy again.

**jbalancer** (`org.j2os.platform.jbalancer`) does exactly that, and very little more. It keeps a **name** for each service together with a list of its addresses, and `getResourceUrl("orders")` returns the next address in turn. The whole library is one registry and four calls. Three properties set it apart:

- **Tiny and predictable.** There is no network, no thread and no timer inside. The rotation is a counter, so what it returns next can always be worked out by hand.
- **Safe for many threads.** Any number of threads can ask for addresses at the same time and each gets a valid one.
- **Meant to be used by other libraries.** `jclient`, the HTTP client of J2OS, takes a service name and asks jbalancer for the address of every request, and `resicord`, the resilience library, turns retries into failover.

The whole library in a handful of lines:

```java
JRoundRobinBalancer balancer = JRoundRobinBalancer.getInstance();

balancer.configurationResource("orders", List.of(            // once, at start-up
        "http://10.0.0.1:8080",
        "http://10.0.0.2:8080"));

String first  = balancer.getResourceUrl("orders");            // http://10.0.0.1:8080
String second = balancer.getResourceUrl("orders");            // http://10.0.0.2:8080
String third  = balancer.getResourceUrl("orders");            // http://10.0.0.1:8080 again
```

The guide moves in this order: a first rotation (chapter 2), names and registration (3), weighting (4), the list that is copied (5), missing names and bad input (6), many threads (7), use with jclient (8), use with resicord (9) and a complete small workflow (10). The appendices are meant for quick lookups.

## Contents

1. [How jbalancer works](#1-how-jbalancer-works)
2. [A first rotation](#2-a-first-rotation)
3. [Names and registration](#3-names-and-registration)
4. [Weighting by repetition](#4-weighting-by-repetition)
5. [The list is copied](#5-the-list-is-copied)
6. [Missing names and bad input](#6-missing-names-and-bad-input)
7. [Many threads](#7-many-threads)
8. [Using it with jclient](#8-using-it-with-jclient)
9. [Using it with resicord](#9-using-it-with-resicord)
10. [A real use](#10-a-real-use)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Every program here runs as it stands: each has one `main` method. They use made-up addresses such as `http://10.0.0.1:8080`; nothing is ever connected to, because jbalancer only hands out text. Replace them by the addresses of real services.

---

## 1. How jbalancer works

Here is the vocabulary in one place:

- **Resource.** The balancer's word for what a program asks for: a service, such as `orders`, together with its list of addresses. In this guide it is simply "a service" or "a name".
- **Address.** A piece of text, usually the base URL of one copy of a service, such as `http://10.0.0.1:8080`.
- **Round robin.** Taking turns in a fixed order, and starting again from the beginning after the last one.
- **Registry.** The single table of names and address lists that the balancer keeps. There is exactly one per JVM, and every part of the program shares it.
- **Registering.** Putting a name and its list of addresses into the registry.

The classes in a program:

| Class | Role |
|-------|------|
| `JRoundRobinBalancer` | The registry. Obtained with `getInstance()`; it has four public calls |
| `ResourceNotFoundException` | A *checked* exception thrown when a name is not registered |
| `JResource` | One list of addresses with its counter. Used inside the registry; rarely needed directly |

Three things to hold on to:

- **There is only one registry per JVM.** It is created when the class is loaded, can neither be replaced nor thrown away, and lives until the program ends. Everything that registers a name in it shares the same table.
- **The balancer never looks at the addresses.** It does not check that they are valid, that they answer, or that they are different. It hands them out, nothing more.
- **Each name has its own turn.** Asking for `orders` does not move the turn of `billing`.

## 2. A first rotation

Registering two services and asking for addresses:

```java
import org.j2os.platform.jbalancer.JRoundRobinBalancer;

import java.util.List;

public class First {
    public static void main(String[] args) throws Exception {
        JRoundRobinBalancer balancer = JRoundRobinBalancer.getInstance();

        balancer.configurationResource("orders", List.of("http://10.0.0.1:8080", "http://10.0.0.2:8080", "http://10.0.0.3:8080"));
        balancer.configurationResource("billing", List.of("http://10.0.1.1:9000", "http://10.0.1.2:9000"));

        for (int i = 1; i <= 4; i++) {
            System.out.println(i + " orders  -> " + balancer.getResourceUrl("orders"));
        }
        System.out.println("5 billing -> " + balancer.getResourceUrl("billing"));
        System.out.println("6 billing -> " + balancer.getResourceUrl("billing"));
        System.out.println("7 orders  -> " + balancer.getResourceUrl("orders") + "  (each name has its own turn)");
    }
}
```

```text
1 orders  -> http://10.0.0.1:8080
2 orders  -> http://10.0.0.2:8080
3 orders  -> http://10.0.0.3:8080
4 orders  -> http://10.0.0.1:8080
5 billing -> http://10.0.1.1:9000
6 billing -> http://10.0.1.2:9000
7 orders  -> http://10.0.0.2:8080  (each name has its own turn)
```

- **Cases 1 to 4.** The three addresses of `orders` come out in the order they were registered, and the fourth call starts again from the first.
- **Cases 5 and 6.** `billing` has its own counter and starts from its first address.
- **Case 7.** Back in `orders`, the turn continues where it stopped, at the second address. The calls for `billing` did not disturb it.

The turn begins at the **first** address of the list. `configurationResource` returns the balancer itself, so calls can be chained.

Registering belongs at program start-up, and the addresses belong in configuration, not in the code:

```java
// ORDERS_URLS=http://10.0.0.1:8080,http://10.0.0.2:8080
String urls = System.getenv("ORDERS_URLS");

JRoundRobinBalancer.getInstance().configurationResource(
        "orders", List.of(urls.split(",")));                  // the addresses come from the environment, not the code
```

## 3. Names and registration

Four things to know about names and about changing the registry:

```java
import org.j2os.platform.jbalancer.JRoundRobinBalancer;

import java.util.List;

public class Names {
    public static void main(String[] args) throws Exception {
        JRoundRobinBalancer balancer = JRoundRobinBalancer.getInstance();
        balancer.configurationResource("Orders", List.of("http://a", "http://b", "http://c"));

        System.out.println("1 same service, any case: " + balancer.getResourceUrl("orders") + " " + balancer.getResourceUrl("ORDERS"));

        balancer.configurationResource("orders", List.of("http://x", "http://y"));
        System.out.println("2 registered again, the turn starts over: " + balancer.getResourceUrl("orders"));

        balancer.removeResource("orders").removeResource("never-existed");
        System.out.println("3 removing is silent, even for a name that was never there");

        JRoundRobinBalancer same = JRoundRobinBalancer.getInstance();
        System.out.println("4 one registry per JVM: " + (same == balancer));
    }
}
```

```text
1 same service, any case: http://a http://b
2 registered again, the turn starts over: http://x
3 removing is silent, even for a name that was never there
4 one registry per JVM: true
```

- **Case 1.** Names are **not case-sensitive**. `Orders`, `orders` and `ORDERS` are one service. (The comparison uses the fixed `ROOT` locale, so a computer set to Turkish behaves like any other.)
- **Case 2.** Registering a name that already exists **replaces** its list, and the turn **starts again** from the first address of the new list.
- **Case 3.** `removeResource` removes a name and is silent when the name is not there. It also returns the balancer, so removals can be chained.
- **Case 4.** There is one registry in the JVM; `getInstance()` always returns the same object.

The registry is kept in memory only. After a restart it is empty, so registering is part of every start-up.

## 4. Weighting by repetition

The balancer has no setting for weights. When one machine is stronger than another, the list says so by **repeating** its address:

```java
balancer.configurationResource("search", List.of(
        "http://big-machine:8080", "http://big-machine:8080",   // twice: twice the share
        "http://small-machine:8080"));
```

```java
import org.j2os.platform.jbalancer.JRoundRobinBalancer;

import java.util.List;

public class Weights {
    public static void main(String[] args) throws Exception {
        JRoundRobinBalancer balancer = JRoundRobinBalancer.getInstance();

        balancer.configurationResource("search", List.of("big", "big", "small"));
        StringBuilder turns = new StringBuilder();
        for (int i = 0; i < 6; i++) turns.append(balancer.getResourceUrl("search")).append(' ');
        System.out.println("1 an address twice gets twice the share: " + turns.toString().trim());

        balancer.configurationResource("odd", List.of("not a url at all", ""));
        System.out.println("2 the text is not checked: '" + balancer.getResourceUrl("odd") + "' then '" + balancer.getResourceUrl("odd") + "'");
    }
}
```

```text
1 an address twice gets twice the share: big big small big big small
2 the text is not checked: 'not a url at all' then ''
```

- **Case 1.** An address that appears twice is handed out twice per round. In a list of three entries, `big` gets two thirds of the requests.
- **Case 2.** The balancer does not look at what it is given: any text works, even an empty string. A typing mistake in an address is found only when somebody tries to call it.

Repeated addresses stay in the order written, so `big, small, big` and `big, big, small` have the same share but a different rhythm. Prefer the first, which spreads the turns.

## 5. The list is copied

When a service is registered, the balancer takes its **own copy** of the list:

```java
import org.j2os.platform.jbalancer.JRoundRobinBalancer;

import java.util.ArrayList;
import java.util.List;

public class Snapshot {
    public static void main(String[] args) throws Exception {
        JRoundRobinBalancer balancer = JRoundRobinBalancer.getInstance();

        List<String> mine = new ArrayList<>(List.of("http://a", "http://b"));
        balancer.configurationResource("shop", mine);

        mine.add("http://c");
        mine.remove("http://a");
        System.out.println("1 my list now has " + mine.size() + " items: " + mine);

        StringBuilder turns = new StringBuilder();
        for (int i = 0; i < 4; i++) turns.append(balancer.getResourceUrl("shop")).append(' ');
        System.out.println("2 the balancer still uses the list it was given: " + turns.toString().trim());

        balancer.configurationResource("shop", mine);
        System.out.println("3 to use the new list, register again: " + balancer.getResourceUrl("shop"));
    }
}
```

```text
1 my list now has 2 items: [http://b, http://c]
2 the balancer still uses the list it was given: http://a http://b http://a http://b
3 to use the new list, register again: http://b
```

- **Cases 1 and 2.** Changing your own list afterwards does nothing to the registry.
- **Case 3.** To make a change count, **register again**. This is also how the list is updated while the program runs: build the new list, register it.
- The balancer's copy cannot be changed by anyone, and there is no call that reads it back or lists the registered names. If your program needs to know the addresses, keep them yourself.
- A list that contains a `null` entry is refused when it is registered, with a `NullPointerException` without a message.

Re-registering replaces the list as a whole at one moment, and a request that arrives at the same moment gets an address from either the old or the new list, never a mixture and never an invalid one.

## 6. Missing names and bad input

```java
import org.j2os.platform.jbalancer.JRoundRobinBalancer;
import org.j2os.platform.jbalancer.exception.ResourceNotFoundException;

import java.util.List;

public class Missing {
    public static void main(String[] args) {
        JRoundRobinBalancer balancer = JRoundRobinBalancer.getInstance();

        try {
            balancer.getResourceUrl("payments");
        } catch (ResourceNotFoundException e) {
            System.out.println("1 unknown name -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            balancer.getResourceUrl(null);
        } catch (Exception e) {
            System.out.println("2 null name -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            balancer.configurationResource("payments", List.of());
        } catch (Exception e) {
            System.out.println("3 no addresses -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            balancer.configurationResource("payments", null);
        } catch (Exception e) {
            System.out.println("4 null list -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
    }
}
```

```text
1 unknown name -> ResourceNotFoundException: Resource PAYMENTS not found, creating a new one using configurationResource(String resourceId, List<String> urls)
2 null name -> NullPointerException: resourceId must not be null
3 no addresses -> IllegalArgumentException: urls must not be null or empty
4 null list -> IllegalArgumentException: urls must not be null or empty
```

- **Case 1.** Asking for a name that is not registered throws `ResourceNotFoundException`. It is a **checked** exception, so the compiler makes the caller decide what to do. The message names the service in **capital letters**, because names are stored that way, and suggests the cure.
- **Case 2.** A `null` name is a `NullPointerException` with a message; it is a programming error and not a missing service.
- **Cases 3 and 4.** An empty or `null` list of addresses is refused at registration, so a registered name always has at least one address.

How to deal with a missing name depends on the program. Most of the time it means the configuration is wrong, and the best reaction is a clear failure:

```java
try {
    String base = balancer.getResourceUrl("payments");
    call(base);
} catch (ResourceNotFoundException notRegistered) {          // checked: the compiler makes you decide
    throw new IllegalStateException("payments is not configured", notRegistered);
}
```

## 7. Many threads

Any number of threads can ask for addresses at the same time:

```java
import org.j2os.platform.jbalancer.JRoundRobinBalancer;

import java.util.List;
import java.util.Map;
import java.util.TreeMap;
import java.util.concurrent.ConcurrentHashMap;

public class Threads {
    public static void main(String[] args) throws Exception {
        JRoundRobinBalancer balancer = JRoundRobinBalancer.getInstance();
        balancer.configurationResource("api", List.of("one", "two", "three"));

        Map<String, Integer> used = new ConcurrentHashMap<>();
        Thread[] workers = new Thread[4];
        for (int i = 0; i < workers.length; i++) {
            workers[i] = new Thread(() -> {
                try {
                    for (int n = 0; n < 1000; n++) {
                        used.merge(balancer.getResourceUrl("api"), 1, Integer::sum);
                    }
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            });
            workers[i].start();
        }
        for (Thread t : workers) t.join();

        System.out.println("1 4000 requests from 4 threads: " + new TreeMap<>(used));
    }
}
```

```text
1 4000 requests from 4 threads: {one=1334, three=1333, two=1333}
```

- **Case 1.** Four threads make 1000 requests each, 4000 in all, over three addresses. The counts, 1334, 1333 and 1333, are exactly what a single thread would have given: every request takes the next turn, and no turn is given twice or lost.
- What the threads cannot be given is a fixed order. Which thread gets which address depends on timing, so strict one-two-three order holds only when the calls come one after another.
- Registering, removing and asking are safe at the same time. What is **not** guaranteed is that two separate calls see the registry unchanged between them: another thread may register or remove a name in between.

## 8. Using it with jclient

`jclient` takes a service name instead of a fixed address and asks jbalancer **for every request**:

```java
JRoundRobinBalancer.getInstance().configurationResource("orders", List.of(
        "http://10.0.0.1:8080",
        "http://10.0.0.2:8080"));                             // base addresses, no path

OrderApi orders = JClient.builder().service("orders").bind(OrderApi.class);
orders.all();                                                 // each call asks the balancer for the next address
```

- The addresses registered for jclient are **base addresses**: no path. The path of each request comes from the client's interface.
- Because the balancer is asked every time, changing the list in the registry changes the destination of the existing client at once.
- A name that is not registered fails when the call is made, not when the client is built (jclient turns `ResourceNotFoundException` into an `IllegalStateException`).
- jclient does not try another address when one fails; the next request simply gets the next turn.

## 9. Using it with resicord

jbalancer hands out turns and knows nothing about failures. `resicord`, the resilience library of J2OS, knows about failures and nothing about addresses. Together they give what each lacks: **failover**.

The key is a fact about resicord's `Try`: when a call fails and a retry is allowed, it runs the **whole work block again**. If that block asks jbalancer for an address, every attempt gets the next address in turn, so a retry after a dead machine goes to another machine:

```java
import org.j2os.platform.jbalancer.JRoundRobinBalancer;
import org.j2os.platform.resicord.Try;

import java.io.IOException;
import java.util.List;

public class Failover {
    public static void main(String[] args) {
        JRoundRobinBalancer balancer = JRoundRobinBalancer.getInstance();
        balancer.configurationResource("orders", List.of("http://10.0.0.1:8080", "http://10.0.0.2:8080", "http://10.0.0.3:8080"));
        balancer.configurationResource("billing", List.of("http://10.0.1.1:9000", "http://10.0.1.2:9000"));
        balancer.configurationResource("search", List.of("http://10.0.2.1:7000", "http://10.0.2.2:7000"));

        for (int i = 1; i <= 3; i++) {                                // the machine 10.0.0.2 is off
            String answer = new Try<String>()
                    .doWork(() -> {
                        String base = balancer.getResourceUrl("orders");
                        if (base.endsWith("0.2:8080")) throw new IOException("connection refused");
                        return "answer from " + base;
                    })
                    .retry(3, 50)
                    .get();
            System.out.println(i + " " + answer);
        }

        String fallback = new Try<String>()                           // every billing machine is off
                .doWork(() -> {
                    String base = balancer.getResourceUrl("billing");
                    throw new IOException("connection refused: " + base);
                })
                .retry(2, 50)
                .onError(error -> "all addresses failed, last error: " + error.getMessage())
                .get();
        System.out.println("4 " + fallback);

        String timed = new Try<String>()                              // the machine 10.0.2.1 is very slow
                .doWork(() -> {
                    String base = balancer.getResourceUrl("search");
                    if (base.endsWith("2.1:7000")) Thread.sleep(2000);
                    return "answer from " + base;
                })
                .retry(2, 50)
                .timeLimit(300)
                .get();
        System.out.println("5 " + timed);
    }
}
```

```text
1 answer from http://10.0.0.1:8080
2 answer from http://10.0.0.3:8080
3 answer from http://10.0.0.1:8080
4 all addresses failed, last error: connection refused: http://10.0.1.2:9000
5 answer from http://10.0.2.2:7000
```

- **Cases 1 to 3.** The machine `10.0.0.2` is off. Request 2 meets it and fails; its second attempt is the next turn, which reaches the third machine, so the caller sees only the answer.
- **Case 4.** When every address fails, the last error is what comes out, unless an `onError` handler gives a fallback instead, as here. The service has two addresses, so `retry(2, 50)` tries both.
- **Case 5.** `timeLimit` gives up on a machine that is slow rather than dead: the first attempt waits 0.3 seconds, is canceled, and the retry goes to the next machine.

What to remember:

- **Ask inside the block.** An address asked for *before* `doWork` is the same for every attempt, and the retry would hit the same machine again.
- **Count the attempts.** `retry(3, 50)` means three attempts in all, the first one included. To be sure of reaching a live machine when `k` machines are down, allow `k + 1` attempts. The delay can be short, because the next attempt usually goes elsewhere.
- **The time limit applies to each attempt**, not to the whole call.
- **A missing name is retried too.** `ResourceNotFoundException` is an ordinary exception for resicord, so a name that was never registered fails only after all attempts. Register names at start-up so this cannot happen.
- **A bulkhead counts all addresses together.** `bulkhead("orders")` limits how many calls through that bulkhead run at once, over every machine of the service, not per machine.
- **Retry only calls that are safe to repeat.** A `GET` can go to a second machine without harm; a `POST` that creates something may already have been done on the first.

The same pattern works around a jclient client, where every request already asks jbalancer:

```java
OrderApi orders = JClient.builder().service("orders").bind(OrderApi.class);   // jbalancer picks the machine

List<Order> all = new Try<List<Order>>()
        .doWork(orders::all)                          // every attempt is a new request, so a new address
        .retry(3, 100)
        .timeLimit(3000)
        .get();
```

The three libraries each do one thing: jbalancer takes turns, jclient makes the request, resicord decides what to do when it fails.

## 10. A real use

The balancer hands out turns, and it has no idea whether an address is alive. Chapter 9 let resicord do the retrying; here the same idea is written by hand, which shows what happens underneath. A program that wants to avoid dead machines asks again, a limited number of times, until it gets a live one:

```java
import org.j2os.platform.jbalancer.JRoundRobinBalancer;

import java.util.List;
import java.util.Set;

public class Workflow {
    public static void main(String[] args) throws Exception {
        JRoundRobinBalancer balancer = JRoundRobinBalancer.getInstance();
        balancer.configurationResource("orders", List.of("http://10.0.0.1:8080", "http://10.0.0.2:8080", "http://10.0.0.3:8080"));
        Set<String> down = Set.of("http://10.0.0.2:8080");            // the machines we know are off

        for (int request = 1; request <= 4; request++) {
            String base = null;
            for (int attempt = 0; attempt < 3 && base == null; attempt++) {
                String next = balancer.getResourceUrl("orders");
                if (!down.contains(next)) base = next;                // a live one: use it
            }
            System.out.println(request + " " + base + "/list");
        }
    }
}
```

```text
1 http://10.0.0.1:8080/list
2 http://10.0.0.3:8080/list
3 http://10.0.0.1:8080/list
4 http://10.0.0.3:8080/list
```

- **The loop** asks for at most three addresses and keeps the first one that is not known to be down. If all three are down, `base` stays `null`, and the program must decide what to do about it.
- **The cost** is only the extra question: the live machines still share the requests evenly (the first and the third address alternate in the output), but every round of turns spends one turn on the dead address. With several dead addresses in a row, three attempts may not be enough. Remove a dead address from the list when it will be down for long.
- For a lasting fix, check health regularly and register only the machines that answer:

```java
// every 30 seconds, in a scheduled task
List<String> healthy = allAddresses.stream().filter(Health::answers).toList();
if (!healthy.isEmpty()) {
    balancer.configurationResource("orders", healthy);         // replaces the list; the turn starts again at the first
}
```

## Appendix A. Quick reference

### A.1 JRoundRobinBalancer

| Call | Purpose |
|------|---------|
| `getInstance()` | The single registry of the JVM |
| `configurationResource(name, urls)` | Registers a name with a list, or replaces the list and restarts the turn. Returns the balancer |
| `removeResource(name)` | Removes a name. Silent if missing. Returns the balancer |
| `getResourceUrl(name)` | The next address of a name. Throws `ResourceNotFoundException` if the name is not registered |

### A.2 Rules for names and lists

| Rule | Detail |
|------|--------|
| Case | Names are not case-sensitive; stored in capital letters |
| Name | Must not be `null` |
| List | Must not be `null`, empty, or hold a `null` entry |
| Copy | The list is copied when registered |
| Order | Handed out in list order, starting from the first |
| Repeats | Allowed; an address repeated gets a larger share |

### A.3 JResource

| Call | Purpose |
|------|---------|
| `new JResource(urls)` | A rotation over a copy of the list. Rejects `null` and empty |
| `getNextUrl()` | The next address; wraps to the first after the last |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| `getResourceUrl` for a name that is not registered | `ResourceNotFoundException` (checked); the message names the service in capital letters |
| A `null` name in any call | `NullPointerException` "resourceId must not be null" |
| `configurationResource` with `null` or an empty list | `IllegalArgumentException` "urls must not be null or empty" |
| A list with a `null` entry | `NullPointerException` at registration |
| `removeResource` for an unknown name | Nothing happens |
| Registering an existing name | The list is replaced and the turn starts again |
| An address that is wrong or down | Handed out like any other; the caller must cope |
| The program restarts | The registry is empty; register again |
| A list changed after registering | No effect until it is registered again |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Expecting health checks.** A dead address is handed out as often as a live one.
- **Forgetting to register at start-up.** The registry is empty after every restart.
- **Changing your list and expecting the balancer to notice.** It uses its own copy; register again.
- **Retrying a call that is not safe to repeat.** Failover through resicord sends the repeat to another machine; a `POST` may already have been done.
- **Re-registering often.** Every registration restarts the turn at the first address, so a program that re-registers more often than it asks would always hit the first address.
- **Using the same name in two libraries.** There is one registry; the second registration replaces the first.
- **Typing mistakes in addresses.** They are not checked.
- **Expecting a strict order under many threads.** Only the share is exact, not who gets which.
- **Ignoring the checked exception.** Catching and swallowing it hides a configuration error.
- **Addresses with paths when used with jclient.** Register base addresses and keep paths in the interface.
- **Expecting the original name in the message.** It shows the name in capital letters.

### C.2 Behavior worth knowing

- The registry is created when the class is loaded and lasts until the JVM ends.
- A turn is a counter that goes round the list; nothing is stored about success or failure.
- Registration builds a new rotation, so the counter is back at zero.
- Asking uses an atomic counter for each name, so callers do not block each other.
- Name matching uses `Locale.ROOT`, so results do not depend on the computer's language settings.
- Nothing connects to an address, so asking is cheap enough to do for every request.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Registries per JVM | One |
| Names | As many as memory allows; no way to list them |
| Addresses per name | At least one; no upper limit |
| Weights | Only by repeating an address |
| Health checks | None |
| Persistence | None; memory only |
| Address checking | None |

### C.4 Tuning and practical advice

- Register all services in one place at start-up, from configuration.
- Keep the master list of addresses yourself; the balancer cannot give it back.
- If you need health awareness, check regularly and re-register only the healthy addresses, but not more often than needed.
- To avoid always starting at the same address after a re-registration, shuffle the list before registering.
- Use short, clear, lower-case names, and one naming scheme for the whole system.
- Treat `ResourceNotFoundException` as a configuration error and let the program stop with a clear message.
- For weighted sharing, repeat the address and spread the repeats through the list.

## Appendix D. Message reference

### D.1 JRoundRobinBalancer and JResource

| Message | Exception | Cause |
|---------|-----------|-------|
| `resourceId must not be null` | `NullPointerException` | A `null` name in `configurationResource`, `removeResource` or `getResourceUrl` |
| `urls must not be null or empty` | `IllegalArgumentException` | `configurationResource` (or `new JResource`) with `null` or an empty list |
| none | `NullPointerException` | A list with a `null` entry |

### D.2 ResourceNotFoundException

| Message | Cause |
|---------|-------|
| `Resource <NAME> not found, creating a new one using configurationResource(String resourceId, List<String> urls)` | `getResourceUrl` for a name that is not registered; `<NAME>` is in capital letters |
