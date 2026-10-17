# jflow

*J2OS Platform 2.4*

Many business processes are not a single action but a path with waiting in it: a leave request that needs the manager, then HR, then payroll; a purchase that goes to a clerk or a director depending on the amount; an order that waits for a payment confirmation. Writing that path as `if` statements and status columns works for a week and then becomes the part of the program nobody dares to touch. The alternative is to draw the path once, as a **process definition**, and let an engine remember where every request currently stands.

**jflow** (`org.j2os.platform.jflow`) puts such an engine, Flowable, behind a small remote interface. Flowable reads processes written in BPMN, the standard XML notation for business processes. jflow publishes the engine over **RMI**, Java's built-in remote-call mechanism, so that any Java program, a web application, a desktop tool, a batch job, can start processes, find the tasks waiting for people and complete them, without embedding the engine itself. Three ideas shape it:

- **One server, many thin clients.** The engine, its database and the code that runs automatic steps live in one place. Clients only hold a handle and a token.
- **Everything is a string, a map or a small descriptor.** Processes are identified by keys, instances by ids, tasks by `JFlowTask` objects. No Flowable types cross the wire.
- **One shared token.** Each call carries a server-wide secret. There are no users or roles inside jflow; deciding who may do what stays with the application.

The whole idea in a few lines:

```java
JFlowClient client = new JFlowClient("rmi://flow-host:1099/jflow", token);

client.deployFromFileAndGetProcessDefinitionKey("/opt/flows/leave-request.bpmn20.xml");

String id = client.startProcessAndGetProcessInstanceId("leaveRequest", Map.of("days", 3));

JFlowTask task = client.getOpenTasksByProcessInstanceId(id).get(0);   // waiting for the manager
client.signalByTaskId(task.getTaskId());                              // the manager approves
```

This guide goes step by step: running a server (chapter 2), connecting (3), deploying definitions (4), starting instances (5), working with tasks (6), automatic steps and waiting (7), moving instances (8), variables (9), suspending (10), diagrams (11), tokens and filters (12), failures (13) and one complete example (14). The appendices are reference material.

## Contents

1. [How jflow works](#1-how-jflow-works)
2. [Starting a server](#2-starting-a-server)
3. [Connecting a client](#3-connecting-a-client)
4. [Deploying process definitions](#4-deploying-process-definitions)
5. [Starting an instance](#5-starting-an-instance)
6. [Working with tasks](#6-working-with-tasks)
7. [Automatic steps and waiting](#7-automatic-steps-and-waiting)
8. [Moving an instance](#8-moving-an-instance)
9. [Variables](#9-variables)
10. [Suspending and activating a definition](#10-suspending-and-activating-a-definition)
11. [Diagrams](#11-diagrams)
12. [Tokens, filters and who may call](#12-tokens-filters-and-who-may-call)
13. [When calls fail](#13-when-calls-fail)
14. [Putting it together: leave requests](#14-putting-it-together-leave-requests)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The example programs are complete: each has a `main` method. Except for `Setup`, `Connect` and `Security`, which start a server themselves, they are clients of the small server program shown in chapter 2. In the printed output, process ids are shown as `<id>`, and where a failure message comes from the engine itself and differs between engine versions, only the word `failed` is printed. A failure message that comes from the server starts with the name of an exception class, as the errors chapter explains.

---

## 1. How jflow works

First the vocabulary, which comes back all the time.

- A **process definition** is the drawing: the steps and the arrows between them, written in a BPMN file. Its **key** is the `id` of the `<process>` element inside the file, for example `leaveRequest`. The file name does not matter.
- Deploying the same key again creates a new **version** (1, 2, 3...). New instances always start on the latest version; instances already running stay on the version they started with.
- A **process instance** is one run of a definition: one concrete leave request. It has an **id**, and it carries **variables**, named values such as `days = 3`.
- An **activity** is one step of the drawing. The ones jflow deals with:
  - a **user task**: a step that waits for a person. It becomes an open **task** that somebody must complete.
  - a **service task**: an automatic step that runs a Java class on the server (a *delegate*) and moves on at once.
  - a **receive task**: a step that waits for an outside signal, not for a person.
  - a **gateway**: a fork. The path taken depends on the variables.
  - **start** and **end** events. When an instance reaches the end event it is finished, and the engine no longer keeps it among the running instances.
- An **assignee** is the name stored on a user task. jflow treats it as plain text and its queries match it exactly.
- **RMI** lets a client call methods of an object living in another JVM. The server publishes its object under a name in an **RMI registry** (port 1099 by default); the client looks it up by URL such as `rmi://host:1099/jflow` and gets a **stub**, an object that forwards calls over the network.

The classes a program meets:

| Class | Role |
|-------|------|
| `JFlowServer` | Wraps a Flowable engine and exposes it over RMI. Created once, on the server |
| `JFlowClient` | Looks the server up and remembers the token. One method per operation |
| `JFlowRemote` | The interface the two share. `JFlowClient` adds the token to every call |
| `JFlowTask` | A small serializable description of one open task |

Keep these three points in mind:

- **Instances end by disappearing.** A finished instance can no longer be read, signalled or drawn. Calls for it fail with *"No process found with this id (it may have already completed)"*.
- **Steps run inside the call that reached them.** `startProcess...` and `signalByTaskId` return when the instance has reached its next waiting point (or its end). Every automatic step in between has already run.
- **The server needs to see the Java classes.** The delegates named in a BPMN file are loaded by the engine, on the server. Clients never need them.

## 2. Starting a server

A server needs an engine configuration, a token, and a place in the RMI registry. The production shape, with the database credentials and the token coming from the environment:

```java
public static void main(String[] args) throws Exception {
    StandaloneProcessEngineConfiguration config = new StandaloneProcessEngineConfiguration();
    config.setJdbcUrl("jdbc:postgresql://localhost:5432/flowable");
    config.setJdbcDriver("org.postgresql.Driver");
    config.setJdbcUsername(System.getenv("FLOW_DB_USER"));
    config.setJdbcPassword(System.getenv("FLOW_DB_PASSWORD"));
    config.setDatabaseSchemaUpdate(ProcessEngineConfiguration.DB_SCHEMA_UPDATE_TRUE);
    config.setAsyncExecutorActivate(false);
    config.setIdGenerator(new StrongUuidGenerator());          // ids are handed to clients: make them unguessable

    JFlowServer server = new JFlowServer(
            System.getenv("FLOW_TOKEN"),                       // from the environment, never from the source
            config,
            "java.lang.*;java.util.*;!*");                     // only plain values may arrive as variables

    LocateRegistry.createRegistry(1099);
    Naming.rebind("rmi://localhost:1099/jflow", server);
}
```

Line by line:

- **The configuration** is a normal Flowable `ProcessEngineConfiguration`. `DB_SCHEMA_UPDATE_TRUE` lets the engine create its tables on first start. `setAsyncExecutorActivate(false)` turns the background job executor off; the work described in this guide all happens inside calls, but jobs such as timers and `async` steps are executed by that background component, so they would not run on a server configured this way.
- **`StrongUuidGenerator`** gives instances and tasks random ids. Because clients receive these ids and use them as handles, they should not be guessable (the default generator counts upward).
- **The token** must not be `null`. An empty string is accepted; then every client that sends an empty token gets in, so use it only when no protection is wanted.
- **The third argument**, the serial filter, is optional and covered in chapter 12.
- **`Naming.rebind`** publishes the server. Once it has run, clients can connect.

There is also a constructor that takes an already built `ProcessEngine` instead of a configuration, for programs that own the engine themselves (for example a Spring application that already created one). Everything else is identical. `getProcessEngine()` returns the engine, should the program need it.

**The server used by the examples.** The client programs in this guide need a server to talk to. This one keeps everything in memory, so each run starts empty, and it uses the token `demo-token`:

```java
import org.flowable.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration;
import org.j2os.platform.jflow.server.JFlowServer;

import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;

public class Server {
    public static void main(String[] args) throws Exception {
        JFlowServer server = new JFlowServer("demo-token", new StandaloneInMemProcessEngineConfiguration());

        LocateRegistry.createRegistry(1099);
        Naming.rebind("rmi://localhost:1099/jflow", server);
        System.out.println("jflow server is ready");
    }
}
```

Start it first and leave it running; start a fresh one before each example so that the examples do not see each other's instances. The three programs that deal with the server itself (`Setup`, `Connect` and `Security`) start their own server inside, so run them while this one is stopped.

The sample process files are in a folder called `processes`, named `<file>.bpmn20.xml`, and the programs read them with `Files.readString(...)`. They are described in the table below.

| File | Key | Shape |
|------|-----|-------|
| `leave-request` | `leaveRequest` | user task for `manager`, then user task for `hr` |
| `leave-automation` | `leaveAutomation` | two service tasks, then a receive task |
| `purchase` | `purchase` | gateway: amount above 1000 goes to `director`, otherwise to a page-assigned clerk task |
| `diagram` | `diagramSample` | one user task, with diagram layout |
| `no-layout` | `noLayout` | one user task, without diagram layout |

What the constructor refuses:

```java
import org.flowable.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration;
import org.j2os.platform.jflow.server.JFlowServer;

public class Setup {
    public static void main(String[] args) throws Exception {
        try {
            new JFlowServer(null, new StandaloneInMemProcessEngineConfiguration());
        } catch (Exception e) {
            System.out.println("1 null token -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            new JFlowServer("t", new StandaloneInMemProcessEngineConfiguration(), " ");
        } catch (Exception e) {
            System.out.println("2 blank filter -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            new JFlowServer("t", new StandaloneInMemProcessEngineConfiguration(), "maxdepth=abc");
        } catch (Exception e) {
            System.out.println("3 broken filter -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }

        JFlowServer server = new JFlowServer("", new StandaloneInMemProcessEngineConfiguration());
        System.out.println("4 empty token is allowed, engine ready: " + (server.getProcessEngine() != null));

        server.close();
        server.close();
        System.out.println("5 closed twice without trouble");
    }
}
```

```text
1 null token -> IllegalArgumentException: token is null (use an empty string "" to run without a token)
2 blank filter -> IllegalArgumentException: serialFilter must be a valid, non-blank pattern:  
3 broken filter -> NumberFormatException: For input string: "abc"
4 empty token is allowed, engine ready: true
5 closed twice without trouble
```

- Cases 1 to 3: a `null` token, a blank filter and a pattern the JDK cannot read are all `IllegalArgumentException` (case 3 is a subclass of it). Each of these also unexports the half-built server, so a failed construction leaves nothing behind. The filter is validated before the engine is built, so a bad pattern costs no engine.
- Case 4: an empty token is a valid token.
- Case 5: `close()` unexports the RMI object and closes the engine, releasing its connections. Calling it twice is harmless. A server stays alive until it is closed or the JVM ends, so close it on shutdown:

```java
Runtime.getRuntime().addShutdownHook(new Thread(server::close));   // unexport, then close the engine
```

## 3. Connecting a client

`new JFlowClient(url, token)` looks the server up once and keeps the stub and the token. From then on every method adds the token and forwards the call. Only the lookup can fail in the constructor; wrong tokens and a dead server show up on the first operation.

```java
import org.flowable.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration;
import org.j2os.platform.jflow.client.JFlowClient;
import org.j2os.platform.jflow.server.JFlowServer;

import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;

public class Connect {
    public static void main(String[] args) throws Exception {
        String url = "rmi://localhost:1099/jflow";
        JFlowServer server = new JFlowServer("demo-token", new StandaloneInMemProcessEngineConfiguration());
        LocateRegistry.createRegistry(1099);
        Naming.rebind(url, server);

        JFlowClient client = new JFlowClient(url, "demo-token");
        System.out.println("1 active definitions: " + client.getActiveProcessDefinitionKeys());

        try {
            new JFlowClient("rmi://localhost:1099/other", "demo-token");
        } catch (Exception e) {
            System.out.println("2 name nobody published -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            new JFlowClient("rmi://localhost:1098/jflow", "demo-token");
        } catch (Exception e) {
            System.out.println("3 nothing listens there -> " + e.getClass().getSimpleName());
        }
        try {
            new JFlowClient(url, "nope").getActiveProcessDefinitionKeys();
        } catch (Exception e) {
            System.out.println("4 wrong token -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        try {
            new JFlowClient(url, null).getActiveProcessDefinitionKeys();
        } catch (Exception e) {
            System.out.println("5 null token -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }

        Naming.unbind(url);
        server.close();
        try {
            client.getActiveProcessDefinitionKeys();
        } catch (Exception e) {
            System.out.println("6 server shut down -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
    }
}
```

```text
1 active definitions: []
2 name nobody published -> NotBoundException: other
3 nothing listens there -> ConnectException
4 wrong token -> Exception: Invalid token, access denied.
5 null token -> Exception: Invalid token, access denied.
6 server shut down -> NoSuchObjectException: no such object in table
```

- Case 1: a client for the right URL and token works at once.
- Case 2: a name nobody published is a `NotBoundException`.
- Case 3: a registry that is not there is a `ConnectException`.
- Cases 4 and 5: a wrong token is refused with a plain `Exception` whose message is *"Invalid token, access denied."* A `null` token is refused the same way on every call; pass `""` for a server that was created with an empty token.
- Case 6: after the server is closed, the client's stub is dead and calls fail with `NoSuchObjectException`. A restarted server is a new object, so the old client does not come back to life: create a new `JFlowClient`.

Because the client only holds a stub and a token, one instance can be shared by the whole application (a single Spring bean is enough), and a lost server is handled by looking it up again:

```java
@Configuration
public class FlowConfig {

    @Bean
    JFlowClient flowClient(@Value("${flow.url}") String url, @Value("${flow.token}") String token) throws Exception {
        return new JFlowClient(url, token);       // one client is enough: it holds one stub and one token
    }
}
```

```java
try {
    client.getAllOpenTasks();
} catch (java.rmi.RemoteException lost) {
    client = new JFlowClient(url, token);      // look the server up again, then retry once
    client.getAllOpenTasks();
}
```

`getRemote()` returns the stub itself for programs that want to call `JFlowRemote` directly. Then the token is the first argument of every call.

## 4. Deploying process definitions

A definition becomes available to clients when it is **deployed**. There are two ways:

| Method | Reads from |
|--------|------------|
| `deployFromSourceAndGetProcessDefinitionKey(name, xml)` | The BPMN text the client sends |
| `deployFromFileAndGetProcessDefinitionKey(path)` | A file on the **server's** disk |

Both return the key. `name` is only the name the XML is stored under; if it does not end in `.bpmn20.xml` or `.bpmn`, the suffix `.bpmn20.xml` is added, which the engine needs to recognise the resource as BPMN. Note that `deployFromFile...` reads a path on the *server*, not on the client.

```java
import org.j2os.platform.jflow.client.JFlowClient;

import java.nio.file.Files;
import java.nio.file.Path;

public class Deploy {
    public static void main(String[] args) throws Exception {
        JFlowClient client = new JFlowClient("rmi://localhost:1099/jflow", "demo-token");

        String key = client.deployFromSourceAndGetProcessDefinitionKey("anything", Files.readString(Path.of("processes/leave-request.bpmn20.xml")));
        System.out.println("1 deployed from text, key = " + key);

        String again = client.deployFromFileAndGetProcessDefinitionKey("processes/leave-request.bpmn20.xml");
        System.out.println("2 deployed from a server file, key = " + again);
        System.out.println("3 active: " + client.getActiveProcessDefinitionKeys());

        client.deployFromSourceAndGetProcessDefinitionKey("p.bpmn20.xml", Files.readString(Path.of("processes/purchase.bpmn20.xml")));
        System.out.println("4 active: " + client.getActiveProcessDefinitionKeys());

        try {
            client.deployFromFileAndGetProcessDefinitionKey("processes/nope.xml");
        } catch (Exception e) {
            System.out.println("5 missing file -> " + e.getMessage());
        }
        try {
            client.deployFromSourceAndGetProcessDefinitionKey("bad", "<hello/>");
        } catch (Exception e) {
            System.out.println("6 not BPMN -> failed");
        }
        System.out.println("7 still active: " + client.getActiveProcessDefinitionKeys());
    }
}
```

```text
1 deployed from text, key = leaveRequest
2 deployed from a server file, key = leaveRequest
3 active: [leaveRequest:1, leaveRequest:2]
4 active: [leaveRequest:1, purchase:1, leaveRequest:2]
5 missing file -> class java.nio.file.NoSuchFileException:processes/nope.xml
6 not BPMN -> failed
7 still active: [leaveRequest:1, purchase:1, leaveRequest:2]
```

- Case 1: the key is the process id written inside the XML. The name `"anything"` played no part.
- Cases 2 and 3: deploying the same definition again is not an error. It becomes version 2, and both versions are active.
- Case 4: `getActiveProcessDefinitionKeys()` returns `key:version` strings ordered by version; among definitions with the same version the order is not promised.
- Cases 5 and 6: a file that does not exist surfaces the JDK's `NoSuchFileException`, and text that is not a BPMN process is rejected by the engine.
- Case 7: a failed deployment changes nothing.

Remember that an application which deploys on start-up creates a new version on every start, even when the file is identical. Deploy when the definition changes, or check the list first.

The file is read with the server's default character set. When a BPMN file contains non-ASCII text (labels in Persian, for example), make sure the server runs with UTF-8 as its default or deploy through `deployFromSource...`, which receives text already decoded.

## 5. Starting an instance

`startProcessAndGetProcessInstanceId(key)` starts the latest version of a definition and returns the id of the new instance. The variant with a map of **initial variables** is how data enters the process.

```java
import org.j2os.platform.jflow.client.JFlowClient;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.HashMap;
import java.util.Map;

public class Start {
    public static void main(String[] args) throws Exception {
        JFlowClient client = new JFlowClient("rmi://localhost:1099/jflow", "demo-token");
        client.deployFromSourceAndGetProcessDefinitionKey("leave", Files.readString(Path.of("processes/leave-request.bpmn20.xml")));

        String plain = client.startProcessAndGetProcessInstanceId("leaveRequest");
        System.out.println("1 variables of a plain start: " + client.getVariablesByProcessInstanceId(plain));

        Map<String, Object> start = new HashMap<>();
        start.put("days", 3);
        start.put("employee", "Sara");
        String withData = client.startProcessAndGetProcessInstanceId("leaveRequest", start);
        System.out.println("2 days = " + client.getVariableByProcessInstanceId(withData, "days"));
        System.out.println("3 employee = " + client.getVariableByProcessInstanceId(withData, "employee"));
        System.out.println("4 two instances: " + !plain.equals(withData));

        try {
            client.startProcessAndGetProcessInstanceId("nothing");
        } catch (Exception e) {
            System.out.println("5 unknown key -> failed");
        }
    }
}
```

```text
1 variables of a plain start: {}
2 days = 3
3 employee = Sara
4 two instances: true
5 unknown key -> failed
```

- Case 1: a plain start has no variables.
- Cases 2 and 3: the initial variables are on the instance from the first step on.
- Case 4: every start creates its own instance.
- Case 5: an unknown key fails.

Starting does not mean finishing. The call returns when the instance reaches its first waiting point (a user task or a receive task). An instance made only of automatic steps runs to its end inside the call, and its id is already invalid when the call returns (chapter 7).

## 6. Working with tasks

When an instance reaches a user task, an open **task** appears. Four queries find tasks, each returning `JFlowTask` objects ordered by creation time:

| Method | Finds |
|--------|-------|
| `getAllOpenTasks()` | Every open task of every process |
| `getOpenTasksByAssignee(name)` | Tasks whose assignee equals `name` exactly |
| `getOpenTasksByProcessInstanceId(id)` | Tasks of one instance |
| `getOpenTasksByProcessDefinitionKey(key)` | Tasks of every instance of one definition |

A `JFlowTask` has `getTaskId()`, `getAssignee()`, `getCreateTime()`, `getProcessDefinitionKey()`, `getProcessInstanceId()` and `getPriority()`. The server fills the object in through its setters; clients only read it.

Completing a task is `signalByTaskId(taskId)`, optionally with a map of variables to set at the same time. The process then moves on until its next waiting point.

```java
import org.j2os.platform.jflow.client.JFlowClient;
import org.j2os.platform.jflow.share.JFlowTask;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Map;

public class Tasks {
    public static void main(String[] args) throws Exception {
        JFlowClient client = new JFlowClient("rmi://localhost:1099/jflow", "demo-token");
        client.deployFromSourceAndGetProcessDefinitionKey("purchase", Files.readString(Path.of("processes/purchase.bpmn20.xml")));
        client.deployFromSourceAndGetProcessDefinitionKey("leave", Files.readString(Path.of("processes/leave-request.bpmn20.xml")));

        String small = client.startProcessAndGetProcessInstanceId("purchase", Map.of("amount", 500));
        String big = client.startProcessAndGetProcessInstanceId("purchase", Map.of("amount", 2000));
        String leave = client.startProcessAndGetProcessInstanceId("leaveRequest");

        System.out.println("1 all open tasks: " + client.getAllOpenTasks().size());
        for (JFlowTask t : client.getAllOpenTasks()) {
            System.out.println("   " + t.getProcessDefinitionKey() + " -> " + t.getAssignee() + " (priority " + t.getPriority() + ")");
        }
        System.out.println("2 director's tasks: " + client.getOpenTasksByAssignee("director").size());
        System.out.println("3 tasks of the leave instance: " + client.getOpenTasksByProcessInstanceId(leave).size());
        System.out.println("4 tasks of every purchase: " + client.getOpenTasksByProcessDefinitionKey("purchase").size());

        JFlowTask clerk = client.getOpenTasksByProcessInstanceId(small).get(0);
        boolean isPage = clerk.getAssignee().startsWith("page:");
        System.out.println("5 small purchase goes to a page: " + isPage + " -> " + clerk.getAssignee().substring(5));

        client.signalByTaskId(clerk.getTaskId());
        System.out.println("6 after completing it, the instance has " + client.getOpenTasksByProcessInstanceId(small).size() + " tasks");
        try {
            client.signalByTaskId(clerk.getTaskId());
        } catch (Exception e) {
            System.out.println("7 completing it again -> failed");
        }
        System.out.println("8 the big one still waits: " + client.getOpenTasksByProcessInstanceId(big).size());
    }
}
```

```text
1 all open tasks: 3
   purchase -> page:purchase-form.jsp (priority 50)
   purchase -> director (priority 50)
   leaveRequest -> manager (priority 50)
2 director's tasks: 1
3 tasks of the leave instance: 1
4 tasks of every purchase: 2
5 small purchase goes to a page: true -> purchase-form.jsp
6 after completing it, the instance has 0 tasks
7 completing it again -> failed
8 the big one still waits: 1
```

- Case 1: the three open tasks come from three instances of two definitions.
- Cases 2 to 4: each query narrows by one criterion.
- Case 5: **the assignee is free text**. The sample purchase process puts `page:purchase-form.jsp` there, a convention for driving a user interface: the application asks which task an instance is waiting at, and routes the user to the page named after `page:`. A value that does not start with `page:` is an ordinary person or group name. The routing code is a few lines:

```java
JFlowTask task = client.getOpenTasksByProcessInstanceId(instanceId).get(0);
String assignee = task.getAssignee();

if (assignee != null && assignee.startsWith("page:")) {
    String page = assignee.substring("page:".length());   // "purchase-form.jsp"
    // send the user to that page
} else {
    // the task belongs to a person: show "waiting for " + assignee
}
```

- Cases 6 and 7: after completion the instance has moved on (here it ended), and the same task cannot be completed twice.
- Case 8: the other instance did not move.

Two things the queries do *not* do. They match only the assignee field: a task offered to a candidate group is not found by `getOpenTasksByAssignee`, though `getAllOpenTasks()` lists it (with a `null` assignee). And nothing checks *who* completes a task: any caller holding the token can complete any task. "Only the manager may approve" must be enforced by the application, before it calls `signalByTaskId`.

## 7. Automatic steps and waiting

A **service task** runs a Java class on the server without any person. The class implements Flowable's `JavaDelegate`, receives the instance's variables through `DelegateExecution`, and writes results back as variables. The BPMN names the class:

```xml
<serviceTask id="calculate" flowable:class="com.shop.flow.CalculateDeduction"/>
```

```java
package com.shop.flow;

public class CalculateDeduction implements JavaDelegate {
    public void execute(DelegateExecution execution) {
        int days = ((Number) execution.getVariable("days")).intValue();
        execution.setVariable("deduction", days * 500_000);
    }
}
```

The class must be on the **server's** classpath, under that exact name. Clients never see it.

The sample automation process has two such steps and then a receive task, a step that waits for an outside signal. For a step like that there is `forceSignalByProcessInstanceId(id)` (with an optional map of variables): it triggers the instance's single waiting execution, so that the process continues.

```java
import org.j2os.platform.jflow.client.JFlowClient;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Map;

public class Automation {
    public static void main(String[] args) throws Exception {
        JFlowClient client = new JFlowClient("rmi://localhost:1099/jflow", "demo-token");
        client.deployFromSourceAndGetProcessDefinitionKey("auto", Files.readString(Path.of("processes/leave-automation.bpmn20.xml")));

        String id = client.startProcessAndGetProcessInstanceId("leaveAutomation", Map.of("days", 3));
        System.out.println("1 the two service tasks already ran: " + client.getVariablesByProcessInstanceId(id));
        System.out.println("2 open user tasks: " + client.getOpenTasksByProcessInstanceId(id).size());

        client.forceSignalByProcessInstanceId(id, Map.of("reviewedBy", "Sara"));
        try {
            client.getVariablesByProcessInstanceId(id);
        } catch (Exception e) {
            System.out.println("3 read after it finished -> " + e.getMessage());
        }
        try {
            client.forceSignalByProcessInstanceId(id);
        } catch (Exception e) {
            System.out.println("4 signal after it finished -> " + e.getMessage());
        }
    }
}
```

```text
1 the two service tasks already ran: {days=3, deduction=1500000, summary=Deduction: 1500000}
2 open user tasks: 0
3 read after it finished -> class java.lang.Exception:No process found with this id (it may have already completed): <id>
4 signal after it finished -> class java.lang.Exception:No process found with this id (it may have already completed): <id>
```

- Case 1: both delegates had already run when the start call returned: the second used the first one's result.
- Case 2: there is no user task, because the instance waits at a receive task.
- Cases 3 and 4: after the forced signal the instance reached its end and disappeared. Reading or signalling it now fails with the *"may have already completed"* message. This is how a finished instance looks to a client.

Use `forceSignal` for steps that wait for an outside event. A user task is normally completed with `signalByTaskId`. The call needs exactly one waiting execution; a process that waits on several parallel paths at once has several, and the call fails with a message saying so. A delegate has to cope with missing variables itself: the engine does not know what a class expects, and a process started without `days` fails inside the first delegate.

An automatic process can also branch on its variables. The condition goes on the arrow leaving a gateway; an arrow without a condition is the fallback:

```xml
<exclusiveGateway id="amountCheck"/>
<sequenceFlow sourceRef="amountCheck" targetRef="directorApproval">
  <conditionExpression xsi:type="tFormalExpression">${amount > 1000}</conditionExpression>
</sequenceFlow>
<sequenceFlow sourceRef="amountCheck" targetRef="clerkApproval"/>
```

## 8. Moving an instance

Sometimes an instance has to go somewhere the drawing does not lead: a request approved by phone must skip the manager's task, or an instance has to be sent back one step. `moveByProcessInstanceId(id, targetActivityId)` takes the instance's current activity and moves it to the activity with the given `id` from the BPMN file.

```java
import org.j2os.platform.jflow.client.JFlowClient;

import java.nio.file.Files;
import java.nio.file.Path;

public class Moving {
    public static void main(String[] args) throws Exception {
        JFlowClient client = new JFlowClient("rmi://localhost:1099/jflow", "demo-token");
        client.deployFromSourceAndGetProcessDefinitionKey("leave", Files.readString(Path.of("processes/leave-request.bpmn20.xml")));

        String id = client.startProcessAndGetProcessInstanceId("leaveRequest");
        System.out.println("1 waiting for: " + client.getOpenTasksByProcessInstanceId(id).get(0).getAssignee());

        client.moveByProcessInstanceId(id, "hrApproval");
        System.out.println("2 after the move: " + client.getOpenTasksByProcessInstanceId(id).get(0).getAssignee());
        System.out.println("3 open tasks in total: " + client.getAllOpenTasks().size());

        try {
            client.moveByProcessInstanceId(id, "nowhere");
        } catch (Exception e) {
            System.out.println("4 unknown target -> failed");
        }
        try {
            client.moveByProcessInstanceId("missing", "hrApproval");
        } catch (Exception e) {
            System.out.println("5 unknown instance -> " + e.getMessage());
        }

        client.signalByTaskId(client.getOpenTasksByProcessInstanceId(id).get(0).getTaskId());
        try {
            client.moveByProcessInstanceId(id, "hrApproval");
        } catch (Exception e) {
            System.out.println("6 move after it finished -> " + e.getMessage());
        }
    }
}
```

```text
1 waiting for: manager
2 after the move: hr
3 open tasks in total: 1
4 unknown target -> failed
5 unknown instance -> class java.lang.Exception:No process found with this id (it may have already completed): missing
6 move after it finished -> class java.lang.Exception:No process found with this id (it may have already completed): <id>
```

- Cases 1 to 3: the manager's task was removed and the HR task created. The total of open tasks stayed one.
- Case 4: an activity id that does not exist in the definition fails.
- Cases 5 and 6: an unknown id, and an instance that has already finished, fail with the usual *"No process found"* message.

Things to know before using it:

- It moves the **first** active activity. An instance that is active in several places (parallel paths) moves only one of them.
- The steps in between are skipped, with their automatic logic. If a skipped service task was supposed to set a variable, set it yourself with `setVariable...`.
- It is a repair tool, not a normal way to advance. The normal way is to complete tasks.

## 9. Variables

Variables are the data of an instance. They can be read and written at any time while the instance is alive.

| Method | Does |
|--------|------|
| `getVariablesByProcessInstanceId(id)` | A map of all variables |
| `getVariableByProcessInstanceId(id, name)` | One value, `null` if it is not set |
| `setVariablesByProcessInstanceId(id, map)` | Sets several |
| `setVariableByProcessInstanceId(id, name, value)` | Sets one |

```java
import org.j2os.platform.jflow.client.JFlowClient;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class Variables {
    public static void main(String[] args) throws Exception {
        JFlowClient client = new JFlowClient("rmi://localhost:1099/jflow", "demo-token");
        client.deployFromSourceAndGetProcessDefinitionKey("leave", Files.readString(Path.of("processes/leave-request.bpmn20.xml")));
        String id = client.startProcessAndGetProcessInstanceId("leaveRequest");

        client.setVariableByProcessInstanceId(id, "note", "urgent");
        Map<String, Object> more = new HashMap<>();
        more.put("days", 2);
        more.put("approvers", new ArrayList<>(List.of("manager", "hr")));
        client.setVariablesByProcessInstanceId(id, more);

        Map<String, Object> all = client.getVariablesByProcessInstanceId(id);
        System.out.println("1 note=" + all.get("note") + ", days=" + all.get("days") + ", approvers=" + all.get("approvers"));
        System.out.println("2 an unset variable: " + client.getVariableByProcessInstanceId(id, "nothing"));

        client.setVariableByProcessInstanceId(id, "note", null);
        System.out.println("3 note set to null: " + client.getVariableByProcessInstanceId(id, "note"));

        try {
            client.setVariableByProcessInstanceId(id, "x", new Object());
        } catch (Exception e) {
            System.out.println("4 not serializable -> " + e.getCause());
        }
        try {
            client.getVariableByProcessInstanceId("missing", "note");
        } catch (Exception e) {
            System.out.println("5 unknown instance -> " + e.getMessage());
        }
    }
}
```

```text
1 note=urgent, days=2, approvers=[manager, hr]
2 an unset variable: null
3 note set to null: null
4 not serializable -> java.io.NotSerializableException: java.lang.Object
5 unknown instance -> class java.lang.Exception:No process found with this id (it may have already completed): missing
```

- Case 1: values of different types come back as they went in, including a list.
- Case 2: reading a variable that was never set returns `null`; there is no error and no way to tell "missing" from "set to null".
- Case 3: setting a value to `null` is allowed.
- Case 4: values are sent over RMI, so they must be `Serializable`. An object that is not fails on the client, before anything reaches the server.
- Case 5: an unknown or finished instance fails.

Choosing what to store:

- **Prefer plain values**: strings, numbers, booleans, dates, and lists or maps of them. The engine stores such values directly, and they are easy to read from SQL and from any language.
- **Avoid storing your own classes.** The engine saves other serializable objects as serialized bytes, which ties the stored data to the class version; the server also needs the class, and a strict serial filter (chapter 12) will refuse it.
- **Store identifiers, not records.** A leave request holds the employee id and the days, not a whole employee object that may change in the meantime.
- Variables belong to the instance and disappear with it. When the data must outlive the process, copy it into your own tables in the last step.

## 10. Suspending and activating a definition

`suspendByProcessDefinitionKey(key)` takes **every version** of a definition out of service; `activateByProcessDefinitionKey(key)` puts them back. A suspended definition cannot start new instances. This is how a process is retired, or paused while a bug is fixed.

```java
import org.j2os.platform.jflow.client.JFlowClient;

import java.nio.file.Files;
import java.nio.file.Path;

public class Suspend {
    public static void main(String[] args) throws Exception {
        JFlowClient client = new JFlowClient("rmi://localhost:1099/jflow", "demo-token");
        client.deployFromSourceAndGetProcessDefinitionKey("leave", Files.readString(Path.of("processes/leave-request.bpmn20.xml")));
        String running = client.startProcessAndGetProcessInstanceId("leaveRequest");

        client.suspendByProcessDefinitionKey("leaveRequest");
        System.out.println("1 active: " + client.getActiveProcessDefinitionKeys());
        System.out.println("2 suspended: " + client.getSuspendedProcessDefinitionKeys());
        try {
            client.startProcessAndGetProcessInstanceId("leaveRequest");
        } catch (Exception e) {
            System.out.println("3 start a new one -> failed");
        }
        System.out.println("4 the running one is still listed: " + client.getOpenTasksByProcessInstanceId(running).size());

        client.activateByProcessDefinitionKey("leaveRequest");
        try {
            client.startProcessAndGetProcessInstanceId("leaveRequest");
            System.out.println("5 start after activating -> ok");
        } catch (Exception e) {
            System.out.println("5 start after activating -> " + e.getMessage());
        }
        try {
            client.suspendByProcessDefinitionKey("nothing");
        } catch (Exception e) {
            System.out.println("6 suspend an unknown key -> failed");
        }
    }
}
```

```text
1 active: []
2 suspended: [leaveRequest:1]
3 start a new one -> failed
4 the running one is still listed: 1
5 start after activating -> ok
6 suspend an unknown key -> failed
```

- Cases 1 and 2: the definition moved from the active list to the suspended list.
- Case 3: starting a new instance fails while it is suspended.
- Case 4: an instance that was already running is untouched; its open task is still there.
- Case 5: after activating, starting works again.
- Case 6: an unknown key fails.

Both calls are written to the server's log, together with every deployment, so there is a trail of who changed which definition.

## 11. Diagrams

jflow can draw a definition as a PNG image: plain, or for a running instance with its current activities highlighted. This is how a page shows an employee "where is my request".

| Method | Draws |
|--------|-------|
| `getProcessDiagramByProcessDefinitionKey(key)` | The latest version, no highlighting |
| `getProcessDiagramByProcessInstanceId(id)` | The instance's definition, with its active activities highlighted |

Both return the PNG bytes. Saving them:

```java
byte[] png = client.getProcessDiagramByProcessInstanceId(instanceId);
Files.write(Path.of("instance.png"), png);
```

```java
import org.j2os.platform.jflow.client.JFlowClient;

import java.nio.file.Files;
import java.nio.file.Path;

public class Diagram {
    static boolean isPng(byte[] data) {
        return data.length > 8 && data[1] == 'P' && data[2] == 'N' && data[3] == 'G';
    }

    public static void main(String[] args) throws Exception {
        JFlowClient client = new JFlowClient("rmi://localhost:1099/jflow", "demo-token");
        client.deployFromSourceAndGetProcessDefinitionKey("d", Files.readString(Path.of("processes/diagram.bpmn20.xml")));
        client.deployFromSourceAndGetProcessDefinitionKey("n", Files.readString(Path.of("processes/no-layout.bpmn20.xml")));

        String id = client.startProcessAndGetProcessInstanceId("diagramSample");
        System.out.println("1 definition image is a PNG: " + isPng(client.getProcessDiagramByProcessDefinitionKey("diagramSample")));
        System.out.println("2 instance image is a PNG: " + isPng(client.getProcessDiagramByProcessInstanceId(id)));

        try {
            client.getProcessDiagramByProcessDefinitionKey("nothing");
        } catch (Exception e) {
            System.out.println("3 unknown definition -> " + e.getMessage());
        }
        try {
            client.getProcessDiagramByProcessDefinitionKey("noLayout");
        } catch (Exception e) {
            System.out.println("4 definition without layout -> " + e.getMessage());
        }

        client.signalByTaskId(client.getOpenTasksByProcessInstanceId(id).get(0).getTaskId());
        try {
            client.getProcessDiagramByProcessInstanceId(id);
        } catch (Exception e) {
            System.out.println("5 finished instance -> " + e.getMessage());
        }
    }
}
```

```text
1 definition image is a PNG: true
2 instance image is a PNG: true
3 unknown definition -> class java.lang.Exception:No process definition was found with this name: nothing
4 definition without layout -> class java.lang.Exception:This process has no diagram layout information (bpmndi:BPMNDiagram); the diagram cannot be rendered: noLayout
5 finished instance -> class java.lang.Exception:No process found with this id (it may have already completed): <id>
```

- Cases 1 and 2: both calls return PNG data.
- Cases 3 and 5: an unknown definition and a finished instance fail.
- Case 4: **a definition without layout information cannot be drawn.** The image needs the `bpmndi:BPMNDiagram` section of the BPMN file, which a modelling tool (Flowable Modeler, bpmn.io, Camunda Modeler) writes automatically. A file written by hand has none.

Drawing happens on the server with Java's graphics classes. On a server without fonts installed, labels can come out as empty boxes; install a font or configure the engine's activity and label font names.

## 12. Tokens, filters and who may call

jflow has two protections, and they guard different things.

**The token** answers "may this caller talk to the server at all?". It is a single secret shared by every client. The server compares it in constant time, so the comparison does not leak how much of it matched, and logs every failed attempt as a warning. Keep it in the environment or a secrets store, give it enough length to be unguessable, and rotate it by restarting the server with a new one. The token tells the server nothing about *who* is calling: with the token, every operation is allowed on every instance and task.

**The serial filter** answers "which Java classes may callers send?". RMI turns arguments back into objects on the server, and deserializing untrusted classes is a classic way to attack Java programs. The optional third constructor argument is a pattern in the format of the JDK's `jdk.serialFilter`: patterns apply in order, the first match wins, and a final `!*` refuses everything not listed. Limits such as `maxdepth=10` or `maxbytes=1000000` can be added with `;`. The filter applies to what clients **send**, not to what the server returns, and it must allow the classes of every variable value clients send, including the classes of everything reachable from them.

```java
import org.flowable.engine.impl.cfg.StandaloneInMemProcessEngineConfiguration;
import org.j2os.platform.jflow.client.JFlowClient;
import org.j2os.platform.jflow.server.JFlowServer;

import java.io.Serializable;
import java.nio.file.Files;
import java.nio.file.Path;
import java.rmi.Naming;
import java.rmi.registry.LocateRegistry;
import java.util.Map;

public class Security {
    public static class Money implements Serializable {
        public int amount = 5;
    }

    public static void main(String[] args) throws Exception {
        String url = "rmi://localhost:1099/jflow";
        JFlowServer server = new JFlowServer("demo-token", new StandaloneInMemProcessEngineConfiguration(), "java.lang.*;java.util.*;!*");
        LocateRegistry.createRegistry(1099);
        Naming.rebind(url, server);

        JFlowClient client = new JFlowClient(url, "demo-token");
        client.deployFromSourceAndGetProcessDefinitionKey("leave", Files.readString(Path.of("processes/leave-request.bpmn20.xml")));

        try {
            client.startProcessAndGetProcessInstanceId("leaveRequest", Map.of("days", 3));
            System.out.println("1 plain values pass the filter -> ok");
        } catch (Exception e) {
            System.out.println("1 plain values pass the filter -> " + e.getMessage());
        }
        try {
            client.startProcessAndGetProcessInstanceId("leaveRequest", Map.of("price", new Money()));
        } catch (Exception e) {
            System.out.println("2 own class is refused -> " + e.getCause().getCause());
        }
        try {
            new JFlowClient(url, "nope").startProcessAndGetProcessInstanceId("leaveRequest", Map.of("price", new Money()));
        } catch (Exception e) {
            System.out.println("3 refused even with a wrong token -> " + e.getCause().getCause());
        }
        try {
            new JFlowClient(url, "nope").startProcessAndGetProcessInstanceId("leaveRequest", Map.of("days", 3));
        } catch (Exception e) {
            System.out.println("4 a wrong token with plain values -> " + e.getMessage());
        }

        Naming.unbind(url);
        server.close();
    }
}
```

```text
1 plain values pass the filter -> ok
2 own class is refused -> java.io.InvalidClassException: filter status: REJECTED
3 refused even with a wrong token -> java.io.InvalidClassException: filter status: REJECTED
4 a wrong token with plain values -> Invalid token, access denied.
```

- Case 1: strings and numbers pass `java.lang.*;java.util.*;!*`.
- Case 2: an own class is refused with `InvalidClassException: filter status: REJECTED`, wrapped by RMI in a `ServerException`.
- Case 3: the filter runs while the arguments are being read, **before the token is checked**, so a caller with a wrong token can't even get to the point of being told so when it sends a forbidden class.
- Case 4: with allowed values, the wrong token is refused by the token check.

How to think about it:

- The token and the filter are the server's defence; **authorisation is the application's job**. A web application that lets users pick a task id from a form must check that the task belongs to that user, and that ids it shows are theirs. Random ids (`StrongUuidGenerator`) mean a user cannot stumble on somebody else's instance, but they are not a permission system.
- The RMI port is not a public service. Put it on an internal network or behind a firewall; the server's object is exported on a port chosen by the system, so firewall rules have to account for that. On a machine with several network addresses, RMI may need the `java.rmi.server.hostname` system property to advertise the right one.
- RMI traffic is not encrypted by default. If the network is not trusted, tunnel it (VPN, SSH, TLS socket factories).

## 13. When calls fail

Failures come in two families, and the catch blocks should separate them.

| Family | Exception | Meaning |
|--------|-----------|---------|
| The call did not complete normally | a `java.rmi.RemoteException` or subclass | Network, lookup or serialization problem: the server is gone, a value was not serializable, the filter refused a class |
| The server ran the call and it failed | plain `Exception` | The token was wrong, or the operation itself failed |

For the second family there are two shapes:

- A wrong or missing token is a plain `Exception` with the message *"Invalid token, access denied."*
- Any failure **inside** an operation is wrapped: a new `Exception` whose message is `class <OriginalClass>:<original message>` and whose cause is the original exception. For server-written problems the original class is `java.lang.Exception`.

```java
import org.j2os.platform.jflow.client.JFlowClient;

public class Errors {
    public static void main(String[] args) throws Exception {
        JFlowClient client = new JFlowClient("rmi://localhost:1099/jflow", "demo-token");

        try {
            client.signalByTaskId("no-such-task");
        } catch (Exception e) {
            System.out.println("1 message starts with: " + e.getMessage().substring(0, e.getMessage().indexOf(':')));
            System.out.println("2 the real cause: " + e.getCause().getClass().getSimpleName());
        }

        try {
            client.getVariablesByProcessInstanceId("missing");
        } catch (Exception e) {
            System.out.println("3 server-written message: " + e.getMessage());
            System.out.println("4 cause: " + e.getCause());
        }

        try {
            new JFlowClient("rmi://localhost:1099/jflow", "wrong").getAllOpenTasks();
        } catch (Exception e) {
            System.out.println("5 " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
    }
}
```

```text
1 message starts with: class org.flowable.common.engine.api.FlowableObjectNotFoundException
2 the real cause: FlowableObjectNotFoundException
3 server-written message: class java.lang.Exception:No process found with this id (it may have already completed): missing
4 cause: java.lang.Exception: No process found with this id (it may have already completed): missing
5 Exception: Invalid token, access denied.
```

- Case 1: the message of an engine failure starts with the engine's exception class name.
- Case 2: and the cause is that exception itself.
- Cases 3 and 4: the server's own *"No process found"* message is the part after the first colon, and the cause is the original exception.
- Case 5: the token failure has no wrapping.

A client usually wants to separate the reason from the class name:

```java
try {
    client.signalByTaskId(taskId);
} catch (java.rmi.RemoteException network) {
    // the call may not have reached the server, or its answer got lost
} catch (Exception failure) {
    String message = failure.getMessage();             // "class <ExceptionClass>:<text>"
    String reason = message.substring(message.indexOf(':') + 1);
    Throwable original = failure.getCause();            // the exception the server actually met
}
```

Because the original exception travels as the cause, the client needs the classes of that exception. A client that lacks the engine's jars can fail to read the cause and get a `RemoteException` about unmarshalling instead of the clean exception. Keep the engine's common library on the client classpath, or at least test an error path before going live.

Two kinds of failure are not retried safely:

- A `RemoteException` on a *write* (start, signal, set variable) does not tell whether the server did it or not. Check the state (open tasks, variables) before repeating.
- The calls are not transactional across several of them: starting an instance and setting variables are two calls. Pass the data as initial variables in the start call so that it happens as one.

## 14. Putting it together: leave requests

An employee asks for three days off. The manager approves, then HR approves, and then payroll runs an automatic process that computes the deduction.

```java
import org.j2os.platform.jflow.client.JFlowClient;
import org.j2os.platform.jflow.share.JFlowTask;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Map;

public class Workflow {
    public static void main(String[] args) throws Exception {
        JFlowClient client = new JFlowClient("rmi://localhost:1099/jflow", "demo-token");
        client.deployFromSourceAndGetProcessDefinitionKey("leave", Files.readString(Path.of("processes/leave-request.bpmn20.xml")));
        client.deployFromSourceAndGetProcessDefinitionKey("auto", Files.readString(Path.of("processes/leave-automation.bpmn20.xml")));

        // 1. an employee asks for 3 days off
        String request = client.startProcessAndGetProcessInstanceId("leaveRequest", Map.of("days", 3, "employee", "Sara"));

        // 2. the manager and then HR each see their own task and approve it
        for (String who : new String[]{"manager", "hr"}) {
            JFlowTask task = client.getOpenTasksByAssignee(who).get(0);
            System.out.println(who + " sees a task of " + client.getVariableByProcessInstanceId(task.getProcessInstanceId(), "employee"));
            client.signalByTaskId(task.getTaskId(), Map.of(who + "Approved", true));
        }
        System.out.println("open tasks left: " + client.getAllOpenTasks().size());

        // 3. payroll runs the automatic process with the approved days
        String payroll = client.startProcessAndGetProcessInstanceId("leaveAutomation", Map.of("days", 3));
        System.out.println("payroll says: " + client.getVariableByProcessInstanceId(payroll, "summary"));
        client.forceSignalByProcessInstanceId(payroll);
        System.out.println("payroll finished: " + client.getAllOpenTasks().isEmpty());
    }
}
```

```text
manager sees a task of Sara
hr sees a task of Sara
open tasks left: 0
payroll says: Deduction: 1500000
payroll finished: true
```

The steps and where they come from:

1. The request is started with the days and the employee as initial variables (chapter 5).
2. Each approver finds their own task by assignee, reads a variable of the instance to show whose request it is, and completes the task with a variable of their own (chapters 6 and 9).
3. Both tasks are done, and the request's instance is gone: no open tasks remain (chapter 1).
4. Payroll's automatic process runs its two service tasks inside the start call, so the summary is readable immediately (chapter 7); then it is released from its receive step (chapter 7).

In a web application, step 2 is a page: the task list of the signed-in user comes from `getOpenTasksByAssignee`, and the approve button calls `signalByTaskId` after the application has checked that the task belongs to that user.

Before going to production:

- The token comes from the environment, and the RMI port is reachable only from the applications that need it.
- The serial filter lists exactly the classes the clients send.
- Variables are plain values; the delegate classes are deployed on the server.
- The application checks the user's right to each task before calling jflow.
- Definitions are deployed when they change, not on every start, and UTF-8 is the server's character set.
- Errors from the second family of chapter 13 are caught and the reason is shown or logged; `RemoteException` is treated as "state unknown".
- The server is closed on shutdown.

---

## Appendix A. Quick reference

### A.1 JFlowServer

| Call | Does |
|------|------|
| `new JFlowServer(token, config)` | Builds an engine from the configuration and exports the server |
| `new JFlowServer(token, config, serialFilter)` | The same, with a filter on the arguments clients send |
| `new JFlowServer(token, engine)` | Uses an engine that already exists |
| `new JFlowServer(token, engine, serialFilter)` | The same, with a filter |
| `getProcessEngine()` | The engine behind the server |
| `close()` | Unexports the server and closes the engine. Safe to repeat |

The token must not be `null`; `""` is valid. The filter, when given, must be a valid, non-blank pattern. The server must still be published with `Naming.rebind(...)` after construction.

### A.2 JFlowClient

| Call | Does |
|------|------|
| `new JFlowClient(rmiUrl, token)` | Looks the server up and stores the token |
| `getRemote()` | The stub (token is then the first argument of each call) |
| `startProcessAndGetProcessInstanceId(key[, variables])` | Starts the latest version, returns the instance id |
| `forceSignalByProcessInstanceId(id[, variables])` | Triggers the single waiting execution |
| `signalByTaskId(taskId[, variables])` | Completes a task |
| `moveByProcessInstanceId(id, targetActivityId)` | Moves the first active activity |
| `getAllOpenTasks()` | All open tasks |
| `getOpenTasksByAssignee(name)` | Tasks of one assignee |
| `getOpenTasksByProcessInstanceId(id)` | Tasks of one instance |
| `getOpenTasksByProcessDefinitionKey(key)` | Tasks of one definition |
| `getProcessDiagramByProcessInstanceId(id)` | PNG with active activities highlighted |
| `getProcessDiagramByProcessDefinitionKey(key)` | Plain PNG of the latest version |
| `getVariablesByProcessInstanceId(id)` | All variables |
| `getVariableByProcessInstanceId(id, name)` | One variable, `null` if unset |
| `setVariablesByProcessInstanceId(id, map)` | Sets several variables |
| `setVariableByProcessInstanceId(id, name, value)` | Sets one variable |
| `suspendByProcessDefinitionKey(key)` | Suspends every version |
| `activateByProcessDefinitionKey(key)` | Activates every version |
| `deployFromSourceAndGetProcessDefinitionKey(name, xml)` | Deploys BPMN text, returns the key |
| `deployFromFileAndGetProcessDefinitionKey(path)` | Deploys a BPMN file on the server |
| `getActiveProcessDefinitionKeys()` | `key:version` of active definitions |
| `getSuspendedProcessDefinitionKeys()` | `key:version` of suspended definitions |

### A.3 JFlowTask

| Member | Meaning |
|--------|---------|
| `getTaskId()` | Id to pass to `signalByTaskId` |
| `getAssignee()` | Free text: a person, a group, or `page:...` (may be `null`) |
| `getCreateTime()` | Creation time (a copy is returned) |
| `getProcessDefinitionKey()` | Key of the definition it belongs to |
| `getProcessInstanceId()` | Instance it belongs to |
| `getPriority()` | Task priority (50 unless the process sets another) |

### A.4 BPMN features that matter to jflow

| Element | Meaning for jflow |
|---------|-------------------|
| `<process id="x">` | `x` is the key used in every call |
| `userTask` with `flowable:assignee` | Creates an open task whose assignee is that text |
| `serviceTask` with `flowable:class` | Runs a delegate on the server |
| `receiveTask` | Waits for `forceSignalByProcessInstanceId` |
| `exclusiveGateway` with conditions | Chooses a path from the variables |
| `bpmndi:BPMNDiagram` | Needed for drawing diagrams |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| `null` token when creating a server | `IllegalArgumentException` |
| Blank or invalid serial filter | `IllegalArgumentException` (or a subclass); nothing is built |
| Name not published in the registry | `NotBoundException` from the `JFlowClient` constructor |
| Registry unreachable | `ConnectException` from the constructor |
| Wrong or `null` token | Plain `Exception`: *"Invalid token, access denied."* |
| Server closed or restarted | `NoSuchObjectException`; create a new client |
| Variable not serializable | `NotSerializableException` inside a `MarshalException`, on the client |
| Class refused by the filter | `InvalidClassException` inside a `ServerException`, before the token is checked |
| Unknown or finished instance | `Exception` with the *"No process found"* message |
| Instance has no waiting execution | `Exception` with the *"No waiting execution"* message |
| Instance has no active activity | `Exception` with the *"No active activity"* message |
| Unknown task id | `Exception` wrapping the engine's not-found exception |
| Unknown process key on start | `Exception` wrapping the engine's not-found exception |
| Start of a suspended definition | `Exception` wrapping an engine exception |
| Deployment of a missing file | `Exception` wrapping `NoSuchFileException` |
| Deployment of invalid BPMN | `Exception` wrapping an engine exception |
| Diagram without layout information | `Exception` with the *"no diagram layout information"* message |
| Move to an activity that does not exist | `Exception` wrapping an engine exception |
| Failure in a delegate | Surfaces as the failure of the call that ran the step |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Empty or default token in production.** An empty token lets in everyone who also sends an empty one.
- **Token or database password in the source.** Read them from the environment or a secrets store.
- **Treating the token as a user check.** Anyone with the token can complete any task. Check rights in the application.
- **Exposing ids to users without checking ownership.** Random ids help but do not replace a check.
- **Default id generator.** Without `StrongUuidGenerator`, ids count upward and can be guessed.
- **Redeploying on every start.** Each deployment makes a new version.
- **Non-ASCII BPMN files read with a non-UTF-8 default charset.** Use UTF-8, or deploy from text.
- **Expecting `deployFromFile...` to read a client file.** The path is on the server.
- **Reading a finished instance.** It no longer exists; keep what you need in your own tables.
- **Using `forceSignal` on a user task or on several parallel waits.** Use `signalByTaskId`, or move/complete explicitly.
- **Relying on `moveBy...` for normal flow.** Skipped steps do not run.
- **Own classes as variables.** They tie stored data to class versions and need the filter's blessing.
- **Treating `null` as "not set".** Reads cannot tell the two apart.
- **Assuming `getOpenTasksByAssignee` covers groups.** It matches only the assignee field.
- **A `RemoteException` on a write is not "it failed".** The server may have done it.
- **Keeping a client after a server restart.** Create a new one.
- **Asynchronous steps and timers.** With the job executor off they do not run.
- **Client without the engine's exception classes.** The wrapped cause may not be readable.

### C.2 Behaviours worth knowing

- Every operation checks the token first, except that argument deserialization (and so the serial filter) happens before it.
- Failures inside an operation are wrapped as `class <Original>:<message>` with the original as cause; token failures are not wrapped.
- Failed token attempts, deployments, suspensions and activations are written to the server's log.
- Queries return tasks ordered by creation time, ascending.
- `getActiveProcessDefinitionKeys` and `getSuspendedProcessDefinitionKeys` return `key:version` ordered by version.
- A process definition's key is the BPMN process id; new instances use the latest version.
- A suspended definition affects only new starts.
- The client holds only a stub and a token, so it is cheap and can be shared.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Authentication | One token per server |
| Users and roles | None inside jflow |
| Task query by assignee | Exact match on the assignee field |
| `forceSignal` | Exactly one waiting execution |
| `moveBy...` | Moves the first active activity |
| Diagram format | PNG, only for definitions with layout information |
| Variable values | Serializable; accepted by the serial filter if one is set |
| Transport | RMI; not encrypted by default |

### C.4 Tuning

- Put the engine's database on a server of its own for anything beyond a demo; give the pool enough connections for the number of simultaneous clients.
- Keep the number of open tasks per assignee small: lists are returned whole.
- Use the narrowest query (by instance or assignee) instead of `getAllOpenTasks()` on busy systems.
- Keep variables small. Large payloads travel on every read of all variables.
- Pass all data a process needs as initial variables of the start call.
- Run jflow close to its database, and clients close to jflow; each call is a network round trip.

## Appendix D. Message reference

### D.1 Messages written by jflow

| Message | Exception | Cause |
|---------|-----------|-------|
| `Invalid token, access denied.` | `Exception` (not wrapped) | Token wrong or `null` |
| `token is null (use an empty string "" to run without a token)` | `IllegalArgumentException` | Server created with a `null` token |
| `serialFilter must be a valid, non-blank pattern: <pattern>` | `IllegalArgumentException` | Blank filter |
| `No process found with this id (it may have already completed): <id>` | wrapped `Exception` | Instance unknown or finished |
| `No waiting execution was found for this process instance: <id>` | wrapped `Exception` | Nothing waits in the instance |
| `No active activity was found for this process instance: <id>` | wrapped `Exception` | Move on an instance with no active activity |
| `No process definition was found with this name: <key>` | wrapped `Exception` | Diagram of an unknown definition |
| `This process has no diagram layout information (bpmndi:BPMNDiagram); the diagram cannot be rendered: <subject>` | wrapped `Exception` | Definition without layout |

### D.2 The wrapping format

| Part | Content |
|------|---------|
| Exception | A plain `java.lang.Exception` |
| Message | `class <fully qualified original class>:<original message>` |
| Cause | The original exception |

### D.3 Messages that come from the engine

The text after the class name in a wrapped message comes from Flowable (unknown process key, unknown task, suspended definition, invalid BPMN, unknown target activity) or from the JDK (a missing file). Its wording belongs to those libraries and may change with their versions; rely on the class name, and show or log the message.
