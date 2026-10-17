# jclient

*J2OS Platform 2.4*

Most applications today talk to other services over HTTP: the shop asks the catalog for a product, the report job posts a file to the archive, the mobile backend calls the payment service. Writing that code by hand means building a URL, opening a connection, encoding a body, reading the answer and turning it back into objects, again and again, with small differences each time.

**jclient** (`org.j2os.platform.jclient`) removes that work. A remote service is described by a plain Java **interface**, with one line above each method saying which HTTP request it stands for. jclient turns the interface into a working client, so a remote call looks like an ordinary method call. It is a thin layer over OpenFeign, a well-known library that does the same job, and adds three things:

- **One short builder.** An address (or a name from the load balancer), two timeouts, and `bind(...)`. That is all the configuration there is.
- **Fixed, written-down rules.** How a request body is encoded and how an answer is decoded follow explicit rules (chapters 5 and 6) and never depend on a guess. A request that cannot be encoded is refused at once, instead of being sent in a wrong shape.
- **Load balancing built in.** Instead of one address, a client can use a service name registered in **jbalancer**, and every request goes to the next instance in turn.

The whole library in a few lines:

```java
public interface ShopApi {
    @RequestLine("GET /products/{id}")
    Product product(@Param("id") String id);          // one line = one remote call

    @RequestLine("POST /orders")
    @Headers("Content-Type: application/json")
    void order(Order order);                           // the object becomes the JSON body
}

ShopApi shop = JClient.builder()
        .url("https://shop.example.com")
        .connectTimeout(2000)
        .readTimeout(5000)
        .bind(ShopApi.class);

Product pen = shop.product("pen-1");                   // an ordinary method call
shop.order(new Order("pen-1", 2));
```

This guide goes step by step: the first client (chapter 2), the builder and its rules (3), describing requests (4), sending data (5), receiving data (6), timeouts (7), several instances with jbalancer (8), what happens when a call fails (9) and a complete small workflow (10). The appendices work as a reference.

## Contents

1. [How jclient works](#1-how-jclient-works)
2. [Your first client](#2-your-first-client)
3. [The builder and its rules](#3-the-builder-and-its-rules)
4. [Describing requests](#4-describing-requests)
5. [Sending data](#5-sending-data)
6. [Receiving data](#6-receiving-data)
7. [Timeouts](#7-timeouts)
8. [Several instances with jbalancer](#8-several-instances-with-jbalancer)
9. [When a call fails](#9-when-a-call-fails)
10. [Putting it together](#10-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The programs of the guide are complete: each has one `main` method and declares the interface it needs inside the class. They talk to a small web service of the guide's own on `localhost:8082` (and a twin on `8083` for chapter 8). Its endpoints, as the programs use them:

| Endpoint | Answer |
|----------|--------|
| `GET /getTree?NODE_PARENT_ID=1` | A JSON list of two categories; for any other parent, an empty list |
| `POST /saveTree` (form) | `{"saved":true,"parent":...}` |
| `GET /search?q=...&page=...` | The text `ok` |
| `GET /user/<id>` | A JSON user with an extra field the client does not know |
| `GET /text`, `GET /download/<id>` | The text `hello`; the bytes `DATA-<id>` |
| `GET /ping`, `POST /upload`, `POST /raw`, `POST /xml` | An empty answer |
| `GET /empty`, `GET /notjson` | Status 200 with no body; status 200 with text that is not JSON |
| `GET /slow` | Answers after 1.5 seconds |
| `GET /flaky`, `GET /count` | Always fails with status 503; how many times it was asked |
| `GET /broken`, `GET /missing` | Status 500; status 404 |
| `POST /orders`, `GET /orders` | Stores a JSON order; lists the stored orders |
| `GET /whoami` | The name of the server (`A` on 8082, `B` on 8083) |
| `GET /last` | A line describing the last request the service received: method and address, content type, body, and the `X-Token` header if there was one |

---

## 1. How jclient works

First, a few terms, because they come up throughout the guide:

- **Feign** is the library underneath. It reads an interface, builds a *proxy* (an object generated at run time that implements the interface) and turns each method call into an HTTP request.
- **`@RequestLine`** is the annotation that says which request a method stands for: the HTTP verb and the path, for example `GET /products/{id}`.
- **`@Param`** names a method argument so that it can fill a `{placeholder}` in the request line or in a header, or become a field of a form.
- **`@Headers`** adds fixed headers to a request. The one that matters most here is `Content-Type`.
- **Encoding and decoding.** *Encoding* is turning a Java object into the bytes of a request body; *decoding* is turning the bytes of an answer into a Java object. JSON is read and written with **Jackson**, the usual JSON library for Java.
- **jbalancer** is the J2OS library that keeps a list of addresses for a service name and hands them out in turn (*round robin*).

The classes a program meets:

| Class | Role |
|-------|------|
| `JClient` | The only door in: `JClient.builder()` starts a builder. It cannot be instantiated |
| `JClient.Builder` | Collects the target and the timeouts, then `bind(...)` |
| Your interface | Describes the remote service with `@RequestLine`, `@Param` and `@Headers` |

The module needs these libraries on the classpath: `io.github.openfeign:feign-core`, `feign-jackson`, `feign-form` and `feign-slf4j`, with Jackson itself, plus jbalancer from J2OS.

Keep these three points in mind:

- **A client is an object made from an interface.** There is nothing to implement. Anything an interface method cannot say (retry, caching, security) is up to the caller.
- **A few things are fixed and cannot be changed.** No retry, logging through slf4j at Feign's `BASIC` level, redirects followed, and the encoding and decoding rules of chapters 5 and 6.
- **The built client is shared freely, the builder is not.** The generated client follows Feign's own rules for use from many threads; a `Builder` is meant for one thread and one `bind`.

## 2. Your first client

A client takes three steps: describe the service, build the client, call it.

```java
import feign.Param;
import feign.RequestLine;
import org.j2os.platform.jclient.JClient;

import java.util.List;
import java.util.Map;

public class First {
    interface Api {
        @RequestLine("GET /getTree?NODE_PARENT_ID={parentId}")
        List<Map<String, Object>> children(@Param("parentId") String parentId);
    }

    public static void main(String[] args) {
        Api api = JClient.builder()
                .url("http://localhost:8082")
                .bind(Api.class);

        List<Map<String, Object>> children = api.children("1");
        System.out.println("1 children of 1: " + children);
        System.out.println("2 first name: " + children.get(0).get("name"));
        System.out.println("3 children of 2: " + api.children("2"));
    }
}
```

```text
1 children of 1: [{id=2, name=Books}, {id=3, name=Pens}]
2 first name: Books
3 children of 2: []
```

- **The interface** has one method, `children`. Its `@RequestLine` is the request: `GET`, the path `/getTree`, and a query `NODE_PARENT_ID={parentId}`. The argument marked `@Param("parentId")` fills the placeholder.
- **The builder** gets the base address with `url(...)` and builds the proxy with `bind(Api.class)`. The path of every request line is added to this base.
- **The call** `api.children("1")` is a real HTTP request. The answer, a JSON array, is decoded into `List<Map<String, Object>>`: one map per element.
- **Case 3** shows that a service which answers with an empty array gives an empty list.

When the service is written in Spring, each endpoint has a twin in the client. Keeping the two next to each other when writing the interface avoids most mistakes:

```java
// the service (Spring)
@GetMapping("/getTree")
public Object getCategory(@RequestParam(value = "NODE_PARENT_ID", required = false) String parentId) { ... }

@PostMapping("/orders")
public void createOrder(@RequestBody OrderDto order) { ... }

// the client (jclient): the same two endpoints
@RequestLine("GET /getTree?NODE_PARENT_ID={parentId}")
List<Map<String, Object>> getTree(@Param("parentId") String parentId);

@RequestLine("POST /orders")
@Headers("Content-Type: application/json")        // @RequestBody on the server means JSON here
void createOrder(OrderDto order);
```

The one detail to watch is the body. A Spring method with `@RequestParam` expects a **form**; with `@RequestBody` it expects **JSON**. Chapter 5 explains how the client picks.

Build the client once and keep it, as the next snippet does with a Spring bean; the proxy can be shared by all threads of the application:

```java
@Configuration
public class ClientConfig {
    @Bean
    public ShopApi shopApi() {                         // one client for the whole application
        return JClient.builder()
                .service("shop")
                .connectTimeout(2000)
                .readTimeout(5000)
                .bind(ShopApi.class);
    }
}
```

## 3. The builder and its rules

The builder has four methods besides `bind`:

| Method | Meaning |
|--------|---------|
| `url(address)` | A fixed base address. Must be absolute, `http` or `https`, with a host |
| `service(name)` | The name of a service registered in jbalancer (chapter 8) |
| `connectTimeout(ms)` | How long to wait for a connection to open. Optional |
| `readTimeout(ms)` | How long to wait for the answer. Optional |

```java
import feign.RequestLine;
import org.j2os.platform.jclient.JClient;

public class Build {
    interface Api {
        @RequestLine("GET /text")
        String text();
    }

    public static void main(String[] args) {
        System.out.println("1 plain url: " + JClient.builder().url("http://localhost:8082").bind(Api.class).text());
        JClient.builder().url("http://order_service:8080").bind(Api.class);
        System.out.println("2 a host with an underscore is accepted");

        try {
            JClient.builder().url("localhost:8082");
        } catch (Exception e) {
            System.out.println("3 no scheme -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            JClient.builder().url("  ");
        } catch (Exception e) {
            System.out.println("4 blank -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            JClient.builder().connectTimeout(2000).bind(Api.class);
        } catch (Exception e) {
            System.out.println("5 no target -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            JClient.builder().url("http://localhost:8082").service("shop").bind(Api.class);
        } catch (Exception e) {
            System.out.println("6 both targets -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
    }
}
```

```text
1 plain url: hello
2 a host with an underscore is accepted
3 no scheme -> IllegalArgumentException: url must start with http:// or https://: localhost:8082
4 blank -> IllegalArgumentException: url must not be blank
5 no target -> IllegalStateException: Either url(...) or service(...) must be specified
6 both targets -> IllegalStateException: url(...) and service(...) cannot both be set
```

- **Case 1 and 2.** The address is checked **at once**, when `url(...)` is called, not at the first request. A host name with an underscore, such as `order_service`, is accepted, because container platforms often use such names.
- **Case 3.** `localhost:8082` is not accepted: the scheme is required. The same holds for any scheme other than `http` and `https`, an address without a host and an address that cannot be parsed.
- **Case 4.** A `null` or blank address is refused.
- **Cases 5 and 6.** **Exactly one** of `url` and `service` must be set. This is checked at `bind(...)`, not before.
- `bind(null)` is refused too.

A builder is for **one client**. Calling `bind` twice on the same builder gives two independent clients with the same settings, which is allowed but rarely useful. Do not share a builder between threads.

## 4. Describing requests

Each method of the interface needs a `@RequestLine`. What goes in it:

```java
import feign.Headers;
import feign.Param;
import feign.RequestLine;
import org.j2os.platform.jclient.JClient;

public class Requests {
    interface Api {
        @RequestLine("GET /search?q={q}&page={page}")
        String search(@Param("q") String q, @Param("page") Integer page);

        @RequestLine("GET /user/{id}")
        String user(@Param("id") String id);

        @RequestLine("GET /search?q={q}")
        @Headers("X-Token: {token}")
        String secured(@Param("q") String q, @Param("token") String token);

        @RequestLine("GET /last")
        String last();
    }

    public static void main(String[] args) {
        Api api = JClient.builder().url("http://localhost:8082").bind(Api.class);

        api.search("pens", 2);
        System.out.println("1 " + api.last());
        api.search("red pen & ink", 1);
        System.out.println("2 " + api.last());
        api.search("pens", null);
        System.out.println("3 " + api.last());
        api.user("sara");
        System.out.println("4 " + api.last());
        api.secured("pens", "abc123");
        System.out.println("5 " + api.last());
    }
}
```

```text
1 GET /search?q=pens&page=2 | - | (no body)
2 GET /search?q=red%20pen%20%26%20ink&page=1 | - | (no body)
3 GET /search?q=pens | - | (no body)
4 GET /user/sara | - | (no body)
5 GET /search?q=pens | - | (no body) | X-Token: abc123
```

- **Case 1.** `{q}` and `{page}` are filled from the arguments named `@Param("q")` and `@Param("page")`. Numbers are written as text.
- **Case 2.** Values are **encoded** for the address: spaces become `%20` and `&` becomes `%26`, so a value can never break the query.
- **Case 3.** A placeholder whose value is `null` is dropped together with its parameter. That makes optional parameters easy: pass `null`.
- **Case 4.** A placeholder can sit in the path as well: `/user/{id}`.
- **Case 5.** A header can hold a placeholder too: `@Headers("X-Token: {token}")`. Fixed headers are written without placeholders. Use this for tokens that change per call.

Two more rules about arguments:

- An argument **without** `@Param` is the **body** of the request (chapter 5). At most one argument can be the body.
- A `@Param` argument whose name appears neither in the address nor in a header becomes a **field of a form** (chapter 5, the file upload).

Since `service(...)` chooses the destination, a request line should hold only a **path**, never a full address (chapter 8).

## 5. Sending data

When a method has a body, jclient decides how to write it by a short list of rules, in this order:

| The body is... | What happens |
|----------------|--------------|
| A `String` or a `byte[]` | Sent **as it is**, whatever `Content-Type` says |
| Anything else, and no `Content-Type` | Sent as a form, `application/x-www-form-urlencoded` |
| Anything else, `Content-Type: application/json` | Written as JSON |
| Anything else, `Content-Type: application/x-www-form-urlencoded` or `multipart/form-data` | Written as a form |
| Anything else, any other `Content-Type` | Refused with `IllegalArgumentException` |

```java
import feign.Headers;
import feign.Param;
import feign.RequestLine;
import org.j2os.platform.jclient.JClient;

import java.io.File;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.Map;

public class Sending {
    record Order(String productId, int quantity, List<String> tags) { }

    interface Api {
        @RequestLine("POST /saveTree")
        Object saveTree(Map<String, Object> fields);

        @RequestLine("POST /orders")
        @Headers("Content-Type: application/json")
        void createOrder(Order order);

        @RequestLine("POST /upload")
        @Headers("Content-Type: multipart/form-data")
        void upload(@Param("avatar") File avatar, @Param("name") String name);

        @RequestLine("POST /raw")
        @Headers("Content-Type: application/json")
        void rawJson(String json);

        @RequestLine("POST /xml")
        @Headers("Content-Type: application/xml")
        void xml(Map<String, Object> data);

        @RequestLine("GET /last")
        String last();
    }

    public static void main(String[] args) throws Exception {
        Api api = JClient.builder().url("http://localhost:8082").bind(Api.class);

        System.out.println("1 form, the default: " + api.saveTree(Map.of("NODE_PARENT_ID", "1")));
        System.out.println("  " + api.last());

        api.createOrder(new Order("SKU-123", 2, List.of("fragile", "express")));
        System.out.println("2 " + api.last());

        File avatar = Files.writeString(Path.of("avatar.png"), "12345").toFile();
        api.upload(avatar, "sara");
        System.out.println("3 " + api.last());

        api.rawJson("{\"already\":\"text\"}");
        System.out.println("4 a String is sent as it is: " + api.last());

        try {
            api.xml(Map.of("a", "b"));
        } catch (Exception e) {
            System.out.println("5 -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        avatar.delete();
    }
}
```

```text
1 form, the default: {saved=true, parent=1}
  POST /saveTree | application/x-www-form-urlencoded | NODE_PARENT_ID=1
2 POST /orders | application/json | {"productId":"SKU-123","quantity":2,"tags":["fragile","express"]}
3 POST /upload | multipart/form-data | avatar.png = file avatar.png (5 bytes); name = sara
4 a String is sent as it is: POST /raw | application/json | {"already":"text"}
5 -> IllegalArgumentException: Unsupported Content-Type in JClient: "application/xml" - only application/json, application/x-www-form-urlencoded, multipart/form-data, or no Content-Type (defaults to form) is supported
```

- **Case 1.** No `Content-Type` means a form, exactly what a Spring `@RequestParam` endpoint wants. A `Map` is the easiest body for a form: each entry becomes a field. The service answered with JSON, which came back as a map.
- **Case 2.** With `Content-Type: application/json`, the record `Order` is written as JSON. Records and ordinary classes both work.
- **Case 3.** With `multipart/form-data`, every `@Param` argument that is not in the address becomes a part: the `File` is sent as a file, the `String` as text.
- **Case 4.** A `String` is never encoded again, so it can carry JSON you wrote or got elsewhere. It is also the way to send any other text (XML, for example) by putting the text in a `String`:

```java
@RequestLine("POST /orders")
@Headers("Content-Type: application/json")
void createRaw(String json);                           // a String is sent exactly as written

shop.createRaw("{\"productId\":\"pen-1\",\"quantity\":2}");
```

- **Case 5.** Anything else is refused **before** anything is sent: the exception lists the types that are allowed. Only the part of the header before `;` counts, and the comparison is exact: `application/json; charset=UTF-8` is JSON, `application/json-fake` is not.

A short rule of thumb: the server's `@RequestParam` means a form (write no `Content-Type`), its `@RequestBody` means JSON (write `Content-Type: application/json`), and its `MultipartFile` means `multipart/form-data`.

## 6. Receiving data

The return type of the method decides how the answer is read:

| The method returns... | What you get |
|-----------------------|--------------|
| `void` or `Void` | Nothing. The body is ignored |
| `String` | The body as text, exactly as it came |
| `byte[]` | The body as bytes, exactly as it came |
| Anything else | The body decoded as JSON with Jackson |

```java
import feign.Param;
import feign.RequestLine;
import org.j2os.platform.jclient.JClient;

import java.util.List;
import java.util.Map;

public class Receiving {
    record User(int id, String name) { }

    interface Api {
        @RequestLine("GET /text")
        String text();

        @RequestLine("GET /download/{id}")
        byte[] download(@Param("id") String id);

        @RequestLine("GET /user/7")
        User user();

        @RequestLine("GET /getTree?NODE_PARENT_ID=1")
        List<Map<String, Object>> tree();

        @RequestLine("GET /ping")
        void ping();

        @RequestLine("GET /empty")
        List<String> empty();

        @RequestLine("GET /notjson")
        Map<String, Object> notJson();
    }

    public static void main(String[] args) {
        Api api = JClient.builder().url("http://localhost:8082").bind(Api.class);

        System.out.println("1 String as it came: " + api.text());
        System.out.println("2 byte[] as it came: " + new String(api.download("report.pdf")));
        System.out.println("3 a record, extra JSON field ignored: " + api.user());
        System.out.println("4 a list of maps: " + api.tree());
        api.ping();
        System.out.println("5 void: nothing to return");
        System.out.println("6 an empty body: " + api.empty());
        try {
            api.notJson();
        } catch (Exception e) {
            System.out.println("7 not JSON -> " + e.getClass().getSimpleName());
        }
    }
}
```

```text
1 String as it came: hello
2 byte[] as it came: DATA-report.pdf
3 a record, extra JSON field ignored: User[id=7, name=Sara]
4 a list of maps: [{id=2, name=Books}, {id=3, name=Pens}]
5 void: nothing to return
6 an empty body: null
7 not JSON -> DecodeException
```

- **Case 1 and 2.** `String` and `byte[]` are never touched by Jackson, so they suit text, HTML, files and images. `byte[]` is read completely into memory.
- **Case 3.** JSON fields that the Java type does not have (here `extra`) are ignored.
- **Case 4.** `Map`, `List` and `Object` give maps and lists. A typed record or class gives typed objects, and `List<Order>` gives a list of them.
- **Case 5.** A `void` method is the right choice when the answer holds nothing you need.
- **Case 6.** An **empty body** with a decoded type gives `null`, not an exception.
- **Case 7.** A body that is not valid JSON, or does not fit the type, throws `DecodeException` (a `FeignException`).

The answer is read in the character set the service names in its `Content-Type`, and a missing one is read as UTF-8.

## 7. Timeouts

Two waits can be limited:

- **`connectTimeout`**: how long to wait for the connection to open.
- **`readTimeout`**: how long to wait for the answer to arrive (also the longest silence while reading it).

```java
import feign.RequestLine;
import org.j2os.platform.jclient.JClient;

public class Timeouts {
    interface Api {
        @RequestLine("GET /slow")
        String slow();
    }

    public static void main(String[] args) {
        Api patient = JClient.builder().url("http://localhost:8082").readTimeout(5000).bind(Api.class);
        System.out.println("1 the service needs 1.5 s, we wait up to 5 s: " + patient.slow());

        Api hasty = JClient.builder().url("http://localhost:8082").readTimeout(300).bind(Api.class);
        try {
            hasty.slow();
        } catch (Exception e) {
            System.out.println("2 we wait only 0.3 s -> " + e.getClass().getSimpleName() + ", cause " + e.getCause().getClass().getSimpleName());
        }

        Api nobody = JClient.builder().url("http://localhost:9").connectTimeout(1000).bind(Api.class);
        try {
            nobody.slow();
        } catch (Exception e) {
            System.out.println("3 nobody listens there -> " + e.getClass().getSimpleName() + ", cause " + e.getCause().getClass().getSimpleName());
        }
        try {
            JClient.builder().readTimeout(-1);
        } catch (Exception e) {
            System.out.println("4 negative -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
    }
}
```

```text
1 the service needs 1.5 s, we wait up to 5 s: late but fine
2 we wait only 0.3 s -> RetryableException, cause SocketTimeoutException
3 nobody listens there -> RetryableException, cause ConnectException
4 negative -> IllegalArgumentException: readTimeout must not be negative
```

- **Case 1 and 2.** A service that needs 1.5 seconds is fine with a 5 second limit and fails with a 0.3 second limit.
- **Case 2 and 3.** A timeout and an unreachable address both throw `RetryableException`; the `cause` tells which one it was (`SocketTimeoutException` or `ConnectException`).
- **Case 4.** A negative value is refused. `0` means no limit, which lets a dead service hold a thread forever; avoid it.
- **If a timeout is not set**, Feign's own default applies (10 seconds to connect, 60 seconds to read in current Feign versions).

Good values depend on the service: a few seconds to connect, and for reading slightly more than the slowest normal answer. A client that serves web requests should wait less than its own caller does.

## 8. Several instances with jbalancer

When a service runs on several machines, `service("name")` replaces `url(...)`. The addresses come from jbalancer:

```java
JRoundRobinBalancer.getInstance().configurationResource("shop", List.of(
        "http://10.0.0.11:8080",                       // base addresses; the @RequestLine path is added to them
        "http://10.0.0.12:8080"));

ShopApi shop = JClient.builder().service("shop").bind(ShopApi.class);
```

```java
import feign.RequestLine;
import org.j2os.platform.jbalancer.JRoundRobinBalancer;
import org.j2os.platform.jclient.JClient;

import java.util.List;

public class Balanced {
    interface Api {
        @RequestLine("GET /whoami")
        String whoami();
    }

    static String ask(Api api, int times) {
        StringBuilder out = new StringBuilder();
        for (int i = 0; i < times; i++) {
            try {
                out.append(api.whoami()).append(' ');
            } catch (Exception e) {
                out.append(e.getClass().getSimpleName()).append(' ');
            }
        }
        return out.toString().trim();
    }

    public static void main(String[] args) {
        JRoundRobinBalancer balancer = JRoundRobinBalancer.getInstance();
        balancer.configurationResource("shop", List.of("http://localhost:8082", "http://localhost:8083"));
        Api api = JClient.builder().service("shop").bind(Api.class);

        System.out.println("1 four calls take turns: " + ask(api, 4));

        balancer.configurationResource("shop", List.of("http://localhost:8083"));
        System.out.println("2 the list changed, the same client follows: " + ask(api, 2));

        balancer.configurationResource("shop", List.of("http://localhost:8082", "http://localhost:9"));
        System.out.println("3 a dead instance is not skipped: " + ask(api, 4));

        balancer.removeResource("shop");
        try {
            api.whoami();
        } catch (Exception e) {
            System.out.println("4 not registered -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
    }
}
```

```text
1 four calls take turns: A B A B
2 the list changed, the same client follows: B B
3 a dead instance is not skipped: A RetryableException A RetryableException
4 not registered -> IllegalStateException: Service 'shop' is not registered in jbalancer
```

- **Case 1.** The addresses are used in turn. The destination is chosen **for every request**, not once when the client is built.
- **Case 2.** Because of that, replacing the list with `configurationResource` takes effect for the existing client at once. (Registering a name again also restarts the rotation from the first address.)
- **Case 3.** **There is no failover.** A dead instance is not skipped: the request that lands on it fails, and the next one goes to the next address. Remove instances that are known to be down.
- **Case 4.** A name that is not registered fails **at the call**, not when the client is built, with `IllegalStateException`.
- Names are case-insensitive: `"shop"`, `"Shop"` and `"SHOP"` are the same service.
- The addresses in jbalancer are **base** addresses. The path of the request line is added to them, and whatever the method's own address says is replaced.
- `removeResource` is silent when the name is not there.

## 9. When a call fails

jclient never retries and never hides an error, so every failure reaches the caller as an exception, all of them unchecked. There are four kinds:

| Failure | Exception |
|---------|-----------|
| The service answered with an error status (`4xx`, `5xx`) | `FeignException`, with `status()` |
| The request never got an answer (connection refused, timeout, network down) | `RetryableException` (a `FeignException`), with the cause |
| The answer could not be decoded | `DecodeException` (a `FeignException`) |
| A request the client refuses to send, or a missing target | `IllegalArgumentException` or `IllegalStateException` from jclient |

```java
import feign.FeignException;
import feign.RequestLine;
import org.j2os.platform.jclient.JClient;

public class Failures {
    interface Api {
        @RequestLine("GET /missing")
        String missing();

        @RequestLine("GET /broken")
        String broken();

        @RequestLine("GET /flaky")
        String flaky();

        @RequestLine("GET /count")
        String count();
    }

    public static void main(String[] args) {
        Api api = JClient.builder().url("http://localhost:8082").bind(Api.class);

        try {
            api.missing();
        } catch (FeignException e) {
            System.out.println("1 the service answered 404 -> status " + e.status());
        }
        try {
            api.broken();
        } catch (FeignException e) {
            System.out.println("2 the service answered 500 -> status " + e.status());
        }
        try {
            api.flaky();
        } catch (FeignException e) {
            System.out.println("3 status " + e.status() + ", and the service saw " + api.count() + " request");
        }
    }
}
```

```text
1 the service answered 404 -> status 404
2 the service answered 500 -> status 500
3 status 503, and the service saw 1 request
```

- **Cases 1 and 2.** Any status outside 200 to 299 is an exception; `status()` has the number and the message has the answer's body.
- **Case 3.** **A call is made once.** The service answered 503 and saw exactly one request. Nothing tried again.

Two habits make calls robust. First, treat "not found" as an answer when that is what it means:

```java
static Product findOrNull(ShopApi shop, String id) {
    try {
        return shop.product(id);
    } catch (FeignException e) {
        if (e.status() == 404) return null;            // "not there" is an answer, not a failure
        throw e;
    }
}
```

Second, repeat only calls that are safe to repeat, and only for failures where no answer arrived:

```java
static <T> T twice(Supplier<T> call) {
    try {
        return call.get();
    } catch (RetryableException lost) {                // the request never got an answer
        return call.get();                             // one more try - only for calls that are safe to repeat
    }
}

Product pen = twice(() -> shop.product("pen-1"));      // a GET can be repeated; a POST /orders could create two orders
```

A `GET` can be asked twice with no harm; a `POST` that creates something cannot, because the first request may have been executed even though the answer was lost. For serious needs (several tries with delays, a circuit breaker), use a resilience library around the client.

To see what the client sends, turn on logging. Each client logs through slf4j under the **name of its interface**, at `DEBUG` level, with the request line and the status:

```xml
<!-- logback.xml: the log of a client is named after its interface -->
<logger name="com.example.shop.ShopApi" level="DEBUG"/>
```

## 10. Putting it together

An order client with three operations, built with timeouts. Orders go out as JSON and come back as typed objects; a missing order is told apart from a failure.

```java
import feign.FeignException;
import feign.Headers;
import feign.Param;
import feign.RequestLine;
import org.j2os.platform.jclient.JClient;

import java.util.List;

public class Workflow {
    record Order(String productId, int quantity, List<String> tags) { }

    interface OrderApi {
        @RequestLine("POST /orders")
        @Headers("Content-Type: application/json")
        void create(Order order);

        @RequestLine("GET /orders")
        List<Order> all();

        @RequestLine("GET /orders/{id}")
        Order one(@Param("id") String id);
    }

    public static void main(String[] args) {
        OrderApi orders = JClient.builder()
                .url("http://localhost:8082")
                .connectTimeout(2000)
                .readTimeout(5000)
                .bind(OrderApi.class);

        orders.create(new Order("SKU-1", 2, List.of("express")));
        orders.create(new Order("SKU-2", 1, List.of()));

        for (Order order : orders.all()) {
            System.out.println("1 " + order.productId() + " x" + order.quantity() + " " + order.tags());
        }
        try {
            orders.one("9");
        } catch (FeignException e) {
            System.out.println("2 order 9 -> status " + e.status());
        }
    }
}
```

```text
1 SKU-1 x2 [express]
1 SKU-2 x1 []
2 order 9 -> status 404
```

- **The interface** is the whole client: three methods, three lines.
- **`create`** returns `void`: the service's answer is not needed.
- **`all`** returns `List<Order>`: Jackson builds one `Order` per element, including the empty `tags` list.
- **`one("9")`** meets a 404, which arrives as a `FeignException` with `status()` 404.

## Appendix A. Quick reference

### A.1 JClient and the builder

| Call | Purpose |
|------|---------|
| `JClient.builder()` | A new builder. Each call gives a fresh one |
| `url(address)` | Fixed base address. Absolute, `http` or `https`, with a host. Checked at once |
| `service(name)` | Name in jbalancer. Looked up for every request |
| `connectTimeout(ms)`, `readTimeout(ms)` | Optional. Not negative; `0` means no limit |
| `bind(Interface.class)` | Builds the client. Exactly one of `url` and `service` must be set |

### A.2 Annotations of the interface

| Annotation | Use |
|------------|-----|
| `@RequestLine("VERB /path?x={x}")` | The request a method stands for. Required |
| `@Param("name")` | Fills `{name}` in the address or a header; otherwise a field of a form |
| `@Headers("Name: value")` | Adds a header. A value may hold a `{placeholder}` |
| An argument without `@Param` | The body of the request |

### A.3 Body rules

| Body | Result |
|------|--------|
| `String`, `byte[]` | Raw, always |
| No `Content-Type` | Form (`application/x-www-form-urlencoded`) |
| `application/json` | JSON by Jackson |
| `application/x-www-form-urlencoded`, `multipart/form-data` | Form |
| Anything else | `IllegalArgumentException` |

### A.4 Answer rules

| Return type | Result |
|-------------|--------|
| `void`, `Void` | `null` |
| `String` | Raw text |
| `byte[]` | Raw bytes |
| Other | JSON by Jackson |

### A.5 jbalancer calls used with jclient

| Call | Purpose |
|------|---------|
| `JRoundRobinBalancer.getInstance()` | The one shared registry |
| `configurationResource(name, urls)` | Registers or replaces a service. The rotation starts again |
| `removeResource(name)` | Removes a service. Silent if missing |
| `getResourceUrl(name)` | The next address (called for you by jclient) |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| `url(...)` null, blank, unparsable, not `http`/`https`, or without a host | `IllegalArgumentException` at `url` |
| `service(...)` null or blank | `IllegalArgumentException` at `service` |
| A negative timeout | `IllegalArgumentException` at the setter |
| `bind(null)` | `IllegalArgumentException` |
| Neither `url` nor `service`, or both | `IllegalStateException` at `bind` |
| A `Content-Type` other than JSON, form or multipart, with an object body | `IllegalArgumentException` at the call; nothing is sent |
| The service answers with an error status | `FeignException` with the status |
| Connection refused, host unknown, network down | `RetryableException` with the cause; one attempt only |
| A timeout | `RetryableException` with `SocketTimeoutException` as the cause |
| The answer is not valid for the return type | `DecodeException` |
| An empty body for a decoded type | `null` |
| `service(...)` with an unregistered name | `IllegalStateException` at the call |
| An instance of the service is down | The requests that land on it fail; no failover |
| A redirect | Followed |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **A body where the server wants a form, or the reverse.** `@RequestParam` means a form (no `Content-Type`), `@RequestBody` means JSON.
- **A `String` body with the wrong content type.** A `String` is never re-encoded; make sure it is already in the form the header says.
- **An object body with `Content-Type: application/xml` or `text/plain`.** Refused. Put the text in a `String`.
- **Forgetting `@Param`.** An argument without it is the body, not a placeholder.
- **Two body arguments.** Only one argument can be the body.
- **A full address in `@RequestLine` together with `service(...)`.** The destination always comes from jbalancer; write only the path.
- **Expecting retries.** There are none. A failed call is a failed call.
- **Retrying a `POST`.** The first request may have been done.
- **Timeouts left at `0`.** A dead service can hold a thread forever.
- **Unregistered service names.** They fail at the first call, not at `bind`.
- **A dead instance in a balanced list.** Its share of the requests fails.
- **Sharing one builder between threads.** Not safe. Share the built client instead.
- **Reading huge answers as `byte[]` or `String`.** They are read completely into memory.
- **Catching only `FeignException`.** A call can also throw `IllegalArgumentException` or `IllegalStateException`.
- **Putting secrets in the address.** URLs end up in logs; use a header.

### C.2 Behavior worth knowing

- The client is generated from the interface at `bind` time; nothing is sent until a method is called.
- Every request is sent once. A redirect is followed; a failure is not retried.
- Logging goes through slf4j under the interface's name; with `DEBUG` off, nothing is written.
- With `service(...)`, the destination is chosen for each request, and it replaces whatever the method's own address says.
- An argument that is `null` and fills a query placeholder is dropped with its parameter.
- JSON fields that the target type does not know are ignored when reading.
- Jackson reads and writes records and ordinary classes alike.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Retries | None, fixed |
| Redirects | Followed, fixed |
| Request body types | `String`, `byte[]`, JSON, form, multipart |
| Answer types | `void`, `String`, `byte[]`, anything Jackson can read |
| Connect and read timeout defaults | Feign's own (10 s and 60 s in current versions) |
| Targets | One of `url` or `service` per client |
| Failover | None |
| Answer size | Held in memory |

### C.4 Tuning and practical advice

- Build one client per remote service, at start-up, and share it.
- Set both timeouts for every client, a few seconds for connecting and a little more than the slowest normal answer for reading.
- Keep interfaces small and close to the service's own endpoint list. When the service changes, change the line.
- Use records for requests and answers; they make the shape of the data visible.
- Return `void` when the answer is not needed, and `String` or `byte[]` when you want the raw answer.
- Put retry and circuit breaking in one place around the client, not in every caller.
- Register service names in jbalancer once at start-up, and remove an address from the list when its machine is known to be down.
- Turn `DEBUG` on for one interface when diagnosing, and off again afterwards.
- Test clients against a small local service, as this guide does.

## Appendix D. Message reference

### D.1 jclient

| Message | Exception | Cause |
|---------|-----------|-------|
| `url must not be blank` | `IllegalArgumentException` | `url` with `null` or blank text |
| `url is invalid: <url>` | `IllegalArgumentException` | The text cannot be parsed as an address |
| `url must start with http:// or https://: <url>` | `IllegalArgumentException` | No scheme, or another scheme |
| `url must have a valid host: <url>` | `IllegalArgumentException` | No usable host |
| `service must not be blank` | `IllegalArgumentException` | `service` with `null` or blank text |
| `connectTimeout must not be negative` | `IllegalArgumentException` | A negative connect timeout |
| `readTimeout must not be negative` | `IllegalArgumentException` | A negative read timeout |
| `apiType must not be null` | `IllegalArgumentException` | `bind(null)` |
| `Either url(...) or service(...) must be specified` | `IllegalStateException` | `bind` with no target |
| `url(...) and service(...) cannot both be set` | `IllegalStateException` | `bind` with two targets |
| `Service '<name>' is not registered in jbalancer` | `IllegalStateException` | A call on a client whose service is not registered |
| `Unsupported Content-Type in JClient: "<value>" - only application/json, application/x-www-form-urlencoded, multipart/form-data, or no Content-Type (defaults to form) is supported` | `IllegalArgumentException` | An object body with another content type |

### D.2 jbalancer

| Message | Exception | Cause |
|---------|-----------|-------|
| `resourceId must not be null` | `NullPointerException` | A `null` name |
| `urls must not be null or empty` | `IllegalArgumentException` | `configurationResource` with no addresses |
| The name in capital letters | `ResourceNotFoundException` | `getResourceUrl` for an unknown name (jclient turns it into `IllegalStateException`) |

### D.3 Feign

| Exception | Meaning |
|-----------|---------|
| `FeignException` | The service answered with an error status; `status()` is the number |
| `RetryableException` | No answer arrived; the cause is the network exception |
| `DecodeException` | The answer could not be turned into the return type |
