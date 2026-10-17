# jfile

*J2OS Platform 2.4*

Java's own file classes are powerful but wordy. Reading a few bytes from the middle of a file, zipping a folder safely, downloading a large file over several connections or finding out when something changes in a directory each take dozens of lines of `Files`, `FileChannel`, `ZipOutputStream`, `HttpClient` and `WatchService`, and each has traps: a half-written archive left behind after a failure, an archive entry that writes outside the target folder, a character cut in half at the edge of a read.

**jfile** (`org.j2os.platform.jfile`) wraps all of that in one small class, `JFile`. One object stands for **one location** (a file, a folder or a web address), and its methods read, write, copy, move, delete, list, search, zip, download and watch. Most methods return the same object, so calls chain. Three ideas shape it:

- **One object, one location.** `new JFile("reports/a.txt")` is bound to that path. Methods like `move` and `back` change where the object points; to work with an unrelated file, create another `JFile`.
- **Text is UTF-8, and characters are never cut.** Reads and writes by byte position are moved to the nearest character boundary, so a Persian or any other multi-byte letter is not split at the edge.
- **One unchecked exception.** Every I/O failure becomes a `JFileException`, so there is nothing to declare and a single `catch` is enough.

A first taste:

```java
JFile report = new JFile("reports/2026-10.txt");           // relative to the working directory
report.write("Total: 120\n");                               // creates the folder and the file
System.out.println(report.lineCount() + " line, " + report.size(JFile.BYTE) + " bytes");

new JFile("reports").zip("backup/reports.zip");            // a whole folder into one archive
new JFile("https://example.com/big.iso").download("downloads/big.iso", 4);   // 4 parallel connections

new JFile("incoming").setListener(event ->                  // react to what happens in a folder
        System.out.println(event.action() + " " + event.relativePath()));
```

The guide goes through the class in the order a program usually needs it: how a location is understood (chapter 1), inspecting (2), reading (3), writing (4), saving objects (5), copying, moving and deleting (6), listing and searching (7), zip files (8), downloading (9), watching for changes (10), failures (11) and a complete example (12). The appendices are reference material.

## Contents

1. [How JFile works](#1-how-jfile-works)
2. [Looking at a file](#2-looking-at-a-file)
3. [Reading](#3-reading)
4. [Writing](#4-writing)
5. [Saving objects](#5-saving-objects)
6. [Copying, moving and deleting](#6-copying-moving-and-deleting)
7. [Listing and searching](#7-listing-and-searching)
8. [Zip files](#8-zip-files)
9. [Downloading](#9-downloading)
10. [Watching for changes](#10-watching-for-changes)
11. [When things fail](#11-when-things-fail)
12. [Putting it together: an inbox](#12-putting-it-together-an-inbox)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The example programs are complete: each has a `main` method. They work inside a folder called `work` under the current directory, and each failure is caught and printed as one line. In the printed output the program's own folder is shortened to `<root>`. `JFile` is in `org.j2os.platform.jfile`; `JFileException` is in `org.j2os.platform.jfile.exception`.

---

## 1. How JFile works

**Paths.** A string that starts with `http://` or `https://` (in any letter case) is a *remote location*. Anything else is a *local path*. An absolute local path is used as it is. A relative one is resolved against the **working directory of the program**, the folder it was started from; JFile reads that folder once, when the class is first used, and keeps it. All relative paths in this guide (`work/...`) therefore live under the folder the examples run in.

**What an object can do depends on what it points to.** A `JFile` for a URL can only `download` (which turns it into a local one). A `JFile` for a local path can do everything except `download`. Calling the wrong kind of method fails with a `JFileException` (chapter 11).

**Sizes and units.** Sizes are given in a unit: `JFile.BYTE`, `KILO_BYTE`, `MEGA_BYTE`, `GIGA_BYTE`, `TERA_BYTE` (powers of 1024). Converting uses integer division, so 1500 bytes are `1` KB and `0` MB: the remainder is dropped, not rounded.

**Objects are cheap, not shared.** A `JFile` holds nothing but its location (and, while watching, a background thread). It does no locking of its own: concurrent reads are fine, concurrent writes behave exactly like writing to a `FileChannel` from two threads. Create a `JFile` per thread or per operation, and coordinate writers yourself.

**Failures.** The class never declares checked exceptions. The one exception type, `JFileException`, carries a message that names the path and keeps the original `IOException` as its cause. A few problems are not wrapped: passing `null`, a path the operating system cannot represent (`InvalidPathException`) and asking `deserialize` for the wrong type (`ClassCastException`).

The types a program meets:

| Type | Role |
|------|------|
| `JFile` | The class itself |
| `JFile.FileSizeUnit` | The size units (`JFile.BYTE` ... `JFile.TERA_BYTE`) |
| `JFileEntry` | One row of a listing: name, relative path, absolute path, directory flag, size |
| `FileEventListener` | A callback that receives `FileEvent`s while watching |
| `FileEvent` | One change: action, entry type, name, paths, last-modified time |
| `FileAction` | `CREATED`, `UPDATED`, `DELETED`, `OVERFLOWED`, `WATCH_TARGET_LOST` (and `CHANGED`, which is never reported) |
| `FileEntryType` | `FILE` or `DIRECTORY` |
| `JFileException` | The unchecked exception (package `org.j2os.platform.jfile.exception`) |

## 2. Looking at a file

The inspection methods answer questions about the location without changing it.

| Method | Answer |
|--------|--------|
| `exists()`, `isFile()`, `isDirectory()` | What is there |
| `length()` | Bytes of a file; `0` when nothing is there |
| `size(unit)` | Size in a unit; for a folder, the sum of every regular file under it |
| `lastModifiedTime()` | An `Instant` |
| `relativePath()`, `absolutePath()` | The path relative to the working directory, or in full |
| `freeSpace(unit)`, `totalSpace(unit)` | Usable and total space of the disk the location lives on |
| `createDirectory()` | Creates the folder and any missing parents. Fine if it exists |
| `back()` | Points the object at the parent folder |

```java
import org.j2os.platform.jfile.JFile;
import org.j2os.platform.jfile.exception.JFileException;

import java.time.Duration;
import java.time.Instant;

public class Inspect {
    public static void main(String[] args) {
        new JFile("work/inspect").createDirectory();
        JFile note = new JFile("work/inspect/note.txt");
        System.out.println("1 exists before writing: " + note.exists());

        note.write("x".repeat(1500));
        System.out.println("2 file: " + note.isFile() + ", directory: " + note.isDirectory());
        System.out.println("3 length: " + note.length());
        System.out.println("4 size in bytes / KB / MB: " + note.size(JFile.BYTE) + " / " + note.size(JFile.KILO_BYTE) + " / " + note.size(JFile.MEGA_BYTE));

        new JFile("work/inspect/second.txt").write("y".repeat(1000));
        System.out.println("5 folder size in bytes: " + new JFile("work/inspect").size(JFile.BYTE));

        System.out.println("6 relative path: " + note.relativePath());
        System.out.println("7 absolute path ends the same way: " + note.absolutePath().endsWith("work" + java.io.File.separator + "inspect" + java.io.File.separator + "note.txt"));
        System.out.println("8 modified a moment ago: " + (Duration.between(note.lastModifiedTime(), Instant.now()).abs().toSeconds() < 60));
        System.out.println("9 free space is positive: " + (note.freeSpace(JFile.BYTE) > 0));

        note.back();
        System.out.println("10 after back(): " + note.relativePath() + ", directory: " + note.isDirectory());

        JFile missing = new JFile("work/inspect/none.txt");
        System.out.println("11 missing file: exists " + missing.exists() + ", length " + missing.length());
        try {
            missing.size(JFile.BYTE);
        } catch (JFileException e) {
            System.out.println("12 size of a missing file -> " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }
    }
}
```

```text
1 exists before writing: false
2 file: true, directory: false
3 length: 1500
4 size in bytes / KB / MB: 1500 / 1 / 0
5 folder size in bytes: 2500
6 relative path: work/inspect/note.txt
7 absolute path ends the same way: true
8 modified a moment ago: true
9 free space is positive: true
10 after back(): work/inspect, directory: true
11 missing file: exists false, length 0
12 size of a missing file -> Failed to compute size: <root>/work/inspect/none.txt  (cause: NoSuchFileException)
```

- Cases 1 to 3: a file that was never written does not exist, and `length()` is `0` for it rather than an error.
- Case 4: the three sizes of one file. The kilobyte and megabyte values are truncated.
- Case 5: for a folder `size` adds up every file under it, in subfolders too. Symbolic links are skipped, so a link to a huge folder elsewhere does not inflate the total.
- Cases 6 and 7: the same file in both path forms.
- Case 8: `lastModifiedTime()` is the file's own timestamp.
- Case 9: space is measured on the disk of the location, and a location that does not exist yet is measured through its nearest existing parent, so it can be asked before creating a file.
- Case 10: `back()` moves the object, it does not return a new one.
- Cases 11 and 12: `exists`, `length`, `isFile` and `isDirectory` accept anything; `size` and `lastModifiedTime` need the file to be there.

## 3. Reading

| Method | Returns |
|--------|---------|
| `read(start, length)` | The text in that byte range, decoded as UTF-8 |
| `readLines()` | Every line, in a list |
| `lineCount()` | The number of lines, counted without loading the file |

`read` is the workhorse: positions and lengths are in **bytes**, but the result is always whole characters.

```java
import org.j2os.platform.jfile.JFile;
import org.j2os.platform.jfile.exception.JFileException;

public class Reading {
    public static void main(String[] args) {
        JFile file = new JFile("work/reading/text.txt");
        file.write("abc");
        file.write("سلام\n");
        file.write("line two\n");

        System.out.println("1 bytes: " + file.length() + ", lines: " + file.lineCount());
        System.out.println("2 whole file: " + file.read(0, Long.MAX_VALUE).replace("\n", "|"));
        System.out.println("3 first 3 bytes: " + file.read(0, 3));
        System.out.println("4 ends inside a letter (0, 4): " + file.read(0, 4));
        System.out.println("5 starts inside a letter (4, 4): " + file.read(4, 4));
        System.out.println("6 past the end: '" + file.read(1000, 10) + "'");
        System.out.println("7 negative start: " + file.read(-5, 3));
        System.out.println("8 empty range: '" + file.read(5, 0) + "'");
        System.out.println("9 lines: " + file.readLines());

        try {
            new JFile("work/reading/none.txt").read(0, 10);
        } catch (JFileException e) {
            System.out.println("10 missing file -> " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }
        try {
            new JFile("work/reading/none.txt").lineCount();
        } catch (JFileException e) {
            System.out.println("11 lineCount of a missing file -> " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }
    }
}
```

```text
1 bytes: 21, lines: 2
2 whole file: abcسلام|line two|
3 first 3 bytes: abc
4 ends inside a letter (0, 4): abcس
5 starts inside a letter (4, 4): سلا
6 past the end: ''
7 negative start: abc
8 empty range: ''
9 lines: [abcسلام, line two]
10 missing file -> Failed to read file: <root>/work/reading/none.txt  (cause: NoSuchFileException)
11 lineCount of a missing file -> Failed to count lines in file: <root>/work/reading/none.txt  (cause: NoSuchFileException)
```

- Case 1: the file is 21 bytes: the Persian word takes 8 bytes (two per letter), so bytes and characters differ. `lineCount` streams through the file and holds nothing in memory.
- Case 2: `Long.MAX_VALUE` as the length means "up to the end".
- Cases 4 and 5: the byte range cut a letter in half. The end of a range moves **forward** to finish the letter, the start moves **backward** to begin it, so no broken character appears (a range may therefore return slightly more than asked).
- Cases 6 to 8: a start past the end gives an empty string, a negative start is treated as `0`, and a length of `0` gives an empty string.
- Cases 10 and 11: a missing file is a `JFileException` whose cause is the JDK's `NoSuchFileException`.

For a large file read it in pieces instead of all at once. `read` returns one `String`, so a single call cannot cover more than about 2 GB, and `readLines` builds a list of every line:

```java
JFile big = new JFile("logs/app.log");
long position = 0;
long length = big.length();

while (position < length) {
    String piece = big.read(position, 1_000_000);        // about 1 MB at a time
    process(piece);
    position += piece.getBytes(StandardCharsets.UTF_8).length;
}
```

The loop assumes valid UTF-8 text. For a file in another encoding, `read` decodes it as UTF-8 and puts a replacement character (`\uFFFD`) where it cannot, while `readLines` and `lineCount` fail with a `JFileException` caused by `MalformedInputException`.

## 4. Writing

| Method | Does |
|--------|------|
| `write(text)` | Adds the text at the **end** of the file (a new or empty file starts at the beginning) |
| `write(position, text)` | Overwrites bytes starting at a position |

Both create the file and any missing parent folders, write UTF-8, and return the object.

```java
import org.j2os.platform.jfile.JFile;

public class Writing {
    public static void main(String[] args) {
        JFile log = new JFile("work/writing/deep/er/log.txt");
        log.write("one;");
        log.write("two;");
        System.out.println("1 appended, parents created: " + log.read(0, Long.MAX_VALUE));

        log.write(0, "ONE");
        System.out.println("2 overwritten at 0: " + log.read(0, Long.MAX_VALUE));

        log.write(8, "three");
        System.out.println("3 at the end: " + log.read(0, Long.MAX_VALUE));

        log.write(4, "TWO");
        System.out.println("4 in the middle: " + log.read(0, Long.MAX_VALUE));

        JFile gap = new JFile("work/writing/gap.txt");
        gap.write("ab");
        gap.write(10, "z");
        System.out.println("5 past the end, length: " + gap.length());

        JFile persian = new JFile("work/writing/persian.txt");
        persian.write("سلام");
        persian.write(1, "ب");
        System.out.println("6 start inside a letter moves back: " + persian.read(0, Long.MAX_VALUE));

        JFile cut = new JFile("work/writing/cut.txt");
        cut.write("سلام");
        cut.write(0, "x");
        System.out.println("7 the end is not protected: " + cut.read(0, Long.MAX_VALUE).contains("�"));
    }
}
```

```text
1 appended, parents created: one;two;
2 overwritten at 0: ONE;two;
3 at the end: ONE;two;three
4 in the middle: ONE;TWO;three
5 past the end, length: 11
6 start inside a letter moves back: بلام
7 the end is not protected: true
```

- Case 1: two `write(text)` calls append. Nothing is replaced, and the folders `deep/er` were created on the way.
- Cases 2 to 4: `write(position, text)` **overwrites** that many bytes, it does not insert. Position 0 replaced the start, position 8 (the end) appended, and position 4 replaced three characters in the middle without moving the rest.
- Case 5: writing past the end extends the file to reach the position. The gap is filled by the operating system, usually with zero bytes.
- Case 6: a position inside a letter is moved **back** to the start of that letter, so the new text replaces the whole letter.
- Case 7: only the *start* is protected. When the new text ends in the middle of a following multi-byte letter, that letter's remaining bytes stay behind and the file contains invalid text. If this matters, overwrite with text of the same byte length or read and rewrite the whole letter.

`write` is not atomic and there is no way to replace a whole file in one step. To publish a complete file, write it elsewhere and `move` it into place (chapter 6), which is also how chapter 12 hands files to a watcher.

## 5. Saving objects

`serialize(object)` writes an object to the file with Java serialization and `deserialize(Class)` reads it back. The object must be `Serializable`.

```java
import org.j2os.platform.jfile.JFile;
import org.j2os.platform.jfile.exception.JFileException;

import java.util.ArrayList;
import java.util.List;

public class Objects {
    public static void main(String[] args) {
        JFile file = new JFile("work/objects/settings.bin");

        ArrayList<String> languages = new ArrayList<>(List.of("fa", "en"));
        file.serialize(languages);
        ArrayList<?> back = file.deserialize(ArrayList.class);
        System.out.println("1 round trip: " + back);

        file.serialize("replaced");
        System.out.println("2 second write replaces the first: " + file.deserialize(String.class));

        try {
            file.deserialize(Integer.class);
        } catch (ClassCastException e) {
            System.out.println("3 wrong type -> " + e.getMessage());
        }
        try {
            new JFile("work/objects/none.bin").deserialize(String.class);
        } catch (JFileException e) {
            System.out.println("4 missing file -> " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }

        try {
            file.serialize(new Object());
        } catch (JFileException e) {
            System.out.println("5 not serializable -> " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }
        System.out.println("6 the old content survived: " + readable(file));
    }

    static boolean readable(JFile file) {
        try {
            file.deserialize(String.class);
            return true;
        } catch (RuntimeException e) {
            return false;
        }
    }
}
```

```text
1 round trip: [fa, en]
2 second write replaces the first: replaced
3 wrong type -> Cannot cast java.lang.String to java.lang.Integer
4 missing file -> Failed to deserialize object from file: <root>/work/objects/none.bin  (cause: NoSuchFileException)
5 not serializable -> Failed to serialize object to file: <root>/work/objects/settings.bin  (cause: NotSerializableException)
6 the old content survived: false
```

- Cases 1 and 2: the file holds one object. Serializing again **replaces** the previous content.
- Case 3: asking for the wrong type is a plain `ClassCastException`, not a `JFileException`.
- Case 4: a missing file is a `JFileException`.
- Cases 5 and 6: the file is opened (and emptied) before the object is written, so an object that cannot be serialized leaves the old content destroyed. Serialize to a temporary name and `move` it over the real one when the old content must survive.

Java deserialization can run code while rebuilding an object. Only read files that your own program, or a source you trust completely, wrote. For data that comes from outside use a text format (JSON, CSV) and `read`.

## 6. Copying, moving and deleting

| Method | Does | The object afterwards |
|--------|------|-----------------------|
| `copy(destination)` | Copies the file, or the whole folder with everything in it | Still the original |
| `move(destination)` | Moves or renames | Points at the new place |
| `delete()` | Deletes the file, or the folder with everything in it | Still the same path |

`destination` is the **new path of the item**, not the folder to put it in: `copy("backup")` makes something called `backup`. Missing parent folders are created, and an existing file at the destination is **replaced** without asking.

```java
import org.j2os.platform.jfile.JFile;
import org.j2os.platform.jfile.exception.JFileException;

public class Moving {
    public static void main(String[] args) {
        JFile original = new JFile("work/moving/a.txt");
        original.write("first");

        original.copy("work/moving/copies/b.txt");
        System.out.println("1 copy: this object still at " + original.relativePath() + ", copy exists: " + new JFile("work/moving/copies/b.txt").exists());

        new JFile("work/moving/c.txt").write("other");
        original.copy("work/moving/c.txt");
        System.out.println("2 copy over an existing file: " + new JFile("work/moving/c.txt").read(0, 100));

        original.move("work/moving/archive/d.txt");
        System.out.println("3 after move() the object is at " + original.relativePath() + ", old place exists: " + new JFile("work/moving/a.txt").exists());

        new JFile("work/moving/tree/sub/leaf.txt").write("leaf");
        new JFile("work/moving/tree/top.txt").write("top");
        new JFile("work/moving/tree").copy("work/moving/tree-copy");
        System.out.println("4 folder copy has " + new JFile("work/moving/tree-copy").search("%.txt").size() + " files");

        try {
            new JFile("work/moving/tree").copy("work/moving/tree/sub/inner");
        } catch (JFileException e) {
            System.out.println("5 folder into itself -> " + e.getMessage());
        }
        try {
            new JFile("work/moving/none.txt").move("work/moving/x.txt");
        } catch (JFileException e) {
            System.out.println("6 move something missing -> " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }

        JFile tree = new JFile("work/moving/tree");
        tree.delete();
        System.out.println("7 folder deleted with its content: " + !tree.exists());
        tree.delete();
        System.out.println("8 deleting what is not there is fine");
    }
}
```

```text
1 copy: this object still at work/moving/a.txt, copy exists: true
2 copy over an existing file: first
3 after move() the object is at work/moving/archive/d.txt, old place exists: false
4 folder copy has 2 files
5 folder into itself -> Cannot copy directory <root>/work/moving/tree into its own subtree: <root>/work/moving/tree/sub/inner
6 move something missing -> Failed to move to: work/moving/x.txt  (cause: NoSuchFileException)
7 folder deleted with its content: true
8 deleting what is not there is fine
```

- Case 1: `copy` leaves the object where it was and creates the missing `copies` folder.
- Case 2: an existing destination file is replaced.
- Case 3: after `move` the object follows the file, and the old place is empty.
- Case 4: copying a folder copies its content recursively.
- Case 5: copying a folder into itself would never end, so it is refused. Symbolic links met while copying a folder are skipped.
- Case 6: moving what is not there fails.
- Cases 7 and 8: `delete` removes a folder with everything in it, and it is not an error when nothing is there.

`delete` is immediate and recursive; there is no trash and no confirmation. Moving onto a non-empty folder fails (see chapter 11), and a folder move between two disks may fail where the same move on one disk is only a rename.

## 7. Listing and searching

`dir(unit)` lists the entries directly inside a folder. `search(pattern)` finds entries by name anywhere below it. Both return `JFileEntry` records with `name()`, `relativePath()`, `absolutePath()`, `isDirectory()` and `size()`.

```java
import org.j2os.platform.jfile.JFile;
import org.j2os.platform.jfile.exception.JFileException;
import org.j2os.platform.jfile.JFileEntry;

import java.util.List;

public class Finding {
    static String names(List<JFileEntry> entries) {
        return entries.stream().map(JFileEntry::name).sorted().toList().toString();
    }

    public static void main(String[] args) {
        new JFile("work/find/Report.txt").write("a".repeat(2048));
        new JFile("work/find/report_old.TXT").write("b");
        new JFile("work/find/notes.md").write("c");
        new JFile("work/find/data/Report2.txt").write("d");
        new JFile("work/find/data/x.log").write("e");

        JFile root = new JFile("work/find");
        System.out.println("1 dir: " + names(root.dir(JFile.BYTE)));
        for (JFileEntry entry : root.dir(JFile.KILO_BYTE)) {
            if (entry.name().equals("Report.txt") || entry.name().equals("data")) {
                System.out.println("   " + entry.name() + " | directory " + entry.isDirectory() + " | size " + entry.size() + " KB | " + entry.relativePath());
            }
        }
        System.out.println("2 report%: " + names(root.search("report%")));
        System.out.println("3 %.txt: " + names(root.search("%.txt")));
        System.out.println("4 _ata (one character): " + names(root.search("_ata")));
        System.out.println("5 %: " + root.search("%").size() + " entries, every level");
        System.out.println("6 *.txt is not a wildcard: " + root.search("*.txt").size());

        try {
            new JFile("work/find/notes.md").dir(JFile.BYTE);
        } catch (JFileException e) {
            System.out.println("7 dir of a file -> " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }
    }
}
```

```text
1 dir: [Report.txt, data, notes.md, report_old.TXT]
   Report.txt | directory false | size 2 KB | work/find/Report.txt
   data | directory true | size 0 KB | work/find/data
2 report%: [Report.txt, Report2.txt, report_old.TXT]
3 %.txt: [Report.txt, Report2.txt, report_old.TXT]
4 _ata (one character): [data]
5 %: 6 entries, every level
6 *.txt is not a wildcard: 0
7 dir of a file -> Failed to list directory: <root>/work/find/notes.md  (cause: NotDirectoryException)
```

- Case 1: the order of `dir()` is whatever the file system gives; sort when order matters (the program sorts the names).
- The `size()` of a folder entry is `0`: it is not the folder's content. Use `size(unit)` on a `JFile` for the total of a folder. In `dir(unit)` a file's size is in the requested unit and truncated; `search` always reports bytes.
- Cases 2 and 3: the pattern follows SQL `LIKE`: `%` stands for any number of characters, `_` for exactly one. The match is against the **name only** and is **not case-sensitive**, so `report%` found `Report.txt` and `report_old.TXT`, also in subfolders.
- Case 4: `_ata` matched the folder `data`; folders are found like files.
- Case 5: `%` returns every entry below the folder (the folder itself is not included).
- Case 6: `*` and `?` are ordinary characters here, so `*.txt` matches nothing. Every other character is taken literally, so a dot in a pattern is just a dot.
- Case 7: `dir` on a file fails.

Symbolic links are not followed during a search. For a huge tree, `search` collects all matches in memory before returning.

## 8. Zip files

`zip(destination)` packs a file or a folder into a standard ZIP archive, and `unzip(folder)` extracts one. There is no password support and no splitting into parts.

```java
import org.j2os.platform.jfile.JFile;
import org.j2os.platform.jfile.exception.JFileException;

import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.zip.ZipEntry;
import java.util.zip.ZipFile;
import java.util.zip.ZipOutputStream;

public class Zipping {
    static List<String> entries(String zip) throws IOException {
        try (ZipFile file = new ZipFile(zip)) {
            List<String> names = new ArrayList<>();
            file.stream().forEach(e -> names.add(e.getName()));
            Collections.sort(names);
            return names;
        }
    }

    public static void main(String[] args) throws Exception {
        new JFile("work/zip/src/a.txt").write("alpha");
        new JFile("work/zip/src/sub/b.txt").write("beta");
        new JFile("work/zip/src/empty").createDirectory();

        new JFile("work/zip/src").zip("work/zip/out/all.zip");
        System.out.println("1 folder: " + entries("work/zip/out/all.zip"));

        new JFile("work/zip/src/a.txt").zip("work/zip/out/one.zip");
        System.out.println("2 single file: " + entries("work/zip/out/one.zip"));

        new JFile("work/zip/src").zip("work/zip/src/self.zip");
        new JFile("work/zip/src").zip("work/zip/src/self.zip");
        System.out.println("3 archive inside the folder, twice: " + entries("work/zip/src/self.zip"));

        new JFile("work/zip/out/all.zip").unzip("work/zip/dest");
        System.out.println("4 unzipped: " + new JFile("work/zip/dest/sub/b.txt").read(0, 100)
                + ", empty folder kept: " + new JFile("work/zip/dest/empty").isDirectory());

        try (ZipOutputStream evil = new ZipOutputStream(new FileOutputStream("work/zip/evil.zip"))) {
            evil.putNextEntry(new ZipEntry("../evil.txt"));
            evil.write(1);
            evil.closeEntry();
        }
        try {
            new JFile("work/zip/evil.zip").unzip("work/zip/safe");
        } catch (JFileException e) {
            System.out.println("5 entry that climbs out -> " + e.getMessage());
        }
        System.out.println("6 nothing written outside: " + !Files.exists(Path.of("work/zip/evil.txt")));

        new JFile("work/zip/plain.txt").write("not an archive");
        try {
            new JFile("work/zip/plain.txt").unzip("work/zip/nothing");
            System.out.println("7 unzip a text file -> ok");
        } catch (JFileException e) {
            System.out.println("7 unzip a text file -> " + e.getMessage());
        }
        try {
            new JFile("work/zip/none").zip("work/zip/none.zip");
        } catch (JFileException e) {
            System.out.println("8 zip something missing -> " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }
    }
}
```

```text
1 folder: [a.txt, empty/, sub/, sub/b.txt]
2 single file: [a.txt]
3 archive inside the folder, twice: [a.txt, empty/, sub/, sub/b.txt]
4 unzipped: beta, empty folder kept: true
5 entry that climbs out -> Zip entry escapes destination directory: ../evil.txt
6 nothing written outside: true
7 unzip a text file -> ok
8 zip something missing -> Failed to zip: <root>/work/zip/none  (cause: NoSuchFileException)
```

- Case 1: a folder is stored with paths **relative to it**; the folder's own name is not an entry, so unzipping into `X` gives `X/a.txt`, not `X/src/a.txt`. Folders, empty ones included, get entries of their own.
- Case 2: a single file is stored under its own name.
- Case 3: an archive written *inside* the folder it packs does not pack itself, even when an older version is already there.
- Case 4: `unzip` creates the destination folder and recreates the structure, empty folders too. Files already there are replaced.
- Cases 5 and 6: **an entry whose path would climb out of the destination** (`../evil.txt`, the classic *Zip Slip* attack) is refused, and nothing is written outside. A path that would pass through an existing symbolic link is refused for the same reason.
- Case 7: a file that is not a ZIP archive does not fail: `unzip` finds no entries and extracts nothing. Check the result if that matters.
- Case 8: a missing source fails.

Two safety properties are worth knowing. `zip` writes to a temporary file next to the destination and renames it into place at the end, so an existing archive survives a failure and a reader never sees a half-written one. `unzip` is not all-or-nothing: if one entry is refused, the entries before it have already been written. Extract untrusted archives into a folder nothing else uses, and only then look at the result:

```java
String work = "work/" + UUID.randomUUID();                 // a folder nothing else writes to
new JFile(uploadedZip).unzip(work);
// ... check the result, then move what you want to keep
new JFile(work).delete();
```

The checks on `unzip` and `copy` are separate from the writes they guard, so a second process that can modify the destination at the same moment could still outwit them. Use a destination nobody else can write to.

## 9. Downloading

`new JFile(url).download(destination, channels)` fetches a web address into a file. With several **channels** the file is cut into equal parts and each part is requested in parallel with an HTTP range request, which is faster on servers that limit the speed of one connection.

The example needs a web server to download from, so the method `serve(data, etag, honourRanges)` at the end of the program starts a tiny one on this machine. It returns the given bytes at `/f`, with or without an `ETag` and with or without support for range requests, and the counter `requests` tells how many ranged requests it received.

```java
import com.sun.net.httpserver.HttpServer;
import org.j2os.platform.jfile.JFile;
import org.j2os.platform.jfile.exception.JFileException;

import java.net.InetSocketAddress;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Arrays;
import java.util.concurrent.atomic.AtomicInteger;

public class Downloading {
    static final AtomicInteger requests = new AtomicInteger();

    public static void main(String[] args) throws Exception {
        byte[] data = new byte[300_000];
        for (int i = 0; i < data.length; i++) data[i] = (byte) (i * 31);

        String strong = serve(data, "\"v1\"", true);
        JFile remote = new JFile(strong);
        remote.download("work/dl/four.bin", 4);
        System.out.println("1 requests with 4 channels: " + requests.get() + ", same bytes: " + Arrays.equals(Files.readAllBytes(Path.of("work/dl/four.bin")), data));
        System.out.println("2 the object now points at the file: " + remote.relativePath() + ", " + remote.length() + " bytes");

        new JFile(serve(data, null, true)).download("work/dl/single.bin", 4);
        System.out.println("3 no ETag, 4 channels asked: " + requests.get() + " request");

        String ignoresRanges = serve(data, "\"v1\"", false);
        new JFile("work/dl/keep.bin").write("old file");
        try {
            new JFile(ignoresRanges).download("work/dl/keep.bin", 4);
        } catch (JFileException e) {
            System.out.println("4 server ignores ranges -> " + e.getMessage() + "  (cause: " + e.getCause().getMessage() + ")");
        }
        System.out.println("5 the old file survived: " + new JFile("work/dl/keep.bin").read(0, 100));
        System.out.println("6 no leftovers: " + new JFile("work/dl").search("jfile-download%").size());

        new JFile(serve(data, null, false)).download("work/dl/whole.bin", 1);
        System.out.println("7 server without ranges, 1 channel: " + new JFile("work/dl/whole.bin").length() + " bytes");

        try {
            new JFile(strong.replace("/f", "/missing")).download("work/dl/none.bin", 2);
        } catch (JFileException e) {
            System.out.println("8 wrong address -> " + e.getMessage());
        }
        try {
            new JFile("work/dl/four.bin").download("work/dl/again.bin", 2);
        } catch (JFileException e) {
            System.out.println("9 local file -> " + e.getMessage());
        }
        try {
            new JFile(strong).length();
        } catch (JFileException e) {
            System.out.println("10 read a URL -> " + e.getMessage());
        }
        System.exit(0);
    }

    /** A stand-in web server: serves data at /f, optionally with an ETag and with or without range support. */
    static String serve(byte[] data, String etag, boolean honourRanges) throws Exception {
        requests.set(0);
        HttpServer server = HttpServer.create(new InetSocketAddress("localhost", 0), 0);
        server.createContext("/f", exchange -> {
            String range = exchange.getRequestHeaders().getFirst("Range");
            if (etag != null) exchange.getResponseHeaders().set("ETag", etag);
            if (exchange.getRequestMethod().equals("HEAD")) {
                exchange.getResponseHeaders().set("Content-Length", String.valueOf(data.length));
                exchange.sendResponseHeaders(200, -1);
            } else if (range != null && honourRanges) {
                requests.incrementAndGet();
                String[] bounds = range.substring("bytes=".length()).split("-");
                int from = Integer.parseInt(bounds[0]), to = Integer.parseInt(bounds[1]);
                exchange.getResponseHeaders().set("Content-Range", "bytes " + from + "-" + to + "/" + data.length);
                exchange.sendResponseHeaders(206, to - from + 1);
                exchange.getResponseBody().write(data, from, to - from + 1);
            } else {
                requests.incrementAndGet();
                exchange.sendResponseHeaders(200, data.length);
                exchange.getResponseBody().write(data);
            }
            exchange.close();
        });
        server.start();
        return "http://localhost:" + server.getAddress().getPort() + "/f";
    }
}
```

```text
1 requests with 4 channels: 4, same bytes: true
2 the object now points at the file: work/dl/four.bin, 300000 bytes
3 no ETag, 4 channels asked: 1 request
4 server ignores ranges -> Failed to download: http://localhost:<port>/f  (cause: Remote server did not honour the range request bytes=0-74999 (HTTP 200); aborting to avoid a corrupted file: http://localhost:<port>/f)
5 the old file survived: old file
6 no leftovers: 0
7 server without ranges, 1 channel: 300000 bytes
8 wrong address -> Remote location returned HTTP 404: http://localhost:<port>/missing
9 local file -> No remote location is set; construct this JFile with a URL before download(...)
10 read a URL -> This JFile was constructed with a remote URL, not a local path
```

- Cases 1 and 2: four channels meant four range requests, the bytes are identical, and afterwards the object points at the **local file**. A `JFile` made from a URL becomes an ordinary one once the download succeeds.
- Case 3: **several channels need a strong `ETag`.** Without it the server could change the file between the parts, and the parts would not fit together, so jfile falls back to one request. (A weak `ETag`, one that starts with `W/`, does not count.)
- Cases 4 to 6: a server that claims support but ignores the range is detected and the download stops. The download goes to a temporary file next to the destination and replaces the destination only at the very end, so the old file survived and nothing was left behind.
- Case 7: a server without range support works with one channel.
- Case 8: any answer other than success (2xx) is a failure carrying the status code.
- Cases 9 and 10: `download` needs a URL, and other methods do not work on a URL.

How a download proceeds: a `HEAD` request learns the size and the `ETag`; the destination is prepared; each channel writes its part directly at its position; the result is moved into place. Further details:

- The server must report `Content-Length` for the `HEAD` request, otherwise the download stops.
- Redirects are followed, except from `https` to `http`.
- The channel count is limited to the number of bytes and to 64, and anything below 1 means 1.
- A server that returns a status other than `206 Partial Content` for a ranged request (or a `Content-Range` that does not match) stops the download. A plain `200` is accepted only for a single channel.
- No timeout is set on the requests. A server that accepts a connection and then says nothing keeps the call waiting. For unattended jobs run the download on a thread you can abandon.
- There is no progress report, authentication, header customisation or resuming of an earlier attempt. For those use `HttpClient` directly.

A simple retry loop is safe because a failed download never touches the destination:

```java
for (int attempt = 1; attempt <= 3; attempt++) {
    try {
        new JFile(url).download("downloads/data.bin", 4);
        break;
    } catch (JFileException failure) {
        if (attempt == 3) throw failure;                   // the destination was never touched
    }
}
```

## 10. Watching for changes

`setListener(listener)` starts watching the location and calls the listener for every change. If the location is a **folder**, everything below it is watched, including folders created later. If it is a **file**, only that file is reported. The listener runs on a **single background thread**, one event at a time.

```java
import org.j2os.platform.jfile.FileAction;
import org.j2os.platform.jfile.JFile;

import java.util.Set;
import java.util.concurrent.ConcurrentSkipListSet;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

public class Watching {
    public static void main(String[] args) throws Exception {
        JFile folder = new JFile("work/watch");
        folder.createDirectory();

        Set<String> seen = new ConcurrentSkipListSet<>();
        CountDownLatch deleted = new CountDownLatch(1);
        folder.setListener(event -> {
            seen.add(event.action() + " " + event.entryType() + " " + event.relativePath());
            if (event.action() == FileAction.DELETED && event.name().equals("a.txt")) deleted.countDown();
        });

        new JFile("work/watch/a.txt").write("x");
        new JFile("work/watch/sub").createDirectory();
        Thread.sleep(300);
        new JFile("work/watch/sub/b.txt").write("y");
        Thread.sleep(300);
        new JFile("work/watch/a.txt").delete();
        deleted.await(5, TimeUnit.SECONDS);
        Thread.sleep(300);

        for (String wanted : new String[]{"CREATED FILE work/watch/a.txt", "UPDATED FILE work/watch/a.txt", "CREATED DIRECTORY work/watch/sub",
                "CREATED FILE work/watch/sub/b.txt", "DELETED FILE work/watch/a.txt"}) {
            System.out.println("1 " + wanted + ": " + seen.contains(wanted));
        }

        folder.stopListening();
        int before = seen.size();
        new JFile("work/watch/after.txt").write("z");
        Thread.sleep(300);
        System.out.println("2 nothing reported after stopListening(): " + (seen.size() == before));

        Set<String> single = new ConcurrentSkipListSet<>();
        JFile one = new JFile("work/watch/one.txt");
        one.write("1");
        one.setListener(event -> single.add(event.name()));
        new JFile("work/watch/neighbour.txt").write("n");
        one.write("2");
        Thread.sleep(500);
        one.stopListening();
        System.out.println("3 watching one file reports only that file: " + single);

        JFile doomed = new JFile("work/doomed");
        doomed.createDirectory();
        CountDownLatch lost = new CountDownLatch(1);
        doomed.setListener(event -> { if (event.action() == FileAction.WATCH_TARGET_LOST) lost.countDown(); });
        doomed.delete();
        System.out.println("4 told when the folder itself is gone: " + lost.await(5, TimeUnit.SECONDS));
    }
}
```

```text
1 CREATED FILE work/watch/a.txt: true
1 UPDATED FILE work/watch/a.txt: true
1 CREATED DIRECTORY work/watch/sub: true
1 CREATED FILE work/watch/sub/b.txt: true
1 DELETED FILE work/watch/a.txt: true
2 nothing reported after stopListening(): true
3 watching one file reports only that file: [one.txt]
4 told when the folder itself is gone: true
```

- Case 1: five kinds of change were reported: a new file, a changed file, a new folder, a file created in that new folder (whose creation the watcher noticed because it registered the new folder), and a deletion.
- Case 2: `stopListening()` ends the watching; later changes are not reported.
- Case 3: when watching one file, its neighbours are ignored.
- Case 4: when the watched folder itself disappears, the listener is told (`WATCH_TARGET_LOST`) and the watcher stops once nothing is left to watch.

A `FileEvent` offers `action()`, `entryType()` (file or folder; for a deletion it is remembered from before), `name()`, `relativePath()`, `absolutePath()` and `lastModifiedTime()` (the entry's own time, or the moment of processing for a deletion).

What to expect from the events:

| Action | Meaning |
|--------|---------|
| `CREATED` | A file or folder appeared (also: moved in, copied in) |
| `UPDATED` | Its content or timestamp changed. A single `write` can produce more than one |
| `DELETED` | It was removed |
| `OVERFLOWED` | The system dropped events because they came too fast; rescan the folder yourself |
| `WATCH_TARGET_LOST` | A watched folder is gone or inaccessible |
| `CHANGED` | Declared but never delivered |

Practical rules:

- **A `CREATED` event can arrive before the file's content is complete**, because the file is created first and filled afterwards. Have producers write the file somewhere else and `move` it into the watched folder, so that it appears all at once (chapter 12 does this).
- **Events can repeat.** `UPDATED` often arrives twice for one write, and an entry inside a brand-new folder can be reported twice. Make handlers safe to run again.
- **The listener must be quick.** It runs on the one watcher thread, so a slow listener delays every event behind it. Hand long work to an executor.
- **A listener that throws does not stop the watcher.** The error is printed to `System.err` and the next event is delivered.
- **The watcher is a daemon thread.** It does not keep the program alive; a `main` that returns ends the JVM and the watching with it. A long-running program must stay alive on purpose:

```java
public static void main(String[] args) throws Exception {
    new JFile("incoming").setListener(event -> handle(event));
    Thread.currentThread().join();       // the watcher is a daemon thread: without this, main ends and the JVM exits
}
```

- **Call `stopListening()` when done.** A watcher keeps running for as long as the JVM does, even when the `JFile` is no longer referenced. Calling `setListener` again on the same object replaces the old watcher. `stopListening()` waits up to a few seconds for the thread to finish.
- Watching depends on the operating system. On some systems (macOS is the usual example) changes can be noticed with a delay of several seconds.
- The watched location is fixed when `setListener` is called. A later `back()` or `move` on the same object does not change what is watched.
- Symbolic links are not followed.

## 11. When things fail

A `JFileException` has a message that names the path (or the destination) and a cause with the real reason.

```java
import org.j2os.platform.jfile.JFile;
import org.j2os.platform.jfile.exception.JFileException;

public class Failures {
    public static void main(String[] args) {
        JFile folder = new JFile("work/failures");
        folder.createDirectory();

        try {
            new JFile("work/failures/none.txt").read(0, 10);
        } catch (JFileException e) {
            System.out.println("1 read a missing file -> " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }
        try {
            new JFile("work/failures/nowhere").dir(JFile.BYTE);
        } catch (JFileException e) {
            System.out.println("2 list a missing folder -> " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }
        try {
            folder.readLines();
        } catch (JFileException e) {
            System.out.println("3 read a folder as a file -> " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }
        try {
            folder.write("text");
        } catch (JFileException e) {
            System.out.println("4 write where a folder is -> " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }
        try {
            new JFile("https://example.com/a.bin").exists();
        } catch (JFileException e) {
            System.out.println("5 a URL used as a path -> " + e.getMessage());
        }
        try {
            folder.download("work/failures/x.bin", 2);
        } catch (JFileException e) {
            System.out.println("6 download without a URL -> " + e.getMessage());
        }
        try {
            new JFile("work/failures/a/b").freeSpace(JFile.BYTE);
            System.out.println("7 free space of a place that does not exist yet -> ok");
        } catch (JFileException e) {
            System.out.println("7 free space of a place that does not exist yet -> " + e.getMessage());
        }
        try {
            new JFile("work/failures/target/inner.txt").write("x");
            new JFile("work/failures/source.txt").write("y");
            new JFile("work/failures/source.txt").move("work/failures/target");
        } catch (JFileException e) {
            System.out.println("8 move onto a non-empty folder -> " + e.getMessage() + "  (cause: " + e.getCause().getClass().getSimpleName() + ")");
        }
    }
}
```

```text
1 read a missing file -> Failed to read file: <root>/work/failures/none.txt  (cause: NoSuchFileException)
2 list a missing folder -> Failed to list directory: <root>/work/failures/nowhere  (cause: NoSuchFileException)
3 read a folder as a file -> Failed to read lines from file: <root>/work/failures  (cause: IOException)
4 write where a folder is -> Failed to write to file: <root>/work/failures  (cause: FileSystemException)
5 a URL used as a path -> This JFile was constructed with a remote URL, not a local path
6 download without a URL -> No remote location is set; construct this JFile with a URL before download(...)
7 free space of a place that does not exist yet -> ok
8 move onto a non-empty folder -> Failed to move to: work/failures/target  (cause: DirectoryNotEmptyException)
```

- Cases 1 and 2: a missing file or folder shows `NoSuchFileException` as the cause.
- Cases 3 and 4: a folder used as a file fails with a cause that names what the system objected to.
- Cases 5 and 6: the two "wrong kind" errors carry fixed messages: *"This JFile was constructed with a remote URL, not a local path"* and *"No remote location is set; construct this JFile with a URL before download(...)"*.
- Case 7: asking for free space of a location that does not exist yet is fine.
- Case 8: `move` onto a non-empty folder fails with `DirectoryNotEmptyException`.

The other messages are in Appendix D. To react to a specific cause, look at `getCause()`:

- `NoSuchFileException`: the thing is not there.
- `FileAlreadyExistsException`, `DirectoryNotEmptyException`: the destination is in the way.
- `AccessDeniedException`: permissions.
- an `IOException` whose message says there is no space left: the disk is full.

## 12. Putting it together: an inbox

A folder receives order files. Every new file must be counted, noted in a log, and moved to an archive; at the end the archive is zipped.

```java
import org.j2os.platform.jfile.FileAction;
import org.j2os.platform.jfile.FileEntryType;
import org.j2os.platform.jfile.JFile;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

public class Workflow {
    public static void main(String[] args) throws Exception {
        JFile inbox = new JFile("work/desk/inbox");
        inbox.createDirectory();
        CountDownLatch handled = new CountDownLatch(2);

        // 1. watch the inbox: every new file is counted, logged and archived
        inbox.setListener(event -> {
            if (event.action() != FileAction.CREATED || event.entryType() != FileEntryType.FILE) return;
            JFile arrived = new JFile(event.absolutePath());
            new JFile("work/desk/report.log").write(event.name() + ": " + arrived.lineCount() + " lines\n");
            arrived.move("work/desk/archive/" + event.name());
            handled.countDown();
        });

        // 2. files arrive complete: written elsewhere first, then moved into the inbox
        new JFile("work/desk/staging/order-1.txt").write("a\nb\nc\n").move("work/desk/inbox/order-1.txt");
        new JFile("work/desk/staging/order-2.txt").write("a\n").move("work/desk/inbox/order-2.txt");
        System.out.println("both handled: " + handled.await(5, TimeUnit.SECONDS));
        inbox.stopListening();

        // 3. the report and the archive
        System.out.println(new JFile("work/desk/report.log").readLines().stream().sorted().toList());
        new JFile("work/desk/archive").zip("work/desk/archive.zip");
        System.out.println("inbox now holds " + inbox.dir(JFile.BYTE).size() + " files, archive size " + new JFile("work/desk/archive").size(JFile.BYTE) + " bytes");
    }
}
```

```text
both handled: true
[order-1.txt: 3 lines, order-2.txt: 1 lines]
inbox now holds 0 files, archive size 8 bytes
```

The steps:

1. The listener watches the inbox and ignores everything except new files (chapter 10).
2. Producers write each file in a staging folder and `move` it into the inbox, so it appears complete (chapters 4, 6 and 10).
3. For each new file the listener counts lines (chapter 3), appends a line to the log (chapter 4) and moves the file to the archive (chapter 6).
4. The program waits until both files are handled, stops watching, zips the archive (chapter 8) and checks that the inbox is empty and the archive size (chapter 2).

Before using the same pattern in production:

- Keep long work out of the listener: queue it.
- Treat repeated events as normal; archive or ignore a file that is already gone.
- Keep the program alive on purpose and stop the watcher on shutdown.
- Never build a path from user input without checking it:

```java
Path base = Path.of("uploads").toAbsolutePath().normalize();
Path wanted = base.resolve(userSuppliedName).normalize();

if (!wanted.startsWith(base)) {
    throw new IllegalArgumentException("Not allowed: " + userSuppliedName);   // "../../etc/passwd"
}
new JFile(wanted.toString()).write(content);
```

---

## Appendix A. Quick reference

### A.1 Creating

| Call | Does |
|------|------|
| `new JFile(path)` | Binds to a local file or folder. Relative paths start at the working directory |
| `new JFile(url)` | Binds to an `http://` or `https://` address; only `download` works |

### A.2 Inspecting

| Call | Returns |
|------|---------|
| `exists()`, `isFile()`, `isDirectory()` | `boolean` |
| `length()` | Bytes; `0` when missing |
| `size(unit)` | Size in the unit; for folders the sum of regular files, links skipped |
| `lastModifiedTime()` | `Instant`; fails if missing |
| `relativePath()`, `absolutePath()` | `String` |
| `freeSpace(unit)`, `totalSpace(unit)` | Usable and total space of the disk |
| `back()` | Points at the parent folder; the object itself |
| `createDirectory()` | Creates the folder with parents |

### A.3 Reading and writing

| Call | Does |
|------|------|
| `read(start, length)` | UTF-8 text of a byte range, edges moved to whole characters; ranges are clamped to the file |
| `readLines()` | All lines |
| `lineCount()` | Number of lines, streamed |
| `write(text)` | Appends (or starts a new file) |
| `write(position, text)` | Overwrites from a position; the start is moved back to a character start |
| `serialize(object)`, `deserialize(type)` | Java serialization of one object |

### A.4 Files and folders

| Call | Does |
|------|------|
| `copy(destination)` | Copies file or folder (recursive); replaces an existing file |
| `move(destination)` | Moves or renames; replaces an existing file; the object follows |
| `delete()` | Deletes file or folder (recursive); fine when missing |
| `dir(unit)` | Direct children as `JFileEntry` list |
| `search(pattern)` | Entries below, by name, `LIKE` pattern, case-insensitive |
| `zip(destination)` | Writes a ZIP archive atomically |
| `unzip(folder)` | Extracts a ZIP archive, refusing escaping entries |
| `download(destination, channels)` | Downloads a URL with parallel range requests |

### A.5 Watching

| Call | Does |
|------|------|
| `setListener(listener)` | Starts watching (replaces an earlier watcher of this object); returns the object |
| `stopListening()` | Stops watching |

### A.6 Types

| Type | Members |
|------|---------|
| `JFileEntry` | `name()`, `relativePath()`, `absolutePath()`, `isDirectory()`, `size()` |
| `FileEvent` | `action()`, `entryType()`, `name()`, `relativePath()`, `absolutePath()`, `lastModifiedTime()` |
| `FileAction` | `CREATED`, `UPDATED`, `DELETED`, `CHANGED`, `OVERFLOWED`, `WATCH_TARGET_LOST` |
| `FileEntryType` | `FILE`, `DIRECTORY` |
| Units | `JFile.BYTE`, `KILO_BYTE`, `MEGA_BYTE`, `GIGA_BYTE`, `TERA_BYTE` |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| Any I/O problem | `JFileException` with the path in the message and the cause attached |
| Local method on a URL object | `JFileException`: *"constructed with a remote URL"* |
| `download` on a local object | `JFileException`: *"No remote location is set"* |
| Missing file or folder | `JFileException`, cause `NoSuchFileException` |
| `deserialize` with the wrong type | `ClassCastException` |
| `serialize` of a non-serializable object | `JFileException`; the old file content is already gone |
| Copy of a folder into itself | `JFileException`: *"Cannot copy directory ... into its own subtree"* |
| Zip entry climbing out of the destination | `JFileException`: *"Zip entry escapes destination directory"* |
| Entry path through a symbolic link | `JFileException`: *"Write would be redirected outside ... through an existing symbolic link"* |
| Unzip of a file that is not a ZIP | Nothing happens, no error |
| Download: server ignores or mishandles ranges | `JFileException`: *"Remote server did not honour the range request"* |
| Download: no `Content-Length` | `JFileException` |
| Download: status other than 2xx | `JFileException` with the status code |
| Failed download or zip | Temporary file removed; the destination is untouched |
| Listener throws | Printed to `System.err`; watching continues |
| `null` argument, invalid path characters | `NullPointerException`, `InvalidPathException` (not wrapped) |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Relative paths depend on where the program started.** The working directory is read once; use absolute paths in services.
- **`write(position, text)` overwrites.** It does not insert, and it can leave a broken character after the written text.
- **`write(text)` always appends.** There is no "replace the file". Delete first, or write elsewhere and `move`.
- **`copy` and `move` replace existing files silently.**
- **`destination` is the new path, not the target folder.**
- **`delete()` on a folder is recursive and final.**
- **`serialize` destroys the old content when it fails.**
- **`deserialize` on untrusted files.** It can run code. Never.
- **Truncating size conversion.** `size(KILO_BYTE)` of 1500 bytes is `1`.
- **`size` of an entry from `dir()` for a folder is `0`.**
- **`search` patterns are `LIKE`, not shell globs.** Use `%`, not `*`.
- **Building paths from user input.** Check that the result stays under the intended folder.
- **Unzipping into a shared folder.** Entries before a refused one stay written; use a private folder.
- **Treating `unzip` of a non-ZIP file as success.** It extracts nothing.
- **Multiple channels without an `ETag`.** jfile falls back to one; do not expect speed.
- **Downloads without a timeout.** A silent server blocks the call.
- **Processing a file on `CREATED`.** It may still be empty. Move complete files in.
- **Expecting one event per change.** Repeats happen.
- **Slow listeners.** They delay all later events.
- **Letting `main` return while watching.** The daemon watcher ends with the JVM.
- **Forgetting `stopListening()`.** The watcher outlives the object.
- **Sharing one `JFile` between threads that change it.** `move`, `back` and `download` change the location.

### C.2 Behaviours worth knowing

- Text is always UTF-8. For a file in another encoding, `read` gives replacement characters, while `readLines` and `lineCount` fail.
- Ranges in `read` are clamped to the file; edges are moved to whole characters (start backward, end forward).
- Symbolic links are skipped by `size`, `copy`, `zip`, `search` and watching, and refused by `unzip` when a write would pass through one.
- `zip` and `download` write to a temporary file in the destination's folder and rename it at the end (a copy is used first if that file ended up on another disk).
- `delete()` and `createDirectory()` are safe to repeat.
- `setListener` on an object that is already watching replaces the watcher.
- `FileAction.CHANGED` exists but is never delivered; `UPDATED` covers changes.

### C.3 Limits

| Limit | Value |
|-------|-------|
| One `read` call | up to about 2 GB (2,147,483,647 bytes); larger ranges fail |
| Download channels | at most 64, never more than the number of bytes |
| Download with several channels | needs `Content-Length`, range support and a strong `ETag` |
| Zip features | standard ZIP; no password, no splitting |
| `stopListening()` wait | up to 5 seconds for the watcher thread |
| Threads | no locking inside `JFile` |
| Watch events | delivered one at a time on one thread; repeats possible |

### C.4 Tuning

- Use `read` in chunks of a few hundred kilobytes to a few megabytes for big files; `lineCount` instead of `readLines().size()`.
- Choose 2 to 8 download channels; more rarely helps and the server may refuse.
- Keep watched folders small and shallow; every subfolder is registered.
- Do the real work of an event on another thread.
- Put temporary and final files on the same disk, so `move` is a rename.

## Appendix D. Message reference

### D.1 Messages of `JFileException`

| Message | When |
|---------|------|
| `Failed to read file: <path>` | `read` |
| `Failed to read lines from file: <path>` | `readLines` |
| `Failed to count lines in file: <path>` | `lineCount` |
| `Failed to read file length: <path>` | `length` |
| `Requested range (<n> bytes) is larger than read() can return as a single String (max 2147483647 bytes): <path>` | `read` of a huge range |
| `Failed to write to file: <path>` | both `write` methods |
| `Failed to serialize object to file: <path>` | `serialize` |
| `Failed to deserialize object from file: <path>` | `deserialize` |
| `Failed to move to: <destination>` | `move` |
| `Failed to copy to: <destination>` | `copy` |
| `Cannot copy directory <source> into its own subtree: <destination>` | `copy` of a folder into itself |
| `Failed to delete: <path>` | `delete` |
| `Failed to create directory: <path>` | `createDirectory` |
| `Failed to compute size: <path>` | `size` |
| `Failed to read free space for: <path>`, `Failed to read total space for: <path>` | `freeSpace`, `totalSpace` |
| `Could not determine an accessible filesystem for: <path>` | `freeSpace`, `totalSpace` |
| `Failed to read last modified time: <path>` | `lastModifiedTime` |
| `Failed to list directory: <path>` | `dir` |
| `Failed to search under: <path>` | `search` |
| `Failed to zip: <path>` | `zip` |
| `Failed to unzip: <path>` | `unzip` |
| `Zip entry escapes destination directory: <entry>` | `unzip` |
| `Write would be redirected outside <folder> through an existing symbolic link: <link>` | `unzip`, `copy` |
| `Failed to watch: <path>` | `setListener` |
| `This JFile was constructed with a remote URL, not a local path` | local method on a URL object |

### D.2 Messages of `download`

| Message | When |
|---------|------|
| `No remote location is set; construct this JFile with a URL before download(...)` | `download` on a local object |
| `Failed to create a temporary file for download: <url>` | no temporary file could be made |
| `Remote location returned HTTP <status>: <url>` | the `HEAD` request was not a success |
| `Remote location returned HTTP <status> on GET: <url>` | the check of an empty resource |
| `Remote location did not report a Content-Length, cannot size the destination file: <url>` | no length |
| `Remote location reported Content-Length 0 on HEAD but the GET returned non-empty content - the resource changed between HEAD and GET: <url>` | an empty resource that was not empty |
| `Remote server did not honour the range request bytes=<a>-<b> (HTTP <status>); aborting to avoid a corrupted file: <url>` | wrong status or `Content-Range` |
| `Remote server did not honour the range request bytes=<a>-<b> (<n> bytes received instead of <m>); aborting to avoid a corrupted file: <url>` | wrong length |
| `Aborted because another channel already failed: bytes=<a>-<b>` | a sibling channel failed first |
| `Failed to download byte range <a>-<b>` | I/O problem in one channel |
| `Failed to download: <url>` | wrapper around the cause of a failed download |

### D.3 Messages written to `System.err`

| Message | When |
|---------|------|
| `JFile: a file event listener threw an exception: <exception>` | the listener threw |
| `JFile: failed to start watching newly created directory <path>: <reason>` | a new subfolder could not be registered |
| `JFile: failed to scan newly created directory <path> for entries created before its registration finished: <reason>` | the catch-up scan failed |
