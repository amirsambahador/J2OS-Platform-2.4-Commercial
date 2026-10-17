# jtelegram

*J2OS Platform 2.4*

A Telegram bot is a small program that talks to people inside Telegram: it answers questions, shows buttons, reports the status of an order, or warns an administrator that a job has failed. Telegram offers a large interface for bots, with photos, files, polls, payments and much more, and a program that only needs to exchange text should not have to learn all of it.

**jtelegram** (`org.j2os.platform.jtelegram`) is a deliberately small wrapper over the *telegrambots* library, limited to what most business bots need: receive text, answer commands, show buttons and send messages. Your code never touches a telegrambots type. It sees four simple classes.

```java
JTelegram bot = JTelegram.create(token);

bot.onCommand("start", "Welcome message", message -> message.reply("Welcome!"));
bot.onMessage(message -> message.reply("You wrote: " + message.text()));

bot.start();
```

- `JTelegram` is the bot. It registers handlers, starts and stops, and sends messages.
- `Message` is one incoming text, with the chat, the sender and the arguments of a command, and a `reply` method.
- `User` describes the sender: id, username, first name and last name.
- `Menu` builds the buttons that appear under a message.

`JTelegramException` is the unchecked exception for every failure that comes from Telegram.

This guide goes step by step: each chapter has a short program and the conversation it produces. The appendices are for quick reference.

## Contents

1. [How a bot works](#1-how-a-bot-works)
2. [A first bot](#2-a-first-bot)
3. [Messages and users](#3-messages-and-users)
4. [Commands](#4-commands)
5. [All handlers run](#5-all-handlers-run)
6. [Command names and the menu](#6-command-names-and-the-menu)
7. [Sending without being asked](#7-sending-without-being-asked)
8. [Buttons: building a menu](#8-buttons-building-a-menu)
9. [Buttons: handling clicks](#9-buttons-handling-clicks)
10. [Rules and limits](#10-rules-and-limits)
11. [Starting and stopping](#11-starting-and-stopping)
12. [In a Spring application](#12-in-a-spring-application)
13. [Putting it together: an order bot](#13-putting-it-together-an-order-bot)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The module is built on *telegrambots* 6.9.7.1. It needs the library on the classpath and a network connection to Telegram's servers.

---

## 1. How a bot works

**The token.** Every bot is created in Telegram by talking to the official bot called *BotFather*. It gives the new bot a name and a secret *token*, a long text such as `123456:ABC-DEF...`. The token is both the identity and the password of the bot. Anyone who has it can act as the bot. Keep it out of the source code and out of version control, and read it from configuration or from an environment variable. The programs in this guide use a placeholder, `123456:DEMO_TOKEN`.

**Long polling.** jtelegram receives messages by *long polling*: the program repeatedly asks Telegram "is there anything new for me?", and Telegram answers as soon as there is. Because the program makes the requests, it needs no public address, no certificate and no open port. It works from a laptop as well as from a server.

**Text only.** The library handles text messages and button clicks. A photo, a sticker, a file, a location or a voice message sent to the bot is ignored. Messages that the bot sends are plain text, without Markdown or HTML formatting.

**Handlers.** You describe what the bot does by registering *handlers*, small functions that Telegram events call. `onMessage` receives every text message. `onCommand` receives the messages that start with a given command, such as `/start`. A *command* is a message that begins with `/` and a name, which Telegram shows to the user as a clickable shortcut.

**How the conversations are shown.** Each chapter shows a program and then the conversation it produces, written like this:

- `Sara: /start` is a message that the user typed.
- `bot: Welcome!` is a message that the bot sent.
- `Sara taps [Laptops]` is a click on a button.
- `[menu] /start - Welcome message` is a line of the command menu that the bot publishes to Telegram when it starts. `[menu cleared]` means that the bot published no commands.
- `bot (edits the message): ...` is a change of an existing message, which chapter 9 explains.
- Lines that begin with `log:` and other plain lines are printed by the program itself.

## 2. A first bot

```java
import org.j2os.platform.jtelegram.JTelegram;

public class FirstBot {

    public static void main(String[] args) {
        JTelegram bot = JTelegram.create("123456:DEMO_TOKEN");

        bot.onMessage(message -> message.reply("You wrote: " + message.text()));

        bot.start();
    }
}
```

```text
[menu cleared]
Sara: hello
bot: You wrote: hello
Sara: how are you?
bot: You wrote: how are you?
```

- `JTelegram.create(token)` creates the bot. The token is not checked yet. Telegram sees it for the first time in `start()`.
- `onMessage(...)` registers a handler for every text message. The handler is a lambda that receives a `Message`.
- `message.reply(text)` answers in the same chat.
- `start()` connects to Telegram, learns the bot's own name, begins long polling and publishes the command menu. It returns at once, and polling continues in background threads, so the program stays alive.
- The first line, `[menu cleared]`, appears because the bot has no commands. `start()` then clears the menu in Telegram, so that commands left over from an earlier version of the bot do not stay visible.

## 3. Messages and users

A `Message` carries what the handler needs to know.

```java
import org.j2os.platform.jtelegram.JTelegram;

public class MessageFields {

    public static void main(String[] args) {
        JTelegram bot = JTelegram.create("123456:DEMO_TOKEN");

        bot.onMessage(message -> {
            System.out.println("text:     " + message.text());
            System.out.println("chat id:  " + message.chatId());
            System.out.println("user id:  " + message.user().id());
            System.out.println("first:    " + message.user().firstName());
            System.out.println("last:     " + message.user().lastName());  // may be null
            System.out.println("username: " + message.user().username());  // may be null
        });

        bot.start();
    }
}
```

```text
[menu cleared]
Sara: Good morning
text:     Good morning
chat id:  1001
user id:  1001
first:    Sara
last:     null
username: null
```

- `text()` is the text that the user sent.
- `chatId()` identifies the conversation. In a private chat it equals the id of the user. Groups have their own ids, which are negative numbers. Keep the chat id if you want to write to this chat later (chapter 7).
- `user()` returns a `User` with `id()`, `firstName()`, `lastName()` and `username()`. The last two are optional on Telegram accounts, so they can be `null`, as in the output. `firstName()` is always filled.
- For a message without a visible sender, `user()` is `null`. Check it before use if your bot is added to groups or channels.

## 4. Commands

A command is the natural way to give a bot an order. `onCommand(name, description, handler)` registers a handler for one command.

```java
import org.j2os.platform.jtelegram.JTelegram;

public class Commands {

    public static void main(String[] args) {
        JTelegram bot = JTelegram.create("123456:DEMO_TOKEN");

        bot.onCommand("start", "Welcome message", message ->
                message.reply("Welcome, " + message.user().firstName() + "!"));

        bot.onCommand("weather", "Weather of a city", message -> {
            String city = message.commandArgs();
            message.reply(city.isEmpty() ? "Usage: /weather <city>" : "Weather in " + city + ": sunny");
        });

        bot.start();
    }
}
```

```text
[menu] /start - Welcome message
[menu] /weather - Weather of a city
Sara: /start
bot: Welcome, Sara!
Sara: /START
bot: Welcome, Sara!
Sara: /start@demo_bot
bot: Welcome, Sara!
Sara: /start@other_bot
Sara: /weather Tehran
bot: Weather in Tehran: sunny
Sara: /weather
bot: Usage: /weather <city>
Sara: /unknown
```

- The first two lines are the menu. A description is required, because Telegram lists each registered command with its description in the menu next to the input field.
- `/start` and `/START` are the same command. Names are not case sensitive.
- `/start@demo_bot` is also accepted. In a group with several bots Telegram adds `@botname` to say which bot is meant. A command that names this bot is handled. A command that names another bot, `/start@other_bot`, is ignored, so the bot never answers someone else's command.
- `message.commandArgs()` is everything after the command: for `/weather Tehran` it is `Tehran`. When there are no arguments it is an empty text.
- `/unknown` is not registered, so nothing happens. The bot has no `onMessage` handler either, so it stays silent.

## 5. All handlers run

A message can be handled by a command handler and by `onMessage` handlers. The rule is simple: a matching command handler runs first, then **every** `onMessage` handler runs, in the order in which they were registered.

```java
import org.j2os.platform.jtelegram.JTelegram;

public class HandlerOrder {

    public static void main(String[] args) {
        JTelegram bot = JTelegram.create("123456:DEMO_TOKEN");

        bot.onCommand("ping", "Answers pong", message -> message.reply("pong"));

        bot.onMessage(message -> System.out.println("log: " + message.text()));
        bot.onMessage(message -> {
            throw new IllegalStateException("a broken handler");
        });
        bot.onMessage(message -> message.reply("received"));

        bot.start();
    }
}
```

```text
[menu] /ping - Answers pong
Sara: /ping
bot: pong
log: /ping
bot: received
Sara: hi
log: hi
bot: received
```

- For `/ping` the command handler answers `pong`, then the first `onMessage` handler writes its log line, and the third answers `received`.
- The second `onMessage` handler throws an exception on every message. It does not stop the third one: the answer `received` still arrives, for both messages.
- Messages that are not commands, like `hi`, reach only the `onMessage` handlers.

If several handlers fail, all of them still run. When the last one is done, the first exception is rethrown to the library, and the others are attached to it as *suppressed* exceptions. It goes back to the library, which keeps polling.

An `onMessage` handler is therefore the right place for things that concern every message, such as logging or counting. Use it with care for replies: a bot with a command handler and a catch-all handler answers twice.

## 6. Command names and the menu

The name of a command is cleaned before it is stored: a leading `/` is removed and the letters are made lower case. A name has 1 to 32 characters, which are lower case English letters, digits or underscores.

```java
import org.j2os.platform.jtelegram.JTelegram;

public class CommandNames {

    public static void main(String[] args) {
        JTelegram bot = JTelegram.create("123456:DEMO_TOKEN");

        bot.onCommand("hello", "First greeting", message -> message.reply("first"));
        bot.onCommand("/Hello", "Second greeting", message -> message.reply("second"));  // same command

        bot.start();
    }
}
```

```text
[menu] /hello - Second greeting
Sara: /hello
bot: second
```

- `"hello"` and `"/Hello"` are the same command. The second registration **replaced** the first: only `second` answers, and the menu has one line.
- The menu is published by `start()`. Register the commands before you call it. A command registered later still works, but it does not appear in Telegram's menu until the bot is started again.
- Every command has a description, so every command appears in the menu. There is no hidden command. For an action that should not be listed, such as an internal button value, handle the text in an `onMessage` handler instead (chapter 9 does this).
- A bot can have up to 100 commands, which is Telegram's limit.

## 7. Sending without being asked

A bot does not only answer. It can start a conversation, for example to report that a job has finished. `send(chatId, text)` writes to a chat from any place in the program.

```java
import org.j2os.platform.jtelegram.JTelegram;
import org.j2os.platform.jtelegram.exception.JTelegramException;

public class SendDirect {

    public static void main(String[] args) {
        JTelegram bot = JTelegram.create("123456:DEMO_TOKEN");

        bot.send(1001, "The nightly backup has finished.");  // start() is not needed

        try {
            bot.send(404, "Hello?");  // a chat that does not exist
        } catch (JTelegramException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
bot -> chat 1001: The nightly backup has finished.
Failed to send message to chat 404
```

- `send` does not need `start()`. Sending and receiving are independent, so a program that only sends notifications never has to start polling.
- A failed send throws `JTelegramException`, with the message `Failed to send message to chat <id>`. The original error of Telegram is attached as the cause.
- Telegram has its own rules for sending. A bot can write only to a user who has started a conversation with it first, and an administrator of a group has to add the bot before it can write there. A text must not be empty and has a maximum length, 4096 characters. Telegram also limits how fast a bot may send, roughly one message per second to the same chat. A text that breaks these rules is rejected by Telegram, and shows up here as a `JTelegramException`.

`send(chatId, text, menu)` sends a message with buttons. Chapter 8 builds them.

## 8. Buttons: building a menu

An *inline keyboard* is a set of buttons shown under a message. `Menu` builds it with a chain of calls. Each call to `addButton` or `addURL` adds one **row**, and the number of label and value pairs in the call is the number of buttons in that row.

```java
import org.j2os.platform.jtelegram.JTelegram;
import org.j2os.platform.jtelegram.Menu;

public class MenuLayout {

    public static void main(String[] args) {
        JTelegram bot = JTelegram.create("123456:DEMO_TOKEN");

        bot.onCommand("shop", "Open the shop", message -> {
            Menu menu = Menu.create()
                    .addURL("Visit the website", "https://example.com")  // a link, on its own row
                    .addButton("Laptops", "cat:laptops")  // one button on a row
                    .addButton("Phones", "cat:phones", "Tablets", "cat:tablets")  // two on a row
                    .addButton("A", "pick:a", "B", "pick:b", "C", "pick:c");  // three on a row
            message.reply("What are you looking for?", menu);
        });

        bot.start();
    }
}
```

```text
[menu] /shop - Open the shop
Sara: /shop
bot: What are you looking for?
     [Visit the website -> https://example.com]
     [Laptops]
     [Phones] [Tablets]
     [A] [B] [C]
```

- `addURL(label, url)` adds a link on its own row. Clicking it opens the address in the user's browser. It never comes back to the bot.
- `addButton(label, value)` adds one button on its own row.
- `addButton(label1, value1, label2, value2)` adds two buttons side by side, and a call with three pairs adds three. A row never has more than three buttons.
- `message.reply(text, menu)` sends the text with the keyboard. `bot.send(chatId, text, menu)` does the same from outside a handler.

The *label* is what the user sees. The *value* is what the bot receives when the button is clicked. Chapter 9 shows how.

## 9. Buttons: handling clicks

When a user clicks a button, jtelegram does three things, in this order:

1. It **acknowledges** the click, so that the small loading indicator on the button stops at once, however long the handler runs.
2. It turns the click into a `Message` whose `text()` is the button's value, as if the user had typed it.
3. It routes this message like any other: a value that starts with `/` goes to the matching `onCommand` handler, and every `onMessage` handler sees the value as text.

```java
import org.j2os.platform.jtelegram.JTelegram;
import org.j2os.platform.jtelegram.Menu;

public class ButtonClicks {

    public static void main(String[] args) {
        JTelegram bot = JTelegram.create("123456:DEMO_TOKEN");

        bot.onCommand("shop", "Open the shop", message -> message.reply("Choose a category:",
                Menu.create().addButton("Laptops", "cat:laptops", "Phones", "cat:phones")));

        bot.onMessage(message -> {
            System.out.println("log: " + message.text());
            if ("cat:laptops".equals(message.text())) {
                message.reply("Laptops: which kind?", Menu.create()
                        .addButton("Gaming", "kind:gaming", "Office", "kind:office")
                        .addButton("Back", "/shop"));
            } else if ("kind:gaming".equals(message.text())) {
                message.reply("Gaming laptops are listed on our website.");
            }
        });

        bot.start();
    }
}
```

```text
[menu] /shop - Open the shop
Sara: /shop
bot: Choose a category:
     [Laptops] [Phones]
log: /shop
Sara taps [Laptops]
log: cat:laptops
bot (edits the message): Laptops: which kind?
     [Gaming] [Office]
     [Back]
Sara taps [Back]
bot (edits the message): Choose a category:
     [Laptops] [Phones]
log: /shop
Sara taps [Laptops]
log: cat:laptops
bot (edits the message): Laptops: which kind?
     [Gaming] [Office]
     [Back]
Sara taps [Gaming]
log: kind:gaming
bot (edits the message): Gaming laptops are listed on our website.
```

- The `log:` lines show that a click arrives as plain text: `cat:laptops` is the value of the button.
- Values are free text. A value that is not a command, such as `cat:laptops` or `kind:gaming`, is handled in `onMessage` and does not clutter the command menu.
- The value of **Back** is `/shop`, so the click runs the `shop` command handler again, which shows the first menu.
- When a message comes from a button click, `reply` **edits** the message that holds the button instead of sending a new one. The conversation stays tidy, and the menu changes in place. This is the `bot (edits the message)` in the output. Your code does not need to tell the two cases apart: `reply` does the right thing for a typed message and for a click.
- The last reply has no menu, so the buttons disappear.

A click is indistinguishable from typed text. A user can type `support:1001` by hand, and the bot will treat it exactly like the click. Never let a button value alone authorise something that the user may not do, and check the data (here, the order number) as you would check any input.

Telegram refuses an edit that changes nothing. If a click would produce exactly the same text and buttons, the edit fails with a `JTelegramException`. Make sure that every click changes the message.

## 10. Rules and limits

The library checks what it can before Telegram does, so that the mistake is reported at the line that caused it.

```java
import org.j2os.platform.jtelegram.JTelegram;
import org.j2os.platform.jtelegram.Menu;
import org.j2os.platform.jtelegram.Message;

import java.util.function.Consumer;

public class Rules {

    public static void main(String[] args) {
        JTelegram bot = JTelegram.create("123456:DEMO_TOKEN");
        Consumer<Message> handler = message -> { };

        Runnable[] attempts = {
                () -> JTelegram.create(" "),
                () -> bot.onMessage(null),
                () -> bot.onCommand("my-command", "A command", handler),
                () -> bot.onCommand("start", "  ", handler),
                () -> bot.onCommand("start", "d".repeat(257), handler),
                () -> bot.onCommand("start", "Start", null),
                () -> Menu.create().addButton("Empty", ""),
                () -> Menu.create().addButton("Long", "x".repeat(65)),
                () -> {
                    for (int i = 0; i <= 100; i++) {
                        bot.onCommand("c" + i, "Command", handler);
                    }
                }};

        for (Runnable attempt : attempts) {
            try {
                attempt.run();
            } catch (RuntimeException e) {
                String message = e.getMessage();
                System.out.println(message.length() > 100 ? message.substring(0, 100) + "..." : message);
            }
        }
    }
}
```

```text
botToken must not be null or blank
handler must not be null
commandName must be 1-32 lowercase English letters, digits, or underscores; was: my-command
description must not be null or blank
description must be 1-256 characters
handler must not be null
Button value must be 1-64 bytes when UTF-8 encoded (Telegram's callback_data limit), was 0 bytes: 
Button value must be 1-64 bytes when UTF-8 encoded (Telegram's callback_data limit), was 65 bytes: x...
Cannot register more than 100 commands (Telegram's limit)
```

- The token must not be `null` or blank.
- A handler must not be `null`.
- A command name must have 1 to 32 lower case English letters, digits or underscores. The message repeats the name you gave. A `-` is not allowed.
- A description must not be blank and must have 1 to 256 characters.
- A button value must have 1 to 64 **bytes** in UTF-8. This is Telegram's limit for the data of a button. Latin letters take one byte each, but Persian and other letters take two, so a value in Persian has at most 32 letters. The label has no such limit.
- The 101st command is rejected: `Cannot register more than 100 commands (Telegram's limit)`.

All of these, except the last, are `IllegalArgumentException`s. The last is an `IllegalStateException`. Registering handlers is safe while the bot is running, and several threads may register at the same time.

## 11. Starting and stopping

```java
import org.j2os.platform.jtelegram.JTelegram;
import org.j2os.platform.jtelegram.exception.JTelegramException;

public class Lifecycle {

    public static void main(String[] args) {
        JTelegram bot = JTelegram.create("123456:DEMO_TOKEN");
        bot.stop();  // never started: harmless
        bot.onCommand("help", "Show help", message -> message.reply("Ask me anything."));

        bot.start();
        try {
            bot.start();  // already running
        } catch (IllegalStateException e) {
            System.out.println(e.getMessage());
        }
        bot.stop();

        try {
            JTelegram.create("bad-token").start();  // a token that Telegram rejects
        } catch (JTelegramException e) {
            System.out.println(e.getMessage() + " (cause attached: " + (e.getCause() != null) + ")");
        }

        JTelegram.create("123456:DEMO_TOKEN").start();  // no commands: the menu is cleared
    }
}
```

```text
[menu] /help - Show help
This bot is already started; call stop() before starting again
[polling stopped]
Failed to start Telegram bot (cause attached: true)
[menu cleared]
```

- `stop()` on a bot that was never started does nothing.
- `start()` asks Telegram for the bot's own name, registers the polling session and publishes the menu. If the bot is already running it throws `IllegalStateException`. To restart a bot, call `stop()` first.
- `stop()` ends the polling. After it, `start()` can be called again.
- A token that Telegram rejects makes `start()` fail with `JTelegramException`: `Failed to start Telegram bot`. The cause is attached and holds Telegram's own explanation. The bot is stopped, so it can be started again after the token is fixed.
- A bot without commands clears the menu when it starts, as chapter 2 showed.

The calls to `start()` and `stop()` are synchronised against each other. Two threads can never create two polling sessions for one bot.

Every `JTelegram` object is independent, with no shared state. One program can run several bots, each with its own token, and a test can create and discard bots freely.

## 12. In a Spring application

In a Spring application the bot is a *bean*, an object that Spring creates once and shares. Its lifetime follows the application: it starts with it and stops with it. A configuration class does both in a few lines:

```java
import org.j2os.platform.jtelegram.JTelegram;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class BotConfig {

    @Bean(destroyMethod = "stop")                       // polling ends when the application stops
    public JTelegram telegramBot(@Value("${telegram.token}") String token, OrderService orders) {
        JTelegram bot = JTelegram.create(token);

        bot.onCommand("status", "Status of an order", message ->
                message.reply(orders.describe(message.commandArgs())));

        bot.start();
        return bot;
    }
}
```

- The token comes from the property `telegram.token`, not from the code.
- The commands are registered, and `start()` is called before the bean is handed out.
- `destroyMethod = "stop"` makes Spring call `stop()` when the application shuts down.
- Handlers can use other beans, such as `orders` here. They are called from threads of the library, so shared data must be safe for several threads.

Because the bot is a bean, any other bean can use it to send. A service that notifies a customer, for example:

```java
import org.j2os.platform.jtelegram.JTelegram;
import org.springframework.stereotype.Service;

@Service
public class ShipmentNotifier {

    private final JTelegram bot;

    public ShipmentNotifier(JTelegram bot) {
        this.bot = bot;
    }

    public void shipped(long chatId, String orderNumber) {
        bot.send(chatId, "Your order " + orderNumber + " has been shipped.");   // from any code
    }
}
```

`send` needs no handler and no running conversation. It needs only the chat id, which the bot learned when the customer first wrote to it. Store that id, for example in the customer table, and the shop can write to the customer whenever an order changes.

## 13. Putting it together: an order bot

A shop wants a bot with which customers check an order, and a way for a customer to ask for help. The bot shows a button after `/start`, answers `/status 1001`, and offers the buttons *Track another* and *Contact support*. A support request is forwarded to the support team's chat.

```java
import org.j2os.platform.jtelegram.JTelegram;
import org.j2os.platform.jtelegram.Menu;

import java.util.Map;

public class OrderBot {

    public static void main(String[] args) {
        Map<String, String> orders = Map.of("1001", "packed", "1002", "shipped");
        long supportChat = 9000;

        JTelegram bot = JTelegram.create("123456:DEMO_TOKEN");  // read the token from configuration

        bot.onCommand("start", "Start the bot", message -> message.reply(
                "Hello " + message.user().firstName() + "! I can tell you where your order is.",
                Menu.create().addButton("Track an order", "/status")));

        bot.onCommand("status", "Status of an order", message -> {
            String number = message.commandArgs();
            if (number.isEmpty()) {
                message.reply("Send the order number like this: /status 1001");
            } else if (!orders.containsKey(number)) {
                message.reply("There is no order with number " + number + ".");
            } else {
                message.reply("Order " + number + " is " + orders.get(number) + ".",
                        Menu.create().addButton("Track another", "/status",
                                "Contact support", "support:" + number));
            }
        });

        bot.onMessage(message -> {
            if (message.text().startsWith("support:")) {
                String number = message.text().substring("support:".length());
                bot.send(supportChat, message.user().firstName() + " (chat " + message.chatId()
                        + ") needs help with order " + number + ".");
                message.reply("A colleague will contact you soon.");
            }
        });

        bot.start();
    }
}
```

```text
[menu] /start - Start the bot
[menu] /status - Status of an order
Sara: /start
bot: Hello Sara! I can tell you where your order is.
     [Track an order]
Sara taps [Track an order]
bot (edits the message): Send the order number like this: /status 1001
Sara: /status 1001
bot: Order 1001 is packed.
     [Track another] [Contact support]
Sara taps [Contact support]
bot -> chat 9000: Sara (chat 1001) needs help with order 1001.
bot (edits the message): A colleague will contact you soon.
Sara: /status 1002
bot: Order 1002 is shipped.
     [Track another] [Contact support]
Sara taps [Track another]
bot (edits the message): Send the order number like this: /status 1001
Sara: /status 5555
bot: There is no order with number 5555.
```

The program uses the whole module:

1. The commands `start` and `status` are registered with descriptions, so they appear in the menu of the bot.
2. `/start` replies with a button whose value is `/status`. A click on it runs the `status` command without arguments, and `commandArgs()` is empty, so the bot explains how to use it. Because the reply comes from a click, it edits the message.
3. `/status 1001` reads the order number with `commandArgs()`, looks it up and answers. The answer carries two buttons: *Track another*, whose value is the command `/status`, and *Contact support*, whose value `support:1001` carries the order number.
4. The `onMessage` handler recognises the value `support:`. It forwards a message to the support chat with `bot.send(...)`, so a person on the team sees the customer's name, the chat and the order. It then answers the customer, which edits the answered message and removes the buttons.
5. An unknown order number gets a clear answer, with no buttons.

A value can carry data. Here `support:1001` holds both the action and the order number, in far fewer than 64 bytes, so a click needs no memory on the server.

---

## Appendix A. Quick reference

### A.1 JTelegram

| Member | Description |
|--------|-------------|
| `JTelegram.create(token)` | creates a bot; the token is checked by `start()` |
| `onMessage(handler)` | handler for every text message, including commands and button clicks |
| `onCommand(name, description, handler)` | handler for one command; the description appears in the menu |
| `start()` | resolves the bot's name, starts long polling, publishes the menu |
| `stop()` | stops long polling; harmless if the bot is not running |
| `send(chatId, text)` | sends a text to a chat; works without `start()` |
| `send(chatId, text, menu)` | the same, with buttons |

### A.2 Message

| Member | Description |
|--------|-------------|
| `text()` | the text, or the value of a clicked button |
| `chatId()` | the chat to answer in |
| `user()` | the sender, as a `User` |
| `commandArgs()` | everything after the first word; an empty text if there is none |
| `reply(text)` | answers in the chat, or edits the message of a clicked button |
| `reply(text, menu)` | the same, with buttons |

### A.3 User

| Member | Description |
|--------|-------------|
| `id()` | the Telegram id of the user |
| `firstName()` | first name |
| `lastName()` | last name; can be `null` |
| `username()` | username; can be `null` |

### A.4 Menu

| Member | Description |
|--------|-------------|
| `Menu.create()` | an empty keyboard |
| `addURL(label, url)` | a link button on its own row |
| `addButton(label, value)` | one button on a row |
| `addButton(l1, v1, l2, v2)` | two buttons on a row |
| `addButton(l1, v1, l2, v2, l3, v3)` | three buttons on a row |

Every method returns the menu, so the calls chain.

### A.5 JTelegramException

`JTelegramException` extends `RuntimeException`. It is thrown with the message `Failed to start Telegram bot`, `Failed to send message to chat <id>`, `Failed to edit message <id> in chat <id>` or `Failed to acknowledge button click`, and carries the error of the telegrambots library as its cause.

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| a `null` or blank token, handler, command name or description | `IllegalArgumentException` |
| a command name or description that breaks the rules | `IllegalArgumentException` |
| a button value of 0 bytes or more than 64 bytes | `IllegalArgumentException` |
| the 101st command | `IllegalStateException` |
| `start()` on a running bot | `IllegalStateException` |
| `start()` with a token that Telegram rejects, or without a network | `JTelegramException`; the bot is stopped |
| `send` that Telegram rejects (unknown chat, a user who has not started the bot, an empty text) | `JTelegramException` |
| a `reply` to a button click that Telegram rejects (for example an edit that changes nothing) | `JTelegramException` |
| the acknowledgement of a click fails | `JTelegramException`, thrown after the handlers have run |
| a handler throws | the other handlers still run; the first exception is rethrown after them |
| `stop()` on a bot that is not running | no effect |
| `send(chatId, text, null)` | `NullPointerException` |
| an update that is not text (photo, sticker, file, ...) | ignored |
| a click that comes from an inline-mode message | ignored |
| a command that names another bot (`/start@other_bot`) | ignored |
| a command that is not registered | only the `onMessage` handlers run |

The messages are listed in Appendix D.

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **The token is a password.** Do not put it in source code, logs or a repository. Anyone who has it controls the bot. If it leaks, create a new one with BotFather.
- **Register commands before `start()`.** The menu is published once, at start. A later command works but is not listed.
- **A catch-all handler answers too.** A message that matches a command also reaches every `onMessage` handler. Do not reply in both unless you want two answers.
- **A click must change the message.** An edit that changes neither the text nor the buttons is refused by Telegram.
- **A button value can be typed by hand.** The bot cannot tell a click from typed text, so validate the value like any other input.
- **Button values are limited.** 64 bytes, and Persian letters take two bytes each. Put an identifier in the value and keep the data on the server.
- **Handlers run on library threads.** Do not block them for long, and protect data that is shared between handlers.
- **A bot cannot write first.** `send` fails for a user who has never started a conversation with the bot.
- **`send(chatId, text, null)` fails.** The menu must be an object. Call the version without a menu to send plain text.
- **Do not call `start()` twice.** Stop the bot first.

### C.2 Behaviour worth knowing

- The matching of commands ignores case, an `@botname` suffix that names this bot, and the arguments after the first space.
- A command is recognised only at the start of the message.
- `commandArgs()` returns everything after the first whitespace, in one piece: for `/weather New York` it is `New York`.
- All `onMessage` handlers run in registration order, after the command handler.
- A button click is acknowledged before its handler runs, so a slow or failing handler never leaves the loading indicator on the button.
- `reply` edits the message of a button click and sends a new message for typed text.
- The bot learns its own username in `start()`. While the username is unknown, an `@botname` suffix is not checked.
- `send` and `start` are independent: a program can send without polling, and poll without ever sending.

### C.3 Limits

- **Text only.** No photos, files, stickers, locations, voice, polls or payments, in either direction.
- **No formatting.** Messages are plain text. There is no Markdown or HTML mode.
- **Long polling only.** There is no webhook mode.
- **Inline keyboards only.** There are no reply keyboards, and clicks from inline-mode messages are ignored.
- **At most three buttons per row** in a single call, 64 bytes per value, 100 commands, 32 characters per command name, 256 per description. Most of these are Telegram's own limits.
- **No answers to callback alerts.** A click is acknowledged without text; there is no way to show a pop-up message.
- **One session per object.** To run two bots, create two `JTelegram` objects.

### C.4 Tuning

- Read the token from configuration, and register all commands before `start()`.
- Use commands for what the user should find in the menu, and button values such as `cat:laptops` for the rest.
- Let each click change the message, and use `reply` so that the chat stays tidy.
- Keep handlers short. Do slow work in another thread or service and answer when it is done.
- Store the chat id of every customer who writes to the bot, so that you can notify them later with `send`.
- In a Spring application, make the bot a bean with `destroyMethod = "stop"`.

## Appendix D. Message reference

**IllegalArgumentException**

| Message | Meaning |
|---------|---------|
| `botToken must not be null or blank` | `create` receives no token |
| `handler must not be null` | `onMessage` or `onCommand` receives `null` |
| `commandName must not be null or blank` | the command name is empty |
| `description must not be null or blank` | the description is empty |
| `description must be 1-256 characters` | the description is longer than 256 characters |
| `commandName must be 1-32 lowercase English letters, digits, or underscores; was: <name>` | the command name breaks the rules |
| `Button value must be 1-64 bytes when UTF-8 encoded (Telegram's callback_data limit), was <n> bytes: <value>` | the button value is empty or too long |

**IllegalStateException**

| Message | Meaning |
|---------|---------|
| `Cannot register more than 100 commands (Telegram's limit)` | the 101st command |
| `This bot is already started; call stop() before starting again` | `start()` on a running bot |

**JTelegramException**

| Message | Meaning |
|---------|---------|
| `Failed to start Telegram bot` | `start()` cannot reach Telegram or the token is rejected |
| `Failed to send message to chat <id>` | `send` or a `reply` to a typed message fails |
| `Failed to edit message <id> in chat <id>` | the edit of a button-click reply fails |
| `Failed to acknowledge button click` | Telegram does not accept the acknowledgement of a click |

The error of the telegrambots library is always attached as the cause.
