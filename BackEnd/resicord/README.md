# resicord

*J2OS Platform 2.4*

Every application talks to something it does not control: a payment gateway, an SMS provider, another team's service, a slow report database. Those calls fail, hang, or pile up under load, and the failure of one dependency should not become the failure of the whole application.

**resicord** (`org.j2os.platform.resicord`) is a small library that wraps a call in four protections, written as one fluent chain:

- **retry**: try again when the call fails;
- **time limit**: stop waiting when the call takes too long;
- **bulkhead**: let only a fixed number of callers into a dependency at once, and turn the rest away quickly instead of letting them pile up;
- **fallback** (`onError`): decide what the caller gets when everything has failed.

The entry point is a single class, `Try<T>`. Policies can be written inline or defined once under a name and shared, so they can be tuned in one place. This guide goes step by step: each chapter has a short program followed by the output it prints. The appendices are for quick reference.

## Contents

1. [The idea and the shape of a Try](#1-the-idea-and-the-shape-of-a-try)
2. [Your first Try](#2-your-first-try)
3. [onError: deciding what the caller gets](#3-onerror-deciding-what-the-caller-gets)
4. [Retry](#4-retry)
5. [Time limit](#5-time-limit)
6. [Named policies](#6-named-policies)
7. [Bulkhead](#7-bulkhead)
8. [Combining the protections](#8-combining-the-protections)
9. [Threads, interrupts and errors](#9-threads-interrupts-and-errors)
10. [In a web application](#10-in-a-web-application)
11. [Putting it together: shipping quotes](#11-putting-it-together-shipping-quotes)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The programs in chapters 2 to 9 and 11 are complete and can be run as they are. They contain real delays, so some numbers in the output are approximate ("about 300 ms").

---

## 1. The idea and the shape of a Try

A `Try` is built from the outside in: first the work, then the protections, then `get()`, which runs everything.

```java
T result = new Try<T>()
        .doWork(() -> riskyCall())      // the work (may throw anything)
        .retry(3, 200)                  // optional: up to 3 attempts, 200 ms apart
        .timeLimit(2000)                // optional: stop waiting after 2 s
        .bulkhead("courier")            // optional: run inside a named, size-limited pool
        .onError(error -> fallback)     // optional: value to use when everything failed
        .get();                         // runs it
```

Only `doWork` is required. Everything else is optional and can be combined freely.

When `get()` runs, each *attempt* passes through the protections in this order:

1. If a **bulkhead** is set, the attempt must first get a place in it. If every place is taken, it waits up to `maxWaitMillis`; if no place becomes free in that time, the attempt does not run and is rejected with a `BulkheadRejectedExecutionException`.
2. If a **time limit** is set, the clock starts when the work starts (not when it queued) and the work is stopped when the clock runs out.
3. The **work** runs.
4. If the attempt failed and attempts remain, **retry** waits the delay and starts a new attempt from step 1.
5. If the attempts are used up, **onError** supplies the result. Without `onError`, the last failure is thrown.

So retry is the outermost loop and each attempt gets a fresh bulkhead place and a fresh time limit.

Three small ideas are worth knowing before the first example:

- A `Try` runs on the **calling thread** unless a time limit or a bulkhead is set. With either of them the work runs on a helper thread and the caller waits for it.
- Policies are plain immutable values (`RetryPolicy`, `TimeLimitPolicy`, `BulkheadPolicy`). Retry and time limit can be written inline or defined under a name; bulkheads are always named.
- A `Try` is a cheap, single-use-at-a-time object. Build one per call; do not share one between threads.

## 2. Your first Try

The work is a `Block<T>`: a lambda that returns a `T` and may throw anything. `new Try<String>(() -> "hello")` and `new Try<String>().doWork(() -> "hello")` are the same thing.

```java
import org.j2os.platform.resicord.Try;

import java.io.IOException;

public class FirstTry {

    public static void main(String[] args) {
        // The work goes into doWork, get() runs it
        String result = new Try<String>()
                .doWork(() -> "hello")
                .get();
        System.out.println("1. " + result);

        // With no time limit and no bulkhead, the work runs on the calling thread
        String thread = new Try<String>()
                .doWork(() -> Thread.currentThread().getName())
                .get();
        System.out.println("2. ran on: " + thread);

        // A checked exception comes out wrapped in a RuntimeException
        try {
            new Try<String>()
                    .doWork(() -> {
                        throw new IOException("disk not ready");
                    })
                    .get();
        } catch (RuntimeException e) {
            System.out.println("3. " + e.getClass().getSimpleName() + ", cause: " + e.getCause());
        }
    }
}
```

```text
1. hello
2. ran on: main
3. RuntimeException, cause: java.io.IOException: disk not ready
```

- **1.** `get()` runs the work and returns its result.
- **2.** With no time limit and no bulkhead nothing special happens: the work is a plain lambda called on the calling thread (`main` here).
- **3.** `get()` never declares `throws`. A checked exception such as `IOException` is wrapped in a `RuntimeException`, with the original as its cause. A `RuntimeException` thrown by the work comes out unchanged.

If `get()` is called with no work set, it throws `BlockNotInitializedException`.

## 3. onError: deciding what the caller gets

`onError` takes a function from the failure to a replacement result. It runs only after every attempt has failed.

```java
import org.j2os.platform.resicord.Try;
import org.j2os.platform.resicord.exception.BlockNotInitializedException;

import java.io.IOException;

public class FallbackDemo {

    public static void main(String[] args) {
        // The work fails, so onError decides what the caller gets
        String price = new Try<String>()
                .doWork(() -> {
                    throw new IOException("price service is down");
                })
                .onError(error -> "unknown (" + error.getClass().getSimpleName() + ")")
                .get();
        System.out.println("1. " + price);

        // The work succeeds, so onError is not used
        String ok = new Try<String>()
                .doWork(() -> "42.00")
                .onError(error -> "never used")
                .get();
        System.out.println("2. " + ok);

        // A missing doWork is a coding mistake: onError does not hide it
        try {
            new Try<String>().onError(error -> "fallback").get();
        } catch (BlockNotInitializedException e) {
            System.out.println("3. " + e.getMessage());
        }
    }
}
```

```text
1. unknown (IOException)
2. 42.00
3. No work block has been set. Call doWork(Block) before get().
```

- **1.** The work fails, so the handler decides the result. It returns the same type as the work (`String` here) and receives the original exception, not wrapped (an `IOException` here).
- **2.** The work succeeds, so the handler is never called.
- **3.** Two failures never reach the handler, because they are mistakes in the code, not failures of the dependency: a missing `doWork` (shown here) and a policy name that was never defined (`NotFoundException`, chapter 6). They are not retried either.

If the handler itself throws, that exception goes to the caller. A fallback that can fail is not a fallback, so keep the handler simple.

> The handler hides the failure from the caller, which also hides it from you. Log the error inside the handler, or the dependency can be down for a week without anyone noticing.

## 4. Retry

`retry(maxAttempts, delayMillis)` allows the work to run up to `maxAttempts` times in total (the first run counts) and pauses `delayMillis` between attempts.

```java
import org.j2os.platform.resicord.Try;

import java.util.concurrent.atomic.AtomicInteger;

public class RetryDemo {

    public static void main(String[] args) {
        AtomicInteger calls = new AtomicInteger();

        // Fails twice, works on the third attempt
        String result = new Try<String>()
                .doWork(() -> {
                    if (calls.incrementAndGet() < 3) {
                        throw new IllegalStateException("not yet");
                    }
                    return "worked on attempt " + calls.get();
                })
                .retry(3, 100)
                .get();
        System.out.println("1. " + result);

        // Always fails: after the last attempt the caller gets the last exception
        calls.set(0);
        try {
            new Try<String>()
                    .doWork(() -> {
                        throw new IllegalStateException("failure " + calls.incrementAndGet());
                    })
                    .retry(3, 100)
                    .get();
        } catch (IllegalStateException e) {
            System.out.println("2. " + e.getMessage() + ", calls: " + calls.get());
        }
    }
}
```

```text
1. worked on attempt 3
2. failure 3, calls: 3
```

- **1.** Two failures and a success: of three allowed attempts, all three are used.
- **2.** When the attempts run out, the **last** exception is the one the caller gets.

Details: there is no pause before the first attempt or after the last one. Without `retry`, the work runs once. `maxAttempts` must be at least 1 and `delayMillis` cannot be negative; otherwise `retry(...)` throws `IllegalArgumentException`.

Almost every failure is retried, including timeouts and bulkhead rejections. The exceptions are the two code mistakes from chapter 3, an interrupt of the calling thread (chapter 9), and a retry that is refused because the previous attempt is still running (section 5.1).

> Retry is only safe for work that can run twice without harm: reading data, or writing with an idempotency key (a unique id sent with the request, so the server can recognize a repeat and ignore it). A timeout does not mean the work did not happen. Retrying "charge the card" can charge it twice.

The pause is a plain `Thread.sleep` on the calling thread. In a web request that thread is held for the whole sequence, so keep attempts and delays small. Chapter 8 shows how to estimate the worst case.

## 5. Time limit

`timeLimit(millis)` stops the caller from waiting longer than `millis`. The work moves to a helper thread and the caller waits for it with a deadline. When the deadline passes, the work is interrupted and a `TimeoutExecutionException` is thrown.

```java
import org.j2os.platform.resicord.Try;
import org.j2os.platform.resicord.exception.TimeoutExecutionException;

public class TimeLimitDemo {

    public static void main(String[] args) throws Exception {
        // Fast work is not affected, but it now runs on a helper thread
        String thread = new Try<String>()
                .doWork(() -> Thread.currentThread().getName())
                .timeLimit(500)
                .get();
        System.out.println("1. ran on: " + thread);

        // Slow work: the caller is released after the limit, not after 5 seconds
        boolean[] interrupted = {false};
        long start = System.nanoTime();
        try {
            new Try<String>()
                    .doWork(() -> {
                        try {
                            Thread.sleep(5000);
                        } catch (InterruptedException e) {
                            interrupted[0] = true;
                            throw e;
                        }
                        return "too late";
                    })
                    .timeLimit(300)
                    .get();
        } catch (TimeoutExecutionException e) {
            long ms = (System.nanoTime() - start) / 1_000_000;
            System.out.println("2. timeout after about " + (ms / 100 * 100) + " ms");
        }

        Thread.sleep(100);   // cancellation is asynchronous: give the work thread a moment
        System.out.println("3. the work was interrupted: " + interrupted[0]);
    }
}
```

```text
1. ran on: resicord-timelimit-watcher-1
2. timeout after about 300 ms
3. the work was interrupted: true
```

- **1.** The work now runs on a helper thread called `resicord-timelimit-watcher-N`, not on the caller.
- **2.** The work needs 5 seconds; the caller is released after about 300 ms. The exception wraps a `java.util.concurrent.TimeoutException`.
- **3.** The work thread receives a real `InterruptedException`. Cancellation is asynchronous, so the work may need a few milliseconds to notice after the caller was already released.

A limit of 0 means "no limit" and the work stays on the calling thread. `onError` receives a timeout like any other failure, so `error instanceof TimeoutExecutionException` lets it answer differently.

### 5.1 Work that ignores the interrupt

A time limit can only *ask* the work to stop, by interrupting it. A loop that never checks for interruption, or a library call that swallows it, keeps running in the background. resicord handles that case carefully: it never starts a retry while the previous attempt is still running.

```java
import org.j2os.platform.resicord.Try;

import java.util.concurrent.atomic.AtomicInteger;

public class StubbornDemo {

    public static void main(String[] args) {
        AtomicInteger started = new AtomicInteger();

        String result = new Try<String>()
                .doWork(() -> {
                    started.incrementAndGet();
                    long end = System.currentTimeMillis() + 3000;
                    while (System.currentTimeMillis() < end) {
                        // busy loop: never checks for interruption
                    }
                    return "finished";
                })
                .timeLimit(200)
                .retry(3, 0)
                .onError(error -> "gave up: " + error.getClass().getSimpleName())
                .get();

        System.out.println(result + ", attempts started: " + started.get());
    }
}
```

```text
gave up: TimeoutExecutionException, attempts started: 1
```

The first attempt passes its 200 ms limit. Instead of starting attempt 2 next to a still-running attempt 1, resicord waits up to one second for attempt 1 to stop, sees that it has not, and gives up with the timeout as the failure. Only one attempt is ever started.

The practical rule: write the work so that it can be interrupted, and also set the real timeouts of the client underneath (HTTP connect and read timeouts, JDBC query timeout). The time limit protects the caller; those timeouts free the resource.

## 6. Named policies

Writing `retry(3, 200)` at every call site means that changing the 3 requires finding every call site. Retry policies and time-limit policies can instead be defined once under a name:

| Registry | Define | Read | List | Remove |
|----------|--------|------|------|--------|
| `RetryPolicies` | `define(name, new RetryPolicy(attempts, delay))` | `get(name)` | `listAll()` | `remove(name)` |
| `TimeLimitPolicies` | `define(name, new TimeLimitPolicy(millis))` | `get(name)` | `listAll()` | `remove(name)` |

and used with `retry("name")` and `timeLimit("name")`. The example uses retry; `TimeLimitPolicies` works the same way.

```java
import org.j2os.platform.resicord.Try;
import org.j2os.platform.resicord.exception.NotFoundException;
import org.j2os.platform.resicord.retry.RetryPolicies;
import org.j2os.platform.resicord.retry.RetryPolicy;

import java.util.concurrent.atomic.AtomicInteger;

public class NamedPolicyDemo {

    public static void main(String[] args) {
        // Define once, read back
        RetryPolicies.define("payment", new RetryPolicy(3, 50));
        System.out.println("1. " + RetryPolicies.get("payment"));

        // A Try that always fails and counts its calls
        AtomicInteger calls = new AtomicInteger();
        Try<String> flaky = new Try<String>()
                .doWork(() -> {
                    calls.incrementAndGet();
                    throw new IllegalStateException("down");
                })
                .retry("payment")
                .onError(error -> "fallback");

        flaky.get();
        System.out.println("2. calls with 3 attempts: " + calls.get());

        // Change the policy: the same Try object follows it
        RetryPolicies.define("payment", new RetryPolicy(5, 10));
        calls.set(0);
        flaky.get();
        System.out.println("3. calls after the change: " + calls.get());

        // A name that was never defined
        try {
            new Try<String>().doWork(() -> "x").retry("shipping").get();
        } catch (NotFoundException e) {
            System.out.println("4. " + e.getMessage());
        }
    }
}
```

```text
1. RetryPolicy[maxAttempts=3, delayMillis=50]
2. calls with 3 attempts: 3
3. calls after the change: 5
4. No retry policy named 'shipping' has been defined
```

- **1.** `define` stores the policy, `get` reads it back.
- **2 and 3.** The important part. The same `Try` object runs with 3 attempts, and then with 5 after `define` replaces the policy. Names are looked up when `get()` runs, not when `retry("payment")` is written. An administrator can change the behaviour of a running application without a restart.
- **4.** A `Try` that uses a name nobody defined throws `NotFoundException`. The message says whether the name was a *retry policy*, a *time limit* or a *bulkhead*.

More details: `listAll()` returns a snapshot, a copy of the registry at that moment that cannot be modified and does not follow later changes. `remove(name)` on a name that does not exist does nothing. A named time limit of 0 means "disabled".

Retry and time-limit names live in separate registries, so `"payment"` can exist in both without conflict. Using the same name for one dependency in all three registries, as the later chapters do, is a convenient convention.

## 7. Bulkhead

A ship is divided into watertight compartments so that a hole floods one of them, not the whole ship. A **bulkhead** does the same for a dependency. If the courier service becomes slow, the threads of your application pile up waiting for it, and soon nothing else works either. A bulkhead gives that dependency a small, separate pool and a waiting room of limited size, and turns everyone else away immediately.

A `BulkheadPolicy` has three numbers:

| Number | Meaning |
|--------|---------|
| `maxConcurrentThreads` | worker threads of this bulkhead = how many calls run at once. At least 1 |
| `maxQueueSize` | how many more callers may wait for a free worker. 0 or more |
| `maxWaitMillis` | how long a caller waits for a place (a worker or a waiting seat) before being rejected. 0 = reject at once if there is no place |

So `new BulkheadPolicy(2, 3, 500)` has 5 places (2 running and 3 waiting). A sixth caller waits up to 500 ms for somebody to leave and then gets a `BulkheadRejectedExecutionException`.

### 7.1 Defining and managing bulkheads

```java
import org.j2os.platform.resicord.Try;
import org.j2os.platform.resicord.bulkhead.BulkheadPolicies;
import org.j2os.platform.resicord.bulkhead.BulkheadPolicy;
import org.j2os.platform.resicord.exception.NotFoundException;

public class BulkheadBasics {

    public static void main(String[] args) {
        // 2 workers, 3 waiting seats, a caller waits at most 500 ms for a place
        BulkheadPolicies.define("sms", new BulkheadPolicy(2, 3, 500));
        BulkheadPolicy policy = BulkheadPolicies.currentConfig("sms");
        System.out.println("1. " + policy + ", places in total: " + policy.totalCapacity());

        // The work runs on a thread that belongs to the bulkhead
        String thread = new Try<String>()
                .doWork(() -> Thread.currentThread().getName())
                .bulkhead("sms")
                .get();
        System.out.println("2. ran on: " + thread);

        // Change the numbers of the running bulkhead
        BulkheadPolicies.reconfigure("sms", new BulkheadPolicy(4, 8, 1000));
        System.out.println("3. " + BulkheadPolicies.currentConfig("sms"));

        // Remove it: the name is gone afterwards
        BulkheadPolicies.remove("sms");
        try {
            BulkheadPolicies.currentConfig("sms");
        } catch (NotFoundException e) {
            System.out.println("4. " + e.getMessage());
        }
    }
}
```

```text
1. BulkheadPolicy[maxConcurrentThreads=2, maxQueueSize=3, maxWaitMillis=500], places in total: 5
2. ran on: resicord-bulkhead-sms-1
3. BulkheadPolicy[maxConcurrentThreads=4, maxQueueSize=8, maxWaitMillis=1000]
4. No bulkhead named 'sms' has been defined
```

- **1.** `define` creates a bulkhead with its own thread pool. `currentConfig` reads the numbers back and `totalCapacity()` is threads plus queue.
- **2.** The work runs on a thread of the bulkhead, named `resicord-bulkhead-<name>-N`. These are daemon threads, so they never keep the JVM alive.
- **3.** `reconfigure` changes the numbers of a **running** bulkhead without losing work in progress. Calling `define` again with an existing name does the same.
- **4.** `remove` shuts the pool down: tasks already accepted finish, new ones are refused. Unlike retry and time-limit policies, a bulkhead owns live threads, so removing it matters.

More details: `reconfigure` and `currentConfig` on an unknown name throw `NotFoundException`. `listAll()` returns a snapshot of all names and numbers. The numbers are validated when the policy is created: `new BulkheadPolicy(0, 0, 0)` throws `IllegalArgumentException`. `BulkheadPolicies.execute(name, callable, timeoutMillis)` runs a callable through a bulkhead without `Try`: no retry, no fallback, and it throws whatever the callable throws.

### 7.2 What happens under load

Six callers arrive at the same moment at a bulkhead with 2 workers, 2 waiting seats and a patience of 100 ms. Each job takes 400 ms.

```java
import org.j2os.platform.resicord.Try;
import org.j2os.platform.resicord.bulkhead.BulkheadPolicies;
import org.j2os.platform.resicord.bulkhead.BulkheadPolicy;
import org.j2os.platform.resicord.exception.BulkheadRejectedExecutionException;

import java.util.concurrent.atomic.AtomicInteger;

public class BulkheadLoad {

    public static void main(String[] args) throws Exception {
        // 2 running + 2 waiting = 4 places; a caller waits 100 ms for a place, then gives up
        BulkheadPolicies.define("reports", new BulkheadPolicy(2, 2, 100));

        AtomicInteger served = new AtomicInteger();
        AtomicInteger rejected = new AtomicInteger();

        // Six callers arrive together, each job needs 400 ms
        Thread[] callers = new Thread[6];
        for (int i = 0; i < callers.length; i++) {
            callers[i] = new Thread(() -> {
                try {
                    new Try<String>()
                            .doWork(() -> {
                                Thread.sleep(400);
                                return "report";
                            })
                            .bulkhead("reports")
                            .get();
                    served.incrementAndGet();
                } catch (BulkheadRejectedExecutionException e) {
                    rejected.incrementAndGet();
                }
            });
            callers[i].start();
        }
        for (Thread caller : callers) {
            caller.join();
        }
        System.out.println("served: " + served.get() + ", rejected: " + rejected.get());
    }
}
```

```text
served: 4, rejected: 2
```

Four callers fit: two running and two waiting. The other two wait their 100 ms, see nobody leave, and are rejected. That is the point of a bulkhead. The rejected callers learn within a tenth of a second instead of joining an unbounded line, and the dependency never sees more than two calls at once.

### 7.3 A place is freed only when the work really ends

A call that times out frees the *caller*, but not necessarily the *place*. The place is returned when the work has truly finished running.

```java
import org.j2os.platform.resicord.Try;
import org.j2os.platform.resicord.bulkhead.BulkheadPolicies;
import org.j2os.platform.resicord.bulkhead.BulkheadPolicy;

public class BulkheadPermit {

    public static void main(String[] args) throws Exception {
        // One place only, nobody waits
        BulkheadPolicies.define("legacy", new BulkheadPolicy(1, 0, 0));

        // This work ignores interruption and runs for 800 ms, but the limit is 100 ms
        String first = new Try<String>()
                .doWork(() -> {
                    long end = System.currentTimeMillis() + 800;
                    while (System.currentTimeMillis() < end) {
                        // busy loop
                    }
                    return "done";
                })
                .timeLimit(100)
                .bulkhead("legacy")
                .onError(error -> "first call: " + error.getClass().getSimpleName())
                .get();
        System.out.println(first);

        // The first caller is free, but its place is not: the work is still running
        String second = new Try<String>()
                .doWork(() -> "ok")
                .bulkhead("legacy")
                .onError(error -> "second call: " + error.getClass().getSimpleName())
                .get();
        System.out.println(second);

        Thread.sleep(900);   // now the first work has really ended

        String third = new Try<String>()
                .doWork(() -> "ok")
                .bulkhead("legacy")
                .onError(error -> "third call: " + error.getClass().getSimpleName())
                .get();
        System.out.println("third call: " + third);
    }
}
```

```text
first call: TimeoutExecutionException
second call: BulkheadRejectedExecutionException
third call: ok
```

The first call times out after 100 ms. Its work ignores the interrupt and keeps the only worker busy, so a second call, made right away, is rejected. After the work ends, the third call is served. This protects the dependency from a hung call that its caller has forgotten but that still uses a thread.

### 7.4 Choosing the numbers

- `maxConcurrentThreads`: calls per second × seconds per call, plus some headroom (Little's law). If the courier is called 20 times a second and answers in 0.2 s, about 4 calls are in progress at any moment, so 8 workers are comfortable.
- `maxQueueSize`: absorbs short bursts. A long queue only turns an overload into long waits, so start small.
- `maxWaitMillis`: much smaller than what the user is willing to wait. Rejecting in 100 to 200 ms and showing a fallback is usually better than a perfect answer after ten seconds.

## 8. Combining the protections

Retry, time limit and bulkhead are used together in one chain. The order in which you write the chain does not matter; the order of execution inside `get()` is the one from chapter 1.

One detail deserves a program: the time limit counts from the moment the work **starts**, not from the moment the caller arrives. A caller who waited in the bulkhead still gets its full limit.

```java
import org.j2os.platform.resicord.Try;
import org.j2os.platform.resicord.bulkhead.BulkheadPolicies;
import org.j2os.platform.resicord.bulkhead.BulkheadPolicy;

public class QueueVsLimit {

    public static void main(String[] args) throws Exception {
        // One worker, one waiting seat, a caller may wait 2 s for a place
        BulkheadPolicies.define("db", new BulkheadPolicy(1, 1, 2000));

        // Caller A keeps the only worker busy for 600 ms
        Thread a = new Thread(() -> new Try<String>()
                .doWork(() -> {
                    Thread.sleep(600);
                    return "A";
                })
                .bulkhead("db")
                .get());
        a.start();
        Thread.sleep(100);

        // Caller B needs a few milliseconds and has a 300 ms limit,
        // but it waits about 500 ms for the worker first
        long start = System.nanoTime();
        String b = new Try<String>()
                .doWork(() -> "B finished")
                .timeLimit(300)
                .bulkhead("db")
                .onError(error -> "B failed: " + error.getClass().getSimpleName())
                .get();
        long ms = (System.nanoTime() - start) / 1_000_000;
        System.out.println(b + ", total time about " + (ms / 100 * 100) + " ms");
        a.join();
    }
}
```

```text
B finished, total time about 500 ms
```

Caller B has a 300 ms limit and waits about 500 ms for the worker, yet succeeds: its work starts only after A releases the worker, and then needs just a few milliseconds. Waiting is controlled by the bulkhead's `maxWaitMillis`; running is controlled by the time limit.

Now all three together, with named policies. Attempt 1 hangs and is cut off by the time limit, attempt 2 fails, attempt 3 works.

```java
import org.j2os.platform.resicord.Try;
import org.j2os.platform.resicord.bulkhead.BulkheadPolicies;
import org.j2os.platform.resicord.bulkhead.BulkheadPolicy;
import org.j2os.platform.resicord.retry.RetryPolicies;
import org.j2os.platform.resicord.retry.RetryPolicy;
import org.j2os.platform.resicord.timelimit.TimeLimitPolicies;
import org.j2os.platform.resicord.timelimit.TimeLimitPolicy;

import java.util.concurrent.atomic.AtomicInteger;

public class Combined {

    public static void main(String[] args) {
        RetryPolicies.define("quotes", new RetryPolicy(3, 100));
        TimeLimitPolicies.define("quotes", new TimeLimitPolicy(300));
        BulkheadPolicies.define("quotes", new BulkheadPolicy(2, 2, 200));

        AtomicInteger calls = new AtomicInteger();

        // Attempt 1 hangs, attempt 2 fails, attempt 3 works
        String quote = new Try<String>()
                .doWork(() -> {
                    int n = calls.incrementAndGet();
                    if (n == 1) {
                        Thread.sleep(2000);
                    }
                    if (n == 2) {
                        throw new IllegalStateException("upstream returned 503");
                    }
                    return "EUR/USD 1.0842 (attempt " + n + ")";
                })
                .retry("quotes")
                .timeLimit("quotes")
                .bulkhead("quotes")
                .onError(error -> "no quote: " + error.getMessage())
                .get();
        System.out.println(quote);
    }
}
```

```text
EUR/USD 1.0842 (attempt 3)
```

The caller never sees the two failures. Chapter 11 shows how `onError` can tell the reasons apart when all attempts fail.

**Worst-case time.** For `A` attempts, delay `D`, time limit `L` and bulkhead wait `W`, the longest a caller can be held is about `A × (W + L) + (A − 1) × D`. For the policies above, `3 × (200 + 300) + 2 × 100 = 1700 ms`. If an attempt ignores its interrupt (section 5.1), add up to one more second: the extra wait before resicord gives up on a retry. Compare the result with what the user, or the web server's own timeout, tolerates, and reduce attempts before reducing limits.

## 9. Threads, interrupts and errors

**An interrupt of the calling thread is respected.** *Interrupting* a thread is how Java asks it to stop: the thread gets an `InterruptedException` if it is waiting, and a flag on the thread is switched on. If the thread that called `get()` is interrupted, for example because the application is shutting down, resicord treats it as a request to stop, not as a failure to retry. It does not retry, it returns the fallback (or throws) at once, and it switches the flag back on, so the code above `get()` can still see that the thread was asked to stop.

**Errors are not handled.** Only `Exception`s go through retry and `onError`. An `Error` such as `OutOfMemoryError` means the JVM itself is in trouble, and it passes straight through.

```java
import org.j2os.platform.resicord.Try;

import java.util.concurrent.atomic.AtomicInteger;

public class InterruptDemo {

    public static void main(String[] args) throws Exception {
        AtomicInteger attempts = new AtomicInteger();

        // A request thread that is cancelled while it waits between two attempts
        Thread request = new Thread(() -> {
            String result = new Try<String>()
                    .doWork(() -> {
                        attempts.incrementAndGet();
                        throw new IllegalStateException("down");
                    })
                    .retry(5, 2000)
                    .onError(error -> "stopped: " + error.getMessage())
                    .get();
            System.out.println(result + ", attempts: " + attempts.get());
            System.out.println("interrupt flag still set: " + Thread.currentThread().isInterrupted());
        });
        request.start();
        Thread.sleep(300);       // attempt 1 has failed, the thread is waiting before attempt 2
        request.interrupt();
        request.join();

        // An Error is not handled: it passes through retry and onError
        try {
            new Try<String>()
                    .doWork(() -> {
                        throw new OutOfMemoryError("simulated");
                    })
                    .retry(3, 0)
                    .onError(error -> "handled")
                    .get();
        } catch (OutOfMemoryError e) {
            System.out.println("Error passed through: " + e.getMessage());
        }
    }
}
```

```text
stopped: down, attempts: 1
interrupt flag still set: true
Error passed through: simulated
```

The request thread is interrupted during the 2 s pause before attempt 2. `get()` returns the fallback at once, after one attempt, and the interrupt flag is still on afterwards. The `OutOfMemoryError` leaves `get()` untouched even though `retry` and `onError` were set.

**Which thread runs the work.** The calling thread when no protection is set; `resicord-timelimit-watcher-N` with a time limit; `resicord-bulkhead-<name>-N` with a bulkhead. Anything tied to the calling thread is not visible to the work on the other two. Examples are `ThreadLocal` values, the logged-in user that Spring Security keeps on the thread, an open database transaction, and objects that exist only during one web request. Read what you need into local variables before `doWork` and use those inside it.

## 10. In a web application

Define the policies once at startup, with values coming from configuration so they can be changed per environment:

```java
import org.j2os.platform.resicord.bulkhead.BulkheadPolicies;
import org.j2os.platform.resicord.bulkhead.BulkheadPolicy;
import org.j2os.platform.resicord.retry.RetryPolicies;
import org.j2os.platform.resicord.retry.RetryPolicy;
import org.j2os.platform.resicord.timelimit.TimeLimitPolicies;
import org.j2os.platform.resicord.timelimit.TimeLimitPolicy;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;

import jakarta.annotation.PostConstruct;

@Configuration
public class ResilienceConfig {

    @Value("${courier.attempts:3}")      private int attempts;
    @Value("${courier.limit-ms:2000}")   private long limitMillis;
    @Value("${courier.threads:8}")       private int threads;
    @Value("${courier.queue:16}")        private int queue;

    @PostConstruct
    void define() {
        RetryPolicies.define("courier", new RetryPolicy(attempts, 100));
        TimeLimitPolicies.define("courier", new TimeLimitPolicy(limitMillis));
        BulkheadPolicies.define("courier", new BulkheadPolicy(threads, queue, 200));
    }
}
```

Then each endpoint that talks to the dependency builds its `Try` per request and chooses what the user should see when the courier is unavailable:

```java
import org.j2os.platform.resicord.Try;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/shipping")
public class ShippingController {

    private final CourierClient courier;          // the class that really calls the courier

    public ShippingController(CourierClient courier) {
        this.courier = courier;
    }

    @GetMapping("/quote")
    public Map<String, Object> quote(@RequestParam String city) {
        return new Try<Map<String, Object>>()
                .doWork(() -> Map.of("city", city, "price", courier.price(city), "exact", true))
                .retry("courier")
                .timeLimit("courier")
                .bulkhead("courier")
                .onError(error -> Map.of("city", city, "price", 15.0, "exact", false))
                .get();
    }
}
```

Two points in this controller:

- The fallback answers with a flat rate and `exact: false`, so the front end can show "approximate price" instead of an error page.
- The values read from the request (`city`) are captured by the lambda; nothing depends on state tied to the request thread, which matters because the work may run on another thread (chapter 9).

An administrator can then resize the bulkhead of a running application:

```java
@PutMapping("/api/admin/courier-capacity")
public BulkheadPolicy resize(@RequestParam int threads, @RequestParam int queue) {
    BulkheadPolicies.reconfigure("courier", new BulkheadPolicy(threads, queue, 200));
    return BulkheadPolicies.currentConfig("courier");
}
```

### 10.1 A page2 list protected by resicord

A typical use inside J2OS is a `page2` query (see the page2 guide) against a database that is sometimes slow or briefly unreachable: a reporting database, a replica, a shard. The pattern has three rules.

```java
import jakarta.annotation.Resource;
import org.j2os.platform.page2.PageDataSQL;
import org.j2os.platform.resicord.Try;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import javax.sql.DataSource;
import java.sql.Connection;
import java.util.Map;

@RestController
public class ReportController {

    @Resource
    private DataSource reportDataSource;

    @GetMapping("/api/report")
    public Map<String, Object> report(@RequestParam Map<String, Object> params) {
        return new Try<Map<String, Object>>()
                .doWork(() -> {
                    // The connection is opened INSIDE the work: every attempt gets its own,
                    // and it is closed even when the attempt fails or is interrupted.
                    try (Connection connection = reportDataSource.getConnection()) {
                        return new PageDataSQL(connection)
                                .searchAndSortOn("b.book_id", "b.title")
                                .getResult("SELECT b.book_id, b.title FROM book b",
                                        "b.book_id", null, params);
                    }
                })
                .retry("report")
                .timeLimit("report")
                .bulkhead("report")
                .get();            // no onError: a failure becomes an HTTP error that DataTable reports
    }
}
```

- **Open the connection inside `doWork`.** Every attempt then gets a fresh connection, and `try-with-resources` closes it whichever way the attempt ends. A connection opened outside and reused by a retry may already be broken.
- **Do not use the request's transaction here.** `@Transactional` and `DataSourceUtils.getConnection(...)` bind the connection to the request thread. With a time limit or a bulkhead the work runs on another thread and would not see it (chapter 9). Use a plain connection from the data source, as above; this is fine for read-only lists.
- **For lists, prefer no `onError`.** Without it the failure leaves the controller as an HTTP error, and DataTable shows its own "error while fetching data" message and lets the user try again. A fallback that returns an empty page would look like "no rows", which is a lie. Use a fallback for values where an approximate answer is honest (such as the shipping price), not for tables of records.

Retrying is safe here because the query only reads.

## 11. Putting it together: shipping quotes

A checkout page asks a courier for a shipping price. The courier is sometimes unavailable and sometimes slow, the page must always answer, and the operator must be able to give the dependency more room without a restart.

The program defines the three policies once, wraps the call in one method, and tries two situations: a single transient failure, and eight customers arriving together, first with the original capacity and then after the operator has raised it.

```java
import org.j2os.platform.resicord.Try;
import org.j2os.platform.resicord.bulkhead.BulkheadPolicies;
import org.j2os.platform.resicord.bulkhead.BulkheadPolicy;
import org.j2os.platform.resicord.exception.BulkheadRejectedExecutionException;
import org.j2os.platform.resicord.retry.RetryPolicies;
import org.j2os.platform.resicord.retry.RetryPolicy;
import org.j2os.platform.resicord.timelimit.TimeLimitPolicies;
import org.j2os.platform.resicord.timelimit.TimeLimitPolicy;

import java.util.Collections;
import java.util.List;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.atomic.AtomicInteger;

public class ShippingQuotes {

    static final AtomicInteger tehranCalls = new AtomicInteger();

    // The one method the rest of the application calls
    static String quote(String city) {
        return new Try<String>()
                .doWork(() -> {
                    // stands in for the real HTTP call to the courier
                    if (city.equals("Tehran")) {
                        if (tehranCalls.incrementAndGet() == 1) {
                            throw new IllegalStateException("courier returned 503");
                        }
                        return "9.50 USD";
                    }
                    Thread.sleep(1000);                    // any other city: the courier is slow today
                    return "11.00 USD";
                })
                .retry("courier")
                .timeLimit("courier")
                .bulkhead("courier")
                .onError(error -> error instanceof BulkheadRejectedExecutionException
                        ? "flat 15.00 USD (courier busy)"
                        : "flat 15.00 USD (courier failed)")
                .get();
    }

    public static void main(String[] args) throws Exception {
        // Configuration, done once at startup
        RetryPolicies.define("courier", new RetryPolicy(3, 50));
        TimeLimitPolicies.define("courier", new TimeLimitPolicy(2000));
        BulkheadPolicies.define("courier", new BulkheadPolicy(2, 2, 100));

        // The first call fails, the retry succeeds
        System.out.println("Tehran: " + quote("Tehran"));

        // Eight customers ask at the same moment, before and after the operator raises the capacity
        ExecutorService customers = Executors.newFixedThreadPool(8);
        for (int round = 1; round <= 2; round++) {
            List<Callable<String>> requests = Collections.nCopies(8, () -> quote("Shiraz"));
            int exact = 0;
            for (Future<String> answer : customers.invokeAll(requests)) {
                if (answer.get().startsWith("11.00")) {
                    exact++;
                }
            }
            System.out.println("Round " + round + ": " + exact + " exact quotes, " + (8 - exact) + " flat rates");
            BulkheadPolicies.reconfigure("courier", new BulkheadPolicy(8, 0, 100));
        }
        customers.shutdown();
    }
}
```

```text
Tehran: 9.50 USD
Round 1: 4 exact quotes, 4 flat rates
Round 2: 8 exact quotes, 0 flat rates
```

Reading the output:

- **Tehran.** The first call fails with a 503, the retry succeeds, and the customer sees a normal price.
- **Round 1.** The bulkhead (2 workers, 2 seats, 100 ms patience) serves four customers. The other four are rejected, try once more after 50 ms, are rejected again, and receive the flat rate. None of them waits more than a fraction of a second, and the courier never sees more than two calls at once. Because `onError` receives the reason, the flat rate is labeled "busy" for a rejection and "failed" for anything else.
- **Round 2.** After `reconfigure` to 8 workers, all eight customers are served in parallel.

What to take into your own code: one policy name per dependency, one method that owns the `Try`, a fallback that returns something useful and says it is approximate, and a capacity that can be changed while the application runs.

---

## Appendix A. Quick reference

### A.1 Try&lt;T&gt;

| Member | Description |
|--------|-------------|
| `new Try<T>()` / `new Try<T>(Block<T>)` | empty `Try` / `Try` with the work already set |
| `doWork(Block<T>)` | sets or replaces the work; `Block<T>` is `T body() throws Exception` |
| `retry(int maxAttempts, long delayMillis)` | inline retry; replaces a previously set named retry |
| `retry(String name)` | named retry, resolved at `get()`; `null` throws `NullPointerException` |
| `timeLimit(long millis)` | inline time limit; 0 = off |
| `timeLimit(String name)` | named time limit, resolved at `get()` |
| `bulkhead(String name)` | run through a named bulkhead; there is no inline form |
| `onError(ErrorBlock<T>)` | fallback; `ErrorBlock<R>` is `R body(Throwable)` |
| `get()` | runs everything and returns the result (or the fallback). Throws unchecked exceptions only |

All configuration methods return the same `Try`, so they chain.

### A.2 RetryPolicy and RetryPolicies

| Member | Description |
|--------|-------------|
| `RetryPolicy(int maxAttempts, long delayMillis)` | record; `maxAttempts >= 1`, `delayMillis >= 0` |
| `RetryPolicy.NONE` | `(1, 0)`: one attempt, used when nothing is set |
| `RetryPolicies.define(name, policy)` | adds or replaces |
| `RetryPolicies.get(name)` | the policy; `NotFoundException` if undefined |
| `RetryPolicies.listAll()` | immutable snapshot `Map<String, RetryPolicy>` |
| `RetryPolicies.remove(name)` | removes; harmless if missing |

### A.3 TimeLimitPolicy and TimeLimitPolicies

| Member | Description |
|--------|-------------|
| `TimeLimitPolicy(long millis)` | record; `millis >= 0` |
| `isEnabled()` | `millis > 0` |
| `TimeLimitPolicy.NONE` | `(0)`: disabled |
| `TimeLimitPolicies.define / get / listAll / remove` | the same four methods as above |

### A.4 BulkheadPolicy and BulkheadPolicies

| Member | Description |
|--------|-------------|
| `BulkheadPolicy(int maxConcurrentThreads, int maxQueueSize, long maxWaitMillis)` | record; `>= 1`, `>= 0`, `>= 0` |
| `totalCapacity()` | threads + queue |
| `BulkheadPolicies.define(name, policy)` | creates the bulkhead; on an existing name reconfigures it in place |
| `BulkheadPolicies.reconfigure(name, policy)` | in place; `NotFoundException` if undefined |
| `BulkheadPolicies.currentConfig(name)` | the policy in effect; `NotFoundException` if undefined |
| `BulkheadPolicies.listAll()` | immutable snapshot `Map<String, BulkheadPolicy>` |
| `BulkheadPolicies.remove(name)` | shuts the pool down (accepted work finishes); harmless if missing |
| `BulkheadPolicies.execute(name, Callable<T>)` | runs through the bulkhead, no time limit |
| `BulkheadPolicies.execute(name, Callable<T>, long timeoutMillis)` | same, with a limit once the work has started; 0 = none |

### A.5 Exceptions

| Exception | Thrown when |
|-----------|-------------|
| `BlockNotInitializedException` | `get()` without work |
| `NotFoundException` | a named retry policy, time limit or bulkhead is not defined |
| `TimeoutExecutionException` | the work exceeded its time limit; the cause is a `TimeoutException` |
| `BulkheadRejectedExecutionException` | no place became free within `maxWaitMillis`, or the bulkhead is shut down |

All four extend `RuntimeException`.

## Appendix B. Failure handling

What the caller of `get()` finally sees, for each kind of failure:

| Failure | Retried? | `onError` called? | Without `onError` |
|---------|----------|-------------------|-------------------|
| exception from the work | yes | yes | last `RuntimeException` as it is, a checked exception wrapped in `RuntimeException` |
| `TimeoutExecutionException` | yes, if the previous attempt has stopped | yes | thrown |
| `BulkheadRejectedExecutionException` | yes | yes | thrown |
| `RejectedExecutionException` (the helper-thread limit, see C.3) | yes | yes | thrown |
| `NotFoundException` | no | **no** | thrown |
| `BlockNotInitializedException` | no | **no** | thrown |
| caller thread interrupted | no | yes | `RuntimeException` wrapping `InterruptedException`; interrupt flag restored |
| exception thrown by the `onError` handler | no | n/a | thrown as it is |
| `Error` (`OutOfMemoryError`, ...) | no | no | thrown |
| invalid policy numbers | n/a | n/a | `IllegalArgumentException` at policy creation |

Notes:

- When the time limit fires and a retry is still allowed, resicord waits up to **one second** for the old attempt to stop. If it has not stopped, no retry is started and the timeout is treated as final.
- A named policy is looked up when `get()` runs, so a missing name fails there, not when the chain is written.

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **`new Try<>()` infers `Object`.** The diamond works, but the result is an `Object` that must be cast. Write `new Try<String>()`.
- **Retrying work that is not repeatable.** A time-out says nothing about whether the work happened. Retry reads, or writes protected by an idempotency key (a unique id that lets the server recognize and ignore a repeated request).
- **A time limit is not a kill switch.** It interrupts; it cannot stop code that ignores interrupts. Set native client timeouts too.
- **Thread-bound state disappears.** With a time limit or a bulkhead the work runs on another thread: no `ThreadLocal`, no logged-in user from the security context, no transaction, no request-scoped objects (chapter 9).
- **A silent fallback hides outages.** Log inside `onError`.
- **A bulkhead name per request.** Each `define` of a new name creates a thread pool. Define bulkheads at startup, one per dependency, and remove those you create dynamically.
- **Redefining by accident.** `define` on an existing bulkhead name reconfigures the live one. Define in one place.
- **Reusing a `Try` across threads.** A `Try` is not thread-safe. Build it per call (it is cheap). Calling `get()` twice in sequence on one object is fine.
- **Sleeping on request threads.** The retry pause is a `Thread.sleep` on the caller. Few attempts and short delays on web threads; longer back-off belongs in a background job.

### C.2 Behaviour worth knowing

- Delays between retries are constant. There is no exponential back-off (a delay that doubles after every failure) and no jitter (a small random change to the delay so that many callers do not retry at the same instant).
- The time limit of a bulkhead call starts when the work starts; time spent waiting for a place is limited only by `maxWaitMillis`.
- Reducing a bulkhead's capacity with `reconfigure` never cancels work in progress; it only lowers the number of places available to new callers.
- Shutting a bulkhead down (`remove`) lets accepted work finish and rejects new calls.
- Policy objects are immutable records; a change is made by defining a new one.

### C.3 Limits

- At most **2000** helper threads serve time-limited calls without a bulkhead across the whole JVM. A request beyond that fails at once with `RejectedExecutionException` instead of queuing. It is treated like any other failure (retried, handled by `onError`).
- Policy names are plain strings; case matters.
- The registries are in-memory and per JVM. In a cluster each node has its own; define them from the same configuration on every node.

### C.4 Tuning

1. One name per dependency, used for retry, time limit and bulkhead alike.
2. Time limit a little above the dependency's normal worst case (its 99th percentile), never above what the user tolerates.
3. Attempts: 2 or 3 for interactive requests. Check the worst-case formula from chapter 8.
4. Bulkhead threads from calls per second × seconds per call; queue small; wait 100 to 200 ms.
5. Fallback that returns something useful and says it is approximate; log inside it.
6. Expose `reconfigure` to an operator so the numbers can be adjusted in production.

## Appendix D. Message reference

| Message | Exception | When |
|---------|-----------|------|
| `No work block has been set. Call doWork(Block) before get().` | `BlockNotInitializedException` | `get()` without `doWork` |
| `No retry policy named '<name>' has been defined` | `NotFoundException` | `retry("<name>")` or `RetryPolicies.get` with an unknown name |
| `No time limit named '<name>' has been defined` | `NotFoundException` | `timeLimit("<name>")` or `TimeLimitPolicies.get` with an unknown name |
| `No bulkhead named '<name>' has been defined` | `NotFoundException` | `bulkhead("<name>")`, `execute`, `reconfigure` or `currentConfig` with an unknown name |
| `java.util.concurrent.TimeoutException` | `TimeoutExecutionException` | the work exceeded its time limit (this is the message of the wrapped cause) |
| `Timed out waiting for a bulkhead permit` | `BulkheadRejectedExecutionException` | no place became free within `maxWaitMillis` |
| `Bulkhead was shut down` | `BulkheadRejectedExecutionException` | the bulkhead is removed while a call is being submitted |
| `The attempt was abandoned before it started` | `CancellationException` | an attempt is dropped before it begins to run (rare) |
| `maxAttempts must be >= 1` | `IllegalArgumentException` | `RetryPolicy` / `retry(...)` |
| `delayMillis must be >= 0` | `IllegalArgumentException` | `RetryPolicy` / `retry(...)` |
| `millis must be >= 0` | `IllegalArgumentException` | `TimeLimitPolicy` / `timeLimit(...)` |
| `maxConcurrentThreads must be >= 1` | `IllegalArgumentException` | `BulkheadPolicy` |
| `maxQueueSize must be >= 0` | `IllegalArgumentException` | `BulkheadPolicy` |
| `maxWaitMillis must be >= 0` | `IllegalArgumentException` | `BulkheadPolicy` |
| `policyName` | `NullPointerException` | `retry(String)`, `timeLimit(String)` or `bulkhead(String)` with `null` |
