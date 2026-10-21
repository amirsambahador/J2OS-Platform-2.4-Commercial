# ai.dl.vision.object

*J2OS Platform 2.4*

A shop wants to know how many items lie on a shelf photo. A moderation step must find the pictures that contain a person. A catalog tool should cut every product out of a group shot. In all of these, a program must say *what* is in a picture and *where*. This is **object detection**.

**ai.dl.vision.object** (`org.j2os.platform.ai.dl.vision.object`) brings it to J2OS. It is built on DJL, the Java library for deep learning, and loads a ready-made neural network. Three things set it apart from the rest:

- **Pretrained.** No training and no data set. The module downloads a model on first use and keeps it.
- **Plain results.** Each object comes back with a name, a confidence and a box. Nothing else to learn.
- **Made for servers.** One loaded model serves any number of threads, and the module protects itself against being closed in the middle of a detection.

The library in use, briefly:

```java
try (ObjectDetection detection = ObjectDetection.load()) {                       // 1. load the model once
    List<DetectedObject> objects = detection.detect("street.jpg");               // 2. what is in the picture?
    objects.get(0).saveImage("first.jpg");                                       // 3. cut one object out
    detection.detect("street.jpg", "street-marked.jpg");                         // 4. or draw boxes on a copy
}
```

This guide goes from the first detection (chapter 2) through boxes and crops (3), annotated pictures (4), ways of giving the image (5) and choosing the model (6) to failures (7), the web (8) and a complete small workflow (9).

## Contents

1. [How ai.dl.vision.object works](#1-how-aidlvisionobject-works)
2. [The first detection](#2-the-first-detection)
3. [Boxes and crops](#3-boxes-and-crops)
4. [A picture with boxes drawn](#4-a-picture-with-boxes-drawn)
5. [Ways to give the image](#5-ways-to-give-the-image)
6. [Choosing the model](#6-choosing-the-model)
7. [When it fails](#7-when-it-fails)
8. [For the web](#8-for-the-web)
9. [A full application](#9-a-full-application)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

None of the programs is a fragment: each has one `main` method and reads its pictures from a folder named `photos`. The pictures they produce go to a folder named `out`.

---

## 1. How ai.dl.vision.object works

A short glossary before we start:

- **Object detection.** Finding the things in a picture. The result is a list: for each thing, its kind, how sure the model is, and where it is.
- **Class name.** The kind of an object, as a word, for example `person` or `car`. The set of names depends on the model.
- **Probability (confidence).** A number from 0 to 1. A value of 0.9 means the model is quite sure; 0.3 means it is guessing.
- **Bounding box.** The rectangle around an object. Here it is given in *fractions* of the picture: `x` and `y` are the left and top edges, `width` and `height` the size, all between 0 and 1. A box with `x=0.5` starts in the middle of the picture, whatever its size in pixels.
- **Model.** The trained neural network. The module offers a few well-known ones (YOLO and SSD). Loading one is slow; using it is fast.

The classes you will use:

| Class | Role |
|-------|------|
| `ObjectDetection` | Loads a model once and detects objects in images |
| `DetectedObject` | One result: name, confidence, box, and a way to save the object's picture |
| `ObjectDetectionModel` | The models to choose from |
| `ObjectDetectionFailedException` | The one checked exception of the module |

Four things shape what follows:

- **Load once, close once.** Loading is expensive. An `ObjectDetection` is created at start-up, used for every image and closed at shutdown.
- **The first load downloads.** DJL fetches the model into a cache directory the first time and reads it from there afterwards. The machine needs network access once.
- **Calls can run side by side.** Only the model is shared; every call makes its own predictor. Many threads can use one instance.
- **Results remember their picture.** Every `DetectedObject` keeps a reference to the image it came from, so it can save its own crop later. Keeping many results keeps their pictures in memory.

## 2. The first detection

`ObjectDetection.load()` loads the default model. `detect` takes the path of a picture and returns the objects in it:

```java
import org.j2os.platform.ai.dl.vision.object.DetectedObject;
import org.j2os.platform.ai.dl.vision.object.ObjectDetection;

import java.util.List;

public class First {
    public static void main(String[] args) throws Exception {
        try (ObjectDetection detection = ObjectDetection.load()) {
            List<DetectedObject> objects = detection.detect("photos/toys.jpg");

            for (DetectedObject object : objects) {
                System.out.printf("%s %.0f%%  x=%.2f y=%.2f width=%.2f height=%.2f%n",
                        object.className(), object.probability() * 100,
                        object.boundingBoxX(), object.boundingBoxY(),
                        object.boundingBoxWidth(), object.boundingBoxHeight());
            }
            System.out.println(objects.size() + " objects");
        }
    }
}
```

```text
ball 84%  x=0.10 y=0.13 width=0.23 height=0.30
box 83%  x=0.70 y=0.13 width=0.19 height=0.25
box 82%  x=0.10 y=0.63 width=0.21 height=0.20
ball 77%  x=0.68 y=0.62 width=0.15 height=0.20
ball 72%  x=0.49 y=0.18 width=0.10 height=0.14
5 objects
```

- **The list.** One entry per object. The order is the model's; do not rely on it.
- **The numbers.** `probability()` is between 0 and 1; the printout multiplies it by 100.
- **The box.** `boundingBoxX()` and `boundingBoxY()` locate the top-left corner, `boundingBoxWidth()` and `boundingBoxHeight()` give the size. All four are fractions of the picture, so the first ball starts a tenth of the way in from the left and covers about a quarter of the width.
- **No filter.** The module returns everything the model reports. Choosing a minimum confidence is the caller's job (chapter 9).
- **`try`-with-resources.** `ObjectDetection` is `AutoCloseable`, so the model is released at the end of the block.

## 3. Boxes and crops

A box is a recipe for a crop. `saveImage` cuts the object out of the original picture and writes it to a file:

```java
import org.j2os.platform.ai.dl.vision.object.DetectedObject;
import org.j2os.platform.ai.dl.vision.object.ObjectDetection;

import javax.imageio.ImageIO;
import java.io.File;
import java.util.List;

public class Crops {
    public static void main(String[] args) throws Exception {
        try (ObjectDetection detection = ObjectDetection.load()) {
            List<DetectedObject> objects = detection.detect("photos/toys.jpg");

            int number = 1;
            for (DetectedObject object : objects) {
                String file = "out/" + object.className() + "-" + number++ + ".jpg";
                object.saveImage(file);
                var crop = ImageIO.read(new File(file));
                System.out.println(file + " -> " + crop.getWidth() + " x " + crop.getHeight());
            }

            var photo = ImageIO.read(new File("photos/toys.jpg"));
            System.out.println("photo is " + photo.getWidth() + " x " + photo.getHeight());
        }
    }
}
```

```text
out/ball-1.jpg -> 181 x 181
out/box-2.jpg -> 151 x 151
out/box-3.jpg -> 171 x 121
out/ball-4.jpg -> 121 x 121
out/ball-5.jpg -> 81 x 81
photo is 800 x 600
```

- **What is saved.** The region of the object from the original picture, without any drawn lines. The format follows the extension; a name without one gives PNG.
- **Size.** The fractions are multiplied by the size of the picture and rounded. A ball with a width of `0.15` in an 800-pixel picture is about 120 pixels wide.
- **Safe at the edges.** A box that reaches past the border is cut at the border, and a box is never smaller than one pixel.
- **Complete files.** The crop is written to a temporary file next to the target and moved into place afterwards, and missing folders are created.
- **Careful with the target.** The path may even be the path of the original picture, which then is replaced by the crop. Use a different name.

## 4. A picture with boxes drawn

For a quick look at what the model sees, `detect` has a form with a second path. It also draws the boxes and labels on a copy of the picture:

```java
import org.j2os.platform.ai.dl.vision.object.ObjectDetection;

import java.io.File;
import java.nio.file.Files;

public class Annotated {
    public static void main(String[] args) throws Exception {
        try (ObjectDetection detection = ObjectDetection.load()) {
            var objects = detection.detect("photos/toys.jpg", "out/toys-marked.jpg");
            System.out.println("found " + objects.size() + ", saved out/toys-marked.jpg: " + new File("out/toys-marked.jpg").exists());

            byte[] png = detection.detectAndDrawBoundingBoxes(new File("photos/toys.jpg"));
            System.out.println("PNG bytes: " + (png[1] == 'P' && png[2] == 'N' && png[3] == 'G'));

            detection.detect("photos/toys.jpg", "out/toys-marked");
            System.out.println("no extension: " + (Files.readAllBytes(new File("out/toys-marked").toPath())[1] == 'P' ? "PNG" : "other"));
        }
    }
}
```

```text
found 5, saved out/toys-marked.jpg: true
PNG bytes: true
no extension: PNG
```

- **`detect(path, outputPath)`.** Detects once, draws and saves. The returned list is the same as from `detect(path)`.
- **Clean crops.** The objects in that list still point to the picture *without* lines, so their `saveImage` crops stay clean.
- **`detectAndDrawBoundingBoxes(file)`.** The same picture as bytes, always PNG. It is the form for a web response; no file is written.
- **The format.** In the file form it follows the extension, and PNG is used when there is none. Transparency is replaced by white for JPEG and BMP.
- **The same path.** The output may equal the input path to mark a picture in place.

## 5. Ways to give the image

Four forms take a picture:

```java
import org.j2os.platform.ai.dl.vision.object.ObjectDetection;

import java.io.File;
import java.io.FileInputStream;
import org.j2os.platform.ai.dl.vision.object.exception.ObjectDetectionFailedException;
import java.nio.file.Path;

public class Inputs {
    public static void main(String[] args) throws Exception {
        try (ObjectDetection detection = ObjectDetection.load()) {
            System.out.println("String: " + detection.detect("photos/shelf.jpg").size());
            System.out.println("File:   " + detection.detect(new File("photos/shelf.jpg")).size());
            System.out.println("Path:   " + detection.detect(Path.of("photos/shelf.jpg")).size());

            try (FileInputStream stream = new FileInputStream("photos/shelf.jpg")) {
                System.out.println("Stream: " + detection.detect(stream).size());
            } catch (ObjectDetectionFailedException e) {
                System.out.println("Stream: " + e.getCause().getClass().getSimpleName());
            }
        }
    }
}
```

```text
String: 4
File:   4
Path:   4
Stream: FileAlreadyExistsException
```

- **`String`, `File`, `Path`.** Equivalent. The picture must exist, or the call fails (chapter 7).
- **`InputStream`.** The module copies the stream to a temporary file, detects, and removes the file. The stream is not closed by the module. In this version the copy fails because the temporary file already exists, and the call ends with a `FileAlreadyExistsException` as its cause. Until that is changed, write the stream to a file yourself and pass the `File`.
- **Formats.** Any format that Java's image readers know, such as JPEG, PNG, GIF and BMP.

## 6. Choosing the model

`ObjectDetectionModel` names the models that DJL can fetch. The module offers five names for four models:

```java
import org.j2os.platform.ai.dl.vision.object.ObjectDetection;
import org.j2os.platform.ai.dl.vision.object.ObjectDetectionModel;

public class Models {
    public static void main(String[] args) throws Exception {
        for (ObjectDetectionModel model : ObjectDetectionModel.values()) {
            System.out.println(model + " -> " + model.getArtifactId());
        }

        try (ObjectDetection detection = ObjectDetection.load("cache", ObjectDetectionModel.YOLO_V8_NANO)) {
            System.out.println("cache directory: " + System.getProperty("DJL_CACHE_DIR"));
            System.out.println("objects: " + detection.detect("photos/one.jpg").size());
        }
    }
}
```

```text
AUTO -> yolo11n
SSD -> ssd
YOLO_V5_SMALL -> yolov5s
YOLO_V8_NANO -> yolov8n
YOLO_11_NANO -> yolo11n
cache directory: cache
objects: 1
```

- **`AUTO`.** The module's default, today `yolo11n`. It is the same as `YOLO_11_NANO`.
- **The choices.** `SSD`, `YOLO_V5_SMALL`, `YOLO_V8_NANO` and `YOLO_11_NANO`. The nano and small variants are the fast ones. `getArtifactId()` returns the name that DJL uses.
- **Where the model lives.** `load()` and `load(model)` use DJL's default cache, `.djl.ai/cache` in the user's home, or the folder named by the environment variable `DJL_CACHE_DIR`.
- **`load(directory)` and `load(directory, model)`.** Use the given folder for downloading and reading. The folder is passed to DJL through the system property `DJL_CACHE_DIR`, which belongs to the whole program.
- **Call it first.** Call the directory form once, at start-up, before other DJL models are loaded. Different folders in different calls would confuse each other.
- **Offline servers.** Run the program once with network access so the cache fills, then ship the folder with the application.

## 7. When it fails

Detection fails when the picture is missing or unreadable, when the model is closed, and when saving a crop fails:

```java
import org.j2os.platform.ai.dl.vision.object.ObjectDetection;

import java.io.File;

public class Failures {
    public static void main(String[] args) throws Exception {
        String here = new File(".").getCanonicalPath();
        ObjectDetection detection = ObjectDetection.load();

        System.out.println("blank picture: " + detection.detect("photos/blank.jpg").size() + " objects, no exception");

        try {
            detection.detect("photos/ghost.jpg");
        } catch (Exception e) {
            System.out.println("missing -> " + e.getClass().getSimpleName() + ": " + e.getMessage().replace(here, "."));
        }

        try {
            detection.detect("photos/notes.txt");
        } catch (Exception e) {
            System.out.println("not an image -> " + e.getClass().getSimpleName() + " caused by " + e.getCause().getClass().getSimpleName());
        }

        var objects = detection.detect("photos/one.jpg");
        try {
            objects.get(0).saveImage("out/ball.xyz");
        } catch (Exception e) {
            System.out.println("odd extension -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }

        detection.close();
        try {
            detection.detect("photos/one.jpg");
        } catch (Exception e) {
            System.out.println("after close -> " + e.getClass().getSimpleName() + ": " + e.getMessage());
        }
        System.out.println("objects found earlier can still be saved: " + objects.get(0).className());
        objects.get(0).saveImage("out/ball.png");
        System.out.println("saved: " + new File("out/ball.png").exists());
    }
}
```

```text
blank picture: 0 objects, no exception
missing -> ObjectDetectionFailedException: java.io.FileNotFoundException: Image file does not exist: ./photos/ghost.jpg
not an image -> ObjectDetectionFailedException caused by IOException
odd extension -> ObjectDetectionFailedException: java.io.IOException: No image writer found for format: xyz
after close -> ObjectDetectionFailedException: This ObjectDetection instance has been closed
objects found earlier can still be saved: ball
saved: true
```

- **A picture with nothing in it** is not an error: the result is an empty list.
- **Missing picture.** The message names the file.
- **Not a picture.** The call ends with an `ObjectDetectionFailedException` caused by an `IOException`.
- **A format nobody can write.** `saveImage` and the annotated forms report the format.
- **After `close`.** New detections are refused with a clear message. Objects found earlier can still save their crops, because each holds its own picture.

The arguments are checked before any work starts:

```java
import org.j2os.platform.ai.dl.vision.object.ObjectDetection;
import org.j2os.platform.ai.dl.vision.object.ObjectDetectionModel;

import java.io.File;

public class Checks {
    public static void main(String[] args) throws Exception {
        try (ObjectDetection detection = ObjectDetection.load()) {
            try { detection.detect((String) null); }
            catch (Exception e) { System.out.println("null path -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

            try { detection.detect("photos/one.jpg", null); }
            catch (Exception e) { System.out.println("null output -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

            try { detection.detect((File) null); }
            catch (Exception e) { System.out.println("null file -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
        }

        try { ObjectDetection.load((ObjectDetectionModel) null); }
        catch (Exception e) { System.out.println("null model -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
    }
}
```

```text
null path -> NullPointerException: imagePath must not be null
null output -> NullPointerException: outputImagePath must not be null
null file -> NullPointerException: imageFile must not be null
null model -> NullPointerException: model must not be null
```

- **`null` values.** A `NullPointerException` that names the argument.
- **Loading failures.** If the model cannot be loaded, for example because it cannot be downloaded, `load` ends with an `ObjectDetectionFailedException` whose cause is the error from DJL.

## 8. For the web

The application loads the model once, as a bean that is closed when the application stops:

```java
@Bean(destroyMethod = "close")
public ObjectDetection objectDetection() throws ObjectDetectionFailedException {
    return ObjectDetection.load("/var/lib/app/models");   // one model for the whole application
}
```

A page sends a photo and shows the same photo with the boxes drawn. A photo that cannot be read is the user's problem, so it becomes a `422`:

```java
@PostMapping(value = "/api/marked", produces = "image/png")
public byte[] marked(@RequestParam MultipartFile photo) throws IOException {
    File upload = File.createTempFile("upload-", ".img");
    try {
        photo.transferTo(upload);
        return objectDetection.detectAndDrawBoundingBoxes(upload);
    } catch (ObjectDetectionFailedException e) {
        throw new ResponseStatusException(HttpStatus.UNPROCESSABLE_ENTITY, "the photo cannot be read");
    } finally {
        upload.delete();
    }
}
```

The page shows the result:

```js
const form = new FormData()
form.append('photo', input.files[0])
const response = await fetch('/api/marked', { method: 'POST', body: form })
if (response.ok) {
  preview.src = URL.createObjectURL(await response.blob())
}
```

Another endpoint answers with the list of objects, for pages that draw their own overlays:

```java
@PostMapping("/api/objects")
public List<Map<String, Object>> objects(@RequestParam MultipartFile photo) throws IOException, ObjectDetectionFailedException {
    File upload = File.createTempFile("upload-", ".img");
    try {
        photo.transferTo(upload);
        return objectDetection.detect(upload).stream()
                .filter(o -> o.probability() >= 0.6)
                .map(o -> Map.<String, Object>of("name", o.className(), "confidence", Math.round(o.probability() * 100)))
                .toList();
    } finally {
        upload.delete();
    }
}
```

- **One instance.** The bean is shared by all requests. The module is built for that.
- **A temporary file.** The upload is saved briefly and always removed.
- **Filter the confidence.** The endpoint keeps objects the model is at least 60 percent sure about.
- **A limit on the upload.** Set a maximum size in the web layer. Large pictures take longer to detect.
- **Log the failure.** The controller turns every `ObjectDetectionFailedException` into a `422`, including a model that failed. Log the exception: a failure that repeats for every photo points to the server, not to the photos.

## 9. A full application

A site receives photos and wants to know which toys appear in them. Detections run in parallel on a small thread pool that shares one loaded model, and only confident results are counted:

```java
import org.j2os.platform.ai.dl.vision.object.DetectedObject;
import org.j2os.platform.ai.dl.vision.object.ObjectDetection;
import org.j2os.platform.ai.dl.vision.object.exception.ObjectDetectionFailedException;

import java.io.File;
import java.util.Arrays;
import java.util.TreeMap;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class Workflow {
    public static void main(String[] args) throws Exception {
        ObjectDetection detection = ObjectDetection.load();
        ExecutorService pool = Executors.newFixedThreadPool(3);
        TreeMap<String, Integer> counts = new TreeMap<>();

        File[] uploads = new File("photos").listFiles((dir, name) -> name.endsWith(".jpg"));
        Arrays.sort(uploads);
        for (File upload : uploads) {
            pool.submit(() -> {
                try {
                    for (DetectedObject object : detection.detect(upload)) {
                        if (object.probability() >= 0.75) {
                            synchronized (counts) { counts.merge(object.className(), 1, Integer::sum); }
                        }
                    }
                } catch (ObjectDetectionFailedException e) {
                    System.out.println(upload.getName() + " failed");
                }
            });
        }

        pool.shutdown();
        pool.awaitTermination(1, TimeUnit.MINUTES);
        detection.close();

        System.out.println("photos: " + uploads.length);
        counts.forEach((name, count) -> System.out.println(name + ": " + count));
    }
}
```

```text
photos: 4
ball: 3
box: 4
```

- **One model, three threads.** Every thread calls `detect` on the same instance.
- **A threshold.** Results under 75 percent are ignored. Pick the value on your own photos: a low value finds more and invents more.
- **A failed photo.** A photo that cannot be read is reported and does not stop the others.
- **Closing at the end.** `close` waits for detections that are still running, so it is safe to call as soon as the pool is finished.

## Appendix A. Quick reference

### A.1 Loading

| Call | Meaning |
|------|---------|
| `ObjectDetection.load()` | Default model (`AUTO`), default DJL cache |
| `ObjectDetection.load(model)` | The given model, default cache |
| `ObjectDetection.load(directory)` | Default model, given cache folder (sets `DJL_CACHE_DIR`) |
| `ObjectDetection.load(directory, model)` | The given model and cache folder |
| `close()` | Waits for running detections, then releases the model; repeated calls are harmless |

### A.2 Detecting

| Call | Meaning |
|------|---------|
| `detect(String)`, `detect(File)`, `detect(Path)` | Objects in a picture |
| `detect(InputStream)` | Objects in a stream; copied to a temporary file first |
| `detect(imagePath, outputPath)` | Objects, and a copy with boxes drawn, saved as a file |
| `detectAndDrawBoundingBoxes(File)` | The copy with boxes as PNG bytes |

### A.3 DetectedObject

| Method | Meaning |
|--------|---------|
| `className()` | The kind of the object |
| `probability()` | Confidence, 0 to 1 |
| `boundingBoxX()`, `boundingBoxY()` | Left and top edge, as fractions of the picture |
| `boundingBoxWidth()`, `boundingBoxHeight()` | Size, as fractions of the picture |
| `saveImage(path)` | Saves the crop of this object from the original picture |

### A.4 ObjectDetectionModel

| Constant | Artifact id |
|----------|-------------|
| `AUTO` | `yolo11n` |
| `SSD` | `ssd` |
| `YOLO_V5_SMALL` | `yolov5s` |
| `YOLO_V8_NANO` | `yolov8n` |
| `YOLO_11_NANO` | `yolo11n` |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| Model cannot be loaded or downloaded | `ObjectDetectionFailedException` caused by the DJL error |
| Picture file missing | `ObjectDetectionFailedException` caused by a `FileNotFoundException` that names the file |
| File is not a picture | `ObjectDetectionFailedException` caused by an `IOException` |
| Detection fails inside the model | `ObjectDetectionFailedException` caused by the underlying error |
| Instance already closed | `ObjectDetectionFailedException`: "This ObjectDetection instance has been closed" |
| Output extension has no writer | `ObjectDetectionFailedException` caused by an `IOException`: "No image writer found for format" |
| Output folder cannot be created or written | `ObjectDetectionFailedException` caused by an `IOException` |
| `detect(InputStream)` | `ObjectDetectionFailedException` caused by a `FileAlreadyExistsException` in this version |
| Nothing found | An empty list, no exception |
| `null` argument | `NullPointerException` that names the argument |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Loading per request.** Loading takes seconds and a lot of memory. Load once and share.
- **Forgetting `close`.** The model holds native memory. Close it at shutdown, or use `try`-with-resources.
- **Keeping results for long.** Each `DetectedObject` holds its picture. If only the numbers are needed, copy them out and let the list go.
- **Trusting every result.** The module does not filter. Compare `probability()` with a threshold you chose.
- **Treating the stream form as working.** See chapter 5: pass a `File` instead.
- **Changing the cache folder repeatedly.** The directory forms of `load` set a program-wide property. Use one folder, set it once, and do it before other DJL models are loaded.
- **Saving a crop over the original.** `saveImage` with the original path replaces the original.
- **Counting on the order.** Results come in the order the model gives them.
- **Expecting names you did not train.** The model reports only the kinds it learned. A rare object may be missed or called something else.

### C.2 Behavior worth knowing

- **A predictor per call.** Only the model is shared, so calls do not disturb each other.
- **Closing waits.** `close` waits until running detections finish; calls started afterwards fail.
- **Detection runs once.** The annotated form detects a single time and returns the objects of that run.
- **Source picture of the results.** In the annotated form it is a copy made before the boxes are drawn.
- **Complete files.** Crops and annotated pictures are written to a temporary file and moved into place.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Models | `ssd`, `yolov5s`, `yolov8n`, `yolo11n` from the DJL model zoo |
| Network | Needed once per model, to fill the cache |
| Boxes | Fractions of the picture, one rectangle per object |
| Confidence filter | None in the module |
| Input | Any picture format that Java can read |
| Output | Any format that Java can write; PNG for bytes |
| Concurrency | One instance serves any number of threads |

### C.4 Tuning

- **Pick the model for the job.** The nano models are fast and good enough for many photos; try `SSD` or another if the results disappoint.
- **Set a threshold from data.** Run a few hundred of your own photos and choose the value that balances missed and invented objects.
- **Shrink big photos first.** The model works on a fixed size anyway, so a huge picture only costs reading time.
- **Warm up at start-up.** The first detection after loading is the slowest; run one small picture when the application starts.
- **Keep the cache folder.** Put it on a persistent disk so a restart does not download the model again.

## Appendix D. Message reference

### D.1 Detection

| Message | Exception | Cause |
|---------|-----------|-------|
| `java.io.FileNotFoundException: Image file does not exist: <path>` | `ObjectDetectionFailedException` | The picture is missing |
| `This ObjectDetection instance has been closed` | `ObjectDetectionFailedException` | A detection was started after `close` |
| `java.io.IOException: No image writer found for format: <ext>` | `ObjectDetectionFailedException` | The output extension cannot be written |
| Text of the cause | `ObjectDetectionFailedException` | The model could not be loaded, the picture could not be read, the model failed, or a file could not be written |

### D.2 Argument checks

| Message | Exception | Cause |
|---------|-----------|-------|
| `<name> must not be null` | `NullPointerException` | `model`, `modelCacheDirectory`, `imagePath`, `imageFile`, `imageInputStream` or `outputImagePath` is `null` |
