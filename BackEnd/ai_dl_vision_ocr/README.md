# ai.dl.vision.ocr

*J2OS Platform 2.4*

A clerk photographs an invoice, and the system should read the total. A scanned contract has to become searchable. A photo of a street sign must turn into text that can be translated. In all of these, a program must read the letters in a picture. This is **optical character recognition**, or **OCR**.

**ai.dl.vision.ocr** (`org.j2os.platform.ai.dl.vision.ocr`) brings it to J2OS. It is built on Tesseract, the best-known open-source OCR engine, and has one class, `OpticalCharacterRecognition`, with one task: *give a picture, get its text*. Three properties set it apart:

- **Offline.** Pictures never leave your servers. No online service, no key, no price per page.
- **Persian included.** Persian, Persian with English, and English are built in, and a Persian document with a few Latin words is handled as one text.
- **Simple to hold.** An instance is immutable and safe for any number of threads. Changing the language gives a new instance.

The whole library in a few lines:

```java
OpticalCharacterRecognition ocr = OpticalCharacterRecognition
        .classpathResourceDataDirectory("ai/ocr/model/tessdata-4.1.0")      // 1. where the language data lives
        .language(OpticalCharacterRecognitionLanguage.PERSIAN);              // 2. Persian, with English words
String text = ocr.recognizeText(new File("invoice.jpg"));                    // 3. the text of the picture
```

This guide goes from the first text (chapter 2) through the languages (3), Persian mixed with English (4) and ways of giving the picture (5) to the training data and how to ship it (6), failures (7), the web (8) and a complete small workflow (9). The appendices work as a reference.

## Contents

1. [How ai.dl.vision.ocr works](#1-how-aidlvisionocr-works)
2. [The first text](#2-the-first-text)
3. [Languages](#3-languages)
4. [Persian mixed with English](#4-persian-mixed-with-english)
5. [Ways to give the picture](#5-ways-to-give-the-picture)
6. [The training data: in a folder or in the application](#6-the-training-data-in-a-folder-or-in-the-application)
7. [When it fails](#7-when-it-fails)
8. [For the web](#8-for-the-web)
9. [Putting it together](#9-putting-it-together)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The programs of the guide are complete: each has one `main` method and reads its pictures from a folder named `photos`. The language data sits in the folder `ai/ocr/model/tessdata-4.1.0`, as shown in chapter 6. Their output goes to a folder named `out`.

---

## 1. How ai.dl.vision.ocr works

First, a few terms, because they come up throughout the guide:

- **OCR.** Turning the letters in a picture into text that a program can search, copy and store.
- **Tesseract.** The OCR engine behind the module. It is a native program, reached through the `Tess4J` library, and it works on pictures of printed text.
- **Training data (`tessdata`).** The files that teach Tesseract a language. Each one is named after its language code and ends with `.traineddata`: `eng.traineddata` for English, `fas.traineddata` for Persian. The folder that holds them is called the *data directory*.
- **Language code.** The short name of a language in Tesseract: `eng`, `fas`. Two codes joined by a plus sign, such as `fas+eng`, mean "both together".

The classes a program meets:

| Class | Role |
|-------|------|
| `OpticalCharacterRecognition` | Immutable reader: holds the data directory and a language, turns a picture into text |
| `OpticalCharacterRecognitionLanguage` | The three languages the module offers |
| `TextRecognitionFailedException` | The one checked exception of the recognition |

Keep these four points in mind:

- **The data is yours to supply.** The module contains no language files. They are put in a folder on disk or inside the application, and the module is told where (chapter 6).
- **A reader never changes.** `language(...)` returns a new reader and leaves the old one as it was. A shared base reader can therefore give a reader for every language.
- **Every call is independent.** A new engine is created for each picture, so one reader can serve many threads at once.
- **The result is text, as Tesseract sees it.** It may contain line breaks, empty lines and wrong letters. Pictures that are sharp and straight give the best text.

## 2. The first text

`OpticalCharacterRecognition.dataDirectory` takes the folder with the training data. `recognizeText` takes a picture and returns its text:

```java
import org.j2os.platform.ai.dl.vision.ocr.OpticalCharacterRecognition;

import java.io.File;

public class First {
    public static void main(String[] args) throws Exception {
        OpticalCharacterRecognition ocr = OpticalCharacterRecognition.dataDirectory("ai/ocr/model/tessdata-4.1.0");

        String text = ocr.recognizeText(new File("photos/invoice.png"));

        System.out.println(text.strip());
    }
}
```

```text
Invoice 2041
Total: 350 USD
Paid by card
```

- **The reader.** `dataDirectory(folder)` makes a reader for English. It does not look at the folder; a wrong folder shows up at the first picture (chapter 7).
- **The text.** One line of the picture is one line of the result.
- **`strip()`.** The text usually ends with an extra line break. The examples remove it; so should a program that compares or stores the text.

## 3. Languages

A reader reads one language. The default is English; `language` makes a reader for another:

```java
import org.j2os.platform.ai.dl.vision.ocr.OpticalCharacterRecognition;
import org.j2os.platform.ai.dl.vision.ocr.OpticalCharacterRecognitionLanguage;

import java.io.File;

public class Languages {
    public static void main(String[] args) throws Exception {
        OpticalCharacterRecognition base = OpticalCharacterRecognition.dataDirectory("ai/ocr/model/tessdata-4.1.0");
        OpticalCharacterRecognition persian = base.language(OpticalCharacterRecognitionLanguage.PERSIAN_ONLY);

        File photo = new File("photos/salam.png");
        System.out.println("PERSIAN_ONLY: " + persian.recognizeText(photo).strip().replace("\n\n", " | "));
        System.out.println("ENGLISH:      " + base.recognizeText(photo).strip().replace("\n\n", " | "));

        for (OpticalCharacterRecognitionLanguage language : OpticalCharacterRecognitionLanguage.values()) {
            System.out.println(language + " -> " + language.getTesseractLanguageCode());
        }
    }
}
```

```text
PERSIAN_ONLY: سلام دنیا | امروز هوا خوب است
ENGLISH:      Lio pdlw | caw! 79> I9d jg rol
ENGLISH -> eng
PERSIAN -> fas+eng
PERSIAN_ONLY -> fas
```

- **`PERSIAN_ONLY`.** Code `fas`. The right choice for pure Persian text, such as the picture above.
- **`PERSIAN`.** Code `fas+eng`. Persian together with English: the right choice for documents that also contain Latin letters, digits or English words, which is most Persian documents.
- **`ENGLISH`.** Code `eng`, the default. English does not know the Persian letters; the output above shows what happens when a Persian picture is read as English.
- **Both files must exist.** `PERSIAN` needs `fas.traineddata` and `eng.traineddata` in the data directory.
- **A new reader each time.** `base.language(...)` leaves `base` unchanged, as the English line above shows.

## 4. Persian mixed with English

A picture with Persian and English lines is the everyday case. Tesseract marks the direction of the text with invisible characters, and the result may contain them:

```java
import org.j2os.platform.ai.dl.vision.ocr.OpticalCharacterRecognition;
import org.j2os.platform.ai.dl.vision.ocr.OpticalCharacterRecognitionLanguage;

import java.io.File;

public class Mixed {
    public static void main(String[] args) throws Exception {
        OpticalCharacterRecognition ocr = OpticalCharacterRecognition.dataDirectory("ai/ocr/model/tessdata-4.1.0")
                .language(OpticalCharacterRecognitionLanguage.PERSIAN);

        String raw = ocr.recognizeText(new File("photos/mixed.png"));
        System.out.println("characters with direction marks: " + raw.strip().length());

        String clean = raw.replaceAll("[\\u200E\\u200F]", "").strip();
        System.out.println("characters after cleaning:        " + clean.length());
        System.out.println(clean);
    }
}
```

```text
characters with direction marks: 47
characters after cleaning:        43
Order 1205
سفارش شما ارسال شد
Shipped today
```

- **`PERSIAN`.** All three lines are read, English and Persian.
- **Invisible marks.** The raw text is four characters longer than the visible text. They are the *left-to-right* and *right-to-left* marks (`‎` and `‏`) around the English lines. Browsers need them to place mixed text correctly, but a search for `Order 1205` would not find the text with a mark in front of it.
- **Cleaning.** Remove the two characters before comparing, searching or storing the text. The workflow in chapter 9 does this.
- **Digits.** Western digits such as `1205` come out as they are written.

## 5. Ways to give the picture

Three forms take a picture:

```java
import org.j2os.platform.ai.dl.vision.ocr.OpticalCharacterRecognition;

import java.io.File;
import java.io.FileInputStream;
import java.nio.file.Path;

public class Inputs {
    public static void main(String[] args) throws Exception {
        OpticalCharacterRecognition ocr = OpticalCharacterRecognition.dataDirectory("ai/ocr/model/tessdata-4.1.0");

        System.out.println("File:   " + ocr.recognizeText(new File("photos/invoice.png")).strip().replace("\n", " / "));
        System.out.println("Path:   " + ocr.recognizeText(Path.of("photos/invoice.png")).strip().replace("\n", " / "));

        try (FileInputStream upload = new FileInputStream("photos/invoice.png")) {
            System.out.println("Stream: " + ocr.recognizeText(upload).strip().replace("\n", " / "));
        }
    }
}
```

```text
File:   Invoice 2041 / Total: 350 USD / Paid by card
Path:   Invoice 2041 / Total: 350 USD / Paid by card
Stream: Invoice 2041 / Total: 350 USD / Paid by card
```

- **`File` and `Path`.** Equivalent. The file must exist.
- **`InputStream`.** Made for uploads. The module copies the stream to a temporary file, reads it and removes the file. The stream is not closed by the module; close it yourself, as the example does with `try`-with-resources.
- **Formats.** Any picture format that Tesseract's image reader knows, such as PNG, JPEG, TIFF and BMP.
- **One reader.** All three calls above use the same reader; it holds no state.

## 6. The training data: in a folder or in the application

Tesseract reads its language files from a folder on disk. The module gives two ways to name it:

```text
src/main/resources/
└── ai/ocr/model/tessdata-4.1.0/
    ├── eng.traineddata
    └── fas.traineddata
```

`dataDirectory(path)` names a folder on disk. `classpathResourceDataDirectory(path)` names a folder inside the application, so the data travels with the program:

```java
import org.j2os.platform.ai.dl.vision.ocr.OpticalCharacterRecognition;

import java.io.File;

public class Resource {
    public static void main(String[] args) throws Exception {
        OpticalCharacterRecognition ocr = OpticalCharacterRecognition.classpathResourceDataDirectory("ai/ocr/model/tessdata-4.1.0");

        System.out.println(ocr.recognizeText(new File("photos/invoice.png")).strip().replace("\n", " / "));
        System.out.println("copied to the temporary folder: " + new File("tmp").list().length + " entries");
    }
}
```

```text
Invoice 2041 / Total: 350 USD / Paid by card
copied to the temporary folder: 0 entries
```

- **Running from the IDE or from a classes folder.** The resource is already a folder on disk, and the module uses it where it is. Nothing is copied: the temporary folder above is still empty.
- **Running from a jar.** Tesseract cannot read inside a jar. The module copies the data folder once to the system temporary directory and uses the copy:

```java
import org.j2os.platform.ai.dl.vision.ocr.OpticalCharacterRecognition;

import java.io.File;
import java.util.Arrays;

public class Packaged {
    public static void main(String[] args) throws Exception {
        OpticalCharacterRecognition first = OpticalCharacterRecognition.classpathResourceDataDirectory("ai/ocr/model/tessdata-4.1.0");
        System.out.println(first.recognizeText(new File("photos/invoice.png")).strip().replace("\n", " / "));

        File cache = new File("tmp").listFiles()[0].listFiles()[0];
        String[] files = cache.list();
        Arrays.sort(files);
        System.out.println("cache: " + Arrays.toString(files));

        File marker = new File(cache, ".extraction-complete");
        long stamp = marker.lastModified();
        OpticalCharacterRecognition.classpathResourceDataDirectory("ai/ocr/model/tessdata-4.1.0");
        System.out.println("second call reuses the cache: " + (marker.lastModified() == stamp));
    }
}
```

```text
Invoice 2041 / Total: 350 USD / Paid by card
cache: [.extraction-complete, eng.traineddata, fas.traineddata]
second call reuses the cache: true
```

- **The cache.** A folder named `ocr-tessdata-cache-<user>` in the temporary directory holds one subfolder per data directory. The name of the subfolder carries the size and date of the jar, so a new version of the application gets a fresh copy.
- **Complete or nothing.** The files are written to a staging folder and moved into place at the end, and the file `.extraction-complete` marks a finished copy. The module is built to cope with two programs starting at the same time.
- **Reuse.** The second call, in the same program or in a later start, finds the marker and copies nothing.
- **Directory entries.** The jar must contain entries for its folders. Build tools do this by default; if the folder is not found, the message says so.
- **Old copies stay.** Copies of older jars are not deleted. A server that is redeployed often should clean the temporary folder now and then.

## 7. When it fails

Recognition fails when the picture is missing or unreadable and when the engine cannot start:

```java
import org.j2os.platform.ai.dl.vision.ocr.OpticalCharacterRecognition;

import java.io.File;

public class Failures {
    public static void main(String[] args) throws Exception {
        String here = new File(".").getCanonicalPath();
        OpticalCharacterRecognition ocr = OpticalCharacterRecognition.dataDirectory("ai/ocr/model/tessdata-4.1.0");

        System.out.println("blank picture: \"" + ocr.recognizeText(new File("photos/blank.png")).strip() + "\"");

        try {
            ocr.recognizeText(new File("photos/ghost.png"));
        } catch (Exception e) {
            System.out.println("missing -> " + e.getClass().getSimpleName() + ": " + e.getMessage().replace(here, "."));
        }

        try {
            ocr.recognizeText(new File("photos/notes.txt"));
        } catch (Exception e) {
            System.out.println("not an image -> " + e.getClass().getSimpleName() + " caused by " + e.getCause().getClass().getSimpleName());
        }

        try {
            OpticalCharacterRecognition.dataDirectory("no-such-folder").recognizeText(new File("photos/invoice.png"));
        } catch (Exception e) {
            System.out.println("no training data -> " + e.getClass().getSimpleName() + " caused by " + e.getCause().getClass().getSimpleName());
        }

        try {
            OpticalCharacterRecognition.classpathResourceDataDirectory("ai/ocr/model/tessdata-9");
        } catch (Exception e) {
            System.out.println("no resource -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
    }
}
```

```text
blank picture: ""
missing -> TextRecognitionFailedException: java.io.FileNotFoundException: Image file does not exist: ./photos/ghost.png
not an image -> TextRecognitionFailedException caused by TesseractException
no training data -> TextRecognitionFailedException caused by TesseractException
no resource -> FileNotFoundException: tessdata resource directory not found on classpath: ai/ocr/model/tessdata-9 (a jar file must contain directory entries for its resource directories)
```

- **Nothing to read** is not an error: a blank picture gives an empty text.
- **Missing file.** The message names the file.
- **Not a picture, or no language data.** Both come from Tesseract and arrive as the cause. Read the cause's message to see which.
- **Wrong data folder.** `dataDirectory` does not check the folder, so the failure appears at the first picture.
- **Resource not found.** `classpathResourceDataDirectory` throws a `FileNotFoundException` at once, and the message explains what a jar needs.

The arguments are checked before any work starts:

```java
import org.j2os.platform.ai.dl.vision.ocr.OpticalCharacterRecognition;

import java.io.File;
import java.io.InputStream;

public class Checks {
    public static void main(String[] args) throws Exception {
        OpticalCharacterRecognition ocr = OpticalCharacterRecognition.dataDirectory("ai/ocr/model/tessdata-4.1.0");

        try { OpticalCharacterRecognition.dataDirectory(null); }
        catch (Exception e) { System.out.println("null directory -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { ocr.language(null); }
        catch (Exception e) { System.out.println("null language -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { ocr.recognizeText((File) null); }
        catch (Exception e) { System.out.println("null file -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { ocr.recognizeText((InputStream) null); }
        catch (Exception e) { System.out.println("null stream -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
    }
}
```

```text
null directory -> NullPointerException: dataDirectory must not be null
null language -> NullPointerException: recognitionLanguage must not be null
null file -> NullPointerException: imageFile must not be null
null stream -> NullPointerException: imageInputStream must not be null
```

- **`null` values.** A `NullPointerException` that names the argument.

## 8. For the web

The application creates one reader and shares it, since it never changes:

```java
@Bean
public OpticalCharacterRecognition ocr() throws IOException {
    return OpticalCharacterRecognition
            .classpathResourceDataDirectory("ai/ocr/model/tessdata-4.1.0")
            .language(OpticalCharacterRecognitionLanguage.PERSIAN);
}
```

A page sends a picture and shows the text. The upload goes straight from the request to the reader, and a picture that cannot be read becomes a `422`:

```java
@PostMapping(value = "/api/ocr", produces = "text/plain;charset=UTF-8")
public String read(@RequestParam MultipartFile photo) throws IOException {
    try (InputStream upload = photo.getInputStream()) {
        return ocr.recognizeText(upload).replaceAll("[\\u200E\\u200F]", "").strip();
    } catch (TextRecognitionFailedException e) {
        throw new ResponseStatusException(HttpStatus.UNPROCESSABLE_ENTITY, "the picture cannot be read");
    }
}
```

The page shows the result:

```js
const form = new FormData()
form.append('photo', input.files[0])
const response = await fetch('/api/ocr', { method: 'POST', body: form })
if (response.ok) {
  result.value = await response.text()
}
```

- **One reader.** The bean is shared by all requests; the module is made for that.
- **No temporary file of your own.** The stream form handles it.
- **Clean text.** The direction marks are removed before the text goes back.
- **A limit on the upload.** Set a maximum size in the web layer. Large pictures take longer to read.
- **Log the failure.** A `422` hides whether the picture or the server is at fault. A failure that repeats for every picture points to the server, for example to missing data files.

## 9. Putting it together

An office scans its paperwork. Each scan is read, the text is saved next to it, and the program reports which scans contain an order number:

```java
import org.j2os.platform.ai.dl.vision.ocr.OpticalCharacterRecognition;
import org.j2os.platform.ai.dl.vision.ocr.OpticalCharacterRecognitionLanguage;
import org.j2os.platform.ai.dl.vision.ocr.exception.TextRecognitionFailedException;

import java.io.File;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Arrays;

public class Workflow {
    public static void main(String[] args) throws Exception {
        OpticalCharacterRecognition ocr = OpticalCharacterRecognition.dataDirectory("ai/ocr/model/tessdata-4.1.0")
                .language(OpticalCharacterRecognitionLanguage.PERSIAN);

        File[] scans = new File("photos").listFiles((dir, name) -> name.endsWith(".png"));
        Arrays.sort(scans);

        String wanted = "1205";
        for (File scan : scans) {
            try {
                String text = ocr.recognizeText(scan).replaceAll("[\\u200E\\u200F]", "").strip();
                Files.writeString(Path.of("out", scan.getName() + ".txt"), text);
                String hit = text.contains(wanted) ? "  <- contains " + wanted : "";
                System.out.println(scan.getName() + ": " + text.length() + " characters" + hit);
            } catch (TextRecognitionFailedException e) {
                System.out.println(scan.getName() + ": failed");
            }
        }
    }
}
```

```text
blank.png: 0 characters
invoice.png: 40 characters
mixed.png: 43 characters  <- contains 1205
order.png: 37 characters
salam.png: 27 characters
tiny.png: 14 characters
```

- **One reader.** Persian with English covers the whole folder.
- **Cleaned text.** The direction marks are removed before the text is saved and searched.
- **A search.** The scan that has `1205` in it is marked. The same loop could index the text in a database instead.
- **A failed scan.** A picture that cannot be read is reported and does not stop the others.
- **Empty results.** The blank scan gives zero characters. Decide in your program whether an empty text is a result or a problem.

## Appendix A. Quick reference

### A.1 Creating readers

| Call | Meaning |
|------|---------|
| `OpticalCharacterRecognition.dataDirectory(path)` | Reader for a data directory on disk; English |
| `OpticalCharacterRecognition.classpathResourceDataDirectory(path)` | Reader for a data directory inside the application; English; throws `IOException` |
| `language(language)` | A new reader with another language; the original is unchanged |

### A.2 Reading

| Call | Meaning |
|------|---------|
| `recognizeText(File)` | The text of a picture file |
| `recognizeText(Path)` | The same, for a path |
| `recognizeText(InputStream)` | The same, for a stream; copied to a temporary file; the stream stays open |

### A.3 Languages

| Constant | Code | Needs |
|----------|------|-------|
| `ENGLISH` | `eng` | `eng.traineddata` |
| `PERSIAN` | `fas+eng` | `fas.traineddata` and `eng.traineddata` |
| `PERSIAN_ONLY` | `fas` | `fas.traineddata` |

### A.4 Defaults

| Setting | Default |
|---------|---------|
| Language | `ENGLISH` |
| Data directory | None: supplied by the caller |
| Cache folder for resources in a jar | `ocr-tessdata-cache-<user>` in the system temporary directory |
| Marker of a complete copy | `.extraction-complete` |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| Picture file missing | `TextRecognitionFailedException` caused by a `FileNotFoundException` that names the file |
| File is not a picture | `TextRecognitionFailedException` caused by a `TesseractException` |
| Data directory or language file missing | `TextRecognitionFailedException` caused by a `TesseractException`, at the first picture |
| Engine fails while reading | `TextRecognitionFailedException` caused by a `TesseractException` |
| Stream cannot be copied | `TextRecognitionFailedException` caused by an `IOException` |
| Resource directory not on the classpath | `FileNotFoundException` from `classpathResourceDataDirectory`, naming the resource |
| Resource directory not in a jar and not on disk | `FileNotFoundException` |
| Jar copy fails, or a jar entry would leave the target folder | `IOException` from `classpathResourceDataDirectory` |
| Nothing recognized | An empty text, no exception |
| `null` argument | `NullPointerException` that names the argument |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Expecting a perfect text.** OCR makes mistakes, more with poor pictures, unusual fonts and mixed languages. Treat the result as a good draft.
- **Reading Persian as English.** The wrong language gives nonsense. Pick the language for each picture.
- **Using `PERSIAN_ONLY` for documents with Latin words.** The Latin parts come out wrong. Use `PERSIAN`.
- **Forgetting the direction marks.** Text from mixed pictures may contain `‎` and `‏`. Remove them before comparing or searching.
- **Forgetting `strip()`.** The text ends with a line break.
- **Missing data files.** `PERSIAN` needs two files. The failure appears only at the first picture.
- **A jar without directory entries.** The resource folder is not found. Rebuild the jar with entries for its folders.
- **Closing the stream too early or not at all.** The module does not close the stream you give it.
- **Piling up copies.** Each new jar version leaves a copy in the temporary folder. Clean it on a schedule.

### C.2 Behavior worth knowing

- **A new engine for every picture.** Readers share no state. This is what makes them thread-safe.
- **The reader is immutable.** Share it freely.
- **Temporary files.** The stream form writes `jocr-…` files and removes them afterwards.
- **The data directory is read, not checked.** Nothing is validated until the first picture.
- **Pictures are read whole.** Neither pictures nor results are streamed.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Engine | Tesseract, through `Tess4J` |
| Languages | English, Persian, Persian with English |
| Input | Printed text in a picture; formats known to Tesseract |
| Output | A plain text; no positions, no confidence |
| Handwriting | Not a goal of the engine |
| Concurrency | One reader serves any number of threads |

### C.4 Tuning

- **Give the engine a sharp picture.** Straight lines, even light and dark text on a light background help more than any setting.
- **Size matters.** Very small letters are read badly. Scan at 300 DPI or more.
- **Pick the narrowest language.** `PERSIAN_ONLY` and `ENGLISH` make fewer mistakes than `PERSIAN` when a picture has only one language.
- **Clean after reading.** Remove the direction marks and extra blank lines once, in one place.
- **Ship the data with the program.** The classpath form avoids a separate installation step.

## Appendix D. Message reference

### D.1 Recognition

| Message | Exception | Cause |
|---------|-----------|-------|
| `java.io.FileNotFoundException: Image file does not exist: <path>` | `TextRecognitionFailedException` | The picture is missing |
| Text of the cause | `TextRecognitionFailedException` | A `TesseractException` (the engine could not read the picture or load its data) or an `IOException` while copying a stream |

### D.2 Resources

| Message | Exception | Cause |
|---------|-----------|-------|
| `tessdata resource directory not found on classpath: <path> (a jar file must contain directory entries for its resource directories)` | `FileNotFoundException` | The resource directory is not on the classpath |
| `tessdata resource directory not found inside a jar: <path>` | `FileNotFoundException` | The resource is neither on disk nor in a jar |
| `Illegal entry name in jar file: <name>` | `IOException` | A jar entry would be written outside the target folder |

### D.3 Argument checks

| Message | Exception | Cause |
|---------|-----------|-------|
| `<name> must not be null` | `NullPointerException` | `dataDirectory`, `resourceDirectoryPath`, `recognitionLanguage`, `imageFile`, `imagePath` or `imageInputStream` is `null` |
