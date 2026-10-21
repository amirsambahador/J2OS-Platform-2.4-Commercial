# ai.ml

*J2OS Platform 2.4*

Many decisions in a program follow a pattern that nobody wrote down as a rule: which loan applications get approved, which support ticket belongs to which team, what a house of a given size should cost, which customers behave alike. The pattern is in the old data. **Machine learning** means letting the computer find it there, so that a new case can be answered in the same way.

**ai.ml** (`org.j2os.platform.ai.ml`) is the classical, small-data side of this in J2OS. It wraps Weka, a well-known machine learning toolkit, behind a handful of classes with one habit in common: *train from a file, keep the model in a file, ask with a `Map`*. Three properties set it apart:

- **Plain inputs and outputs.** A question is a `Map` of field names and values; an answer is a small object with a label, a number or a cluster index. No Weka class appears in a program that uses the library.
- **Six ready models.** A decision tree, a text classifier, a linear regression, a clustering, a closest-row memory, and a writer that produces the dataset file for all of them.
- **Models are files.** Training writes the model to a path (replacing the old file in one step) and `load` brings it back in another program.

All of it in a few lines:

```java
try (JArffWriter w = new JArffWriter("loans", "data/loans.arff",          // 1. rows become a dataset
        JArffAttribute.numeric("income"), JArffAttribute.nominal("approved", "yes", "no"))) {
    w.addRow(70, "yes");
    w.addRow(20, "no");
    // ... more rows
}

JC45 tree = JC45.train("data/loans.arff", "models/loans.model");         // 2. learn, and keep the model

ClassificationResult r = tree.classify(Map.of("income", 55));            // 3. ask
System.out.println(r.getPredictedLabel() + " " + r.getConfidencePercentage() + "%");
```

Read it in this order: making a dataset (chapter 2), deciding with a tree (3), classifying text (4), predicting a number (5), grouping similar rows (6), recalling the closest row (7), keeping models (8) and a complete small workflow (9).

## Contents

1. [How ai.ml works](#1-how-aiml-works)
2. [Making a dataset](#2-making-a-dataset)
3. [Deciding with a tree](#3-deciding-with-a-tree)
4. [Classifying text](#4-classifying-text)
5. [Predicting a number](#5-predicting-a-number)
6. [Grouping similar rows](#6-grouping-similar-rows)
7. [Recalling the closest row](#7-recalling-the-closest-row)
8. [Keeping models](#8-keeping-models)
9. [The last example](#9-the-last-example)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

The programs are complete, not fragments: each has one `main` method and builds its own small dataset, in a folder named `data`, and its models in a folder named `models`. The data is invented and tiny so that the output is easy to follow; real models need far more rows.

---

## 1. How ai.ml works

Before anything else, the terms you need:

- **Dataset.** A table of examples, stored as a file. Each line is a **row**, each column an **attribute**.
- **ARFF.** The file format of Weka: a header that names the columns and their kinds, then the rows. `JArffWriter` writes it.
- **Class column.** The column that a model should learn to answer, such as `approved`. In this library it is always the **last** column.
- **Label.** One value of a class column, such as `yes`.
- **Training.** Letting the model study the dataset. It happens once, and the result is a **model**.
- **Classification.** Answering with a label. **Regression**: answering with a number. **Clustering**: finding groups of similar rows, without any class column.
- **Evaluation.** Measuring how good a model is. Precision says how many of the rows announced as `yes` really are `yes`; recall says how many of the real `yes` rows were found; the F-measure joins the two.

The classes you will use:

| Class | Role |
|-------|------|
| `JArffWriter`, `JArffAttribute` | Write a dataset file, row by row |
| `JC45` | Decision tree (Weka `J48`, an implementation of `C4.5`). Classifies a record |
| `JNaiveBayes` | Text classifier. Classifies a sentence |
| `JLinearRegression` | Predicts a number from other numbers |
| `JKMeans` | Groups rows into clusters and assigns new rows to them |
| `JAssociativeMemory` | Finds the stored row most like a query |
| `ClassificationResult`, `RegressionResult`, `ClusterAssignmentResult`, `ClusterCentroidResult`, `PatternMatchResult` | The answers |
| `Evaluation`, `RegressionEvaluation`, `ClusterEvaluation` | The quality measures |

Keep these three points in mind:

- **The class column is last.** `JC45`, `JNaiveBayes` and `JLinearRegression` treat the last column as the answer. `JKMeans` and `JAssociativeMemory` have no class column; every column counts.
- **A question is a `Map`.** Field names are matched to column names without regard to case and surrounding spaces. A column the map leaves out is treated as *missing*, and fields that match no column are ignored, unless *none* matches.
- **A model file is a Java-serialized object.** Load only files that you created or trust.

## 2. Making a dataset

`JArffWriter` collects rows and writes the file when it is closed:

```java
import org.j2os.platform.ai.ml.arff.*;
import java.nio.file.*;

public class Arff {
    public static void main(String[] args) throws Exception {
        try (JArffWriter writer = new JArffWriter("customers", "data/customers.arff",
                JArffAttribute.numeric("purchaseCount"),
                JArffAttribute.numeric("purchaseAmount"),
                JArffAttribute.nominal("city", "Tehran", "Shiraz"),
                JArffAttribute.string("note"))) {
            writer.addRow(3, 120.5, "Tehran", "likes discounts");
            writer.addRow(10, 980, "Shiraz", null);
            writer.addRow(null, 40, "Tehran", "new, from a friend");
        }
        System.out.println(Files.readString(Path.of("data/customers.arff")));

        JArffWriter other = new JArffWriter("t", "data/t.arff", JArffAttribute.numeric("a"), JArffAttribute.nominal("b", "x", "y"));
        try { other.addRow(1); }
        catch (Exception e) { System.out.println("one value for two columns -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { other.addRow("abc", "x"); }
        catch (Exception e) { System.out.println("text in a numeric column -> " + e.getClass().getSimpleName()); }

        try { other.addRow(1, "z"); }
        catch (Exception e) { System.out.println("value not in the list -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        other.close();
        try { other.addRow(2, "y"); }
        catch (Exception e) { System.out.println("after close -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
    }
}
```

```text
@relation customers

@attribute purchaseCount numeric
@attribute purchaseAmount numeric
@attribute city {Tehran,Shiraz}
@attribute note string

@data
3,120.5,Tehran,'likes discounts'
10,980,Shiraz,?
?,40,Tehran,'new, from a friend'

one value for two columns -> IllegalArgumentException: Expected 2 values (one per attribute), got 1
text in a numeric column -> ClassCastException
value not in the list -> IllegalArgumentException: Value not defined for given nominal attribute!
after close -> IllegalStateException: addRow() called after close(): this writer's file has already been saved.
```

- **The file.** The header comes first, with one line per column: `numeric`, a list of allowed values in braces for a nominal column, or `string` for free text. Text is quoted when it needs to be, and a `null` becomes `?`, the sign of a missing value.
- **Three kinds of column.** `numeric(name)` for numbers, `nominal(name, values...)` for a fixed list of words, `string(name)` for free text. A model that must answer with a label needs a nominal class column.
- **Numbers** are written in plain decimal form, whatever the language settings of the computer, so the file reads back to the same value.
- **Wrong number of values.** The count of values must match the count of columns.
- **Text in a numeric column.** The value must be a `Number`; text fails with a `ClassCastException`.
- **A value outside the list.** A nominal column accepts only the words it was declared with.
- **After `close()`.** The file is written when the writer is closed, so use `try`-with-resources; adding a row after that is an error. Closing twice is harmless.

The file is first written beside the target and then moved into place, and missing folders are created. All rows are held in memory until then.

Rows usually come from a database:

```java
try (JArffWriter w = new JArffWriter("customers", "data/customers.arff",
             JArffAttribute.numeric("purchaseCount"),
             JArffAttribute.numeric("purchaseAmount"));
     ResultSet rs = statement.executeQuery("select purchase_count, purchase_amount from customer")) {
    while (rs.next()) {
        w.addRow(rs.getInt("purchase_count"), rs.getDouble("purchase_amount"));   // one row per record
    }
}                                                         // the file is written here, when the writer closes
```

## 3. Deciding with a tree

A decision tree learns a chain of questions ("is the income above 45?") that leads to an answer. `JC45` trains one from a dataset whose last column holds the labels:

```java
import org.j2os.platform.ai.ml.arff.*;
import org.j2os.platform.ai.ml.c45.JC45;
import org.j2os.platform.ai.ml.evaluation.Evaluation;
import org.j2os.platform.ai.ml.result.ClassificationResult;
import java.util.Map;

public class Tree {
    public static void main(String[] args) throws Exception {
        try (JArffWriter w = new JArffWriter("loans", "data/loans.arff",
                JArffAttribute.numeric("age"), JArffAttribute.numeric("income"),
                JArffAttribute.nominal("ownsHome", "yes", "no"),
                JArffAttribute.nominal("approved", "yes", "no"))) {
            for (int i = 0; i < 24; i++) {
                int income = 20 + (i * 7) % 60;
                w.addRow(20 + i * 2, income, i % 3 == 0 ? "yes" : "no", income >= 45 ? "yes" : "no");
            }
        }

        JC45 tree = JC45.train("data/loans.arff", "models/loans.model");

        Evaluation ev = tree.evaluate();
        System.out.printf("yes: precision %.0f%%, recall %.0f%%, f-measure %.0f%%%n",
                ev.precisionPercentage("yes"), ev.recallPercentage("yes"), ev.fMeasurePercentage("yes"));

        ClassificationResult a = tree.classify(Map.of("age", 35, "income", 70, "ownsHome", "no"));
        System.out.printf("applicant A: %s (%.0f%%)%n", a.getPredictedLabel(), a.getConfidencePercentage());

        ClassificationResult b = tree.classify(Map.of(" INCOME ", "30", "colour", "red"));
        System.out.printf("applicant B: %s (%.0f%%)%n", b.getPredictedLabel(), b.getConfidencePercentage());

        try { tree.classify(Map.of("colour", "red")); }
        catch (Exception e) { System.out.println("no known field -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { tree.classify(Map.of("income", "a lot")); }
        catch (Exception e) { System.out.println("not a number -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { tree.classify(Map.of("ownsHome", "maybe")); }
        catch (Exception e) { System.out.println("bad nominal -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { ev.recallPercentage("perhaps"); }
        catch (Exception e) { System.out.println("bad label -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
    }
}
```

```text
yes: precision 100%, recall 100%, f-measure 100%
applicant A: yes (100%)
applicant B: no (100%)
no known field -> IllegalArgumentException: None of the given fields matches an attribute of the model. Attribute names: [age, income, ownsHome], given fields: [colour]
not a number -> IllegalArgumentException: Attribute "income" expects a number but got "a lot"
bad nominal -> IllegalArgumentException: "maybe" is not a valid value of attribute "ownsHome". Valid values: [yes, no]
bad label -> IllegalArgumentException: Invalid class label: "perhaps". Valid values: [yes, no]
```

- **The data.** Twenty-four applicants whose approval depends on income alone. The tree finds that rule by itself.
- **`train`** reads the file, builds the tree, writes the model to the second path and returns it ready to use.
- **`evaluate()`** measures the tree on the same rows it learned from. The result is `100%` here and is *optimistic*: it says the tree has learned its own examples, not that it will do well on new ones.
- **Applicant A.** A map with `age`, `income` and `ownsHome`. The answer is the label and the confidence, in percent.
- **Applicant B.** The key `" INCOME "` still finds the column `income`, the number may be given as text (`"30"`), and the field `colour` matches nothing and is ignored. Columns the map leaves out are treated as missing.
- **No known field.** If fields are given and none of them is a column, the call fails and the message lists the column names and the fields given.
- **Not a number.** A numeric column needs a `Number` or a text that holds one.
- **Bad nominal value.** The message lists the valid values.
- **Bad label.** Precision, recall and F-measure are asked for a label that exists in the class column.

An honest check needs rows the tree did not see. Keep some rows aside, train on the rest and count:

```java
// train.arff holds 80% of the rows, the other 20% are kept as maps with their true label
JC45 tree = JC45.train("data/train.arff", "models/loans.model");

int right = 0;
for (Map<String, Object> row : testRows) {
    String truth = (String) row.remove("approved");                       // take the answer out of the question
    if (tree.classify(row).getPredictedLabel().equals(truth)) right++;
}
System.out.println(right + " of " + testRows.size() + " right on rows the tree never saw");
```

## 4. Classifying text

`JNaiveBayes` learns which words go with which label. The dataset needs a string column named **`text`** and, as its last column, the label:

```java
import org.j2os.platform.ai.ml.arff.*;
import org.j2os.platform.ai.ml.bayes.JNaiveBayes;
import org.j2os.platform.ai.ml.result.ClassificationResult;

public class Text {
    public static void main(String[] args) throws Exception {
        String[][] news = {
                {"the team won the football match", "sports"},
                {"the striker scored two goals", "sports"},
                {"a great game for the home team", "sports"},
                {"the coach praised the players after the match", "sports"},
                {"the new phone has a faster processor", "tech"},
                {"the software update fixes the battery problem", "tech"},
                {"a startup released a new laptop", "tech"},
                {"the processor and the memory are faster", "tech"}};

        try (JArffWriter w = new JArffWriter("news", "data/news.arff",
                JArffAttribute.string("text"), JArffAttribute.nominal("topic", "sports", "tech"))) {
            for (String[] row : news) w.addRow(row[0], row[1]);
        }

        JNaiveBayes bayes = JNaiveBayes.train("data/news.arff", "models/news.model");

        for (String text : new String[]{"the team scored a goal", "The Team Scored a Goal", "a faster laptop processor", "tomorrow it may rain"}) {
            ClassificationResult r = bayes.classify(text);
            System.out.printf("%-28s -> %s (%.0f%%)%n", text, r.getPredictedLabel(), r.getConfidencePercentage());
        }

        System.out.printf("training-set precision for sports: %.0f%%%n", bayes.evaluate().precisionPercentage("sports"));

        try (JArffWriter w = new JArffWriter("bad", "data/bad.arff",
                JArffAttribute.string("body"), JArffAttribute.nominal("topic", "sports", "tech"))) {
            w.addRow("some words", "tech");
        }
        JNaiveBayes noText = JNaiveBayes.train("data/bad.arff", "models/bad.model");
        try { noText.classify("any words"); }
        catch (Exception e) { System.out.println("no column called text -> " + e.getClass().getSimpleName()); }
    }
}
```

```text
the team scored a goal       -> sports (100%)
The Team Scored a Goal       -> tech (100%)
a faster laptop processor    -> tech (100%)
tomorrow it may rain         -> tech (99%)
training-set precision for sports: 100%
no column called text -> NullPointerException
```

- **The data.** Eight short sentences, four per topic.
- **Words.** The text is cut into words, and each word becomes a signal. The model weighs the signals of a new sentence and answers with the label and its confidence.
- **Case matters.** `the team scored a goal` is `sports`, but the same words in capitals are not recognized, and the model answers `tech` with full confidence. Words are compared as written; clean every text the same way, for training and for questions:

```java
static String clean(String text) {
    return text.toLowerCase(Locale.ROOT).strip();          // use the same cleaning for training rows and for questions
}

writer.addRow(clean(title), topic);                         // training
bayes.classify(clean(incomingText));                        // asking
```

- **Confidence is not familiarity.** `tomorrow it may rain` has no word the model knows, and still comes out as `tech` with `99%`. The percentage tells how the model splits its belief between the known labels, never whether the text is like anything it has seen. Do not treat a high percentage as proof.
- **`evaluate()`** again measures on the training rows.
- **No `text` column.** A dataset whose string column has another name trains without complaint, and the first `classify` fails with a bare `NullPointerException`.

## 5. Predicting a number

`JLinearRegression` fits a straight-line relation between numeric columns and a numeric last column:

```java
import org.j2os.platform.ai.ml.arff.*;
import org.j2os.platform.ai.ml.evaluation.RegressionEvaluation;
import org.j2os.platform.ai.ml.regression.JLinearRegression;
import java.util.Map;

public class Regression {
    public static void main(String[] args) throws Exception {
        try (JArffWriter w = new JArffWriter("houses", "data/houses.arff",
                JArffAttribute.numeric("area"), JArffAttribute.numeric("rooms"), JArffAttribute.numeric("price"))) {
            for (int i = 0; i < 20; i++) {
                int area = 50 + i * 5, rooms = 1 + i % 4;
                int noise = (i * 37) % 7 - 3;                       // a little scatter, so the fit is not perfect
                w.addRow(area, rooms, 10 * area + 40 * rooms + noise * 5);
            }
        }

        JLinearRegression model = JLinearRegression.train("data/houses.arff", "models/houses.model");

        RegressionEvaluation ev = model.evaluate();
        System.out.printf("correlation %.3f, mean error %.1f, root mean squared error %.1f%n",
                ev.correlationCoefficient(), ev.meanAbsoluteError(), ev.rootMeanSquaredError());

        System.out.printf("100 m2, 3 rooms -> %.0f%n", model.predict(Map.of("area", 100, "rooms", 3)).getPredictedValue());
        System.out.printf("only the area 100 -> %.0f%n", model.predict(Map.of("area", 100)).getPredictedValue());

        try { model.predict(Map.of("floor", 3)); }
        catch (Exception e) { System.out.println("no known field -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
    }
}
```

```text
correlation 0.999, mean error 8.5, root mean squared error 9.9
100 m2, 3 rooms -> 1120
only the area 100 -> 1100
no known field -> IllegalArgumentException: None of the given fields matches an attribute of the model. Attribute names: [area, rooms], given fields: [floor]
```

- **The data.** Twenty houses where the price follows the area and the rooms, with a little scatter added.
- **Quality.** The correlation coefficient (close to 1 is good), the mean absolute error and the root mean squared error, both in the unit of the price column. The root measure punishes large mistakes more.
- **A full question.** `100 m2, 3 rooms` gives the price the line predicts.
- **A missing column.** With only the area, the missing `rooms` is replaced by its average over the training rows, and the answer changes a little.
- **No known field.** As for the tree, a map that matches no column is refused.
- The class column must be numeric. A nominal class makes training fail with Weka's `UnsupportedAttributeTypeException`.

## 6. Grouping similar rows

`JKMeans` has no class column. It looks for groups of rows that are close to each other, described by their centers:

```java
import org.j2os.platform.ai.ml.arff.*;
import org.j2os.platform.ai.ml.kmeans.JKMeans;
import org.j2os.platform.ai.ml.result.ClusterCentroidResult;
import java.util.Map;

public class Clusters {
    public static void main(String[] args) throws Exception {
        double[][] customers = {{2, 20}, {10, 120}, {25, 400}, {3, 25},
                                {12, 110}, {28, 420}, {2, 30}, {11, 130},
                                {26, 390}, {3, 22}, {9, 115}, {27, 410}};
        try (JArffWriter w = new JArffWriter("customers", "data/customers.arff",
                JArffAttribute.numeric("purchaseCount"), JArffAttribute.numeric("purchaseAmount"))) {
            for (double[] c : customers) w.addRow(c[0], c[1]);
        }

        JKMeans fixed = JKMeans.train("data/customers.arff", "models/fixed.model", 3);
        System.out.printf("3 clusters, squared error %.3f%n", fixed.evaluate().getSquaredError());
        for (ClusterCentroidResult c : fixed.getClusterCentroidResults()) {
            System.out.printf("cluster %d: count %.1f, amount %.1f%n", c.getClusterIndex(),
                    (Double) c.getValues().get(0), (Double) c.getValues().get(1));
        }
        System.out.println("a customer with 11 purchases and 125 -> cluster "
                + fixed.assignCluster(Map.of("purchaseCount", 11, "purchaseAmount", 125)).getClusterIndex());

        JKMeans auto = JKMeans.train("data/customers.arff", "models/auto.model");
        System.out.println("clusters chosen by the library: " + auto.getClusterCentroidResults().size());

        try { JKMeans.train("data/customers.arff", "models/zero.model", 0); }
        catch (Exception e) { System.out.println("0 clusters -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try { fixed.assignCluster(Map.of("colour", "red")); }
        catch (Exception e) { System.out.println("no known field -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
    }
}
```

```text
3 clusters, squared error 0.021
cluster 0: count 2.5, amount 24.3
cluster 1: count 10.5, amount 118.8
cluster 2: count 26.5, amount 405.0
a customer with 11 purchases and 125 -> cluster 1
clusters chosen by the library: 3
0 clusters -> Exception: Number of clusters must be > 0
no known field -> IllegalArgumentException: None of the given fields matches an attribute of the model. Attribute names: [purchaseCount, purchaseAmount], given fields: [colour]
```

- **The data.** Twelve customers who fall into three kinds of behavior.
- **Fixed number.** `train(dataset, model, 3)` asks for three clusters. The **squared error** is the sum of squared distances from each row to the center of its cluster; lower means tighter groups, and the number is meaningful only for comparing models of the same dataset.
- **Centers.** Each `ClusterCentroidResult` holds one value per column: a `Double` for numeric columns and a `String` for the others.
- **A new customer** is assigned to the nearest center, and its index is returned.
- **Automatic number.** `train(dataset, model)` without a count tries 2 to 10 clusters (fewer for a small dataset), looks at how the squared error falls, and picks the "elbow" where more clusters stop paying off. It builds many models, so it is slower.
- **Zero clusters.** A count below 1 is refused by Weka with a plain `Exception`.
- **No known field.** A map that matches no column is refused.

The groups depend on where the search starts. The same twelve customers listed in another order can give an uneven split with a larger squared error. When the result looks odd, compare the squared error of runs on the same data.

## 7. Recalling the closest row

`JAssociativeMemory` stores the rows and answers with the one that is most like the query:

```java
import org.j2os.platform.ai.ml.arff.*;
import org.j2os.platform.ai.ml.associativememory.JAssociativeMemory;
import org.j2os.platform.ai.ml.result.PatternMatchResult;
import java.util.Map;

public class Memory {
    public static void main(String[] args) throws Exception {
        try (JArffWriter w = new JArffWriter("phones", "data/phones.arff",
                JArffAttribute.numeric("screen"), JArffAttribute.numeric("battery"),
                JArffAttribute.nominal("colour", "black", "white", "blue"))) {
            w.addRow(6.1, 4000, "black");
            w.addRow(6.7, 5000, "white");
            w.addRow(5.4, 3000, "blue");
        }

        JAssociativeMemory memory = JAssociativeMemory.train("data/phones.arff", "models/phones.model");

        PatternMatchResult r = memory.recall(Map.of("screen", 6.5, "colour", "white"));
        System.out.println("closest row " + r.getMatchedPatternIndex() + ": " + r.getMatchedPattern());
        System.out.printf("score %.2f%n", r.getSimilarityScore());

        try { memory.recall(Map.of("weight", 150)); }
        catch (Exception e) { System.out.println("no known field -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        try (JArffWriter w = new JArffWriter("none", "data/none.arff", JArffAttribute.numeric("x"))) { }
        try { JAssociativeMemory.train("data/none.arff", "models/none.model").recall(Map.of("x", 1)); }
        catch (Exception e) { System.out.println("empty memory -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }
    }
}
```

```text
closest row 1: {screen=6.7, battery=5000.0, colour=white}
score 44.55
no known field -> IllegalArgumentException: None of the given fields matches an attribute of the model. Attribute names: [screen, battery, colour], given fields: [weight]
empty memory -> IllegalStateException: No patterns are stored to compare against; the training dataset was empty.
```

- **The data.** Three phones. There is no class column; every column is part of the pattern.
- **Recall.** The query names only the screen and the color. Columns the query leaves out do not count. The result holds the row, its position (starting at 0) and the score.
- **The score.** For each numeric column, the query value times the stored value; for each other column, 1 if the words are equal and 0 if not; all added. Here `6.5 × 6.7 = 43.55`, plus 1 for the white color, gives `44.55`.
- **No known field.** A query that matches no column is refused.
- **Empty memory.** Recalling from a dataset without rows fails with an `IllegalStateException`.

Because the score multiplies numbers, a column measured in thousands outweighs one measured in single digits. Bring numeric columns to a similar scale before writing the rows and before asking:

```java
// the closest-pattern score multiplies numbers, so a column in thousands outweighs a column in single digits
double battery = batteryMah / 6000.0;                       // bring every numeric column to about 0..1
double screen  = screenInches / 7.0;                        // before it is written, and before it is asked
writer.addRow(screen, battery, colour);
memory.recall(Map.of("screen", wantedInches / 7.0, "colour", "white"));
```

## 8. Keeping models

Every `train` writes the model to the path it is given, and `load` reads it back. This is how a training job and a service share a model:

```java
import org.j2os.platform.ai.ml.arff.*;
import org.j2os.platform.ai.ml.c45.JC45;
import org.j2os.platform.ai.ml.kmeans.JKMeans;
import java.nio.file.*;
import java.util.Map;

public class Models {
    public static void main(String[] args) throws Exception {
        try (JArffWriter w = new JArffWriter("pets", "data/pets.arff",
                JArffAttribute.numeric("weight"), JArffAttribute.nominal("kind", "cat", "dog"))) {
            for (int i = 0; i < 10; i++) { w.addRow(3 + i % 3, "cat"); w.addRow(15 + i % 5, "dog"); }
        }

        JC45.train("data/pets.arff", "models/pets.model");          // the training program stops here

        JC45 loaded = JC45.load("models/pets.model");                // a later program starts here
        System.out.println("loaded model says: " + loaded.classify(Map.of("weight", 17)).getPredictedLabel());

        JKMeans.train("data/pets.arff", "models/groups.model", 2);
        try { JC45.load("models/groups.model"); }
        catch (Exception e) { System.out.println("wrong kind of model -> " + e.getClass().getSimpleName() + ": " + e.getMessage()); }

        Files.writeString(Path.of("models/broken.model"), "this is not a model");
        try { JC45.load("models/broken.model"); }
        catch (Exception e) { System.out.println("damaged file -> " + e.getClass().getSimpleName()); }

        try { JC45.load("models/missing.model"); }
        catch (Exception e) { System.out.println("missing file -> " + e.getClass().getSimpleName()); }

        try { JC45.train("data/nothing.arff", "models/x.model"); }
        catch (Exception e) { System.out.println("missing dataset -> " + e.getClass().getSimpleName()); }
    }
}
```

```text
loaded model says: dog
wrong kind of model -> IllegalStateException: Model file "models/groups.model" is invalid or corrupted. Delete the file and rebuild it with train(...).
damaged file -> StreamCorruptedException
missing file -> FileNotFoundException
missing dataset -> NullPointerException
```

- **Train and load.** After `load`, the object behaves as the one that was trained.
- **Wrong kind of model.** Loading a clustering file as a tree is refused by the library with a message that says to delete the file and train again.
- **A damaged file.** A file that is not a serialized model at all is refused by Weka's reader with a `StreamCorruptedException`.
- **A missing file.** `FileNotFoundException`.
- **A missing dataset.** A dataset path that does not exist fails with a `NullPointerException`.

A model is written to a temporary file and moved into place, so a reader never sees a half-written model. A running object keeps the model it has in memory; it does not notice a new file. A service therefore loads once, and switches to a fresh model on purpose:

```java
public final class Scoring {
    private static final JC45 TREE = load();                // read the model file once, when the class loads

    private static JC45 load() {
        try { return JC45.load("/var/lib/shop/models/loans.model"); }
        catch (Exception e) { throw new ExceptionInInitializerError(e); }
    }

    public static boolean approve(Map<String, Object> applicant) throws Exception {
        return TREE.classify(applicant).getPredictedLabel().equals("yes");
    }
}
```

```java
private volatile JC45 current;

void start() throws Exception {
    current = JC45.load("models/loans.model");                             // when the service starts
}

void nightlyJob() throws Exception {
    JC45 fresh = JC45.train("data/loans.arff", "models/loans.model");     // the file is replaced in one step
    current = fresh;                                                       // requests switch to the new model
}
```

Loading deserializes a Java object. Never load a model file from a source that you do not trust.

## 9. The last example

Support tickets are sorted into two teams. The history becomes a dataset, a model is trained and stored, and a service loads it and routes new tickets:

```java
import org.j2os.platform.ai.ml.arff.*;
import org.j2os.platform.ai.ml.bayes.JNaiveBayes;
import org.j2os.platform.ai.ml.result.ClassificationResult;

public class Workflow {
    public static void main(String[] args) throws Exception {
        String[][] tickets = {
                {"I was charged twice for my invoice", "billing"},
                {"please refund the payment on my invoice", "billing"},
                {"the card payment failed again", "billing"},
                {"my invoice shows the wrong amount", "billing"},
                {"the app crashes when I open the report", "support"},
                {"I cannot log in to the app", "support"},
                {"the report page shows an error", "support"},
                {"the app is slow and then crashes", "support"}};

        // 1. the history becomes a dataset
        try (JArffWriter w = new JArffWriter("tickets", "data/tickets.arff",
                JArffAttribute.string("text"), JArffAttribute.nominal("team", "billing", "support"))) {
            for (String[] t : tickets) w.addRow(t[0], t[1]);
        }

        // 2. train once and keep the model
        JNaiveBayes.train("data/tickets.arff", "models/tickets.model");

        // 3. the service loads the model and routes new tickets
        JNaiveBayes router = JNaiveBayes.load("models/tickets.model");
        for (String text : new String[]{"refund for a double payment", "login error in the app", "do you sell gift cards?"}) {
            ClassificationResult r = router.classify(text);
            System.out.printf("%-30s -> %s (%.0f%%)%n", text, r.getPredictedLabel(), r.getConfidencePercentage());
        }
    }
}
```

```text
refund for a double payment    -> billing (100%)
login error in the app         -> support (100%)
do you sell gift cards?        -> billing (100%)
```

- **Step 1.** `JArffWriter` turns past tickets into a dataset.
- **Step 2.** `JNaiveBayes.train` learns from it and stores the model.
- **Step 3.** The service loads the model and classifies each new text.
- **The last ticket** is about gift cards, which neither team handled in the history, yet it is routed to `billing` with `100%`. A model can only choose among the labels it knows. For a real service, add a label such as `other` with examples, and keep a human check for important cases.

---

## Appendix A. Quick reference

### A.1 JArffWriter and JArffAttribute

| Call | Purpose |
|------|---------|
| `new JArffWriter(relation, path, attributes...)` | A writer for a file at `path` with the given columns |
| `addRow(values...)` | Adds a row; one value per column; `null` is a missing value |
| `close()` | Writes the file (temporary file, then moved). Safe to call twice |
| `JArffAttribute.numeric(name)` | A numeric column |
| `JArffAttribute.nominal(name, values...)` | A column with a fixed list of words |
| `JArffAttribute.string(name)` | A free-text column |

### A.2 Models with a class column

| Class | Calls | Class column |
|-------|-------|--------------|
| `JC45` | `train(dataset, model)`, `load(model)`, `evaluate()`, `classify(map)` | Nominal, last |
| `JNaiveBayes` | `train(dataset, model)`, `load(model)`, `evaluate()`, `classify(text)` | Nominal, last; a string column named `text` |
| `JLinearRegression` | `train(dataset, model)`, `load(model)`, `evaluate()`, `predict(map)` | Numeric, last |

### A.3 JKMeans

| Call | Purpose |
|------|---------|
| `train(dataset, model, k)` | Trains with `k` clusters and stores the model |
| `train(dataset, model)` | Chooses the number of clusters (2 to 10) by the elbow method |
| `load(model)` | Loads a stored model |
| `evaluate()` | The squared error |
| `getClusterCentroidResults()` | The center of every cluster |
| `assignCluster(map)` | The cluster of a new row |

### A.4 JAssociativeMemory

| Call | Purpose |
|------|---------|
| `train(dataset, model)` | Stores the rows as patterns and saves them |
| `load(model)` | Loads the stored patterns |
| `recall(map)` | The closest pattern, its index and its score |

### A.5 Answers and measures

| Type | Calls |
|------|-------|
| `ClassificationResult` | `getPredictedLabel()`, `getConfidencePercentage()` (0 to 100) |
| `RegressionResult` | `getPredictedValue()` |
| `ClusterAssignmentResult` | `getClusterIndex()` |
| `ClusterCentroidResult` | `getClusterIndex()`, `getValues()` |
| `PatternMatchResult` | `getMatchedPattern()`, `getMatchedPatternIndex()`, `getSimilarityScore()` |
| `Evaluation` | `precisionPercentage(label)`, `recallPercentage(label)`, `fMeasurePercentage(label)` |
| `RegressionEvaluation` | `correlationCoefficient()`, `meanAbsoluteError()`, `rootMeanSquaredError()` |
| `ClusterEvaluation` | `getSquaredError()` |

### A.6 Rules for a question

| Rule | Detail |
|------|--------|
| Names | Matched to column names ignoring case and surrounding spaces; an exact match wins |
| Numbers | A `Number` or a text that holds one |
| Nominal columns | The value must be one of the declared words |
| Missing columns | Treated as missing |
| Unknown fields | Ignored, unless no field matches any column |
| Class column | Never read from the question |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| `addRow` with the wrong number of values | `IllegalArgumentException` |
| `addRow` after `close()` | `IllegalStateException` |
| A non-number in a numeric column | `ClassCastException` |
| A word outside a nominal column's list | `IllegalArgumentException` (from Weka) |
| A dataset path that does not exist | `NullPointerException` |
| A tree or text classifier on a numeric class column | Weka's `UnsupportedAttributeTypeException` |
| A regression on a nominal class column | Weka's `UnsupportedAttributeTypeException` |
| `classify`, `predict`, `assignCluster` or `recall` with no matching field | `IllegalArgumentException` listing the columns |
| A text for a numeric column | `IllegalArgumentException` |
| A word that is not valid for a nominal column | `IllegalArgumentException` listing the valid values |
| A label that does not exist in an evaluation call | `IllegalArgumentException` listing the labels |
| `JNaiveBayes` on a dataset without a `text` column | `NullPointerException` at `classify` |
| `JKMeans.train(..., 0)` | Weka's plain `Exception`, "Number of clusters must be > 0" |
| `JKMeans.train` without a count on fewer than two rows | `IllegalStateException` |
| `load` of a file with the wrong kind of content | `IllegalStateException` |
| `load` of a file that is not a serialized object | `StreamCorruptedException` |
| `load` of a file that does not exist | `FileNotFoundException` |
| `recall` on an empty memory | `IllegalStateException` |
| A text that no word of the model knows | Answered anyway, usually with a high confidence |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Believing the evaluation.** `evaluate()` measures on the training rows. Hold some rows out.
- **Believing the confidence.** It is a share between known labels, not a sign of familiarity.
- **Capital letters in text.** Words are compared as written; clean the text the same way everywhere.
- **Too few rows.** A model of eight sentences is a demonstration, not a tool.
- **Rows that do not represent the future.** A model repeats the past, including its mistakes.
- **Unbalanced labels.** If 95% of the rows say `no`, a model that always says `no` looks very good.
- **Numbers on different scales in the closest-row memory.** The big column decides.
- **Believing in one clustering.** The result depends on the starting point; compare squared errors, and look at the centers.
- **Choosing the cluster count by the squared error alone.** It always falls as the count rises.
- **A class column that is not last.** The library takes the last column; reorder before writing.
- **A string column not named `text`.** `JNaiveBayes` fails only when asked.
- **A dataset path that is wrong.** It shows up as a `NullPointerException`.
- **Loading model files from outside.** They are serialized Java objects.
- **Loading a model on every request.** Load once.
- **Expecting a live object to see a new file.** It keeps the model it loaded.
- **Sharing one instance among threads without care.** See C.2.

### C.2 Behavior worth knowing

- Everything rests on Weka. The first model of a run may print warnings about native numeric libraries that could not be loaded; Weka falls back to Java code and works the same.
- `train` always replaces the model file at its path, creating missing folders.
- The decision tree uses Weka's default settings: it prunes, and a leaf needs at least two rows. There is no way to change them through this library.
- Missing values are allowed in questions. The tree handles them inside the tree, the regression replaces them by the column average, and the clustering by the average or the most common word.
- `JNaiveBayes` turns text into one column per word with Weka's default settings: no lowercase, no stemming, no stop-word list.
- `classify` of `JNaiveBayes`, `evaluate` and `predict` of `JLinearRegression`, `evaluate` of `JNaiveBayes` and `assignCluster` of `JKMeans` are synchronized on the instance. The other methods read the stored model and do not change it.
- Weka reads a dataset by the extension of its file name, so a `.csv` file can be trained on as well; the examples use ARFF, which states the column kinds explicitly.
- A dataset without rows trains without an error, and a cluster count larger than the row count is accepted by Weka.

### C.3 Limits

| Limit | Value |
|-------|-------|
| Class column | Always the last column |
| Models | Tree, Naive Bayes for text, linear regression, K-Means, closest row |
| Settings of the algorithms | None; Weka defaults |
| Automatic cluster count | 2 to 10 |
| Evaluation data | The training rows only |
| Clustering evaluation | The squared error only |
| Text | One column named `text`; words as written |
| Model storage | A Java-serialized file |
| Dataset size | Held in memory while written and while trained |
| Incremental training | None; every `train` starts over |

### C.4 Tuning and practical advice

- Write the dataset with `JArffWriter` from the same code that holds the real records, so training and asking use the same names.
- Hold out a part of the rows and count the right answers on it (chapter 3).
- Clean text the same way for training and asking.
- Give every label enough examples, and add an `other` label for text the model should refuse.
- Scale numeric columns before the closest-row memory.
- Try several cluster counts and look at the centers, not at the squared error alone.
- Train in a job and let services only `load`; switch models with a volatile reference.
- Keep model files in a folder that only your programs can write.
- Retrain on a schedule; the data changes, and a model does not notice.

## Appendix D. Message reference

### D.1 JArffWriter

| Message | Exception | Cause |
|---------|-----------|-------|
| `addRow() called after close(): this writer's file has already been saved.` | `IllegalStateException` | A row added after the writer was closed |
| `Expected %d values (one per attribute), got %d` | `IllegalArgumentException` | The count of values differs from the count of columns |
| none | `ClassCastException` | A value that is not a `Number` in a numeric column |
| `Value not defined for given nominal attribute!` | `IllegalArgumentException` | A word outside a nominal column's list (message from Weka) |

### D.2 Questions

| Message | Exception | Cause |
|---------|-----------|-------|
| `None of the given fields matches an attribute of the model. Attribute names: [...], given fields: [...]` | `IllegalArgumentException` | Fields given, none of them a column |
| `Attribute "%s" expects a number but got "%s"` | `IllegalArgumentException` | A text that is not a number in a numeric column |
| `Attribute "%s" expects a number but got a value of type %s` | `IllegalArgumentException` | A value that is neither a number nor text |
| `"%s" is not a valid value of attribute "%s". Valid values: [...]` | `IllegalArgumentException` | A word outside a nominal column's list |
| `Invalid class label: "%s". Valid values: [...]` | `IllegalArgumentException` | `precisionPercentage`, `recallPercentage` or `fMeasurePercentage` with an unknown label |

### D.3 Models

| Message | Exception | Cause |
|---------|-----------|-------|
| `Model file "%s" is invalid or corrupted. Delete the file and rebuild it with train(...).` | `IllegalStateException` | `load` of a file that holds another kind of model |
| `No patterns are stored to compare against; the training dataset was empty.` | `IllegalStateException` | `recall` on an empty memory |
| `Not enough dataset samples to automatically determine the cluster count.` | `IllegalStateException` | `JKMeans.train(dataset, model)` with fewer than two rows |
| `Number of clusters must be > 0` | `Exception` (from Weka) | `JKMeans.train(..., k)` with `k` below 1 |
| `...: Cannot handle numeric class!` | `UnsupportedAttributeTypeException` (from Weka) | A tree or text classifier trained on a numeric class column |
| `...: Cannot handle binary class!` | `UnsupportedAttributeTypeException` (from Weka) | A regression trained on a nominal class column; Weka names the kind of the column, here a two-valued one |
