# ai.dl.vision.face

*J2OS Platform 2.4*

A user uploads a holiday photo, and the profile page needs a small square with only the face in it. A group picture has to be split into one picture per person. A recognition step needs faces, not rooms and trees around them. In all of these, a program must find the faces in a photograph and cut them out. This is **face detection** followed by **cropping**.

**ai.dl.vision.face** (`org.j2os.platform.ai.dl.vision.face`) brings it to J2OS. It has one static class, `FaceCrop`, built on OpenCV and its classic face detector. Three properties set it apart:

- **Nothing to download.** The detector's data file is bundled with the library. There is no model to fetch and no service to call.
- **Plain inputs and outputs.** A path or a `File` goes in; a cropped image file or PNG bytes come out.
- **Careful with photos.** Phone photos that are stored sideways are turned upright first, and files are written in a way that never leaves a half-written picture behind.

The library in a few lines:

```java
FaceCrop.crop("photo.jpg", "face.png");                              // 1. the largest face, saved to a file
byte[] png = FaceCrop.crop(new File("photo.jpg"));                   // 2. or as PNG bytes, no file
List<byte[]> all = FaceCrop.cropAll(new File("group.jpg"));          // 3. every face in the photo
```

This guide goes from the first crop (chapter 2) through the margin around the face (3), photos with several faces (4), file formats (5) and phone photos (6) to failures (7), the detector's own file (8), the web (9) and a complete small workflow (10). The appendices are meant for quick lookups.

## Contents

1. [How ai.dl.vision.face works](#1-how-aidlvisionface-works)
2. [The first crop](#2-the-first-crop)
3. [The margin around the face](#3-the-margin-around-the-face)
4. [Photos with several faces](#4-photos-with-several-faces)
5. [File formats](#5-file-formats)
6. [Photos from a phone](#6-photos-from-a-phone)
7. [When it fails](#7-when-it-fails)
8. [The detector's own file](#8-the-detectors-own-file)
9. [For the web](#9-for-the-web)
10. [In practice](#10-in-practice)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The programs are complete, not fragments: each has one `main` method and reads its pictures from a folder named `photos`. The faces they produce go to a folder named `out`.

---

## 1. How ai.dl.vision.face works

First the vocabulary, which comes back all the time:

- **Face detection.** Finding where faces are in a picture. The result for each face is a rectangle: where it starts and how wide and high it is.
- **Cascade.** The data file that tells the detector what a face looks like. The one in this module is the classic *Haar cascade* for faces seen from the front. It ships inside the library.
- **Crop.** Cutting a rectangle out of a picture.
- **Margin.** A little extra picture kept around the face, so the head is not cut tight. It is a fraction of the face size: `0.1` means 10 percent of the face width on the left and right, and 10 percent of the face height on top and bottom.
- **EXIF orientation.** A note inside a JPEG that says how the camera was held. Phones often store the picture sideways and rely on this note to show it upright.

The classes a program meets:

| Class | Role |
|-------|------|
| `FaceCrop` | Static class: finds faces in an image and crops them, as files or as PNG bytes |
| `FaceCropFailedException` | The one checked exception of the module |

Four points matter most:

- **The detector looks at faces from the front.** A face turned to the side, or tilted a lot, may be missed. Now and then something that is not a face, such as an ear, is reported as one.
- **Small faces are ignored.** A face must be at least a tenth of the shorter side of the picture. In a photo of 1600 by 1600 pixels, a face smaller than 160 pixels is not found.
- **The largest face wins.** Methods that return one face choose the biggest; methods named `cropAll` return every face.
- **The output has the face and the margin only.** Nothing is resized or changed: the crop is a plain cut from the original pixels.

## 2. The first crop

`FaceCrop.crop` takes the path of a photo and the path where the face goes:

```java
import org.j2os.platform.ai.dl.vision.face.FaceCrop;

import javax.imageio.ImageIO;
import java.io.File;

public class First {
    public static void main(String[] args) throws Exception {
        FaceCrop.crop("photos/portrait.jpg", "out/face.png");

        var photo = ImageIO.read(new File("photos/portrait.jpg"));
        var face = ImageIO.read(new File("out/face.png"));
        System.out.println("photo: " + photo.getWidth() + " x " + photo.getHeight());
        System.out.println("face:  " + face.getWidth() + " x " + face.getHeight());
    }
}
```

```text
photo: 512 x 512
face:  110 x 110
```

- **The call.** It reads the photo, finds the largest face and writes it to `out/face.png`. The folder `out` is created if it does not exist.
- **The size.** The face is about 110 pixels wide in a photo of 512. That is the face itself plus the default margin of 10 percent on each side.
- **Where the detector comes from.** The first call extracts the bundled cascade and remembers it for the rest of the program. Chapter 8 shows how to use another one.

## 3. The margin around the face

A tight crop cuts the forehead and the chin. A bigger margin gives an avatar more room. Every cropping method has a form that takes the margin:

```java
import org.j2os.platform.ai.dl.vision.face.FaceCrop;

import javax.imageio.ImageIO;
import java.io.ByteArrayInputStream;
import java.io.File;

public class Margin {
    public static void main(String[] args) throws Exception {
        File photo = new File("photos/portrait.jpg");

        for (float margin : new float[]{0f, 0.1f, 0.5f}) {
            var face = ImageIO.read(new ByteArrayInputStream(FaceCrop.crop(photo, margin)));
            System.out.println("margin " + margin + " -> " + face.getWidth() + " x " + face.getHeight());
        }

        FaceCrop.useMargin(0.3f);
        var usual = ImageIO.read(new ByteArrayInputStream(FaceCrop.crop(photo)));
        System.out.println("default now 0.3 -> " + usual.getWidth() + " x " + usual.getHeight());
    }
}
```

```text
margin 0.0 -> 92 x 92
margin 0.1 -> 110 x 110
margin 0.5 -> 184 x 184
default now 0.3 -> 148 x 148
```

- **Per call.** `crop(photo, margin)` uses its own margin and ignores any default. The three crops above come from the same photo.
- **Zero means a tight crop.** The rectangle of the detector, nothing more.
- **The default.** Without a margin, calls use the default of `0.1`. `FaceCrop.useMargin` changes it, and the next call without a margin shows the new value.
- **Global.** The default belongs to the whole program, not to one caller. A server that serves several pages should pass the margin in every call and leave `useMargin` alone (Appendix C).
- **At the edge.** If the margin reaches past the picture, the crop is simply cut at the border.

## 4. Photos with several faces

A group photo has many faces. `crop` keeps the largest; `cropAll` returns them all, as bytes or as files:

```java
import org.j2os.platform.ai.dl.vision.face.FaceCrop;

import javax.imageio.ImageIO;
import java.io.ByteArrayInputStream;
import java.io.File;
import java.util.List;

public class Many {
    public static void main(String[] args) throws Exception {
        File team = new File("photos/team.jpg");

        List<byte[]> faces = FaceCrop.cropAll(team);
        System.out.println("faces found: " + faces.size());
        for (byte[] png : faces) {
            var image = ImageIO.read(new ByteArrayInputStream(png));
            System.out.println("  " + image.getWidth() + " x " + image.getHeight());
        }

        List<String> names = FaceCrop.cropAll(team, "out/team");
        for (String name : names) {
            System.out.println("saved: " + name.substring(0, 5) + "... (" + name.length() + " characters)");
        }
        System.out.println("files in folder: " + new File("out/team").list().length);

        byte[] largest = FaceCrop.crop(team);
        var big = ImageIO.read(new ByteArrayInputStream(largest));
        System.out.println("crop() keeps the largest: " + big.getWidth() + " x " + big.getHeight());
    }
}
```

```text
faces found: 2
  117 x 117
  71 x 71
saved: team-... (45 characters)
saved: team-... (45 characters)
files in folder: 2
crop() keeps the largest: 117 x 117
```

- **`cropAll(file)`.** A list of PNG byte arrays, one per face. The list can be empty.
- **`cropAll(file, folder)`.** Saves each face as a PNG in the folder and returns the file names. Each name is the photo's name, a dash and a unique code, so two faces never overwrite each other and the same photo can be processed again safely.
- **`crop` on the same photo.** It returns only the largest of the faces.
- **The order.** Faces come in the order the detector finds them. Do not rely on it for anything.

## 5. File formats

The format of the output file follows its extension. A name without an extension gives PNG:

```java
import org.j2os.platform.ai.dl.vision.face.FaceCrop;

import java.io.File;
import java.nio.file.Files;

public class Formats {
    public static void main(String[] args) throws Exception {
        FaceCrop.crop("photos/portrait.jpg", "out/avatar.jpg");
        FaceCrop.crop("photos/portrait.jpg", "out/avatar.png");
        FaceCrop.crop("photos/portrait.png", "out/from-png.jpg");
        FaceCrop.crop("photos/portrait.jpg", "out/plain");

        for (String name : new String[]{"avatar.jpg", "avatar.png", "from-png.jpg", "plain"}) {
            byte[] head = Files.readAllBytes(new File("out/" + name).toPath());
            String kind = (head[0] & 0xFF) == 0xFF ? "JPEG" : head[1] == 'P' ? "PNG" : "other";
            System.out.println(name + " -> " + kind);
        }
        System.out.println("files in out: " + new File("out").list().length);
    }
}
```

```text
avatar.jpg -> JPEG
avatar.png -> PNG
from-png.jpg -> JPEG
plain -> PNG
files in out: 4
```

- **`.jpg` and `.png`.** Any format that Java can write works. A photo in one format can be cropped into another.
- **No extension.** The file is written as PNG.
- **Bytes are always PNG.** The forms that return `byte[]` do not take a format.
- **Transparency.** If a PNG with a transparent background is cropped into a JPEG, the transparent parts become white.
- **Only complete files.** The picture is first written to a temporary file next to the target and moved into place when it is finished, so a reader never finds half a picture. If writing fails, the temporary file is removed.

## 6. Photos from a phone

Phones often store a photo as the sensor saw it, which may be sideways, and add an EXIF note that says "turn this by 90 degrees". Viewers obey the note; a detector that ignores it sees a sideways face and finds nothing. `FaceCrop` applies the note first:

```java
import org.j2os.platform.ai.dl.vision.face.FaceCrop;

import javax.imageio.ImageIO;
import java.io.File;

public class Phone {
    public static void main(String[] args) throws Exception {
        var stored = ImageIO.read(new File("photos/phone.jpg"));
        System.out.println("stored: " + stored.getWidth() + " x " + stored.getHeight());

        FaceCrop.crop("photos/phone.jpg", "out/phone-face.png");

        var face = ImageIO.read(new File("out/phone-face.png"));
        System.out.println("face:   " + face.getWidth() + " x " + face.getHeight());
    }
}
```

```text
stored: 400 x 512
face:   115 x 115
```

- **The stored picture** is 400 wide and 512 high, with the face lying on its side. Its note says it must be turned to be shown upright.
- **The result.** The face is found and cut upright.
- **All eight values.** Rotated and mirrored variants of the note are all handled.
- **JPEG only.** Other formats carry no such note and are read as they are.

## 7. When it fails

Cropping fails when the picture cannot be read, when the folder cannot be written, and when no face is found:

```java
import org.j2os.platform.ai.dl.vision.face.FaceCrop;

import java.io.File;
import java.util.List;

public class Failures {
    public static void main(String[] args) throws Exception {
        String here = new File(".").getCanonicalPath();

        String[][] cases = {
                {"no face", "photos/coffee.jpg"},
                {"face too small", "photos/wide.jpg"},
                {"missing file", "photos/ghost.jpg"},
                {"not an image", "photos/notes.txt"},
        };
        for (String[] test : cases) {
            try {
                FaceCrop.crop(test[1], "out/x.png");
            } catch (Exception e) {
                System.out.println(test[0] + " -> " + e.getClass().getSimpleName() + ": " + e.getMessage().replace(here, "."));
            }
        }

        List<byte[]> none = FaceCrop.cropAll(new File("photos/coffee.jpg"));
        System.out.println("cropAll on a photo without faces: " + none.size() + " results, no exception");
    }
}
```

```text
no face -> FaceCropFailedException: No face detected in the given image
face too small -> FaceCropFailedException: No face detected in the given image
missing file -> FaceCropFailedException: java.io.FileNotFoundException: Image file does not exist: ./photos/ghost.jpg
not an image -> FaceCropFailedException: Unsupported or unreadable image format: ./photos/notes.txt
cropAll on a photo without faces: 0 results, no exception
```

- **No face.** `crop` throws a `FaceCropFailedException`. This is the common case: a photo of a cup, a face seen from the side, a face too small for the picture.
- **Too small.** The second photo has a face, but it is smaller than a tenth of the picture, so it is skipped. Give the detector a closer photo, or crop the picture first.
- **Missing and unreadable files.** The message names the file. A text file, or a file of bytes that no reader understands, gives "Unsupported or unreadable image format". A picture that is only partly damaged may still be read, and then usually yields no face.
- **`cropAll` is different.** A photo without faces is not an error there: the result is an empty list. Check the size of the list.

The arguments are checked before any work starts:

```java
import org.j2os.platform.ai.dl.vision.face.FaceCrop;

import java.io.File;

public class Checks {
    public static void main(String[] args) throws Exception {
        try { FaceCrop.crop("photos/portrait.jpg", "out/x.png", -0.5f); }
        catch (Exception e) { System.out.println("negative margin -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { FaceCrop.useMargin(-1f); }
        catch (Exception e) { System.out.println("useMargin(-1) -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { FaceCrop.crop("photos/portrait.jpg", "out/portrait.xyz"); }
        catch (Exception e) { System.out.println("odd extension -> " + e.getClass().getSimpleName() + ": " + e.getMessage().replace(new File(".").getCanonicalPath(), ".")); }

        try { FaceCrop.crop((String) null, "out/x.png"); }
        catch (Exception e) { System.out.println("null path -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        System.out.println("leftover files: " + new File("out").list().length);
    }
}
```

```text
negative margin -> IllegalArgumentException: margin must not be negative: -0.5
useMargin(-1) -> IllegalArgumentException: fraction must not be negative: -1.0
odd extension -> FaceCropFailedException: No image writer available for format "xyz" (from out/portrait.xyz). Use a supported extension such as .png or .jpg.
null path -> NullPointerException: imagePath must not be null
leftover files: 0
```

- **Negative margin.** An `IllegalArgumentException`, in the per-call form and in `useMargin`.
- **An extension nobody can write.** The crop is made, but no writer exists for the format. The message names the file; no temporary file stays behind.
- **`null` values.** A `NullPointerException` that names the argument.

## 8. The detector's own file

The bundled cascade is found on the classpath at `ai/face/model/haarcascade_frontalface_default.xml`. Two methods let a program use another file instead:

```text
src/main/resources/
└── ai/face/model/
    └── haarcascade_frontalface_default.xml
```

```java
import org.j2os.platform.ai.dl.vision.face.FaceCrop;

public class Cascade {
    public static void main(String[] args) throws Exception {
        FaceCrop.useCascadeFile("ai/face/model/haarcascade_frontalface_default.xml");
        FaceCrop.crop("photos/portrait.jpg", "out/face.png");
        System.out.println("own cascade file works");

        try { FaceCrop.useCascadeFile("models/ghost.xml"); }
        catch (Exception e) { System.out.println("missing file -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { FaceCrop.useCascadeResource("ai/face/model/ghost.xml"); }
        catch (Exception e) { System.out.println("missing resource -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        FaceCrop.crop("photos/portrait.jpg", "out/again.png");
        System.out.println("the earlier cascade is still in use");
    }
}
```

```text
own cascade file works
missing file -> IllegalStateException: Failed to load cascade file (check the path exists and is readable): models/ghost.xml
missing resource -> IllegalStateException: Cascade file not found on classpath at ai/face/model/ghost.xml
the earlier cascade is still in use
```

- **`useCascadeFile(path)`.** A file on disk.
- **`useCascadeResource(path)`.** A file on the classpath, for example inside the application's jar.
- **Checked at once.** The file is loaded when the method is called. A wrong path fails right there with an `IllegalStateException`, not at the first photo.
- **A failed call changes nothing.** The detector in use before stays in use, as the last line shows.
- **Global.** Like the margin, the cascade is one choice for the whole program.

## 9. For the web

A profile page takes a photo and returns the avatar. The server receives the file, crops the face as PNG bytes and answers with the image. A photo without a face is the user's problem, not the server's, so it becomes a `422`:

```java
@PostMapping(value = "/api/avatar", produces = "image/png")
public byte[] avatar(@RequestParam MultipartFile photo) throws IOException {
    File upload = File.createTempFile("upload-", ".img");
    try {
        photo.transferTo(upload);
        return FaceCrop.crop(upload, 0.3f);
    } catch (FaceCropFailedException e) {
        throw new ResponseStatusException(HttpStatus.UNPROCESSABLE_ENTITY, "no usable face in the photo");
    } finally {
        upload.delete();
    }
}
```

The page shows the result:

```js
const form = new FormData()
form.append('photo', input.files[0])
const response = await fetch('/api/avatar', { method: 'POST', body: form })
if (response.ok) {
  preview.src = URL.createObjectURL(await response.blob())
}
```

- **A temporary file.** `crop(File, margin)` reads a file, so the upload is saved briefly and always removed.
- **A fixed margin.** The call passes `0.3` and does not touch the global default.
- **Which status.** `422` tells the page that the upload arrived but cannot be used, which is different from a failure of the server.
- **A limit on the upload.** Set a maximum size for uploads in the web layer. Detection time grows with the size of the picture.

## 10. In practice

A site collects photos from its users. For each photo it makes a square avatar with a generous margin and notes why the others were skipped:

```java
import org.j2os.platform.ai.dl.vision.face.FaceCrop;
import org.j2os.platform.ai.dl.vision.face.exception.FaceCropFailedException;

import java.io.File;
import java.util.Arrays;

public class Workflow {
    public static void main(String[] args) {
        int made = 0;
        int skipped = 0;

        File[] uploads = new File("photos").listFiles();
        Arrays.sort(uploads);

        for (File upload : uploads) {
            String base = upload.getName().replace('.', '-');
            try {
                FaceCrop.crop(upload.getPath(), "avatars/" + base + ".jpg", 0.3f);
                System.out.println(upload.getName() + " -> avatars/" + base + ".jpg");
                made++;
            } catch (FaceCropFailedException e) {
                String reason = e.getMessage().startsWith("No face") ? "no face found" : "not a usable image";
                System.out.println(upload.getName() + " -> skipped (" + reason + ")");
                skipped++;
            }
        }

        System.out.println(made + " avatars, " + skipped + " skipped");
        System.out.println("files in avatars: " + new File("avatars").list().length);
    }
}
```

```text
coffee.jpg -> skipped (no face found)
notes.txt -> skipped (not a usable image)
phone.jpg -> avatars/phone-jpg.jpg
portrait.jpg -> avatars/portrait-jpg.jpg
portrait.png -> avatars/portrait-png.jpg
team.jpg -> avatars/team-jpg.jpg
wide.jpg -> skipped (no face found)
4 avatars, 3 skipped
files in avatars: 4
```

- **One call per photo.** `crop(photoPath, avatarPath, margin)` does the whole job: it reads, finds the face, cuts and writes.
- **Two kinds of skip.** A `FaceCropFailedException` is caught for a text file as well as for a photo without a face. The message tells the two apart.
- **A face too small.** The wide photo is skipped for the reason in chapter 7: its face is under a tenth of its width.
- **The group photo.** It gets an avatar from its largest face.
- **Next step.** The avatars can go straight into a face-comparison step, as the next snippet shows:

```java
FaceCrop.crop("full-body-photo.jpg", "face-only.jpg");
float[] feature = imageSimilarity.extractFeature("face-only.jpg");   // compare faces, not backgrounds
```

## Appendix A. Quick reference

### A.1 Settings

| Method | Meaning |
|--------|---------|
| `useMargin(fraction)` | Default margin, as a fraction of the face size; default `0.1`; global |
| `useCascadeResource(classpathPath)` | Use a cascade file from the classpath; global |
| `useCascadeFile(filePath)` | Use a cascade file from disk; global |

### A.2 One face (the largest)

| Call | Meaning |
|------|---------|
| `crop(imagePath, outputPath)` | Writes the face; the format comes from the extension, PNG without one |
| `crop(imagePath, outputPath, margin)` | The same, with its own margin |
| `crop(File)` | Returns the face as PNG bytes |
| `crop(File, margin)` | The same, with its own margin |

### A.3 All faces

| Call | Meaning |
|------|---------|
| `cropAll(File)` | A list of PNG byte arrays; empty when there are no faces |
| `cropAll(File, margin)` | The same, with its own margin |
| `cropAll(File, folder)` | Saves a PNG per face and returns the file names |
| `cropAll(File, folder, margin)` | The same, with its own margin |

### A.4 Defaults

| Setting | Default |
|---------|---------|
| Margin | `0.1` (10 percent of the face on each side) |
| Cascade | The bundled frontal-face file |
| Smallest face | A tenth of the shorter side of the picture |
| Output of the byte forms | PNG |
| Output of the file forms | By extension; PNG without one |
| File names of `cropAll(File, folder)` | `<photo name>-<unique code>.png` |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| `crop` finds no face | `FaceCropFailedException`: "No face detected in the given image" |
| `cropAll` finds no face | An empty list; the folder form still creates the folder |
| Image file missing | `FaceCropFailedException` caused by a `FileNotFoundException` that names the file |
| File is not a picture, or is damaged | `FaceCropFailedException`: "Unsupported or unreadable image format" |
| Output extension has no writer | `FaceCropFailedException` naming the format and the path; no file is left |
| Output folder cannot be created or written | `FaceCropFailedException` caused by an `IOException` |
| Detection itself fails | `FaceCropFailedException` caused by the underlying error |
| Cascade file missing or invalid | `IllegalStateException` from the method that sets it; if the bundled file is missing from the classpath, the first detection reports it as a `FaceCropFailedException` caused by an `IllegalStateException` |
| Negative or `NaN` margin | `IllegalArgumentException` |
| Negative value to `useMargin` | `IllegalArgumentException` |
| `null` argument | `NullPointerException` that names the argument |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Expecting every face.** The detector is strict: it prefers missing a face to inventing one. Blurred, tilted, covered or small faces are often skipped.
- **Profile photos.** The cascade is trained on faces seen from the front. A side view may be missed, and an ear may be reported as a face.
- **Using `crop` for group photos.** It returns only the largest face. Use `cropAll` when the count matters.
- **Treating "no face" as a server error.** It is usually a property of the photo. Report it to the user.
- **Changing the globals in a shared server.** `useMargin`, `useCascadeFile` and `useCascadeResource` affect every thread. Set them once at start-up, and pass the margin in each call when pages need different ones.
- **Overwriting by name.** When a folder gets avatars from several photos, build the target names from the full source name, or two photos named alike will replace each other.
- **A `NaN` default.** `useMargin` refuses negative values but accepts `NaN`. Every later call without a margin then fails with `margin must not be negative: NaN`. Pass real numbers only.
- **Huge pictures.** A 50-megapixel photo is read fully into memory before detection. Limit the upload size.

### C.2 Behavior worth knowing

- **A new detector for every detection.** Calls do not share state, so they are safe to run in parallel. The cascade path itself is resolved once.
- **The cascade is extracted once.** The bundled file is copied to a temporary file on first use and removed when the program ends.
- **The orientation is applied before detection.** And the crops are cut from the upright picture.
- **Transparency in other formats.** Formats without it get a white background.
- **Whole pictures are read.** Neither the photo nor the crop is streamed.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Detector | The classic frontal-face Haar cascade |
| Faces | Seen from the front; at least a tenth of the shorter side of the picture |
| Detection settings | Fixed in the module (scale step 1.1, 8 neighbors) |
| Input | Any picture format Java can read; the orientation note only in JPEG |
| Output | Any format Java can write; PNG for bytes |
| Output of `cropAll` into a folder | Always PNG |
| Concurrency | Calls are independent and safe to run in parallel |

### C.4 Tuning

- **Pick the margin for the use.** `0` for a recognition step, `0.3` or more for avatars.
- **Resize large photos first.** Smaller pictures detect faster; keep faces above a tenth of the side.
- **Use a better cascade when needed.** Another file from OpenCV may fit your photos better; chapter 8 shows how to switch.
- **Cache the avatars.** The crop of a photo does not change; store it and do not detect twice.
- **Detect once per photo.** For avatars, call `crop`; for galleries, call `cropAll` once and keep the results.

## Appendix D. Message reference

### D.1 Cropping

| Message | Exception | Cause |
|---------|-----------|-------|
| `No face detected in the given image` | `FaceCropFailedException` | `crop` found no face |
| `java.io.FileNotFoundException: Image file does not exist: <path>` | `FaceCropFailedException` | The photo is missing |
| `Unsupported or unreadable image format: <path>` | `FaceCropFailedException` | Not a picture, or damaged |
| `No image writer available for format "<ext>" (from <path>). Use a supported extension such as .png or .jpg.` | `FaceCropFailedException` | The output extension cannot be written |
| Text of the cause | `FaceCropFailedException` | An `IOException` while reading, writing or moving a file, or an error inside the detector |

### D.2 Settings

| Message | Exception | Cause |
|---------|-----------|-------|
| `Cascade file not found on classpath at <path>` | `IllegalStateException` | The classpath resource is missing; during a detection it arrives as the cause of a `FaceCropFailedException` |
| `Failed to load cascade file (check the path exists and is readable): <path>` | `IllegalStateException` | The file is missing or is not a cascade |
| `fraction must not be negative: <value>` | `IllegalArgumentException` | `useMargin` with a negative value |

### D.3 Argument checks

| Message | Exception | Cause |
|---------|-----------|-------|
| `margin must not be negative: <value>` | `IllegalArgumentException` | A negative or `NaN` margin |
| `<name> must not be null` | `NullPointerException` | `imagePath`, `outputImagePath`, `imageFile`, `outputDirectory`, `classpathResourcePath` or `cascadeFilePathToUse` is `null` |
