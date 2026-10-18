# ai.dl.vision.similarity

*J2OS Platform 2.4*

A shop wants to show "similar products" under a picture. A gallery must refuse a photo that is already in it. A sign-in step has to check whether two faces belong to the same person. In all of these, a program must decide how much one picture resembles another. This is **image similarity**, and for faces it is **face comparison**.

**ai.dl.vision.similarity** (`org.j2os.platform.ai.dl.vision.similarity`) brings it to J2OS. It has one class, `ImageSimilarity`, which turns each picture into a short list of numbers and compares the lists. The same code serves faces and any other kind of picture; only the model changes. What makes it different comes down to three properties:

- **Your model.** The neural network is a file that you supply, for example a face model for faces or a general model for products. The module does not download anything.
- **A dataset folder with a memory.** Pictures are kept in an ordinary folder, and the numbers computed for them are saved next to it, so a search does not recompute them.
- **Made for servers.** One loaded model serves any number of threads, and the module protects itself against being closed in the middle of a job.

Everything in one small example:

```java
try (ImageSimilarity similarity = ImageSimilarity.load("models", "face_feature")) {      // 1. load the model once
    float score = similarity.compare("a.jpg", "b.jpg");                                  // 2. how alike are two pictures?
    similarity.addToDataset("shop", "new-product.jpg");                                  // 3. keep a picture in a folder
    List<SimilarityMatch> matches = similarity.findMatches("query.jpg", "shop");         // 4. find the closest ones
}
```

This guide goes from the first feature (chapter 2) through comparing two pictures (3), a dataset and searching it (4) and its cache (5) to faces (6), failures (7), the web (8) and a complete small workflow (9). The appendices are meant for quick lookups.

## Contents

1. [How ai.dl.vision.similarity works](#1-how-aidlvisionsimilarity-works)
2. [The first feature](#2-the-first-feature)
3. [Comparing two pictures](#3-comparing-two-pictures)
4. [A dataset and searching it](#4-a-dataset-and-searching-it)
5. [The cache and removing](#5-the-cache-and-removing)
6. [Faces](#6-faces)
7. [When it fails](#7-when-it-fails)
8. [For the web](#8-for-the-web)
9. [A realistic scenario](#9-a-realistic-scenario)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Every program here runs as it stands: each has one `main` method, loads the model from a folder named `models` and reads its pictures from a folder named `photos`. The dataset folders they create are under `data`.

---

## 1. How ai.dl.vision.similarity works

Some terms first; later chapters take them for granted:

- **Feature (embedding).** A list of numbers that describes a picture, for example 128 of them. Pictures that look alike get lists that are alike. The model computes it; the numbers have no meaning to a person.
- **Similarity score.** A number from 0 to 1 that says how close two features are. `1.0` means the same, around `0.5` means unrelated, `0.0` means opposite. It is the cosine of the angle between the two lists, moved from the range -1 to 1 into 0 to 1.
- **Model.** The neural network that makes features. Here it is a *TorchScript* file with the extension `.pt`, for example a FaceNet or ArcFace model that has been converted. It expects pictures resized to 160 by 160 pixels, and the module prepares them that way.
- **Dataset.** A folder of pictures that you search in. The module stores copies of the pictures you add, under new names.
- **Query.** The picture you look for. It is never stored.

The classes a program meets:

| Class | Role |
|-------|------|
| `ImageSimilarity` | Loads a model once; extracts features, compares pictures, manages a dataset and searches in it |
| `SimilarityMatch` | One search result: the name of a dataset picture and its score |
| `ImageSimilarityFailedException` | The one checked exception of the module |

The four points to carry forward:

- **The model is yours.** The module has no model of its own. A file `<name>.pt` is put in a folder, and the module is told where (the layout is in chapter 4).
- **Load once, close once.** Loading is expensive. An `ImageSimilarity` is created at start-up, used for every request and closed at shutdown.
- **Only dataset pictures are cached.** Single comparisons and queries are computed fresh every time. Pictures added to a dataset are computed once and remembered.
- **A search looks at every picture.** `findMatches` compares the query with each picture in the folder. This is fine for thousands of pictures and slow for millions.

## 2. The first feature

`ImageSimilarity.load` takes the folder of the model and its name without the extension. `extractFeature` returns the feature of one picture:

```java
import org.j2os.platform.ai.dl.vision.similarity.ImageSimilarity;

public class First {
    public static void main(String[] args) throws Exception {
        try (ImageSimilarity similarity = ImageSimilarity.load("models", "face_feature")) {
            float[] feature = similarity.extractFeature("photos/astronaut.jpg");

            System.out.println("numbers in the feature: " + feature.length);
            System.out.printf("first three: %.3f %.3f %.3f%n", feature[0], feature[1], feature[2]);
        }
    }
}
```

```text
numbers in the feature: 128
first three: -0.622 0.001 0.112
```

- **`load("models", "face_feature")`.** Reads `models/face_feature.pt`. A missing file is reported at once (chapter 7).
- **The feature.** A `float[]`. Its length is decided by the model.
- **Not cached.** `extractFeature` is meant for single query pictures. Every call computes the picture again.
- **`try`-with-resources.** `ImageSimilarity` is `AutoCloseable`, so the model is released at the end of the block.

## 3. Comparing two pictures

`compare` takes two paths and returns the score:

```java
import org.j2os.platform.ai.dl.vision.similarity.ImageSimilarity;

public class Compare {
    public static void main(String[] args) throws Exception {
        try (ImageSimilarity similarity = ImageSimilarity.load("models", "face_feature")) {
            String photo = "photos/astronaut.jpg";

            System.out.printf("same photo:        %.2f%n", similarity.compare(photo, photo));
            System.out.printf("brighter copy:     %.2f%n", similarity.compare(photo, "photos/astronaut-bright.jpg"));
            System.out.printf("smaller copy:      %.2f%n", similarity.compare(photo, "photos/gallery/astronaut-small.jpg"));
            System.out.printf("another picture:   %.2f%n", similarity.compare(photo, "photos/gallery/coffee.jpg"));
            System.out.printf("order does not matter: %b%n",
                    similarity.compare(photo, "photos/gallery/coffee.jpg") == similarity.compare("photos/gallery/coffee.jpg", photo));
        }
    }
}
```

```text
same photo:        1.00
brighter copy:     1.00
smaller copy:      1.00
another picture:   0.55
order does not matter: true
```

- **The same picture** scores `1.00`. So does a copy that differs only a little in brightness or size.
- **Unrelated pictures** score near `0.5`.
- **The order does not matter.** `compare(a, b)` equals `compare(b, a)`.
- **Which value is a match?** It depends on the model. There is no universal limit: take pairs that you know are the same and pairs that you know are different, look at their scores, and choose a threshold between them.
- **No caching.** Both features are computed on every call. For many comparisons against the same picture, use a dataset (chapter 4).

## 4. A dataset and searching it

A dataset is a folder. `addToDataset` copies a picture into it, `addAllToDataset` does it for a whole folder, and `findMatches` searches:

```text
models/
└── face_feature.pt            (a TorchScript model)

shop/                          (the dataset folder: the pictures)
shop.cache/                    (made by the module, next to it: one .feature file per picture)
```

```java
import org.j2os.platform.ai.dl.vision.similarity.ImageSimilarity;
import org.j2os.platform.ai.dl.vision.similarity.SimilarityMatch;

import java.util.List;

public class Dataset {
    public static void main(String[] args) throws Exception {
        try (ImageSimilarity similarity = ImageSimilarity.load("models", "face_feature")) {
            String stored = similarity.addToDataset("data/shop", "photos/gallery/chelsea.jpg");
            System.out.println("stored as: " + stored.replaceAll("[0-9a-f]{8}-[0-9a-f-]{27}", "<id>"));

            List<String> names = similarity.addAllToDataset("data/shop", "photos/gallery");
            System.out.println("added " + names.size() + " more");

            List<SimilarityMatch> matches = similarity.findMatches("photos/astronaut.jpg", "data/shop");
            for (SimilarityMatch match : matches.stream().limit(3).toList()) {
                System.out.printf("%s -> %.2f%n", match.imageName().replaceAll("-[0-9a-f]{8}-[0-9a-f-]{27}", ""), match.score());
            }
            System.out.println("images in the dataset: " + matches.size());
        }
    }
}
```

```text
stored as: chelsea-<id>.jpg
added 6 more
astronaut-small.jpg -> 1.00
camera.jpg -> 0.62
coffee.jpg -> 0.55
images in the dataset: 7
```

- **`addToDataset(folder, picture)`.** The folder is created if needed. The picture is copied under a new name, the original name plus a unique code, and the stored name is returned. Keep it next to your other data about the picture.
- **Safe copying.** The copy first gets a temporary name and is renamed when it is complete, so a search that runs at the same time never sees half a file.
- **`addAllToDataset(folder, sourceFolder)`.** Adds every picture directly inside the source folder; other files, such as the text file in the example, are skipped. The names come back in the order the pictures were processed.
- **All or nothing.** If one picture fails, the ones added by this call are removed again and the dataset is as it was; at most an empty folder remains if the call had created it.
- **Same picture twice.** Adding a picture again stores another copy. The example adds `chelsea.jpg` once by itself and once with the whole folder, and the dataset ends with seven pictures instead of six. Search before adding if that matters (chapter 9).
- **`findMatches(query, folder)`.** Returns every readable picture in the folder, the most similar first. Files that are not pictures are ignored; the decision is made from the content of the file, not from its name.
- **The best ones.** The list is complete. Take the first few with `matches.stream().limit(10).toList()`.
- **Only the folder itself.** Subfolders are not searched.

## 5. The cache and removing

The feature of every dataset picture is saved in a folder next to the dataset, with the same name and `.cache` added:

```java
import org.j2os.platform.ai.dl.vision.similarity.ImageSimilarity;
import org.j2os.platform.ai.dl.vision.similarity.SimilarityMatch;

import java.io.File;

public class Cache {
    public static void main(String[] args) throws Exception {
        try (ImageSimilarity similarity = ImageSimilarity.load("models", "face_feature")) {
            similarity.addAllToDataset("data/shop", "photos/gallery");
            System.out.println("pictures: " + new File("data/shop").list().length);
            System.out.println("cached features in data/shop.cache: " + new File("data/shop.cache").list().length);

            String first = new File("data/shop").list()[0];
            System.out.println("cache file ends with .feature: " + new File("data/shop.cache/" + first + ".feature").exists());

            for (SimilarityMatch match : similarity.findMatches("photos/astronaut.jpg", "data/shop").subList(0, 1)) {
                similarity.removeFromDataset("data/shop", match.imageName());
            }
            System.out.println("after removing the best match: " + new File("data/shop").list().length
                    + " pictures, " + new File("data/shop.cache").list().length + " cached features");
        }
    }
}
```

```text
pictures: 6
cached features in data/shop.cache: 6
cache file ends with .feature: true
after removing the best match: 5 pictures, 5 cached features
```

- **One file per picture.** `data/shop.cache/<picture>.feature`. It is created when the picture is added, and a search reads it instead of running the model.
- **It checks itself.** Each entry records the date and size of the picture and a fingerprint of the model file. If the picture was replaced, or the model changed, the entry is ignored and computed again.
- **Damaged entries.** An unreadable entry is also computed again. A failure to write the cache is ignored; the search still works, only slower.
- **`removeFromDataset(folder, storedName)`.** Deletes the picture and its cache file. The name must be a plain file name, as returned by `addToDataset`; a name with `/`, `\` or `..` is refused.
- **Removing something that is not there** is not an error.
- **The query is not cached.** Only dataset pictures are.

## 6. Faces

For faces, compare the faces and not the photographs. A photograph contains a background, clothes and light that have nothing to do with who is in it. Cut the face out first with the face module:

```java
FaceCrop.crop("full-body-photo.jpg", "face-only.jpg");                  // cut the face out first
float score = similarity.compare("face-only.jpg", "other-face.jpg");     // then compare faces, not backgrounds
```

```java
import org.j2os.platform.ai.dl.vision.face.FaceCrop;
import org.j2os.platform.ai.dl.vision.similarity.ImageSimilarity;

public class Faces {
    public static void main(String[] args) throws Exception {
        FaceCrop.crop("photos/portrait.jpg", "out/portrait-face.png");
        FaceCrop.crop("photos/team.jpg", "out/team-face.png");

        try (ImageSimilarity similarity = ImageSimilarity.load("models", "face_feature")) {
            System.out.printf("whole photos: %.2f%n", similarity.compare("photos/portrait.jpg", "photos/team.jpg"));
            System.out.printf("only faces:   %.2f%n", similarity.compare("out/portrait-face.png", "out/team-face.png"));
        }
    }
}
```

```text
whole photos: 0.52
only faces:   0.98
```

- **Whole photos** carry everything around the face, and their score is low.
- **Faces only** give a high score. The faces here come from two different photographs: one from a portrait and one, the largest face, from a group photo.
- **The same model for both.** A face model is trained on faces, and gives its best results on cropped faces.
- **Several faces in one photo.** Use `cropAll` and compare each face separately.
- **Decide the threshold with real pairs.** For sign-in and similar checks, a wrong "same" is worse than a wrong "different". Choose a high threshold and test it on your own photographs.

## 7. When it fails

The operations fail when a file or a folder is missing, when the instance is closed, and when the model cannot do its job:

```java
import org.j2os.platform.ai.dl.vision.similarity.ImageSimilarity;

import java.io.File;

public class Failures {
    public static void main(String[] args) throws Exception {
        String here = new File(".").getCanonicalPath();

        try { ImageSimilarity.load("models", "other_model"); }
        catch (Exception e) { System.out.println("no model file -> " + e.getClass().getSimpleName() + ": " + e.getMessage().replace(here, ".")); }

        ImageSimilarity similarity = ImageSimilarity.load("models", "face_feature");

        try { similarity.extractFeature("photos/ghost.jpg"); }
        catch (Exception e) { System.out.println("no picture -> " + e.getClass().getSimpleName() + ": " + e.getMessage().replace(here, ".")); }

        try { similarity.findMatches("photos/astronaut.jpg", "data/none"); }
        catch (Exception e) { System.out.println("no folder -> " + e.getClass().getSimpleName() + ": " + e.getMessage().replace(here, ".")); }

        try { similarity.removeFromDataset("data/shop", "../secret.jpg"); }
        catch (Exception e) { System.out.println("odd name -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        System.out.println("matches in a folder with only a text file: " + similarity.findMatches("photos/astronaut.jpg", "photos/text-only").size());

        similarity.close();
        try { similarity.extractFeature("photos/astronaut.jpg"); }
        catch (Exception e) { System.out.println("after close -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
    }
}
```

```text
no model file -> ImageSimilarityFailedException: java.io.FileNotFoundException: Model file does not exist: ./models/other_model.pt
no picture -> ImageSimilarityFailedException: java.io.FileNotFoundException: Image file does not exist: ./photos/ghost.jpg
no folder -> ImageSimilarityFailedException: java.io.FileNotFoundException: Dataset folder does not exist: ./data/none
odd name -> ImageSimilarityFailedException: storedFileName must be a plain file name, not a path: ../secret.jpg
matches in a folder with only a text file: 0
after close -> ImageSimilarityFailedException: This ImageSimilarity instance has been closed
```

- **No model file.** `load` names the file it looked for.
- **No picture.** The message names the picture.
- **No folder.** The message names the dataset or source folder.
- **A name that is not plain.** `removeFromDataset` refuses `../secret.jpg`, so a stored name that comes from outside can never reach other folders.
- **A folder without pictures** is not an error: the result is an empty list.
- **After `close`.** New extractions are refused with a clear message.

The arguments are checked before any work starts:

```java
import org.j2os.platform.ai.dl.vision.similarity.ImageSimilarity;

public class Checks {
    public static void main(String[] args) throws Exception {
        try { ImageSimilarity.load(null, "face_feature"); }
        catch (Exception e) { System.out.println("null directory -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try (ImageSimilarity similarity = ImageSimilarity.load("models", "face_feature")) {
            try { similarity.compare("photos/astronaut.jpg", null); }
            catch (Exception e) { System.out.println("null second picture -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

            try { similarity.findMatches(null, "data/shop"); }
            catch (Exception e) { System.out.println("null query -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

            try { similarity.addToDataset("data/shop", null); }
            catch (Exception e) { System.out.println("null picture -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
        }
    }
}
```

```text
null directory -> NullPointerException: modelDirectory must not be null
null second picture -> NullPointerException: imagePath must not be null
null query -> NullPointerException: queryImagePath must not be null
null picture -> NullPointerException: imagePath must not be null
```

- **`null` values.** A `NullPointerException` that names the argument.
- **Loading failures.** If the model file exists but cannot be loaded, `load` ends with an `ImageSimilarityFailedException` whose cause is the error from the model engine.

## 8. For the web

The application loads the model once, as a bean that is closed when the application stops:

```java
@Bean(destroyMethod = "close")
public ImageSimilarity imageSimilarity() throws ImageSimilarityFailedException {
    return ImageSimilarity.load("/var/lib/app/models", "face_feature");   // one model for the whole application
}
```

A page sends a picture and shows the five most similar pictures of the shop. A picture that cannot be used becomes a `422`:

```java
@PostMapping("/api/similar")
public List<Map<String, Object>> similar(@RequestParam MultipartFile photo) throws IOException {
    File upload = File.createTempFile("query-", ".img");
    try {
        photo.transferTo(upload);
        return imageSimilarity.findMatches(upload.getPath(), "/var/lib/app/shop").stream()
                .limit(5)
                .map(m -> Map.<String, Object>of("image", m.imageName(), "score", Math.round(m.score() * 100)))
                .toList();
    } catch (ImageSimilarityFailedException e) {
        throw new ResponseStatusException(HttpStatus.UNPROCESSABLE_ENTITY, "the picture cannot be used");
    } finally {
        upload.delete();
    }
}
```

The page shows the result:

```js
const form = new FormData()
form.append('photo', input.files[0])
const response = await fetch('/api/similar', { method: 'POST', body: form })
if (response.ok) {
  matches.value = await response.json()   // [{ image: '...', score: 87 }, ...]
}
```

- **One instance.** The bean is shared by all requests. The module is built for that.
- **A temporary file.** The upload is saved briefly and always removed. The query itself is never stored.
- **Percentages.** Scores are turned into whole percentages for the page.
- **The dataset is changed elsewhere.** New products are added with `addToDataset` when they are created, not at search time.
- **A limit on the upload.** Set a maximum size in the web layer. Log the exception: a failure that repeats for every request points to the model or the folder, not to the picture.

## 9. A realistic scenario

A catalog accepts pictures from sellers and refuses those that are already in it. Each upload is searched for in the catalog; a near-copy is rejected, anything else is added:

```java
import org.j2os.platform.ai.dl.vision.similarity.ImageSimilarity;
import org.j2os.platform.ai.dl.vision.similarity.SimilarityMatch;

import java.io.File;
import java.util.List;

public class Workflow {
    public static void main(String[] args) throws Exception {
        try (ImageSimilarity similarity = ImageSimilarity.load("models", "face_feature")) {
            similarity.addAllToDataset("data/catalog", "photos/gallery");

            String[] uploads = {"photos/astronaut-bright.jpg", "photos/horse.jpg", "photos/portrait.jpg"};
            for (String upload : uploads) {
                List<SimilarityMatch> matches = similarity.findMatches(upload, "data/catalog");
                SimilarityMatch best = matches.get(0);
                String name = new File(upload).getName();

                if (best.score() >= 0.9f) {
                    System.out.printf("%s: duplicate of %s (%.2f), not added%n", name,
                            best.imageName().replaceAll("-[0-9a-f]{8}-[0-9a-f-]{27}", ""), best.score());
                } else {
                    similarity.addToDataset("data/catalog", upload);
                    System.out.printf("%s: new, closest was %.2f, added%n", name, best.score());
                }
            }
            System.out.println("catalog now has " + new File("data/catalog").list().length + " pictures");
        }
    }
}
```

```text
astronaut-bright.jpg: duplicate of astronaut-small.jpg (1.00), not added
horse.jpg: new, closest was 0.73, added
portrait.jpg: duplicate of astronaut-small.jpg (1.00), not added
catalog now has 7 pictures
```

- **Search before adding.** The best score of the search decides: `0.9` or more is a duplicate.
- **The threshold.** `0.9` suits this model and these pictures. Choose yours from real examples (chapter 3).
- **A copy with a changed brightness** is recognized as a duplicate of the smaller copy that is in the catalog.
- **A new picture** is added, and the next search will find it, because its feature was computed when it was added.
- **The result.** The catalog has the pictures that were already there plus the new one.

## Appendix A. Quick reference

### A.1 Loading

| Call | Meaning |
|------|---------|
| `ImageSimilarity.load(modelDirectory, modelName)` | Loads `<modelDirectory>/<modelName>.pt` |
| `close()` | Waits for running work, then releases the model; repeated calls are harmless |

### A.2 Features and comparison

| Call | Meaning |
|------|---------|
| `extractFeature(imagePath)` | The feature of one picture; not cached |
| `compare(imagePath1, imagePath2)` | Score from 0 to 1; not cached |

### A.3 Dataset

| Call | Meaning |
|------|---------|
| `addToDataset(datasetFolder, imagePath)` | Copies the picture under a unique name, computes its feature, returns the stored name |
| `addAllToDataset(datasetFolder, sourceFolder)` | Adds every picture of a folder; all or nothing; returns the stored names |
| `removeFromDataset(datasetFolder, storedFileName)` | Deletes the picture and its cached feature |
| `findMatches(queryImagePath, datasetFolder)` | All pictures of the folder, most similar first |

### A.4 SimilarityMatch

| Member | Meaning |
|--------|---------|
| `imageName()` | The file name of the picture in the dataset folder |
| `score()` | The similarity, 0 to 1 |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| Model file missing | `ImageSimilarityFailedException` caused by a `FileNotFoundException` that names the file |
| Model cannot be loaded | `ImageSimilarityFailedException` caused by the error of the model engine |
| Picture missing | `ImageSimilarityFailedException` caused by a `FileNotFoundException` that names the picture |
| Dataset or source folder missing | `ImageSimilarityFailedException` caused by a `FileNotFoundException` that names the folder |
| Stored name contains a separator or `..` | `ImageSimilarityFailedException`: "storedFileName must be a plain file name, not a path" |
| Picture cannot be copied or its folder cannot be created | `ImageSimilarityFailedException` caused by an `IOException`; no partial file is left |
| Feature cannot be extracted | `ImageSimilarityFailedException` caused by the underlying error; in `addToDataset` the copy and its cache entry are removed |
| One picture of `addAllToDataset` fails | The exception of that picture; everything added by the call is removed |
| Instance already closed | `ImageSimilarityFailedException`: "This ImageSimilarity instance has been closed" |
| Folder without pictures | An empty list, no exception |
| `null` argument | `NullPointerException` that names the argument |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Loading per request.** Loading takes seconds and a lot of memory. Load once and share.
- **Forgetting `close`.** The model holds native memory. Close it at shutdown, or use `try`-with-resources.
- **A fixed threshold copied from elsewhere.** Scores depend on the model. Choose the threshold from your own pairs.
- **Comparing whole photos for faces.** Backgrounds and light dominate the score. Crop the faces first.
- **Adding the same picture twice.** Each add stores a new copy. Search first.
- **Expecting a sorted limit.** `findMatches` returns everything. Cut the list yourself.
- **A huge dataset.** Every search looks at each picture. Beyond tens of thousands, keep the features in an index of your own.
- **Storing paths instead of stored names.** Keep the name that `addToDataset` returns; the original name is not kept.
- **A read-only parent folder.** The cache sits next to the dataset folder. If it cannot be written, searches still work but are slower.
- **Replacing the model file.** Every cached feature of the old model becomes invalid and is recomputed on the next search, so the first search is slow.

### C.2 Behavior worth knowing

- **A predictor per extraction.** Only the model is shared, so threads do not disturb each other.
- **Closing waits.** `close` waits until running extractions finish; extractions that start afterwards fail.
- **The cache is next to the folder.** `data/shop` has `data/shop.cache`.
- **Only readable pictures count.** The content decides, not the extension; unfinished copies are never listed.
- **Pictures removed during a search** are skipped quietly.
- **The same score both ways.** The score of a pair does not depend on the order.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Model | A TorchScript `.pt` file that you supply; input is 160 by 160 pixels |
| Score | 0 to 1; `1.0` identical, about `0.5` unrelated |
| Search | A linear scan of the dataset folder |
| Dataset | One folder; subfolders are not searched |
| Input | Any picture format that Java can read |
| Concurrency | One instance serves any number of threads |

### C.4 Tuning

- **Pick the model for the job.** A face model for faces; a general model for products or scenes.
- **Choose thresholds from data.** Collect pairs of the same and of different things and look at the scores.
- **Crop first for faces.** The face module makes this one call per picture.
- **Fill the cache early.** Add pictures when they arrive, so searches never compute features.
- **Warm up at start-up.** The first extraction after loading is the slowest; run one small picture when the application starts.
- **Keep the cache folder.** Put the dataset on a persistent disk so a restart does not recompute anything.

## Appendix D. Message reference

### D.1 Loading and extraction

| Message | Exception | Cause |
|---------|-----------|-------|
| `java.io.FileNotFoundException: Model file does not exist: <path>` | `ImageSimilarityFailedException` | `<name>.pt` is not in the model folder |
| `java.io.FileNotFoundException: Image file does not exist: <path>` | `ImageSimilarityFailedException` | The picture is missing |
| `java.io.FileNotFoundException: Dataset folder does not exist: <path>` | `ImageSimilarityFailedException` | The folder given to `findMatches` is missing |
| `java.io.FileNotFoundException: Source folder does not exist: <path>` | `ImageSimilarityFailedException` | The folder given to `addAllToDataset` is missing |
| `This ImageSimilarity instance has been closed` | `ImageSimilarityFailedException` | An extraction was started after `close` |
| Text of the cause | `ImageSimilarityFailedException` | The model could not be loaded, the picture could not be read or copied, or the model failed |

### D.2 Dataset

| Message | Exception | Cause |
|---------|-----------|-------|
| `storedFileName must be a plain file name, not a path: <name>` | `ImageSimilarityFailedException` | The name has a separator or `..` |

### D.3 Argument checks

| Message | Exception | Cause |
|---------|-----------|-------|
| `<name> must not be null` | `NullPointerException` | `modelDirectory`, `modelName`, `imagePath`, `queryImagePath`, `datasetFolder`, `sourceFolder` or `storedFileName` is `null` |
