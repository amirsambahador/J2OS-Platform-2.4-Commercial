# jterminal

*J2OS Platform 2.4*

Sooner or later a Java program has to run something outside Java: a backup script, a database client, a command on a remote machine. The usual tools start one process for one command and give back its output. That is enough for `ls`, but it breaks as soon as the work has several steps. A directory changed in the first command is forgotten by the second, a variable set in one call is gone in the next, and a program that asks a question cannot be answered.

**jterminal** (`org.j2os.platform.jterminal`) solves this by opening a real, persistent terminal. A program types into it exactly as a person would, and reads what appears on the screen. The shell stays alive between the calls, so it remembers its directory, its variables and its history, and any interactive program (a shell, `psql`, `sqlplus`, `ssh`) can be driven from Java.

```java
try (JTerminal terminal = new JTerminal()) {
    terminal.type("ls -l").enter();           // type a command and press Enter
    System.out.print(terminal.output());      // read what the shell printed
}
```

The module has three public types:

- `JTerminal` is the terminal. It types, presses Enter, reads output and shuts down.
- `ControlKey` lists the special keys that can be sent: `Ctrl+C`, `Ctrl+D`, the arrow keys and others.
- `JTerminal.JTerminalException` is the exception thrown when the terminal cannot start or cannot be written to.

This guide goes step by step: each chapter has a short program followed by the output it prints. The appendices are for quick reference.

## Contents

1. [What a terminal session is](#1-what-a-terminal-session-is)
2. [A first session](#2-a-first-session)
3. [The shell remembers](#3-the-shell-remembers)
4. [Reading output](#4-reading-output)
5. [Knowing when a command has finished](#5-knowing-when-a-command-has-finished)
6. [Commands that never end](#6-commands-that-never-end)
7. [Control keys and raw bytes](#7-control-keys-and-raw-bytes)
8. [Programs that ask questions](#8-programs-that-ask-questions)
9. [Clean text](#9-clean-text)
10. [Input rules and failures](#10-input-rules-and-failures)
11. [Starting and stopping](#11-starting-and-stopping)
12. [In a web application](#12-in-a-web-application)
13. [Putting it together: a maintenance run](#13-putting-it-together-a-maintenance-run)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The module depends on the *pty4j* library, which provides the pseudo-terminal and contains native code for each platform. It runs on Linux and on Windows. The examples in this guide use the Linux shell (`bash`) and were written for a Linux machine; on Windows the same Java code drives `cmd.exe`, and only the commands you type change.

---

## 1. What a terminal session is

A *pseudo-terminal*, or *pty*, is a software imitation of the screen and keyboard of an old terminal. A program started inside it believes that a person is sitting in front of it. This is why interactive programs work: they print prompts, wait for typed answers, and understand keys such as `Ctrl+C`.

`JTerminal` starts a shell in a pty (`bash -i` on Linux, `cmd.exe` on Windows) in the working directory of the Java process, and from then on does two things:

- **Writing.** `type(...)` sends characters to the keyboard, and `enter()` presses Enter. Nothing runs until Enter is pressed, just as in a real terminal.
- **Reading.** A background thread collects everything the shell prints. `output()` hands over the text that is new since the last call.

Three facts follow from this design, and the rest of the guide returns to them.

**Output is what a screen would show.** The terminal echoes what you type, so the output starts with your own command, and it ends with the next prompt. Lines end with `\r\n`.

**The end of a command is not visible.** A terminal does not announce "finished". `output()` has to guess that the command is done by waiting for a short silence. Chapter 4 explains how, and chapter 5 shows a reliable alternative.

**The shell is shared by all calls.** Whatever one command does to the shell remains for the next.

## 2. A first session

```java
import org.j2os.platform.jterminal.JTerminal;

public class FirstSession {

    public static void main(String[] args) {
        try (JTerminal terminal = new JTerminal()) {
            System.out.println("start-up output is not empty: " + !terminal.output().isEmpty());

            terminal.type("PS1='$ '").enter();  // a short prompt, the same on every machine
            terminal.output();

            terminal.type("echo hello").enter();
            String text = terminal.output();
            System.out.println(text.replace("\r", "\\r").replace("\n", "\\n\n"));
        }
    }
}
```

```text
start-up output is not empty: true
echo hello\r\n
\rhello\r\n
$ 
```

- `JTerminal` is `AutoCloseable`, so `try-with-resources` closes it and ends the shell, even if an exception is thrown.
- The first `output()` returns what the shell printed at start-up: its first prompt. The prompt differs from machine to machine (it usually holds the user name and the directory), so the program sets its own, short prompt with `PS1='$ '` and reads, and ignores, the echo of that line. This keeps the output of the guide the same everywhere. A real program does not have to do it.
- `type("echo hello").enter()` types the command and presses Enter. Both methods return the terminal, so they chain.
- The output has three parts: the typed command (echoed by the terminal), the result `hello`, and the new prompt `$ `. The program prints `\r` and `\n` as visible text to show that every line ends with `\r\n`.

## 3. The shell remembers

Because one shell lives through all the calls, the effect of a command stays.

```java
import org.j2os.platform.jterminal.JTerminal;

public class ShellState {

    public static void main(String[] args) {
        try (JTerminal terminal = new JTerminal()) {
            terminal.output();
            terminal.type("PS1='$ '").enter();
            terminal.output();

            terminal.type("cd /tmp").enter();
            terminal.output();
            terminal.type("export COLOR=blue").enter();
            terminal.output();

            terminal.type("pwd").enter();  // a later command still sees the new directory
            System.out.print(terminal.output());
            terminal.type("echo $COLOR").enter();  // and the variable
            System.out.print(terminal.output());
        }
    }
}
```

```text
pwd
/tmp
$ echo $COLOR
blue
$ 
```

- `cd /tmp` changes the directory of the shell, and `pwd` in a later call reports `/tmp`.
- `export COLOR=blue` sets a variable, and `echo $COLOR` in a later call sees it.
- Each text ends with the prompt of the next line, so the echo of the following command continues on the same line as `$ `. A real terminal looks the same, and the next chapters show how to cut it up.

A program started with `Runtime.exec` forgets both settings after the first command. This is the main reason to use a terminal.

## 4. Reading output

`output()` returns the text that arrived since the last call, and *consumes* it. The next call starts from where this one stopped.

```java
import org.j2os.platform.jterminal.JTerminal;

public class ReadingOutput {

    public static void main(String[] args) {
        try (JTerminal terminal = new JTerminal()) {
            terminal.output();
            terminal.type("PS1='$ '").enter();
            terminal.output();

            terminal.type("echo one").enter();
            System.out.println("first:  " + terminal.output().replace("\r\n", " | "));
            System.out.println("again:  [" + terminal.output(100) + "]");  // nothing new has arrived

            terminal.type("sleep 1; echo two").enter();
            System.out.println("early:  " + terminal.output().replace("\r\n", " | "));
            System.out.println("late:   " + terminal.output().replace("\r\n", " | "));
        }
    }
}
```

```text
first:  echo one | one | $ 
again:  []
early:  sleep 1; echo two | 
late:   two | $ 
```

- `output()` waits for the first piece of text to arrive, then keeps waiting until the terminal has been silent for 150 milliseconds, and takes that as "the command has finished".
- The second line shows an empty result. Nothing new has arrived, so `output(100)` waits at most 100 milliseconds and returns an empty text. Chapter 6 explains this overload.
- The last two lines show the weak point of the silence rule. The command `sleep 1; echo two` is quiet for a whole second before it prints. The first call sees only the echo and gives up after 150 milliseconds. The result `two` arrives later and comes out of the **next** call.

The consequence is important: a call to `output()` may return less than the command has produced, and the rest then appears in a later call. A program must never assume that one call returns one whole command. For commands that print slowly, use the method of the next chapter.

## 5. Knowing when a command has finished

A reliable way to detect the end of a command is to ask the shell to say so. After the command, add `echo` with a marker word and the exit status `$?`, and read until the marker appears:

```java
import org.j2os.platform.jterminal.JTerminal;

import java.time.Duration;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class WaitForMarker {

    public static void main(String[] args) {
        try (JTerminal terminal = new JTerminal()) {
            terminal.output();
            terminal.type("PS1='$ '").enter();
            terminal.output();

            terminal.type("echo start; sleep 1; echo end; echo __END__$?").enter();

            StringBuilder text = new StringBuilder();
            Matcher done = Pattern.compile("__END__(\\d+)").matcher(text);
            while (!done.reset().find()) {
                text.append(terminal.output(Duration.ofMillis(300)));
            }

            System.out.println(text.toString().replace("\r\n", " | "));
            System.out.println("exit code: " + done.group(1));
        }
    }
}
```

```text
echo start; sleep 1; echo end; echo __END__$? | start | end | __END__0 | $ 
exit code: 0
```

- The typed line ends with `; echo __END__$?`. When the command is over, the shell prints `__END__` followed by its exit status, here `0` for success.
- The terminal also echoes the typed line, which contains the text `__END__$?`. The pattern `__END__(\d+)` needs digits after the marker, so the echo does not match it. Only the real marker does.
- The loop reads in slices of 300 milliseconds and appends them to a text, so a slow command and a split output do no harm. It ends when the marker is found, and `group(1)` is the exit status.
- This also solves the problem of chapter 4: the program waits as long as the command needs, however long its pauses.

The technique works for any shell command and gives, besides the end, the exit status. Chapter 13 builds a complete program on it.

## 6. Commands that never end

Some commands never finish by themselves: `ping` without a count, `tail -f`, a monitoring loop. `output()` waits until the terminal falls silent, which never happens, so it blocks forever. For these commands there are two overloads that wait for a limited time: `output(Duration)` and `output(long)` (milliseconds).

```java
import org.j2os.platform.jterminal.ControlKey;
import org.j2os.platform.jterminal.JTerminal;

import java.time.Duration;

public class Streaming {

    public static void main(String[] args) {
        try (JTerminal terminal = new JTerminal()) {
            terminal.output();
            terminal.type("PS1='$ '").enter();
            terminal.output();

            terminal.type("while true; do echo tick; sleep 0.3; done").enter();

            StringBuilder seen = new StringBuilder();
            for (int i = 0; i < 5; i++) {
                seen.append(terminal.output(Duration.ofMillis(200)));  // wait at most 200 ms
            }
            System.out.println("ticks arrived: " + seen.toString().contains("tick"));

            terminal.type(ControlKey.CTRL_C);  // stop the loop
            System.out.println("interrupted: " + terminal.output().contains("^C"));
        }
    }
}
```

```text
ticks arrived: true
interrupted: true
```

- `output(Duration.ofMillis(200))` waits at most 200 milliseconds for new text. As soon as something has arrived it returns that, without waiting for silence. If nothing arrives it returns an empty text. It never blocks for longer than the time given.
- The loop collects five slices, so about one second of the stream.
- `type(ControlKey.CTRL_C)` sends `Ctrl+C`, the key that interrupts a running command. The terminal shows `^C`, and the shell returns to its prompt.

Use the limited overloads whenever you are not sure that a command ends, and in any loop that reads while a program is running.

## 7. Control keys and raw bytes

`type(ControlKey)` sends a key that is not a printable character. The enum contains `CTRL_A` to `CTRL_Z`, `ESCAPE`, `BACKSPACE` and the four arrow keys. The raw version, `type(byte...)`, writes any bytes you give.

```java
import org.j2os.platform.jterminal.ControlKey;
import org.j2os.platform.jterminal.JTerminal;

public class ControlKeys {

    public static void main(String[] args) throws InterruptedException {
        try (JTerminal terminal = new JTerminal()) {
            terminal.output();
            terminal.type("PS1='$ '").enter();
            terminal.output();

            terminal.type("echo one").enter();
            terminal.output();

            terminal.type(ControlKey.ARROW_UP).enter();  // recalls "echo one" from the history
            System.out.print(terminal.output());

            terminal.type(new byte[]{'e', 'c', 'h', 'o', ' ', 'h', 'i'}).enter();  // raw bytes: "echo hi"
            System.out.print(terminal.output());

            terminal.type(ControlKey.CTRL_D);  // end of input: the shell exits
            terminal.output();
            Thread.sleep(200);
            System.out.println("\nalive: " + terminal.isAlive());
        }
    }
}
```

```text
echo one
one
$ echo hi
hi
$ 
alive: false
```

- `ARROW_UP` recalls the previous command from the shell's history. Pressing Enter runs it again, and the output shows `one` a second time.
- `type(byte...)` writes the bytes as they are. The letters of `echo hi` are sent without any text conversion.
- `CTRL_D` means "end of input". On an empty prompt it makes the shell exit, and `isAlive()` becomes `false`.

The most useful keys are listed in Appendix A. A key is not followed by Enter automatically: `type(ControlKey.CTRL_C)` interrupts at once, and `ARROW_UP` only changes the line that is being edited until you call `enter()`.

## 8. Programs that ask questions

Everything typed goes to whichever program is reading the keyboard at that moment. If a program waits for an answer, the next `type(...).enter()` is that answer.

```java
import org.j2os.platform.jterminal.JTerminal;

public class Interactive {

    public static void main(String[] args) {
        try (JTerminal terminal = new JTerminal()) {
            terminal.output();
            terminal.type("PS1='$ '").enter();
            terminal.output();

            terminal.type("read -p 'Your name? ' name; echo Hello, $name").enter();
            System.out.println("program says: " + terminal.output().replace("\r\n", " | "));

            terminal.type("Sara").enter();  // the answer goes to the waiting program
            System.out.println("program says: " + terminal.output().replace("\r\n", " | "));
        }
    }
}
```

```text
program says: read -p 'Your name? ' name; echo Hello, $name | Your name? 
program says: Sara | Hello, Sara | $ 
```

- The command `read -p 'Your name? ' name` prints a question and waits. The first `output()` returns the echo and the question `Your name? `, and the program stays waiting.
- `type("Sara").enter()` is the answer. The same shell line then continues, and the second `output()` shows the greeting and the new prompt.

The same pattern drives a database client or a remote login: start the program with `type("psql mydb").enter()`, read the prompt, send each statement, read the answer, and finish with the program's own exit command. Keep in mind that a program decides itself when to print its prompt, so the markers of chapter 5 are the safest way to find where an answer ends.

## 9. Clean text

A terminal does not print only text. It prints *escape sequences*, hidden codes that set colors, move the cursor or change the window title. A program such as `ls` or `git` colors its output when it believes that a person is watching, and in a pty it does. These codes are meaningless to a program that wants the text.

```java
import org.j2os.platform.jterminal.JTerminal;

public class CleanOutput {

    public static void main(String[] args) {
        try (JTerminal terminal = new JTerminal()) {
            terminal.output();
            terminal.type("PS1='$ '").enter();
            terminal.output();

            terminal.type("printf '\\033[31mred\\033[0m and plain\\n'").enter();  // red text
            System.out.print(terminal.output());

            terminal.type("echo سلام دنیا").enter();  // non-Latin text
            System.out.print(terminal.output());
        }
    }
}
```

```text
printf '\033[31mred\033[0m and plain\n'
red and plain
$ echo سلام دنیا
سلام دنیا
$ 
```

- The command prints `red` in red, with a code before and after it. The output holds only `red and plain`: `JTerminal` removes color and formatting codes, cursor movements, window-title codes and similar sequences before it hands the text over.
- The echo of the command still shows its own text exactly as typed, because that is what was typed.
- Text in other scripts works as it does in Latin. Input and output are encoded as UTF-8, and a character that arrives split into two pieces is put back together.

Not everything is removed. Plain control characters, such as the carriage return that ends every line, remain in the text.

## 10. Input rules and failures

`type(String)` accepts one line of text and nothing else.

```java
import org.j2os.platform.jterminal.JTerminal;

public class InputRules {

    public static void main(String[] args) {
        JTerminal terminal = new JTerminal();
        String[] texts = {null, "ls\nrm file", "a\0b"};
        for (String text : texts) {
            try {
                terminal.type(text);
            } catch (IllegalArgumentException e) {
                System.out.println(e.getMessage());
            }
        }

        terminal.close();
        try {
            terminal.type("ls");
        } catch (JTerminal.JTerminalException e) {
            System.out.println(e.getMessage());
        }
    }
}
```

```text
Input text must not be null
type(String) input must not contain a newline; use enter() to send Enter
Input text must not contain a null byte
Failed to write to terminal
```

- `null` is rejected.
- A text with a line break is rejected, because a line break would press Enter in the middle of your text. To send several lines, call `type(...)` and `enter()` for each line, so that every Enter is a visible, deliberate step.
- A null character (`\0`) is rejected, because it cannot be typed.
- All three are `IllegalArgumentException`, thrown before anything is written, so the terminal is unchanged.
- Writing to a closed terminal fails with `JTerminal.JTerminalException`, here with the message `Failed to write to terminal`. This exception also reports a failed start (`Failed to start terminal`), for example when the native library of pty4j is missing.

`type(ControlKey)` and `type(byte...)` do no checks, because every byte is allowed.

## 11. Starting and stopping

```java
import org.j2os.platform.jterminal.JTerminal;

public class Lifecycle {

    public static void main(String[] args) throws InterruptedException {
        JTerminal terminal = new JTerminal();
        terminal.output();
        System.out.println("alive after start: " + terminal.isAlive());

        terminal.type("exit").enter();  // the shell ends by itself
        terminal.output();
        Thread.sleep(200);
        System.out.println("alive after exit:  " + terminal.isAlive());
        System.out.println("output after exit: [" + terminal.output() + "]");  // returns at once

        terminal.close();  // safe to call anyway
    }
}
```

```text
alive after start: true
alive after exit:  false
output after exit: []
```

- The constructor starts the shell. `isAlive()` tells whether the shell is still running.
- When the shell ends by itself, for example after `exit` or `Ctrl+D`, `isAlive()` becomes `false`. A moment may pass between the last output and the change, so a loop that waits for the shell to end should poll `isAlive()` instead of reading it once.
- When the shell has ended and nothing is pending, `output()` returns an empty text at once. It does not wait for text that can never arrive, so a loop such as `while (terminal.isAlive())` ends cleanly.
- `close()` ends the shell by force, closes its input and stops the reader thread. It is safe to call after the shell has ended, and it is called automatically by `try-with-resources`.

A terminal owns a native process and a thread. Always close it, and do not keep unused terminals open.

## 12. In a web application

A browser terminal, an administration console or a job page can use `JTerminal` through a REST service. A session is created once, kept on the server under an identifier, written to by one request and read by another, and closed at the end:

```java
import jakarta.annotation.PreDestroy;
import org.j2os.platform.jterminal.JTerminal;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.server.ResponseStatusException;

import java.time.Duration;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

@RestController
@RequestMapping("/api/terminal")
public class TerminalController {

    private final Map<String, JTerminal> sessions = new ConcurrentHashMap<>();

    @PostMapping
    public Map<String, String> open() {
        JTerminal terminal = new JTerminal();
        String id = UUID.randomUUID().toString();
        sessions.put(id, terminal);
        return Map.of("id", id);
    }

    @PostMapping("/{id}/input")
    public void input(@PathVariable String id, @RequestBody Map<String, String> body) {
        JTerminal terminal = find(id);
        synchronized (terminal) {                       // one writer at a time
            terminal.type(body.get("text")).enter();
        }
    }

    @GetMapping("/{id}/output")
    public Map<String, String> output(@PathVariable String id) {
        return Map.of("text", find(id).output(Duration.ofSeconds(1)));   // never waits longer
    }

    @DeleteMapping("/{id}")
    public void close(@PathVariable String id) {
        JTerminal terminal = sessions.remove(id);
        if (terminal != null) {
            terminal.close();
        }
    }

    @PreDestroy
    public void closeAll() {
        sessions.values().forEach(JTerminal::close);    // no shell outlives the application
    }

    private JTerminal find(String id) {
        JTerminal terminal = sessions.get(id);
        if (terminal == null) {
            throw new ResponseStatusException(HttpStatus.NOT_FOUND);
        }
        return terminal;
    }
}
```

- `open()` creates a terminal and returns its identifier. Each user session needs its own terminal, and the identifier is the handle.
- `input(...)` types a line and presses Enter. The `synchronized` block makes sure that two requests never write to one terminal at the same moment. `JTerminal` allows only one writer at a time.
- `output(...)` reads with a limit of one second, so the request always ends. The page asks again and again, as long as the user watches the terminal, and appends the text that comes back.
- `close(...)` and `closeAll()` end the shells. The second one is called when the application stops, so that no shell stays behind.

**Security.** A terminal gives its user the full rights of the account under which the application runs. Place such a service behind authentication and authorisation, allow only trusted administrators, log every command, and run the application with an account that can do no more than it must.

## 13. Putting it together: a maintenance run

A maintenance job has to run several shell steps in order: go to a fresh directory, create a report, check its size, read another file. If a step fails, the job must stop and report why. The program runs all steps in one shell and uses the marker of chapter 5 to learn when each step ends and how it ended.

```java
import org.j2os.platform.jterminal.JTerminal;

import java.time.Duration;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class MaintenanceRun {

    public static void main(String[] args) {
        String[] steps = {
                "cd $(mktemp -d)",
                "echo 'report' > report.txt",
                "wc -c report.txt",
                "cat missing.txt",
                "echo never reached"};

        try (JTerminal terminal = new JTerminal()) {
            terminal.output();
            terminal.type("PS1='$ '").enter();
            terminal.output();

            for (int i = 0; i < steps.length; i++) {
                terminal.type(steps[i] + "; echo __END__$?").enter();

                StringBuilder text = new StringBuilder();
                Matcher end = Pattern.compile("__END__(\\d+)").matcher(text);
                while (!end.reset().find()) {
                    text.append(terminal.output(Duration.ofMillis(300)));
                }

                int exitCode = Integer.parseInt(end.group(1));
                System.out.println("step " + (i + 1) + ": exit code " + exitCode);
                if (exitCode != 0) {
                    String message = text.substring(text.indexOf("\n") + 1, end.start()).trim();
                    System.out.println("stopped: " + message);
                    break;
                }
            }
        }
    }
}
```

```text
step 1: exit code 0
step 2: exit code 0
step 3: exit code 0
step 4: exit code 1
stopped: cat: missing.txt: No such file or directory
```

The program uses the whole module:

1. One `JTerminal` in a `try-with-resources` block runs every step in the same shell. Step 1 changes into a new temporary directory, and steps 2 to 4 work in it, which would be impossible with separate processes.
2. After each command the program appends `; echo __END__$?` and reads with a limit of 300 milliseconds until the marker appears, so a slow step cannot be cut in half.
3. The number after the marker is the exit status. Zero means success. Step 4 reads a file that does not exist, so it ends with status 1.
4. For a failed step the program takes the text between the echoed command line and the marker, which is the message of the command, reports it and stops. Step 5 never runs.

The same loop fits a deployment, a backup or a database script. Only the list of steps changes.

---

## Appendix A. Quick reference

### A.1 JTerminal

| Member | Description |
|--------|-------------|
| `new JTerminal()` | starts a shell in a pty, in the working directory of the Java process |
| `type(String)` | types one line of text, without Enter; returns the terminal |
| `type(ControlKey)` | sends a control key; returns the terminal |
| `type(byte...)` | writes raw bytes; returns the terminal |
| `enter()` | presses Enter; returns the terminal |
| `output()` | waits for new text and for 150 ms of silence; returns and consumes the new text |
| `output(Duration)` | waits at most that long for new text; returns what has arrived |
| `output(long)` | the same, with the time in milliseconds |
| `isAlive()` | whether the shell is running |
| `close()` | ends the shell, closes the input and stops the reader thread |

`JTerminal` implements `AutoCloseable`.

### A.2 ControlKey

| Keys | Meaning |
|------|---------|
| `CTRL_A` to `CTRL_Z` | `Ctrl` plus the letter; the codes are 1 to 26 |
| `CTRL_C` | interrupts the running command |
| `CTRL_D` | end of input; exits an idle shell |
| `CTRL_Z` | suspends the running command |
| `CTRL_L` | clears the screen |
| `CTRL_A` / `CTRL_E` | start / end of the line being edited |
| `CTRL_U` / `CTRL_W` | deletes the line / the last word |
| `CTRL_I` | Tab (completion) |
| `CTRL_J`, `CTRL_M` | line feed, carriage return (Enter) |
| `ESCAPE` | the Escape key |
| `BACKSPACE` | deletes the character before the cursor |
| `ARROW_UP`, `ARROW_DOWN` | previous / next command in the history |
| `ARROW_LEFT`, `ARROW_RIGHT` | moves the cursor |

The effect of the letters depends on the program that reads them. The meanings above are those of the shell.

### A.3 JTerminalException

`JTerminal.JTerminalException` extends `RuntimeException`. It is thrown with the message `Failed to start terminal` or `Failed to write to terminal`, and carries the original `IOException` as its cause.

## Appendix B. Failure handling

| Situation | Result |
|-----------|--------|
| the pty or the shell cannot be started | `JTerminalException`: `Failed to start terminal` |
| `type(String)` with `null`, a line break or a null character | `IllegalArgumentException`; nothing is written |
| writing to a closed terminal | `JTerminalException`: `Failed to write to terminal` |
| `output()` when the shell has ended and nothing is pending | empty text, at once |
| `output()` on a command that never goes quiet | does not return; use `output(Duration)` |
| `output()` when nothing ever arrives (a command that prints nothing and has no prompt) | does not return; use `output(Duration)` |
| `output(Duration)` when nothing arrives in time | empty text |
| the reading thread is interrupted | `output()` returns what has arrived and leaves the interrupt flag set |
| `close()` called twice, or after the shell ended | no effect, no exception |

The only exceptions that jterminal throws on purpose are `IllegalArgumentException` for the input rules of `type(String)` and `JTerminalException` for a pty that cannot start or be written to. A command that fails inside the shell is not an exception: it shows as text, and as an exit status that you read with `echo $?`.

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **One call is not one command.** The silence rule can cut a slow command in two (chapter 4). Use a marker (chapter 5) when the whole answer matters.
- **Unread output waits for the next call.** Text that nobody reads is returned by a later `output()`, and it looks like the answer to the wrong command. Read the output after every command, or discard it on purpose.
- **`output()` can wait forever.** It does so on a command that never goes quiet, and on a command that prints nothing when the prompt is also empty. Use `output(Duration)` when in doubt.
- **The echo is part of the output.** The text starts with the typed command and ends with the next prompt. Strip them, or look only for what you need.
- **The prompt differs between machines.** Do not compare output with a fixed prompt. Set your own prompt, as in chapter 2, or search for a marker.
- **Some programs redraw the line while you type.** An editor-like prompt, such as the one of a REPL (a program that reads one statement at a time and prints its result), can redraw its line after every character, and the redraws end up in the output. For such programs search the output for the answer instead of comparing it as a whole.
- **Newlines in `type(String)` are an error.** Send several lines as several `type(...).enter()` calls.
- **Dangerous commands run for real.** A terminal has the rights of the Java process. Never build the typed text from unchecked user input.

### C.2 Behaviour worth knowing

- The silence that ends `output()` is **150 milliseconds**.
- The output buffer is emptied by every read, so memory does not grow with the total amount of output. A single read can return a large amount of text at once.
- An escape sequence that has only just started to arrive is held back for a moment so that it is not returned cut in half.
- The text is decoded as UTF-8 incrementally, so a multi-byte character split between two reads stays intact.
- The shell's standard output and its error output arrive together, in the order in which they were printed, because both go to the same pty.
- Reading from one thread while another thread writes is safe.
- On Windows the constructor first adds the System32 folder to `PATH` if it is missing, then switches the console to UTF-8 (`chcp 65001`) and discards the output of that step, so the first `output()` of the program starts clean.

### C.3 Limits

- **One writer at a time.** Concurrent calls of `type` may interleave their bytes. Do the writes one at a time, as the controller of chapter 12 does.
- **One reader thread per terminal.** Each instance starts a background thread that lives as long as its pty. Many open terminals mean many threads and many processes.
- **The end of a command is a guess.** The fixed 150 ms window cannot be changed. A command that pauses longer may be split.
- **Native library.** pty4j needs its native library on the host platform. If it is missing the constructor throws `JTerminalException`.
- **The shell is fixed.** `bash -i` on Linux and `cmd.exe /D` on Windows. There is no parameter to choose another shell or directory; the directory is that of the Java process, and another shell is started by typing its name.
- **No API for terminal size or signals.** The size is not configurable, and keys such as `Ctrl+C` are the way to send signals.

### C.4 Tuning

- Put the terminal in a `try-with-resources` block.
- Wait for the end of a command with a marker, and use `output(Duration)` in loops.
- Check the exit status (`echo $?`) of every step that matters.
- Write one command at a time, and give each user or job its own terminal.
- Protect any service that exposes a terminal with authentication, and log what is typed.

## Appendix D. Message reference

The messages below are the texts of the exceptions that jterminal throws.

| Message | Exception | Meaning |
|---------|-----------|---------|
| `Input text must not be null` | `IllegalArgumentException` | `null` is passed to `type(String)` |
| `type(String) input must not contain a newline; use enter() to send Enter` | `IllegalArgumentException` | the text contains `\n` or `\r` |
| `Input text must not contain a null byte` | `IllegalArgumentException` | the text contains `\0` |
| `Failed to start terminal` | `JTerminalException` | the pty or the shell cannot be started; the cause is attached |
| `Failed to write to terminal` | `JTerminalException` | a write to the terminal fails, for example after `close()`; the cause is attached |
