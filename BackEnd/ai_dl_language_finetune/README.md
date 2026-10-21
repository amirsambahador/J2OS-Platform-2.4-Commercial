# ai.dl.language.finetune

*J2OS Platform 2.4*

A general-purpose language model knows a lot about the world, but it knows nothing about your own business, and it answers in its own style. Suppose a support chatbot must answer questions such as "When do you open?", "Do you deliver?" or "Can I return a product?" with your rules and in your words.

The cure is **fine-tuning**. You take a model that already knows language, show it a few hundred examples of the answers you want, and let it learn from them. Afterwards the model answers like your business.

**ai.dl.language.finetune** (`org.j2os.platform.ai.dl.language.finetune`) brings this to J2OS, and you do not need to know the field to use it. You give two things, a base model and a file of examples, and call `run()`. The library does the rest: it prepares its own Python environment, checks your examples, looks at your graphics card and the model, chooses safe settings, trains, and saves the result. If you do know the field, every setting can be chosen by hand.

Three properties set the module apart:

- **Two inputs are enough.** Everything else has a sensible automatic value.
- **Everything runs on your own machine.** Your examples never leave it.
- **It explains itself.** Wrong files and impossible settings are stopped early, with messages that say what to change.

The library in use, briefly:

```java
FineTuneResult result = JFineTune.create()
        .baseModel("Qwen/Qwen2.5-1.5B")                  // 1. a model that already knows language
        .dataset(Path.of("examples.jsonl"))              // 2. your own examples
        .run();                                          // 3. train; the result is a small "adapter"
System.out.println(result.adapterDirectory());
```

Here is how the guide is arranged: what fine-tuning is and how the module works (chapter 1), checking your machine (2), the examples file (3), the first training (4), what the library decides (5), choosing settings yourself (6), watching and stopping (7), using the result (8), mistakes (9) and a complete small program (10). The appendices are meant for quick lookups.

## Contents

1. [What fine-tuning is, and how the module works](#1-what-fine-tuning-is-and-how-the-module-works)
2. [Checking your machine](#2-checking-your-machine)
3. [The examples file](#3-the-examples-file)
4. [The first training](#4-the-first-training)
5. [What the library decides](#5-what-the-library-decides)
6. [Choosing settings yourself](#6-choosing-settings-yourself)
7. [Watching and stopping](#7-watching-and-stopping)
8. [Using the result](#8-using-the-result)
9. [Mistakes](#9-mistakes)
10. [The last example](#10-the-last-example)
- [Appendix A. Quick reference](#appendix-a-quick-reference)
- [Appendix B. Failure handling](#appendix-b-failure-handling)
- [Appendix C. Pitfalls, limits and tuning](#appendix-c-pitfalls-limits-and-tuning)
- [Appendix D. Message reference](#appendix-d-message-reference)

Each program stands on its own: each has one `main` method. They use a small examples file `examples.jsonl` with eight typical support questions. (A real file needs many more; chapter 3 explains how many.)

---

## 1. What fine-tuning is, and how the module works

Fine-tuning has its own vocabulary. Here are the words you need, in plain language:

- **Language model.** A program that reads text and predicts what comes next. Chat assistants are language models. It "knows" things because it has seen a huge amount of text.
- **Weights (parameters).** The numbers inside the model that hold what it learned. A "1.5B" model has about 1.5 billion of them. More weights usually mean a smarter model, but also a bigger file and more memory.
- **Training.** Changing the weights so that the model gets better at some text. Training a model from nothing costs a fortune. Nobody does it for a small project.
- **Base model.** A model that was already trained by someone else. Qwen and Gemma are examples. You start from it.
- **Fine-tuning.** Continuing the training of a base model on your own small set of examples. It does not teach the model a new language. It teaches your style, your format and your facts.
- **Hugging Face.** A website where people share models. A name such as `Qwen/Qwen2.5-1.5B` means "the model Qwen2.5-1.5B of the owner Qwen". The library downloads the model by this name the first time.
- **LoRA and adapter.** Changing all weights needs a lot of memory. LoRA is a trick: leave the base model untouched and train a small add-on called an **adapter**. It is only a few megabytes. Using the model later means loading the base model *and* the adapter.
- **QLoRA.** LoRA with one more trick: the base model is stored compressed (4 bits per weight) while training. It needs much less memory and the result is almost as good.
- **Graphics card, video memory.** Training is mostly arithmetic, and graphics cards do it far faster than processors. Their own memory is called video memory (VRAM). If the model does not fit into it, training fails. Only NVIDIA cards are supported.
- **GGUF.** A single-file format that many programs for running models on a normal computer understand, for example llama.cpp, Ollama and LM Studio.
- **Quantization.** Storing each number with fewer bits. The file becomes much smaller and the model a little less precise. `Q4_K_M` is the common balance.

Fine-tuning has a place and a limit. It is good for **style** ("answer short and politely"), **format** ("always answer with JSON") and **a small amount of stable facts** (the opening hours). It is a poor way to store information that changes every day, such as a price list. For that, give the facts to the model at question time.

The module runs these steps in order:

1. Prepare the private Python environment (only the first time).
2. Check the examples file, line by line.
3. Look at the graphics card and at the model.
4. Choose the settings that you did not choose.
5. Train and save the adapter.
6. If you asked for it, convert the result to GGUF.

The classes you will use:

| Class | Role |
|-------|------|
| `JFineTune` | The entry point: `create()`, `environment()`, `probeHardware()` |
| `FineTuneBuilder` | Describes one training: model, examples and optional settings |
| `FineTuneJob` | A training that runs in the background: wait, cancel |
| `FineTuneResult` | The result: adapter folder, optional GGUF file, merge and export |
| `FineTuneEnvironmentBuilder`, `FineTuneEnvironment` | The private Python environment |
| `HardwareReport` | What is known about the graphics card and memory |
| `ProgressEvent` | One progress report of the training |
| `DatasetFormat`, `TrainingMethod`, `Precision`, `GgufQuantization` | Choices as enums |
| `FineTuneException` and its subclasses | The failures; all unchecked |

You need Python 3.10 or newer, an internet connection for the first run (to install PyTorch, which is several gigabytes, and to download the model) and enough free disk space. An NVIDIA graphics card is strongly recommended. Training on the processor works, but only for small models, and it is slow.

Three things shape what follows:

- **The first run is long.** Installing the environment and downloading the model can take a long time. After that, both are kept.
- **Your examples are everything.** The model becomes as good as the examples are. Few, sloppy or contradicting examples give a poor model.
- **The result is an adapter, not a model.** To use it you need the same base model next to it, unless you merge or export (chapter 8).

## 2. Checking your machine

Before training, find out what you have. `JFineTune.probeHardware()` asks the NVIDIA driver about the graphics card. It does not need Python and downloads nothing, so it is instant:

```java
import org.j2os.platform.ai.dl.language.finetune.HardwareReport;
import org.j2os.platform.ai.dl.language.finetune.JFineTune;

public class F01Hardware {
    public static void main(String[] args) {
        HardwareReport report = JFineTune.probeHardware();
        System.out.println("Graphics card found: " + report.graphicsCardDetected());
        System.out.println("Name: " + report.graphicsCardName());
        System.out.println("Video memory (MB): " + report.videoMemoryInMegabytes());
    }
}
```

On a machine with a GeForce RTX 3060 it prints:

```text
Graphics card found: true
Name: NVIDIA GeForce RTX 3060
Video memory (MB): 12288
```

The returned `HardwareReport` has more fields: free video memory, the compute capability (a version number of the card's abilities), the driver version, the installed RAM and a `problem` text. On a machine without an NVIDIA card, `graphicsCardDetected()` is `false`, the name is empty and the memory is 0. Training is then done on the processor.

A rough guide for sizes. These are not promises, because the length of your examples matters too:

| Video memory | What is realistic |
|--------------|-------------------|
| No NVIDIA card | Models under 1 billion weights, slowly |
| 6 GB | Models up to about 1.5 billion weights, with QLoRA |
| 12 GB | Models up to about 3 to 7 billion weights, with QLoRA |
| 24 GB and more | Larger models, with LoRA or QLoRA |

## 3. The examples file

The examples file is a text file in the **JSON Lines** format (extension `.jsonl`): one example per line, and each line is a small JSON object. It must be saved as UTF-8. Persian and any other language are fine.

There are four layouts. The library finds out which one you use by looking at the first line; you can also tell it with `datasetFormat(...)`.

**Question and answer** (`PROMPT_COMPLETION`). The simplest form, and the one used in this guide:

```json
{"prompt": "What are your opening hours?", "completion": "We are open every day from 9 am to 8 pm."}
{"prompt": "Do you deliver?", "completion": "We deliver inside the city for 3 dollars."}
```

**Instruction style** (`ALPACA`). The fields are `instruction`, an optional `input`, and `output`. The library joins instruction and input for you:

```json
{"instruction": "Translate to French", "input": "Good morning", "output": "Bonjour"}
{"instruction": "Say hello", "output": "Hello!"}
```

**Conversations** (`CHAT`). A list of messages. The roles are `system` (optional, first), `user` and `assistant`, and they must alternate and end with an assistant message. The ShareGPT names (`human`, `gpt`, `from`, `value`) are converted automatically:

```json
{"messages": [{"role": "system", "content": "You are a helpful assistant."}, {"role": "user", "content": "When do you open?"}, {"role": "assistant", "content": "Every day at 9 am."}]}
{"messages": [{"role": "user", "content": "Do you deliver?"}, {"role": "assistant", "content": "Yes, inside the city."}]}
```

**Plain text** (`TEXT`). One field, `text`. The model learns to continue this kind of text:

```json
{"text": "We open at nine in the morning and close at eight in the evening."}
```

Rules that apply to every layout:

- **One layout per file.** A line that does not match the earlier ones is refused.
- **No empty values.** An empty question or answer is refused.
- **Blank lines are skipped.** They are not counted.
- **CHAT needs a chat model.** The model must know how to format a conversation (it has a "chat template"). Use an "instruct" or "chat" version of the model, or choose another layout.

The whole file is checked before the real work starts, and an error always names the line number. The program below feeds four broken files to the library on purpose:

```java
import java.nio.file.Path;
import org.j2os.platform.ai.dl.language.finetune.JFineTune;
import org.j2os.platform.ai.dl.language.finetune.exception.DatasetFormatException;

public class F07BadFiles {
    public static void main(String[] args) {
        for (String file : new String[]{"bad_line", "mixed", "bad_chat", "empty"}) {
            try {
                JFineTune.create().baseModel("Qwen/Qwen2.5-1.5B").dataset(Path.of(file + ".jsonl")).run();
            } catch (DatasetFormatException problem) {
                System.out.println(file + ".jsonl\n  " + problem.getMessage());
            }
        }
    }
}
```

```text
bad_line.jsonl
  Line 2 is not valid JSON. Invalid JSON at column 29: expected ',' or '}' in an object
mixed.jsonl
  Line 2 looks like TEXT but the earlier lines are PROMPT_COMPLETION; one file must use a single format.
bad_chat.jsonl
  Line 1: a conversation needs pairs of user and assistant messages (after an optional system message) and must end with an assistant message.
empty.jsonl
  The dataset /home/user/work/empty.jsonl contains no samples.
```

The four mistakes are a missing comma on line 2, two layouts in one file, a conversation that has no answer, and an empty file. Fix the file and run again.

How many examples? Eight, as in this guide, are enough to test that everything works, but not to learn anything reliable. For a real result, plan **hundreds**, and for a broader skill thousands. Quality matters more than quantity: examples that are correct, consistent and written the way you want the answers to sound.

## 4. The first training

Now the real thing. Only the model and the file are given:

```java
import java.nio.file.Path;
import org.j2os.platform.ai.dl.language.finetune.*;
import org.j2os.platform.ai.dl.language.finetune.exception.FineTuneException;

public class F02FirstRun {
    public static void main(String[] args) {
        try {
            FineTuneResult result = JFineTune.create()
                    .baseModel("Qwen/Qwen2.5-1.5B")
                    .dataset(Path.of("examples.jsonl"))
                    .onLog(System.out::println)
                    .run();
            System.out.println("Adapter: " + result.adapterDirectory());
        } catch (FineTuneException problem) {
            System.err.println("Fine-tuning failed: " + problem.getMessage());
        }
    }
}
```

The program prints, on a machine whose environment is already prepared:

```text
Dataset: 8 samples, format PROMPT_COMPLETION
Inspecting Qwen/Qwen2.5-1.5B (this may download the model configuration)
WARNING: the dataset has only 8 samples. That is enough to test the setup, but a model needs hundreds of examples to learn something reliable.
Training plan: method=LORA, precision=BF16, loraRank=16, batchSize=1, gradientAccumulationSteps=1, maximumSequenceLength=96, learningRate=2.0E-4, epochCount=13, device=graphics card, gradientCheckpointing=true
Starting the training
Finished. Adapter: /home/user/work/finetune-Qwen2.5-1.5B/adapter
Adapter: /home/user/work/finetune-Qwen2.5-1.5B/adapter
```

Read it line by line:

1. *Dataset: 8 samples, format PROMPT_COMPLETION* — the file was checked and the layout detected.
2. *Inspecting ...* — the library looks at the model (its size, the length of your examples). It may download the model's small description files.
3. *WARNING: the dataset has only 8 samples* — the library tells you that this is a test, not a real training.
4. *Training plan: ...* — the settings that were chosen (chapter 5 explains them).
5. *Starting the training* — the real work, which takes most of the time.
6. *Finished. Adapter: ...* — the adapter is saved in that folder.

The environment preparation is the part you did not see. On the very first run of a machine, `onLog` shows it too:

```text
Preparing the python environment in /home/user/.j2os/finetune-env
Found python 3.11.4
Creating the virtual environment
Found graphics card NVIDIA GeForce RTX 3060 (compute capability 8.6, driver 550.54)
Installing PyTorch (...) from ... - this downloads several gigabytes
Installing the training packages
PyTorch 2.5.1+cu121 is installed.
```

This is the long part: PyTorch is several gigabytes. It happens once, and the environment is kept in a folder (by default `.j2os/finetune-env` in the home folder, see chapter 6).

`run()` works on the calling thread and returns when everything is finished. If you give no `outputDirectory`, the results go to a folder named `finetune-<model name>` next to your examples file. With `outputDirectory(Path.of("my-model"))` the folder looks like this:

```text
my-model/
├── adapter/          (the trained adapter: this is the result)
├── my-model.gguf     (only when you asked for GGUF)
└── .work/            (scratch files; removed after a successful run, kept after a failure)
```

If the job fails, the `.work` folder is kept, so you can look at the intermediate files. After a successful job or a cancellation it is deleted.

> **Advice.** Use `onLog` always. A training takes a long time, and the log is the only way to see what it is doing.

## 5. What the library decides

When you do not give a setting, the library chooses it. It does so from three facts: your graphics card, the model and your examples. Here are the settings in plain words:

| Setting | What it means | Automatic choice |
|---------|---------------|------------------|
| `method` | `LORA` or `QLORA` | `LORA` if the model fits comfortably in video memory, otherwise `QLORA`; always `LORA` without a graphics card |
| `precision` | How exactly numbers are stored while training: `BF16`, `FP16` or `FP32` | `BF16` on newer cards (RTX 30 series and later), `FP16` on others, `FP32` for GTX 10 series (where it is faster) and on the processor |
| `loraRank` | The size of the adapter; larger learns more but needs more memory | 16 |
| `batchSize` | Examples handled at the same moment | 1 to 4, from the free video memory |
| `gradientAccumulationSteps` | How many batches are collected before the model is updated | Chosen to reach about 16 examples per update |
| `maximumSequenceLength` | The longest example in *tokens* (pieces of words); longer ones are cut | From the length of your examples, up to the model's limit |
| `learningRate` | How big each update is | 0.0002 |
| `epochCount` | How many times the whole file is used | At least 3; more for small files so that about 100 updates happen |

The same settings once more, in everyday words. Think of a student who prepares for an exam with a pile of practice sheets (your examples):

| Setting | In everyday words | When it matters |
|---------|-------------------|-----------------|
| `epochCount` | How many times the student goes through the whole pile | More if the model has not learned your answers yet; fewer if it repeats them word for word |
| `batchSize` | How many sheets lie on the desk at the same moment | Larger is faster but needs more desk space (video memory); the library uses 1 on small cards |
| `gradientAccumulationSteps` | How many sheets the student solves, noting the mistakes, before learning from all of them at once | Works together with `batchSize`; their product counts, and the library aims at about 16 |
| `learningRate` | The size of each step the model takes when it corrects itself | Smaller if the loss jumps or becomes `NaN`; a very small value makes learning slow |
| `maximumSequenceLength` | The longest sheet the student reads in one piece, counted in tokens | Longer examples are cut; lower it if memory runs out, raise it if your answers are long |
| `loraRank` | The thickness of the notebook that lies next to the textbook (the adapter) | Larger when there is more to learn; 16 is enough for style and short facts |
| `precision` | How many digits the student writes for each number | `FP32` is exact but heavy; `BF16` and `FP16` are lighter; the library picks by card |

The library prints its decision as the *Training plan* line. To see what it would decide on different machines, its planner was asked about the same model and the same 400 examples in six situations. The machines are made up; the answers are the real output of the library's own planner:

```text
== RTX 3060, everything automatic
Training plan: method=LORA, precision=BF16, loraRank=16, batchSize=1, gradientAccumulationSteps=16, maximumSequenceLength=96, learningRate=2.0E-4, epochCount=4, device=graphics card, gradientCheckpointing=true
== GTX 1060, everything automatic
Training plan: method=QLORA, precision=FP32, loraRank=16, batchSize=1, gradientAccumulationSteps=16, maximumSequenceLength=96, learningRate=2.0E-4, epochCount=4, device=graphics card, gradientCheckpointing=true
== GT 1030 with 2 GB, LoRA
InsufficientVideoMemoryException: The model needs about 6840 MB of video memory for its weights and training state with LORA, but only 1900 MB of the graphics card are free. Try TrainingMethod.QLORA, choose a smaller model, or close programs that use the graphics card.
== No graphics card, everything automatic
Training plan: method=LORA, precision=FP32, loraRank=16, batchSize=1, gradientAccumulationSteps=16, maximumSequenceLength=96, learningRate=2.0E-4, epochCount=4, device=processor, gradientCheckpointing=false
== No graphics card, QLORA requested
InvalidConfigurationException: TrainingMethod.QLORA needs a usable NVIDIA graphics card. PyTorch was installed without CUDA support or no NVIDIA graphics card is present.
== GTX 1060, BF16 requested
InvalidConfigurationException: Precision.BF16 needs compute capability 8.0 or newer, but the graphics card NVIDIA GeForce GTX 1060 has 6.1. Use Precision.FP16.
```

What the output teaches:

- On the **RTX 3060**, the library picks `LORA` with `BF16`.
- On the **GTX 1060** (6 GB), it picks `QLORA`, because the model would not fit otherwise, and `FP32`, because half-precision numbers are very slow on this card family.
- With a **2 GB card**, the plain `LORA` request is stopped with an `InsufficientVideoMemoryException`, and the message suggests `QLORA` or a smaller model.
- **Without a graphics card**, the plan uses the processor, `LORA` and `FP32`. Asking for `QLORA` there is refused, because it needs an NVIDIA card.
- `BF16` on a card that cannot do it is refused, and the message names the right choice.

The library also warns, in the log, about two risky situations: fewer than 50 examples, and a `maximumSequenceLength` that is shorter than about 5 percent of your examples (they would be cut, and the model could learn incomplete answers).

Memory numbers in the planner are estimates. A training can still run out of memory; then you get an `InsufficientVideoMemoryException` with advice (a smaller model, `QLORA`, a smaller `batchSize` or a shorter `maximumSequenceLength`).

## 6. Choosing settings yourself

A setting that you give is used **exactly as given**, or the job stops with a clear message. It is never changed silently. Everything else stays automatic. These are all the settings of the builder:

```java
FineTuneJob job = JFineTune.create()
        .baseModel("Qwen/Qwen2.5-1.5B")
        .dataset(Path.of("examples.jsonl"))
        .datasetFormat(DatasetFormat.PROMPT_COMPLETION)  // normally detected from the first line
        .outputDirectory(Path.of("my-model"))
        .method(TrainingMethod.QLORA)                    // LORA or QLORA
        .precision(Precision.FP32)                       // FP16, BF16 or FP32
        .loraRank(16)                                    // capacity of the adapter, 1 to 512
        .batchSize(1)                                    // examples at the same time
        .gradientAccumulationSteps(16)                   // batches collected before one update
        .maximumSequenceLength(512)                      // longest example, in tokens
        .learningRate(2e-4)                              // size of one update
        .epochCount(3)                                   // how many passes over the dataset
        .start();
```

The ranges that are checked at once, when you call the method:

| Method | Allowed values |
|--------|----------------|
| `loraRank` | 1 to 512 |
| `batchSize` | 1 to 4096 |
| `gradientAccumulationSteps` | 1 to 4096 |
| `maximumSequenceLength` | 16 to 1,000,000 |
| `learningRate` | any positive number |
| `epochCount` | 1 to 10,000 |

Some advice for the beginner who wants to change something:

- **More epochs, if the model does not learn your answers yet.** Fewer, if it repeats your examples word for word (it has memorized them).
- **A bigger `loraRank`, if the model must learn a lot.** 16 is fine for style and short facts.
- **A smaller `learningRate`, if the loss jumps or turns into `NaN`.** The job stops with "Training diverged" in that case.
- **Do not change `batchSize` and `gradientAccumulationSteps` casually.** Their product is what counts: the number of examples per update.

Two more options are about the environment and the model:

```java
FineTuneEnvironment environment = JFineTune.environment()
        .directory(Path.of("D:/finetune-env"))          // default: ~/.j2os/finetune-env
        .onLog(System.out::println)
        .prepare();                                     // fast when it is already prepared

FineTuneResult result = JFineTune.create()
        .environment(environment)
        .baseModel("Qwen/Qwen2.5-1.5B")
        .dataset(Path.of("examples.jsonl"))
        .run();
```

- **`environment(...)`.** Chooses the folder of the private Python environment. Preparing an environment that is already prepared is fast. Prepare it first when you want to see the installation separately from the training, or when several jobs share one environment.
- **`huggingFaceToken(token)`.** Some models, such as Gemma, need you to accept a license on Hugging Face and give a token to download them. The token goes only to the job's Python processes and is never written to a file or to the log. Without this call, the `HF_TOKEN` environment variable of your program is used.
- **`baseModel(...)`** can also be the folder of a model that you downloaded earlier. A `.gguf` file is **refused**: GGUF is a compressed format meant for running a model, not for training it. Train the original model and export to GGUF afterwards.

## 7. Watching and stopping

A training can last minutes or hours. Use `start()` instead of `run()`: it begins the training on its own background thread and returns at once, so your program stays free. With `onProgress`, you get one report per update step:

```java
import java.nio.file.Path;
import org.j2os.platform.ai.dl.language.finetune.*;

public class F03Progress {
    public static void main(String[] args) {
        FineTuneJob job = JFineTune.create()
                .baseModel("Qwen/Qwen2.5-1.5B")
                .dataset(Path.of("examples.jsonl"))
                .epochCount(2)
                .onProgress(p -> {
                    if (p.currentStep() % 4 == 0) {
                        System.out.printf("step %d/%d  epoch %d/%d  loss=%.2f%n",
                                p.currentStep(), p.totalSteps(), p.epoch(), p.epochCount(), p.trainingLoss());
                    }
                })
                .start();
        FineTuneResult result = job.await();
        System.out.println("Done: " + result.adapterDirectory().getFileName());
        Path merged = result.mergeIntoBaseModel(Path.of("merged"));
        System.out.println("Merged model: " + merged.getFileName());
    }
}
```

```text
step 4/16  epoch 1/2  loss=1.29
step 8/16  epoch 1/2  loss=0.64
step 12/16  epoch 2/2  loss=0.36
step 16/16  epoch 2/2  loss=0.23
Done: adapter
Merged model: merged
```

The first lines show the progress report: the step, the epoch and the **loss**. The loss is a number that tells how wrong the model still is on your examples. It should fall while the training runs, as it does here. If it stays flat from the start, the model learns nothing; if it falls to almost zero too early, it has memorized the examples. A `ProgressEvent` also has the learning rate and the seconds since the start. The loss can be `NaN` when it is not available for a step.

The last two lines show something new: `mergeIntoBaseModel(folder)` writes a complete model, the base model with the adapter already inside. Chapter 8 explains why you might want that.

A `FineTuneJob` offers:

| Method | Meaning |
|--------|---------|
| `await()` | Waits and returns the `FineTuneResult`, or throws the failure |
| `cancel()` | Asks the job to stop as soon as possible |
| `isRunning()` | `true` until the job has finished, failed or been canceled |
| `isCancelled()` | `true` after `cancel()` was called |

When your program ends, the training is stopped too. Canceling is clean:

```java
import java.nio.file.Path;
import org.j2os.platform.ai.dl.language.finetune.*;
import org.j2os.platform.ai.dl.language.finetune.exception.FineTuneCancelledException;

public class F04Cancel {
    public static void main(String[] args) throws InterruptedException {
        FineTuneJob job = JFineTune.create()
                .baseModel("Qwen/Qwen2.5-1.5B")
                .dataset(Path.of("examples.jsonl"))
                .start();
        Thread.sleep(2000);
        job.cancel();
        try {
            job.await();
        } catch (FineTuneCancelledException cancelled) {
            System.out.println("Cancelled. Was it asked for? " + job.isCancelled());
        }
    }
}
```

```text
Cancelled. Was it asked for? true
```

`await()` throws a `FineTuneCancelledException` after `cancel()`. A button named "Stop training" in an admin page would do exactly this.

Two jobs may exist at the same time, but they compete for the graphics card and the memory. Start them one after another, for example from a single-thread executor. Two jobs that use the same output folder are refused.

## 8. Using the result

A finished job gives a `FineTuneResult` with three useful things:

- **`adapterDirectory()`.** The adapter, a few megabytes. It works only together with the **same** base model. This is the natural result for a program that loads models with the Hugging Face libraries.
- **`mergeIntoBaseModel(folder)`.** Writes a *complete* model into that folder: the base model with the adapter built in. It does not need the adapter any more, but it is as big as the base model.
- **`exportGguf(...)`.** Writes one GGUF file. This is the form for programs such as llama.cpp, Ollama or LM Studio that run models on a normal computer. Internally, the adapter is merged first and then converted.

Asking for GGUF can be done during the training, or later from the result:

```java
FineTuneResult result = JFineTune.create()
        .baseModel("Qwen/Qwen2.5-1.5B")
        .dataset(Path.of("examples.jsonl"))
        .exportGguf(Path.of("my-model/my-model.gguf"), GgufQuantization.Q4_K_M)
        .run();
System.out.println("GGUF: " + result.ggufFile());

// later, from the same result:
result.exportGguf(Path.of("my-model/my-model-q8.gguf"), GgufQuantization.Q8_0);
result.mergeIntoBaseModel(Path.of("my-model/merged"));
```

The log of the first export:

```text
Merging the adapter into the base model
Downloading the llama.cpp conversion script (...)
Installing the packages needed by the GGUF converter
Converting to GGUF (f16)
Quantizing to Q4_K_M
Finished. Adapter: /home/user/work/my-model/adapter
GGUF: /home/user/work/my-model/my-model.gguf
```

The first export needs the internet, because the conversion script of llama.cpp is fetched. The `GgufQuantization` value decides size and quality:

| Value | Bits | Meaning |
|-------|------|---------|
| `F16` | 16 | Best quality, largest file |
| `Q8_0` | 8 | Almost no loss of quality |
| `Q6_K` | 6 | Very good quality |
| `Q5_K_M` | 5 | Good quality |
| `Q4_K_M` | 4 | Small file; the usual choice, and the default |

`F16` and `Q8_0` are written by the converter directly. The other three need the program `llama-quantize`. On Windows, the library downloads it by itself. On other systems, install llama.cpp and put `llama-quantize` on the `PATH`, or use `F16` or `Q8_0`.

If the export fails, you get a `GgufExportException`. The adapter is still saved; only the GGUF file is missing.

## 9. Mistakes

The builder methods check their arguments immediately. The rest is checked when the job runs, in the order of the steps: examples first, then the model and machine. The program below makes three configuration mistakes on purpose. (Broken examples files, which you saw in chapter 3, are reported in the same way.)

```java
import java.nio.file.Path;
import org.j2os.platform.ai.dl.language.finetune.*;
import org.j2os.platform.ai.dl.language.finetune.exception.FineTuneException;

public class F05Mistakes {
    static void attempt(String title, Runnable action) {
        try {
            action.run();
        } catch (FineTuneException problem) {
            System.out.println(title + "\n  " + problem.getClass().getSimpleName() + ": " + problem.getMessage());
        }
    }

    public static void main(String[] args) {
        attempt("GGUF as base model", () -> JFineTune.create().baseModel("tiny.gguf"));
        attempt("LoRA rank 0", () -> JFineTune.create().loraRank(0));
        attempt("Missing dataset file", () -> JFineTune.create().baseModel("Qwen/Qwen2.5-1.5B")
                .dataset(Path.of("nothing.jsonl")).run());
    }
}
```

```text
GGUF as base model
  InvalidConfigurationException: A GGUF file cannot be fine-tuned because GGUF stores compressed weights for running models. Give the original Hugging Face model as baseModel, train it, and call exportGguf() to get a GGUF file afterwards.
LoRA rank 0
  InvalidConfigurationException: loraRank must be between 1 and 512.
Missing dataset file
  InvalidConfigurationException: The dataset file does not exist: nothing.jsonl
```

Every failure is a subclass of the unchecked `FineTuneException`. You can catch the base class, or the cases you want to treat in a special way:

| Exception | Meaning |
|-----------|---------|
| `InvalidConfigurationException` | A setting is wrong or impossible on this machine |
| `DatasetFormatException` | The examples file is wrong; the message has the line number |
| `PythonNotFoundException` | The `python` command is missing or does not work |
| `UnsupportedPythonVersionException` | Python is older than 3.10 |
| `EnvironmentPreparationException` | The environment could not be prepared (installation, old driver) |
| `InsufficientVideoMemoryException` | The model does not fit into the video memory |
| `InsufficientSystemMemoryException` | The model does not fit into RAM (processor training) |
| `TrainingFailedException` | The training stopped with an error or diverged |
| `GgufExportException` | The GGUF conversion failed; the adapter is saved |
| `FineTuneCancelledException` | The job was canceled |
| `ProgramNotStartedException` | The operating system cannot start an external program |

## 10. The last example

The task: train a model on `examples.jsonl`. The program checks the machine, trains with progress, and reports the folder of the adapter. It handles a bad examples file separately from other failures:

```java
import java.nio.file.Path;
import org.j2os.platform.ai.dl.language.finetune.*;
import org.j2os.platform.ai.dl.language.finetune.exception.DatasetFormatException;
import org.j2os.platform.ai.dl.language.finetune.exception.FineTuneCancelledException;
import org.j2os.platform.ai.dl.language.finetune.exception.FineTuneException;

public class F06Complete {
    public static void main(String[] args) {
        HardwareReport hardware = JFineTune.probeHardware();
        System.out.println(hardware.graphicsCardDetected()
                ? "Training on " + hardware.graphicsCardName()
                : "No NVIDIA card found. Training on the processor will be slow.");
        try {
            FineTuneResult result = JFineTune.create()
                    .baseModel("Qwen/Qwen2.5-1.5B")
                    .dataset(Path.of("examples.jsonl"))
                    .outputDirectory(Path.of("my-model"))
                    .onLog(line -> System.out.println("[log] " + line))
                    .onProgress(p -> {
                        if (p.currentStep() % 40 == 0 || p.currentStep() == p.totalSteps()) {
                            System.out.printf("step %d/%d  loss=%.2f%n", p.currentStep(), p.totalSteps(), p.trainingLoss());
                        }
                    })
                    .run();
            System.out.println("Your adapter is in: " + result.adapterDirectory());
        } catch (DatasetFormatException problem) {
            System.err.println("Fix the dataset first: " + problem.getMessage());
        } catch (FineTuneCancelledException cancelled) {
            System.err.println("Stopped by the user.");
        } catch (FineTuneException problem) {
            System.err.println("Fine-tuning failed: " + problem.getMessage());
        }
    }
}
```

```text
Training on NVIDIA GeForce RTX 3060
[log] Dataset: 8 samples, format PROMPT_COMPLETION
[log] Inspecting Qwen/Qwen2.5-1.5B (this may download the model configuration)
[log] WARNING: the dataset has only 8 samples. That is enough to test the setup, but a model needs hundreds of examples to learn something reliable.
[log] Training plan: method=LORA, precision=BF16, loraRank=16, batchSize=1, gradientAccumulationSteps=1, maximumSequenceLength=96, learningRate=2.0E-4, epochCount=13, device=graphics card, gradientCheckpointing=true
[log] Starting the training
step 40/104  loss=0.88
step 80/104  loss=0.34
step 104/104  loss=0.23
[log] Finished. Adapter: /home/user/work/my-model/adapter
Your adapter is in: /home/user/work/my-model/adapter
```

The adapter in `my-model/adapter` now carries what your examples taught it. The usual routine is this: test the model with questions that are *not* in the file, add examples where it answers wrongly, and train again. That is how fine-tuning is done in practice: small rounds, each with better examples.

The core of it, in code:

- Collect good examples in a JSON Lines file.
- `JFineTune.create().baseModel(...).dataset(...).run()` is the whole call; the rest is automatic.
- Use `onLog` and `onProgress`; use `start()` when you do not want to wait.
- The result is an adapter; merge it or export it to GGUF when you need a standalone model.

## Appendix A. Quick reference

### A.1 The entry point

| Call | Meaning |
|------|---------|
| `JFineTune.create()` | Starts the description of a training |
| `JFineTune.environment()` | Starts the description of the Python environment |
| `JFineTune.probeHardware()` | Reports the graphics card and memory from the NVIDIA driver; no Python needed |

### A.2 The training builder (`FineTuneBuilder`)

| Call | Meaning |
|------|---------|
| `baseModel(name)` | Required. A Hugging Face name or a model folder; not `.gguf` |
| `dataset(path)` | Required. The `.jsonl` file |
| `datasetFormat(format)` | The layout; detected when omitted |
| `outputDirectory(path)` | Result folder; default `finetune-<model>` next to the examples |
| `environment(env)` | A prepared environment |
| `method(m)` | `LORA` or `QLORA` |
| `precision(p)` | `FP16`, `BF16` or `FP32` |
| `loraRank(n)` | 1 to 512 |
| `batchSize(n)` | 1 to 4096 |
| `gradientAccumulationSteps(n)` | 1 to 4096 |
| `maximumSequenceLength(n)` | 16 to 1,000,000 tokens |
| `learningRate(x)` | A positive number |
| `epochCount(n)` | 1 to 10,000 |
| `exportGguf()` | Also writes a GGUF with `Q4_K_M` |
| `exportGguf(quantization)` | Also writes a GGUF with this quantization |
| `exportGguf(path)` | Also writes a GGUF to this path with `Q4_K_M` |
| `exportGguf(path, quantization)` | Both |
| `huggingFaceToken(token)` | Token for restricted models; `HF_TOKEN` is used otherwise |
| `onLog(consumer)` | Receives every log line |
| `onProgress(consumer)` | Receives a `ProgressEvent` per update step |
| `run()` | Trains on the calling thread and returns the result |
| `start()` | Trains in the background and returns a `FineTuneJob` |

### A.3 Environment, job and result

| Call | Meaning |
|------|---------|
| `environment().directory(path)` | The environment folder; default `.j2os/finetune-env` in the home folder |
| `environment().onLog(consumer)` | Receives the installation output |
| `environment().prepare()` | Creates or checks the environment and returns it |
| `FineTuneEnvironment.directory()` | The environment folder |
| `FineTuneEnvironment.pythonExecutable()` | The Python of the environment |
| `FineTuneEnvironment.scriptsDirectory()` | Where the library's scripts are written |
| `job.await()` / `cancel()` / `isRunning()` / `isCancelled()` | Wait, stop, ask |
| `result.adapterDirectory()` | The adapter folder |
| `result.ggufFile()` | The GGUF file, or `null` |
| `result.mergeIntoBaseModel(folder)` | Writes a full merged model |
| `result.exportGguf(file)` | GGUF with `Q4_K_M` |
| `result.exportGguf(file, quantization)` | GGUF with the quantization |

### A.4 The enums and records

| Type | Values or fields |
|------|------------------|
| `DatasetFormat` | `PROMPT_COMPLETION`, `ALPACA`, `CHAT`, `TEXT` |
| `TrainingMethod` | `LORA`, `QLORA` |
| `Precision` | `FP16`, `BF16`, `FP32` |
| `GgufQuantization` | `F16`, `Q8_0`, `Q6_K`, `Q5_K_M`, `Q4_K_M` |
| `ProgressEvent` | `currentStep`, `totalSteps`, `epoch`, `epochCount`, `trainingLoss`, `learningRate`, `elapsedSeconds` |
| `HardwareReport` | `graphicsCardDetected`, `graphicsCardName`, `videoMemoryInMegabytes`, `freeVideoMemoryInMegabytes`, `computeCapability`, `driverVersion`, `systemMemoryInMegabytes`, `verifiedByTorch`, `torchVersion`, `graphicsCardUsableByTorch`, `fourBitQuantizationSupported`, `problem` |

## Appendix B. Failure handling

| Situation | What happens |
|-----------|--------------|
| A builder argument is out of range or `null` | `InvalidConfigurationException` at once |
| `baseModel` or `dataset` is missing, or the file does not exist | `InvalidConfigurationException` when the job is built |
| Two jobs use the same output folder | The second gets `InvalidConfigurationException` |
| Python is missing or too old | `PythonNotFoundException` or `UnsupportedPythonVersionException` |
| The environment cannot be prepared | `EnvironmentPreparationException` |
| An examples line is wrong, mixed or empty | `DatasetFormatException` with the line number |
| `CHAT` layout with a model that has no chat template | `DatasetFormatException` |
| The model cannot be inspected (name, internet, login) | `FineTuneException` with details |
| The model does not fit into video memory | `InsufficientVideoMemoryException` before or during training |
| The model does not fit into RAM | `InsufficientSystemMemoryException` |
| The loss becomes invalid | `TrainingFailedException` ("Training diverged") |
| Any other training error | `TrainingFailedException` with the exit code |
| GGUF conversion fails | `GgufExportException`; the adapter stays |
| `cancel()` | `FineTuneCancelledException` from `await()`; scratch files are removed |
| Any other failure | The `.work` folder is kept for inspection |

## Appendix C. Pitfalls, limits and tuning

### C.1 Pitfalls

- **Too few or poor examples.** The model learns what you show, including your mistakes. Check the examples before you blame the settings.
- **Expecting new knowledge from a few lines.** Fine-tuning shapes style and format. For changing facts, give the facts at question time.
- **Training a `.gguf` file.** Refused. Train the original model.
- **Using the adapter with another base model.** It works only with the model it was trained from.
- **Testing with the training questions.** The model will answer them well because it saw them. Test with new questions.
- **Starting two trainings on one graphics card.** They slow each other down or run out of memory.
- **Choosing `FP16` on a GTX 10 series card.** It is about 64 times slower than `FP32`; the log warns about it.
- **Closing the program and expecting the job to continue.** The training stops when the program ends.

### C.2 Behavior worth knowing

- The environment is prepared once and kept. It carries a version number: when a new version of the library needs different packages, the environment is rebuilt automatically.
- The library checks the whole examples file before it touches the graphics card.
- The random seed is always 42, so the same file and settings train in nearly the same way each time.
- Automatic settings come from the free video memory at that moment; close other programs that use the card before you start.
- Memory use is estimated, not measured; out-of-memory errors during training are reported with advice.

### C.3 Limits

- Only NVIDIA cards are used for training; other cards mean training on the processor.
- QLoRA needs an NVIDIA card and the 4-bit library; the hardware check reports when it is not available.
- BF16 needs compute capability 8.0 or higher (RTX 30 series and later).
- Python 3.10 or newer; NVIDIA driver 525 or newer for the CUDA builds of PyTorch.
- `Q6_K`, `Q5_K_M` and `Q4_K_M` need `llama-quantize`, which is downloaded automatically on Windows only.
- The first run and the first GGUF export need the internet.

### C.4 Tuning and practical advice

- Begin with the automatic settings and a small model; look at the log and the loss before you change anything.
- Put the environment folder on a disk with plenty of free space.
- Use `start()` in server applications, and keep a reference to the job so that a "Stop" button can call `cancel()`.
- Increase `epochCount` before you increase `loraRank`.
- Save every good adapter under a name that tells the base model and the examples file it came from.

## Appendix D. Message reference

### D.1 Configuration

| Message | Cause |
|---------|-------|
| `baseModel must not be empty.` | The name is `null` or blank |
| `A GGUF file cannot be fine-tuned because GGUF stores compressed weights for running models. ...` | A `.gguf` file was given as the base model |
| `dataset must not be null.` | No dataset |
| `baseModel is required.` / `dataset is required.` | The training was started without them |
| `The dataset file does not exist: <path>` | Wrong path |
| `<name> must be between A and B.` | An integer setting is out of range |
| `learningRate must be a positive number.` | Zero, negative or infinite |
| `Another fine-tuning job is already writing to <folder>. Give every job its own outputDirectory.` | Two jobs, one folder |
| `TrainingMethod.QLORA needs a usable NVIDIA graphics card. ...` | QLoRA without a usable card |
| `TrainingMethod.QLORA is not available on this machine. ...` | The 4-bit library does not work here |
| `Precision <P> needs a usable NVIDIA graphics card; training on the processor supports only Precision.FP32. ...` | Half precision on the processor |
| `Precision.BF16 needs compute capability 8.0 or newer, but the graphics card <name> has <version>. Use Precision.FP16.` | BF16 on an older card |

### D.2 Examples file

| Message | Cause |
|---------|-------|
| `Line N is not valid JSON. ...` | The line is not JSON |
| `Line N must be a JSON object.` | The line is a list, number or text |
| `Line N has none of the supported field sets: prompt+completion, instruction+output(+input), messages (or conversations), text.` | No known layout |
| `Line N looks like X but the earlier lines are Y; one file must use a single format.` | Two layouts in one file |
| `Line N: "<field>" must be a non-empty string.` | A field is missing or empty |
| `Line N: "input" must be a string.` | The optional `input` is not text |
| `Line N: "messages" must be a list.` | The conversation is not a list |
| `Line N, message M must be an object.` | A message is not an object |
| `Line N, message M: the role must be system, user or assistant (ShareGPT: human or gpt).` | Unknown role |
| `Line N, message M needs non-empty text in "content" (or "value").` | An empty message |
| `Line N: a conversation needs pairs of user and assistant messages (after an optional system message) and must end with an assistant message.` | A broken conversation |
| `Line N, message M: expected a <role> message here; roles must alternate user, assistant.` | The roles do not alternate |
| `The dataset is not valid UTF-8 text (problem near line N). Save the file as UTF-8.` | Wrong encoding |
| `The dataset <path> contains no samples.` | Empty file |
| `The dataset uses the CHAT format but the tokenizer of <model> has no chat template. ...` | CHAT with a model that is not a chat model |

### D.3 Environment and machine

| Message | Cause |
|---------|-------|
| `The 'python' command was not found. Install Python 3.10 or newer and make sure that typing 'python --version' works in a terminal.` | Python is missing |
| `The 'python' command did not report a version. On Windows this happens when only the Microsoft Store shortcut exists. ...` | A fake Python shortcut |
| `Python X.Y is too old; version 3.10 or newer is required.` | An old Python |
| `The NVIDIA driver <version> is too old for the CUDA builds of PyTorch. Update the driver to version 525 or newer.` | An old driver |
| `Creating the virtual environment failed (exit code N). On Debian or Ubuntu the package python3-venv may be missing. Output: ...` | `venv` failed |
| `PyTorch could not be installed. Last output: ...` | No PyTorch source worked |
| `PyTorch was installed but cannot be imported: ...` | A broken PyTorch |
| `Installing the training packages failed (exit code N): ...` | `pip` failed |
| `The model needs about N MB of video memory ... but only M MB of the graphics card are free. ...` | The model does not fit |
| `Training on the processor needs about N MB of memory but the machine has M MB. Choose a smaller model.` | Not enough RAM |
| `The model <name> could not be inspected. Check the model name, the internet connection and whether the model needs a Hugging Face login. Details: ...` | Wrong name, no internet or restricted model |

### D.4 Training and export

| Message | Cause |
|---------|-------|
| `The graphics card ran out of memory during training (...). Use a smaller model, TrainingMethod.QLORA, a smaller batchSize or a shorter maximumSequenceLength.` | Out of video memory |
| `The machine ran out of memory during training (...). ...` | Out of RAM |
| `Training diverged: ... Try a lower learningRate or Precision.FP32.` | Invalid loss values |
| `Training failed (exit code N): ...` | Another training error |
| `Merging the adapter into the base model failed: ...` | The merge failed |
| `The adapter directory does not exist: <path>` | A wrong adapter folder |
| `GGUF export failed: ...` | The conversion failed |
| `This quantization needs the llama-quantize program. Automatic download is available on Windows only; ...` | Q4/Q5/Q6 on another system |
| `No prebuilt llama-quantize for Windows was found in the recent llama.cpp releases ...` | No download available |
| `The conversion finished but <file> was not created.` | The output is missing |

### D.5 Log lines

| Line | Meaning |
|------|---------|
| `Preparing the python environment in <folder>` | Start of the environment setup |
| `Found python X.Y.Z` | The detected Python |
| `Found graphics card <name> (compute capability ..., driver ...)` | A card was found |
| `No NVIDIA graphics card found; training will run on the processor.` | No card |
| `Installing PyTorch (...) - this downloads several gigabytes` | The long download |
| `PyTorch <version> is installed.` | PyTorch is ready |
| `Dataset: N samples, format F` | The examples were checked |
| `Inspecting <model> (this may download the model configuration)` | The model is examined |
| `Training plan: ...` | The settings in use |
| `WARNING: ...` | A risk, with advice |
| `Starting the training` | The training begins |
| `Finished. Adapter: <folder>` | The adapter was saved |
