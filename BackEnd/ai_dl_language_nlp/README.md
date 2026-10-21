# ai.dl.language.nlp

*J2OS Platform 2.4*

A program that handles human text soon needs more than `String.split`. Where does one word end and the next begin? Which words are nouns? Is "jumped" the same word as "jump"? What is an article about, which sentence answers a question, and is the text even in the expected language? These are the everyday questions of **natural language processing** (NLP).

**ai.dl.language.nlp** (`org.j2os.platform.ai.dl.language.nlp`) answers them in J2OS. It wraps Apache OpenNLP behind small classes that take a plain `String` and return plain records. What makes it different comes down to three properties:

- **Text in, records out.** No OpenNLP class appears in a program that uses the library. Answers are `String[]`, `List<PosTaggedToken>`, `List<Keyword>`, `NamedEntity` and similar.
- **Built for Persian, open to any language.** Tokenization keeps Persian compound words whole, tags come with Persian descriptions, and the models are OpenNLP files, so any language that has models works.
- **Safe to share.** An instance holds only an immutable model. One loaded object can serve every thread of an application.

The whole thing at a glance:

```java
String[] words = Tokenization.tokenize("The cats jumped over the walls.");   // 1. no model needed

Lemmatization lemmas = Lemmatization.load("models/en-lemmas.bin", "models/ud-pos.bin");
List<LemmatizedToken> tokens = lemmas.lemmatize("The cats jumped.");        // 2. words, tags, lemmas

List<Keyword> keywords = KeywordExtraction.extract(article, 5, lemmas);     // 3. what the text is about
String[] summary = Summarization.summarize(article, 2, lemmas);             //    its main sentences
```

This guide goes from small pieces to larger ones: tokens and sentences (chapter 2), parts of speech (3), lemmas (4), keywords (5), summaries (6), searching a text (7), detecting the language (8), named entities (9), training an entity model (10) and a complete small workflow (11). The appendices are meant for quick lookups.

## Contents

1. [How ai.dl.language.nlp works](#1-how-aidllanguagenlp-works)
2. [Tokens and sentences](#2-tokens-and-sentences)
3. [Parts of speech](#3-parts-of-speech)
4. [Lemmas](#4-lemmas)
5. [Keywords](#5-keywords)
6. [Summaries](#6-summaries)
7. [Searching a text](#7-searching-a-text)
8. [Detecting the language](#8-detecting-the-language)
9. [Named entities](#9-named-entities)
10. [Training an entity model](#10-training-an-entity-model)
11. [Everything on one page](#11-everything-on-one-page)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Every program here runs as it stands: each has one `main` method and loads its models from a folder named `models`. The texts are invented and short so that the output is easy to follow.

---

## 1. How ai.dl.language.nlp works

Here is the vocabulary in one place:

- **Token.** One word or one punctuation mark of a text.
- **Part of speech (POS).** The grammatical role of a token: noun, verb, adjective and so on. The tags follow *Universal Dependencies*: `NOUN`, `PROPN` (a name), `VERB`, `ADJ`, `ADV`, `DET`, `ADP`, `PUNCT` and the rest.
- **Lemma.** The dictionary form of a word: `cats` has the lemma `cat`, `jumped` has `jump`.
- **Entity.** A name found in a text, with a type such as `person`, `org` or `city`.
- **Model.** A file made by training that carries what a task needs to know about one language. Loading it is the expensive part; using it is cheap.

The classes at a glance:

| Class | Role |
|-------|------|
| `Tokenization` | Splits a text into tokens. No model |
| `SentenceDetection` | Splits a paragraph into sentences with a model |
| `PartOfSpeechTagging` | Tags every token. Returns `PosTaggedToken` |
| `Lemmatization` | Tags and finds the lemma of every token. Returns `LemmatizedToken` |
| `KeywordExtraction` | The most important words of a text. Returns `Keyword` |
| `Summarization` | The most important sentences of a text |
| `TextSearch` | The sentence that best answers a question. Returns `SearchResult` |
| `LanguageDetection` | The language of a text, as a code |
| `NamedEntityRecognition` | Finds names. Returns `NamedEntity` |
| `NamedEntityRecognitionTraining` | Trains an entity model from a file |
| `NlpFailedException` | The one checked exception of the module |

Remember three things:

- **Models are files, loaded once.** Each class has a static `load` that reads the model file; keep the result and reuse it. OpenNLP's ready-made models are listed on its website; they come for many languages, Persian included.
- **Keywords, summaries and searches need Universal Dependencies tags.** They give weight to `PROPN`, `NOUN`, `VERB` and `ADJ` and ignore the rest, so the part-of-speech model passed to `Lemmatization` must use those tag names.
- **Instances are immutable.** Every call creates its own OpenNLP working objects, so one instance can be used by many threads at once. Training is the only operation that writes a file.

## 2. Tokens and sentences

`Tokenization.tokenize` needs no model. `SentenceDetection` needs a sentence model:

```java
import org.j2os.platform.ai.dl.language.nlp.core.Tokenization;
import org.j2os.platform.ai.dl.language.nlp.core.SentenceDetection;

public class Tokens {
    public static void main(String[] args) throws Exception {
        // tokens: words and punctuation, no model needed
        String[] tokens = Tokenization.tokenize("Hello! I am Sara, e-mail me at 5pm.");
        System.out.println(String.join(" | ", tokens));

        // a zero-width non-joiner between letters keeps one word
        String[] fa = Tokenization.tokenize("من می‌روم");
        System.out.println(fa.length + " tokens: " + String.join(" | ", fa));

        // sentences need a model
        SentenceDetection detection = SentenceDetection.load("models/en-sent.bin");
        for (String s : detection.detect("Acme opened an office. It is in Berlin! Is it big? Yes, Mr. Smith says so.")) {
            System.out.println("sentence: " + s);
        }
    }
}
```

```text
Hello | ! | I | am | Sara | , | e | - | mail | me | at | 5 | pm | .
2 tokens: من | می‌روم
sentence: Acme opened an office.
sentence: It is in Berlin!
sentence: Is it big?
sentence: Yes, Mr. Smith says so.
```

- **Tokens.** Words and punctuation marks come out one by one. A hyphenated word and a time like `5pm` are split where the kind of character changes: `e | - | mail`, `5 | pm`.
- **Persian.** The zero-width non-joiner (the half-space inside `می‌روم`) stays inside the word, so the Persian sentence has two tokens, not three. Only a half-space that touches letters on both sides is merged; one that stands beside a space does not join two words.
- **Sentences.** The model decides; `Mr.` does not end a sentence in an English model. The sentences come back as they were written.
- **A null text.** Every method rejects `null` with a `NullPointerException` that names the argument.

## 3. Parts of speech

`PartOfSpeechTagging` tokenizes and tags in one call:

```java
import org.j2os.platform.ai.dl.language.nlp.core.PartOfSpeechTagging;
import org.j2os.platform.ai.dl.language.nlp.core.PosTaggedToken;

public class Tags {
    public static void main(String[] args) throws Exception {
        PartOfSpeechTagging tagging = PartOfSpeechTagging.load("models/ud-pos.bin");
        for (PosTaggedToken t : tagging.tag("The team plans another office in Paris.")) {
            System.out.println(t.token() + "  " + t.tag() + "  " + t.description());
        }

        // a model that uses other tag names (Penn Treebank) is accepted,
        // but the descriptions fall back to the default text
        PartOfSpeechTagging penn = PartOfSpeechTagging.load("models/en-pos-maxent.bin");
        PosTaggedToken first = penn.tag("The team plans.").get(0);
        System.out.println(first.token() + "  " + first.tag() + "  " + first.description());
    }
}
```

```text
The  DET  معرفه/نکره
team  NOUN  اسم
plans  VERB  فعل
another  DET  معرفه/نکره
office  NOUN  اسم
in  ADP  حرف اضافه
Paris  PROPN  اسم خاص
.  PUNCT  علامت نگارشی
The  DT  نامشخص
```

- **`PosTaggedToken`.** The token as written, its tag, and a short *Persian* description of the tag (`اسم` for a noun, `فعل` for a verb).
- **The tags.** They come from the model. A model trained with Universal Dependencies tags gives `NOUN`, `VERB` and so on.
- **Other tag sets.** A model with different tag names, such as the Penn Treebank `NN` and `VBZ`, loads without any complaint, but every description becomes `نامشخص` ("unknown"), and the keyword, summary and search classes find no content words.

## 4. Lemmas

`Lemmatization` needs two models, a lemmatizer and a part-of-speech model, because the lemma of a word depends on its role:

```java
import org.j2os.platform.ai.dl.language.nlp.core.Lemmatization;
import org.j2os.platform.ai.dl.language.nlp.core.LemmatizedToken;

public class Lemmas {
    public static void main(String[] args) throws Exception {
        Lemmatization lemmatization = Lemmatization.load("models/en-lemmas.bin", "models/ud-pos.bin");
        for (LemmatizedToken t : lemmatization.lemmatize("The cats jumped over the walls.")) {
            System.out.println(t.token() + "  " + t.tag() + "  -> " + t.lemma());
        }
    }
}
```

```text
The  DET  -> _
cats  NOUN  -> cat
jumped  VERB  -> jump
over  ADP  -> _
the  DET  -> _
walls  NOUN  -> wall
.  PUNCT  -> _
```

- **`LemmatizedToken`.** The token, its tag, the Persian description of the tag and the lemma.
- **Not every word has a lemma.** A model may answer `_` for a word it cannot reduce, as it does for `The`. The extraction classes treat `_` and an empty lemma as "use the token itself".
- **Same order.** There is one entry per token, in the order of the text.

## 5. Keywords

`KeywordExtraction.extract` returns the words that carry the most meaning:

```java
import org.j2os.platform.ai.dl.language.nlp.core.Lemmatization;
import org.j2os.platform.ai.dl.language.nlp.extraction.Keyword;
import org.j2os.platform.ai.dl.language.nlp.extraction.KeywordExtraction;

public class Keywords {
    public static void main(String[] args) throws Exception {
        Lemmatization lemmatization = Lemmatization.load("models/en-lemmas.bin", "models/ud-pos.bin");
        String text = "Acme Corp opened a new office in Berlin. The office will host the engineering team. "
                + "Engineers build fast payment systems for banks. Customers praise the new payment system.";
        for (Keyword k : KeywordExtraction.extract(text, 5, lemmatization)) {
            System.out.println(k.word() + "  " + k.score());
        }
    }
}
```

```text
office  10.0
payment  10.0
system  10.0
Acme  6.0
Corp  6.0
```

- **How a word earns a score.** Each occurrence of a lemma counts by its role: a name (`PROPN`) 3.0, a noun 2.5, a verb 1.5, an adjective 1.0. All other roles are ignored.
- **The scale.** The best word gets exactly `10.0`; the others are scaled against it and rounded to one decimal. Here `office`, `payment` and `system` appear twice as nouns, and `Acme` and `Corp` once each as names.
- **Ties.** Words with the same score keep the order of their first appearance.
- **The limit.** The second argument is the most words to return. It must be positive, otherwise `IllegalArgumentException`.
- **Case.** A lemma that the model does not know falls back to the token as written, so `Office` and `office` count as two words unless the model lowercases.

## 6. Summaries

`Summarization.summarize` picks the sentences that contain the most important words of the whole text:

```java
import org.j2os.platform.ai.dl.language.nlp.core.Lemmatization;
import org.j2os.platform.ai.dl.language.nlp.extraction.Summarization;

public class Summary {
    public static void main(String[] args) throws Exception {
        Lemmatization lemmatization = Lemmatization.load("models/en-lemmas.bin", "models/ud-pos.bin");
        String text = "Acme Corp opened a new office in Berlin. The office will host the engineering team. "
                + "Engineers build fast payment systems for banks. Berlin offers great talent and cheap rent. "
                + "Customers praise the new payment system. The team plans another office in Paris.";
        for (String s : Summarization.summarize(text, 2, lemmatization)) {
            System.out.println("- " + s);
        }
    }
}
```

```text
- Acme Corp opened a new office in Berlin.
- Customers praise the new payment system.
```

- **How it works.** The words of the text are weighed as for keywords; each sentence gets the average weight of its tokens; the best sentences are returned in their original order.
- **A short text.** If the text has no more sentences than asked for, all of them are returned unchanged.
- **Its own splitter.** Sentences are cut after `.`, `!`, `?` or the Persian `؟` followed by white space. No sentence model is used, so an abbreviation such as `Mr. Smith` ends a sentence here.
- **The limit.** It must be positive, otherwise `IllegalArgumentException`.

## 7. Searching a text

`TextSearch.search` finds the sentence of a text that shares the most important words with a question:

```java
import org.j2os.platform.ai.dl.language.nlp.core.Lemmatization;
import org.j2os.platform.ai.dl.language.nlp.extraction.SearchResult;
import org.j2os.platform.ai.dl.language.nlp.extraction.TextSearch;

public class Search {
    public static void main(String[] args) throws Exception {
        Lemmatization lemmatization = Lemmatization.load("models/en-lemmas.bin", "models/ud-pos.bin");
        String text = "Acme Corp opened a new office in Berlin. Engineers build fast payment systems for banks. "
                + "The team plans another office in Paris.";
        for (String q : new String[]{"Where does the team plan an office?", "Who builds the payment system?", "What is the weather?"}) {
            SearchResult r = TextSearch.search(text, q, lemmatization);
            System.out.println(q + "\n   " + r.score() + "  " + r.result());
        }
    }
}
```

```text
Where does the team plan an office?
   6.0  The team plans another office in Paris.
Who builds the payment system?
   7.5  Engineers build fast payment systems for banks.
What is the weather?
   0.0  Acme Corp opened a new office in Berlin.
```

- **The score.** The share of the question's content words (nouns, names, verbs, adjectives) that appear in the sentence, written on a scale of 0 to 10.
- **Only content words count.** Function words such as `the` or `an` are ignored. If a question has no content word at all, all of its words are used.
- **No match.** The last question shares nothing with the text. The answer is the *first* sentence with the score `0.0`. Check the score before trusting the sentence.
- **Ties.** The earliest of equally good sentences wins. An empty text gives an empty result with the score `0.0`.

## 8. Detecting the language

`LanguageDetection` returns the code of the most likely language:

```java
import org.j2os.platform.ai.dl.language.nlp.extraction.LanguageDetection;

public class Language {
    public static void main(String[] args) throws Exception {
        LanguageDetection detection = LanguageDetection.load("models/langdetect.bin");
        System.out.println(detection.detect("The weather is great in the garden and the team is happy."));
        System.out.println(detection.detect("Das ist nicht der Garten und die Katze ist mit dem Hund."));
        System.out.println(detection.detect("این یک متن است که در آن از واژه‌ها استفاده شده و با دقت نوشته شده است"));
    }
}
```

```text
eng
deu
pes
```

- **A code.** The answer is the language code of the model, such as `eng`. The model always names its best guess, and the class does not report how sure it is.
- **Use it as a gate.** Detect the language first, and send the text to the models of that language, or skip it.
- **Short texts.** A few words are not enough to be sure; the guess gets better with length.

## 9. Named entities

`NamedEntityRecognition` finds names and their types. The model decides which types exist:

```java
import org.j2os.platform.ai.dl.language.nlp.ner.NamedEntity;
import org.j2os.platform.ai.dl.language.nlp.ner.NamedEntityRecognition;

public class Entities {
    public static void main(String[] args) throws Exception {
        NamedEntityRecognition ner = NamedEntityRecognition.load("models/en-ner-person.bin");
        for (NamedEntity e : ner.find("Yesterday John Smith met Mary Johnson in Berlin.")) {
            System.out.println(e.text() + " -> " + e.type());
        }
    }
}
```

```text
John Smith -> person
Mary Johnson -> person
```

- **`NamedEntity`.** The matched text and its type.
- **Tokens joined by a space.** A name of several tokens comes back with single spaces between them, whatever the text had.
- **Only what the model knows.** This model finds people. `Berlin` is a city, but the model has no such type, so it is not reported.
- **Order.** Entities come in the order of the text; the result may be empty.

## 10. Training an entity model

A model for your own names is trained from a file in which names are marked inline:

```text
<START:org> Acme Corp <END> opened a new office in <START:city> Berlin <END> .
<START:person> Sara <END> works for <START:org> Acme Corp <END> in <START:city> Berlin <END> .
Rent is cheap in <START:city> Madrid <END> .
```

One line is one sample, tokens are separated by spaces, and a name is wrapped in `<START:type>` and `<END>`. A file that starts with a byte-order mark is accepted.

```java
import java.io.PrintStream;
import org.j2os.platform.ai.dl.language.nlp.ner.*;

public class Training {
    public static void main(String[] args) throws Exception {
        PrintStream console = System.out;
        System.setOut(new PrintStream(java.io.OutputStream.nullOutputStream())); // OpenNLP prints its progress
        try {
            NamedEntityRecognitionTraining.train("models/company.train", "models/company.bin", "en");
        } finally {
            System.setOut(console);
        }

        NamedEntityRecognition ner = NamedEntityRecognition.load("models/company.bin");
        for (NamedEntity e : ner.find("Sara joined Globex in Madrid.")) {
            System.out.println(e.text() + " -> " + e.type());
        }
        for (NamedEntity e : ner.find("Omid visited Initech in Paris and Acme Corp in Rome.")) {
            System.out.println(e.text() + " -> " + e.type());
        }
    }
}
```

```text
Sara -> person
Globex -> org
Madrid -> city
Omid -> person
Initech -> org
Paris -> org
Acme Corp -> org
Rome -> city
```

- **`train`.** Reads the file, trains and writes the model. The third argument is the language code stored in the model.
- **Safe writing.** The model is written to a temporary file next to the target and then moved into place, so a reader never sees a half-written model. Missing folders are created.
- **Progress.** OpenNLP prints its training progress on the standard output; the program above switches it off for a clean output.
- **The result.** `Sara`, `Globex`, `Madrid`, `Omid`, `Initech`, `Acme Corp` and `Rome` are right, but `Paris` is labeled as an organization. The file has sixteen lines. A model learns only what its examples show; give every type many varied examples, and check the model on sentences that are not in the file.
- **Tuning.** The overload with two more arguments sets the number of passes (default 150) and the cutoff, the number of times a feature must occur to be kept (default 1):

```java
// 300 passes, and drop features seen fewer than 2 times
NamedEntityRecognitionTraining.train("data/train.txt", "models/ner.bin", "en", 300, 2);
```

## 11. Everything on one page

A small service loads the models once and then handles many texts at the same time. It checks the language, then extracts keywords, the main sentence and the names:

```java
import java.util.*;
import java.util.concurrent.*;
import org.j2os.platform.ai.dl.language.nlp.core.Lemmatization;
import org.j2os.platform.ai.dl.language.nlp.extraction.*;
import org.j2os.platform.ai.dl.language.nlp.ner.NamedEntityRecognition;

public class Workflow {
    // models are loaded once and shared by every thread
    static Lemmatization lemmas;
    static LanguageDetection language;
    static NamedEntityRecognition ner;

    static String digest(String text) {
        if (!language.detect(text).equals("eng")) return "skipped (not English)";
        StringBuilder sb = new StringBuilder();
        KeywordExtraction.extract(text, 3, lemmas).forEach(k -> sb.append(k.word()).append(' '));
        sb.append("| ").append(Summarization.summarize(text, 1, lemmas)[0]);
        ner.find(text).forEach(e -> sb.append(" | ").append(e.text()).append(':').append(e.type()));
        return sb.toString();
    }

    public static void main(String[] args) throws Exception {
        lemmas = Lemmatization.load("models/en-lemmas.bin", "models/ud-pos.bin");
        language = LanguageDetection.load("models/langdetect.bin");
        ner = NamedEntityRecognition.load("models/en-ner-person.bin");

        List<String> articles = List.of(
            "John Smith opened a new office in Berlin. The office will host the engineering team. The team is happy with the plan.",
            "Das ist nicht der Garten und die Katze ist mit dem Hund.",
            "Engineers build payment systems for banks. Mary Johnson praises the new system and the team.",
            "The weather is great and the garden is small but cheap.");

        ExecutorService pool = Executors.newFixedThreadPool(4);
        List<Future<String>> results = new ArrayList<>();
        for (String a : articles) results.add(pool.submit(() -> digest(a)));
        for (int i = 0; i < results.size(); i++) System.out.println((i + 1) + ") " + results.get(i).get());
        pool.shutdown();
    }
}
```

```text
1) office team Berlin | The office will host the engineering team. | John Smith:person
2) skipped (not English)
3) system Mary Johnson | Engineers build payment systems for banks. | Mary Johnson:person
4) weather garden great | The weather is great and the garden is small but cheap.
```

- **Load once.** Three `load` calls at the start, three shared objects afterwards.
- **Four threads, one set of models.** No lock is needed; each call builds its own working objects.
- **The gate.** The German text is skipped before any expensive work.
- **Names and keywords differ.** The keywords are content words from the part-of-speech model; the names come from the entity model, so `Berlin` is a keyword in the first text but not an entity.
- **Reuse the answers.** Lemmatizing is the slow part of keywords, summaries and searches, and it is repeated for each call. Keep the result for texts that are asked about again:

```java
// the expensive part is lemmatizing: do it once per text, not once per question
List<Keyword> keywords = KeywordExtraction.extract(text, 10, lemmas);
String[] summary = Summarization.summarize(text, 3, lemmas);
cache.put(articleId, new Digest(keywords, summary));
```

## Appendix A. Quick reference

### A.1 Core

| Call | Returns |
|------|---------|
| `Tokenization.tokenize(text)` | `String[]` |
| `SentenceDetection.load(path)` then `detect(paragraph)` | `String[]` |
| `PartOfSpeechTagging.load(path)` then `tag(text)` | `List<PosTaggedToken>` (`token`, `tag`, `description`) |
| `Lemmatization.load(lemmatizerPath, posPath)` then `lemmatize(text)` | `List<LemmatizedToken>` (`token`, `tag`, `description`, `lemma`) |

### A.2 Extraction

| Call | Returns |
|------|---------|
| `KeywordExtraction.extract(text, maxCount, lemmatization)` | `List<Keyword>` (`word`, `score`), best first |
| `Summarization.summarize(text, maxSentences, lemmatization)` | `String[]`, in text order |
| `TextSearch.search(text, question, lemmatization)` | `SearchResult` (`result`, `score`) |
| `LanguageDetection.load(path)` then `detect(text)` | the language code |

### A.3 Entities

| Call | Returns |
|------|---------|
| `NamedEntityRecognition.load(path)` then `find(text)` | `List<NamedEntity>` (`text`, `type`) |
| `NamedEntityRecognitionTraining.train(data, model, language)` | nothing; writes the model |
| `train(data, model, language, iterations, cutoff)` | the same, with settings |

### A.4 Weights and scores

| Role | Weight |
|------|--------|
| `PROPN` | 3.0 |
| `NOUN` | 2.5 |
| `VERB` | 1.5 |
| `ADJ` | 1.0 |
| any other role | ignored |

Keyword scores run from 0 to 10, with the best word at 10. Search scores run from 0 to 10, the share of the question's content words found in the sentence.

## Appendix B. Failure handling

```java
import org.j2os.platform.ai.dl.language.nlp.core.*;
import org.j2os.platform.ai.dl.language.nlp.extraction.*;
import org.j2os.platform.ai.dl.language.nlp.ner.*;

public class Checks {
    interface Step { void run() throws Exception; }

    static void attempt(String label, Step step) {
        try {
            step.run();
            System.out.println(label + " -> ok");
        } catch (Exception e) {
            System.out.println(label + " -> " + e.getClass().getSimpleName() + ": " + e.getMessage().replace(System.getProperty("user.dir"), "."));
        }
    }

    public static void main(String[] args) throws Exception {
        Lemmatization lemmas = Lemmatization.load("models/en-lemmas.bin", "models/ud-pos.bin");

        attempt("missing model", () -> PartOfSpeechTagging.load("models/nope.bin"));
        attempt("null text", () -> Tokenization.tokenize(null));
        attempt("zero keywords", () -> KeywordExtraction.extract("Some text.", 0, lemmas));
        attempt("missing training file", () -> NamedEntityRecognitionTraining.train("nope.train", "m.bin", "en"));
    }
}
```

```text
missing model -> NlpFailedException: java.io.FileNotFoundException: Model file does not exist: ./models/nope.bin
null text -> NullPointerException: text must not be null
zero keywords -> IllegalArgumentException: maximumKeywordCount must be positive, was 0
missing training file -> NlpFailedException: java.io.FileNotFoundException: Training data file does not exist: ./nope.train
```

| Situation | What happens |
|-----------|--------------|
| A model file that does not exist | `NlpFailedException` caused by a `FileNotFoundException` |
| A model file that cannot be read | `NlpFailedException` caused by the `IOException` |
| A file that is not a model of that kind | A failure inside OpenNLP; the exact exception depends on its version |
| A training data file that does not exist | `NlpFailedException` caused by a `FileNotFoundException` |
| An unwritable output path | `NlpFailedException` caused by the `IOException` |
| A training file that is empty or has a broken mark (a `<START>` without `<END>`) | `IllegalArgumentException` from OpenNLP, "Model not compatible with name finder!" (the text depends on its version) |
| Any `null` argument | `NullPointerException` naming the argument |
| A keyword or sentence count of 0 or less | `IllegalArgumentException` |
| Iterations of 0 or less, or a negative cutoff | `IllegalArgumentException` |
| A tag the module has no description for | The description `نامشخص` |
| A text with no content words | Keywords: an empty list. Search: the first sentence, score `0.0` |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Wrong tag names.** A part-of-speech model without Universal Dependencies tags gives empty keywords, flat summaries and zero-score searches, with no error.
- **Mixing languages.** A model knows one language. Check the language first, and use the models of that language for all steps.
- **Trusting the first sentence.** `TextSearch` always returns a sentence. Read the score.
- **Case.** Lemmas that fall back to the written token keep their case, so keyword counts and search matches are case-sensitive.
- **Abbreviations in summaries.** The summary splitter is a simple rule; `Mr.` ends a sentence for it. Split with `SentenceDetection` first if abbreviations are common.
- **Loading in the request path.** Reading a model takes far longer than using it. Load at startup.
- **A tiny training file.** The model repeats its examples and misses the rest. Check it on new sentences.
- **Noisy progress output.** OpenNLP prints training progress on the standard output.

### C.2 Behavior worth knowing

- **Instances are thread-safe.** Only the immutable model is shared.
- **`load` reads the whole file.** The file may be deleted afterwards.
- **Order is kept.** Tokens, sentences, entities and summary sentences come in the order of the text.
- **Entity text is rebuilt.** Tokens of one entity are joined with single spaces.
- **Training is atomic.** The old model stays in place until the new one is complete.
- **Arabic letters are unified.** For keywords, summaries and search, the Arabic forms of `ي`, `ى` and `ك` are replaced by the Persian `ی` and `ک`.
- **Lemmas need a letter.** A lemma with no letter in it, such as `5` or `.`, is never a keyword.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Models | OpenNLP model files |
| Tag set for extraction | Universal Dependencies (`PROPN`, `NOUN`, `VERB`, `ADJ` count) |
| Descriptions of tags | Persian, 17 tags |
| Sentence cut for summaries and search | After `.`, `!`, `?` or `؟` and white space |
| Tokenizer | Rule-based, by character class; no model |
| Training algorithm | Perceptron only |
| Incremental training | None; every `train` starts over |
| Evaluation of an entity model | None built in |
| Language detection | A code only; no confidence |

### C.4 Tuning and practical advice

- **Load once, share.** Keep the loaded objects in fields or a holder for the whole application.
- **Pick the right model for each step.** A sentence model for splitting, a Universal Dependencies model for tags and lemmas, an entity model per set of types.
- **Cache results.** Keywords and summaries of the same text do not change; see the sketch in chapter 11.
- **Give entity types many examples.** Several dozen varied lines per type is a reasonable start; more is better.
- **Raise iterations and cutoff with data.** Start with the defaults. A larger file may use a higher cutoff to ignore rare noise.
- **Keep a test file.** Sentences that are not in the training file show the real quality.

## Appendix D. Message reference

### D.1 Failures from the module

| Message | Exception | Cause |
|---------|-----------|-------|
| `java.io.FileNotFoundException: Model file does not exist: <path>` | `NlpFailedException` | A model path that is not a file |
| `java.io.FileNotFoundException: Training data file does not exist: <path>` | `NlpFailedException` | A training file that is not a file |
| the text of the cause | `NlpFailedException` | An `IOException` while reading a model or writing the trained model |
| `Model not compatible with name finder!` | `IllegalArgumentException` | Training data with no valid sample, from OpenNLP |

### D.2 Argument checks

| Message | Exception | Cause |
|---------|-----------|-------|
| `<name> must not be null` | `NullPointerException` | `modelFilePath`, `lemmatizerModelFilePath`, `posModelFilePath`, `text`, `paragraph`, `question`, `lemmatization`, `trainingDataFilePath`, `outputModelFilePath` or `languageCode` is `null` |
| `maximumKeywordCount must be positive, was %d` | `IllegalArgumentException` | A keyword count of 0 or less |
| `maximumSentenceCount must be positive, was %d` | `IllegalArgumentException` | A sentence count of 0 or less |
| `iterations must be positive, was %d` | `IllegalArgumentException` | Iterations of 0 or less |
| `cutoff must not be negative, was %d` | `IllegalArgumentException` | A negative cutoff |
