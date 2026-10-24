+++
title = 'How LLMs Actually Work: From Internet Text to Reasoning Engine: '
date = 2026-10-24T12:28:09-07:00
+++
An LLM is best understood as a very large Transformer neural network trained to predict the next token in a sequence.
## 1. The central idea

That simple training objective is used in several increasingly sophisticated stages:

1. **Pre-training:** learn statistical patterns from a huge, filtered collection of internet text.
2. **Supervised fine-tuning (SFT):** train on examples of conversations so the base model behaves like an assistant.
3. **Reinforcement learning (RL):** let the model try many solutions, score the outcomes, and reinforce sequences that work well—especially on tasks with verifiable answers.
4. **Inference:** freeze the learned parameters and repeatedly generate one token at a time from a prompt and its accumulated context.

The most important mental model is that ChatGPT is not a tiny person living behind a text box. It is a learned simulation of patterns in data, executed as a finite mathematical computation over tokens. It can be extraordinarily capable, but its capabilities are uneven and its output must be checked.

## 2. The entire pipeline at a glance

```text
Internet documents
        ↓
Filtering, cleaning, deduplication, PII reduction
        ↓
Tokenization into a one-dimensional token stream
        ↓
Transformer pre-training: predict the next token
        ↓
Base model: an internet-text/token simulator
        ↓
Conversation data and special turn markers
        ↓
Supervised fine-tuning: imitate good assistant behavior
        ↓
Reinforcement learning: discover solution strategies
        ↓
Deployed model + prompt/context + optional tools
        ↓
Autoregressive token generation
```

A useful analogy is education:

| LLM stage | Education analogy | What is learned |
|---|---|---|
| Pre-training | Reading exposition | Broad background knowledge and patterns |
| SFT | Studying worked examples | How an expert or assistant responds |
| RL | Solving practice problems | Which strategies reliably produce good outcomes |

## 3. Pre-training data: turning the internet into training material

### 3.1 The goal

The first stage starts with a very large collection of text. The goal is not merely to collect as much text as possible. The dataset should be:

- **Large:** enough examples for the model to learn broad patterns.
- **High quality:** useful documents are more valuable than noise.
- **Diverse:** the model should encounter many subjects, styles, languages, and domains.

Hugging Face’s FineWeb dataset is a good example of what a production-scale pre-training dataset looks like. It begins from large web crawls such as Common Crawl and ends with a much smaller, heavily processed representation of the web.

### 3.2 Why raw web data is not ready

A web crawl contains navigation menus, duplicated pages, spam, advertisements, malicious content, boilerplate, poor-quality text, and personally identifying information. Building a useful dataset therefore requires a substantial data-engineering pipeline.

1. **URL/domain filtering:** remove known undesirable sources and categories.
2. **Content extraction:** keep the meaningful article or page text rather than the surrounding web-page structure.
3. **Language filtering:** decide which languages to retain and in what proportions.
4. **Quality filtering:** remove low-value or noisy text.
5. **Deduplication:** reduce repeated copies of the same content.
6. **PII reduction/removal:** try to detect and remove information such as addresses or government identification numbers.

### 3.3 Scale is large, but not infinite

FineWeb, for instance, contains roughly 44 TB of filtered text and about 15 trillion tokens. The point is not the exact number; it is that the final training corpus is a carefully selected and transformed subset of the internet, not the entire raw web.

### 3.4 Important lesson: data quality is part of model quality

Model behavior is strongly shaped by the distribution of the training data. If a language or domain is underrepresented, the model will generally be weaker there. If a source is repeatedly included, the model may memorize and later reproduce parts of it. If low-quality patterns dominate, the model can learn those patterns too.

## 4. Tokenization: the model does not see words the way we do

### 4.1 The model needs a finite alphabet

Neural networks operate on numerical sequences. Text must therefore be converted into a one-dimensional sequence of discrete IDs.

At the lowest level, text can be represented as bits, but bit-level sequences are extremely long. A longer sequence consumes more context and computation. The system therefore trades off:

- **Vocabulary size:** how many distinct symbols are available.
- **Sequence length:** how many symbols are needed to represent the text.

### 4.2 From bytes to tokens

The intuition builds in stages:

- Bits use only two symbols but produce very long sequences.
- Grouping eight bits gives a byte, with 256 possible values.
- Byte Pair Encoding (BPE) repeatedly merges common adjacent byte/symbol pairs into new symbols.
- The resulting symbols are called **tokens**.

Modern tokenizers often have vocabularies on the order of 100,000 symbols. A token may be a whole word, part of a word, punctuation, whitespace plus a word fragment, or a less intuitive chunk.

### 4.3 Tokenization is not characterization

Tokenization changes with:

- capitalization;
- spaces and the number of spaces;
- punctuation;
- the exact spelling of a string;
- the language and text distribution used to train the tokenizer.

For example, using a GPT-style tokenizer:

```text
"hello world"   → [15339, 1917]            (2 tokens)
"Hello World"   → [9906, 4435]             (2 tokens, completely different IDs)
"hello  world"  → [15339, 220, 1917]       (3 tokens — the extra space becomes its own token)
"你好世界"       → [57668, 16325, 98185]     (3 tokens for 4 Chinese characters)
```

The same meaning can map to entirely different token sequences depending on casing, spacing, and language. This is not a minor detail—it directly shapes what the model "sees."

### 4.4 Why tokenization matters

Tokenization affects:

- context-window usage;
- cost and latency;
- how much text fits in a prompt;
- the model’s ability to manipulate characters, count, or spell;
- how efficiently different languages are represented.

The model’s native world is a sequence of token IDs, not a sequence of human-visible characters. This explains several later weaknesses.

## 5. Neural-network input/output and next-token training

### 5.1 Input

The input is a window of tokens from the training stream. The window can be short or can extend up to the model’s maximum context length.

For a toy example, suppose the context is:

```text
token_1 token_2 token_3 token_4
```

The training data tells us which token comes next. That next token is the label the model should learn to predict.

### 5.2 Output

If the vocabulary contains approximately 100,000 tokens, the model produces approximately 100,000 output scores for the next position. After normalization, those scores form a probability distribution over possible next tokens.

The model is not directly outputting a sentence. It is outputting a distribution such as:

```text
token A: 0.04
token B: 0.02
token C: 0.03
...
```

The correct next token from the dataset provides the learning signal.

### 5.3 Training update

The model begins with randomly initialized parameters, so its predictions are effectively random. Training repeatedly nudges the parameters so that the correct next token receives higher probability.

The basic loop is:

1. Sample a batch of token windows.
2. Run the neural network forward.
3. Compare predicted probabilities with the known next tokens.
4. Compute a loss.
5. Use backpropagation and an optimizer to update the parameters.
6. Repeat over many batches and many training steps.

Because many positions and examples can be processed in parallel, training uses large batches and highly parallel hardware.

### 5.4 Parameters as learned “knobs”

The parameters are the numerical settings that determine how the network transforms inputs into predictions. Think of them as an enormous collection of knobs: training searches for a setting that makes the network’s predictions statistically consistent with the training corpus.

The parameters are also a form of **lossy compression** of the training data. They do not contain a clean database of facts. They contain distributed statistical patterns that can sometimes reproduce, generalize, or distort what appeared in the data.

## 6. Transformer internals: a high-level picture

The important conceptual picture is:

1. Token IDs are converted to vector representations called **embeddings**.
2. Those vectors pass through repeated Transformer blocks.
3. The blocks contain operations such as layer normalization, matrix multiplications, attention, and multilayer perceptrons.
4. The final representation is converted into logits and probabilities over the vocabulary.

The network is a fixed mathematical function once its parameters are fixed. It has no biological-neuron-like persistent memory inside a single forward pass. The changing information at inference time is primarily the token sequence in the context window.

### 6.1 Attention and MLP blocks

At a high level:

- **Attention** lets a position use information from relevant earlier tokens in the context.
- **MLP blocks** apply learned nonlinear transformations to the representations.
- Repeating these operations many times allows the network to build increasingly rich internal features before producing the next-token distribution.

The core point is not that each operation is mysterious. The surprising behavior comes from the scale, composition, data, and optimization of many simple mathematical operations.

## 7. Inference: generating one token at a time

Training changes the parameters. **Inference** uses the resulting fixed parameters to generate new text.

The loop is:

```text
Start with prompt tokens
        ↓
Run the Transformer
        ↓
Get probabilities for the next token
        ↓
Sample or select one token
        ↓
Append it to the context
        ↓
Repeat
```

This is autoregressive generation. The next token becomes part of the input used to choose the following token.

### 7.1 Why outputs vary

The model generally samples from a probability distribution rather than always selecting a single deterministic token. Different samples can lead to different continuations. Once an early token differs, the rest of the generated sequence can move into a completely different region.

The result is usually not a verbatim copy of a training document. It is a statistically related remix—although repeated or highly overrepresented material can sometimes be reproduced surprisingly closely.

### 7.2 Inference is not normally additional training

When using a deployed model, its parameters are generally fixed. The model is executing inference over the current prompt and context. It can adapt its answer to examples in the context, but that is not the same as permanently updating its weights.

## 8. GPT-2 as a concrete example

GPT-2  contains the recognizable structure of modern LLMs while being old and accessible enough to study.

| Example | Parameters | Context length | Training tokens |
|---|---:|---:|---:|
| GPT-2 | about 1.5–1.6B | 1,024 | about 100B |
| Llama 3.1 405B base model | 405B | much larger by modern standards | about 15T |

The point is not that GPT-2 and current models work differently at the conceptual level. The same broad recipe has been scaled up with more data, larger networks, longer contexts, better data processing, and more efficient hardware/software.

### 8.1 What a base-model release contains

A usable base-model release needs at least:

- code describing the forward computation;
- learned parameter values, often called weights.

The code describes the architecture. The weights contain the trained numerical settings.

### 8.2 Training progress

During training, researchers watch the loss. A decreasing loss indicates that the model is improving its next-token predictions on the training data. Early in training, generated text is random or only locally coherent. Later, it becomes more fluent as the model learns more of the statistical structure of the corpus.

### 8.3 Why GPUs matter

Training requires enormous amounts of matrix computation. GPUs are effective because they can perform many similar arithmetic operations in parallel. Large systems combine multiple GPUs into nodes and many nodes into clusters.

The important systems lesson is that the algorithmic idea—predict the next token—is simple, while the distributed engineering, memory movement, data pipeline, numerical efficiency, and hardware scale are extremely difficult.

## 9. Base model versus assistant model

### 9.1 The base model

The result of pre-training is a **base model**. It is fundamentally an internet-text simulator. Given a prefix, it tries to continue the kind of text that statistically fits that prefix.

If the prompt looks like:

```text
Once upon a time...
```

the model can continue a story. If it looks like an article, it may continue an article. But a base model is not necessarily trained to answer a direct user question in a helpful assistant format.
Base models  are “dreaming” or remixing internet documents. They contain useful knowledge but lack the post-training behavior expected from ChatGPT.

### 9.2 Few-shot prompting can simulate an assistant

Even a base model can be induced to act like an assistant if the prompt contains:

- a description of the desired roles;
- several example Human/Assistant turns;
- a new Human question at the end.

The base model is still only continuing the token sequence. It appears to take on the assistant role because the prompt establishes a strong pattern for continuation. This is an example of **in-context learning**: the model adapts its behavior within the prompt without changing its parameters.

## 10. Post-training and supervised fine-tuning

### 10.1 Turning a document simulator into an assistant

Post-training uses the base model and changes the training data. Instead of raw internet documents, the system trains on conversation examples.

The dataset can contain:

- user requests and assistant answers;
- multi-turn conversations;
- examples of explanations and transformations;
- refusals for requests the system should not fulfill;
- examples of tone, helpfulness, and formatting.

The algorithm is still next-token training. What changes is the distribution of sequences the model is trained to imitate.

### 10.2 Conversation protocol

A conversation that looks structured to a human—system, user, assistant, multiple turns—is encoded as a one-dimensional token sequence. Special tokens or markers indicate boundaries and roles.

Conceptually:

```text
<start><user> question <end>
<start><assistant> ideal answer <end>
```

At inference time, the serving system builds this structured context, places an assistant-start marker at the end, and asks the model to continue the assistant’s response token by token.

### 10.3 Human data labeling

In the InstructGPT-style approach, human labelers are given instructions for creating desirable answers. At a high level, the desired behavior is often described as helpful, truthful, and harmless, with detailed guidelines behind those principles.

Human labelers may:

1. write or select prompts;
2. produce ideal assistant responses;
3. follow detailed labeling policies;
4. create examples across many topics and interaction styles.

Modern pipelines use substantial synthetic-data assistance from other LLMs, followed by filtering, editing, or human review. The underlying behavior is still being shaped by curated examples.

### 10.4 The personality is statistical

The assistant’s personality is not usually written as a giant collection of explicit rules. It is learned from examples. With enough examples, the model learns a statistical pattern of how a helpful assistant tends to respond.

The practical mental model is:

> A response from a chat model is a neural-network simulation of the kinds of answers produced by the data-labeling and instruction process used to train it.

This does not mean every output is copied from one labeler. It means the style and behavior are learned as a distribution.

## 11. Hallucinations, knowledge, and tools

### 11.1 Why hallucinations happen

A language model is trained to produce plausible continuations, not to consult a truth database before every statement. If the prompt asks about an obscure or fictional person, the model may have learned that questions of the form “Who is X?” are normally answered confidently. It may therefore generate a plausible-sounding answer even when it lacks the fact.

The failure is not random noise in the ordinary sense. It is often a coherent imitation of the style and structure of a factual answer without reliable factual grounding.

### 11.2 Parameter knowledge is vague recollection

Knowledge stored in the weights is distributed and statistical. Frequently repeated information is more likely to be recalled correctly. Rare, ambiguous, outdated, or poorly represented information is less reliable.

The model can also memorize and reproduce high-quality or repeatedly sampled documents. This is why exact regurgitation is possible even though the parameters are not a normal database.

### 11.3 Improving “I don’t know” behavior

One mitigation is to:

1. empirically test what the model knows and does not know;
2. identify questions near the boundary of its knowledge;
3. add training examples where the correct behavior is to acknowledge uncertainty.

This helps connect an internal signal of uncertainty to an explicit response such as “I don’t know” or “I cannot verify that.”

### 11.4 Working memory versus long-term memory

A useful psychology analogy is:

- **Parameters:** long-term, vague, compressed recollection learned during training.
- **Context window:** working memory containing the current prompt, conversation, retrieved documents, and tool results.

If you want a model to summarize a specific document accurately, putting the document in the context is generally safer than asking it to recall the document from its parameters.

### 11.5 Tools extend the model

Tools change the reliability profile by moving certain operations outside the neural network:

- **Web search:** retrieves current or obscure information and inserts the results into the context.
- **Code interpreter:** lets the model write a program and have a real interpreter perform arithmetic, counting, or data manipulation.
- **Other tools:** can provide databases, APIs, files, browsers, or specialized computation.

The model decides to emit a tool call or structured action. The external system executes it, returns the result, and the model continues from that result.

This is the basic pattern behind tool-using agents:

```text
Model proposes action → external tool executes → result enters context → model continues
```

#### How tool-usage training data is formatted

To teach this behavior, the SFT training set includes examples where the assistant emits a structured tool call instead of (or before) answering directly. A simplified training example might look like:

```text
<|system|>
You have access to the following tools:
- web_search(query: str) → returns search results
- calculator(expression: str) → returns numeric result
<|end|>

<|user|>
What is the mass of Jupiter in kilograms?
<|end|>

<|assistant|>
<tool_call>{"name": "web_search", "arguments": {"query": "mass of Jupiter in kg"}}</tool_call>
<|end|>

<|tool_result|>
The mass of Jupiter is approximately 1.898 × 10^27 kg, making it the
largest planet in our solar system.
<|end|>

<|assistant|>
Jupiter’s mass is approximately **1.898 × 10²⁷ kg**, about 318 times
the mass of Earth.
<|end|>
```

Notice the structure: special tokens mark role boundaries, the assistant’s first turn is a tool call rather than a direct answer, the tool result is injected back into the context, and only then does the assistant produce the final human-readable response. By training on many examples like this, the model learns *when* to call a tool ("I need current or exact information") and *how* to format the call so the external system can parse it.

A calculator example follows the same pattern:

```text
<|user|>
What is 7,842 × 9,153?
<|end|>

<|assistant|>
<tool_call>{"name": "calculator", "arguments": {"expression": "7842 * 9153"}}</tool_call>
<|end|>

<|tool_result|>
71,785,626
<|end|>

<|assistant|>
7,842 × 9,153 = **71,785,626**.
<|end|>
```

The model never performs the multiplication itself—it delegates to a tool that can do it exactly.

## 12. Knowledge of self

At a basic level, each interaction consists of:

1. constructing a context;
2. running the model over that context;
3. generating tokens;
4. ending the computation.

The model does not automatically possess a persistent autobiographical self across conversations. Statements about its identity are generated from learned patterns in the context and training data. They can be useful operationally, but they should not be treated as introspective evidence of a conscious, continuously existing entity.

## 13. Models need tokens to think

### 13.1 Finite compute per generated token

For each next token, the Transformer performs a finite amount of computation. A hard problem may require more computation than can be packed into one forward pass before the answer token is produced.

The model can often improve by generating intermediate tokens that externalize or distribute the computation:

```text
Problem → intermediate step → intermediate step → check → final answer
```

Each intermediate token gives the model another opportunity to transform the state and use the generated text as a scratchpad.

### 13.2 Arithmetic example

For a simple word problem, a model may produce the right answer in a short response. If the numbers become larger, forcing it to answer immediately can cause an error. Allowing intermediate calculations makes each individual step easier.

However, even a long natural-language calculation should not automatically be trusted. The model can make an error in any intermediate step.

### 13.3 Prefer tools for exact computation

For arithmetic, counting, or data manipulation, a practical workflow is:

- ask the model to use code or a calculator;
- inspect the generated code when appropriate;
- rely on the interpreter for the exact operation;
- use the model to explain the result.

The model may be good at translating a problem into code even when it is unreliable at performing the computation internally.

## 14. Tokenization explains spelling and counting failures

### 14.1 Character-level tasks are unnatural for token models

Humans visually see individual letters. The model may see a word as a few multi-character tokens. Asking it to select every third character or count letters requires it to recover character-level structure from token-level representations.

This is why tasks such as:

- spelling a word backward;
- selecting every third character;
- counting repeated letters;
- counting a long sequence of dots;

can fail in surprising ways.

### 14.2 The “strawberry” example

A well-known example is counting the letter `r` in “strawberry.” The apparent contradiction is that a model may solve advanced mathematics while failing a simple spelling/counting question.

The explanation is a combination of:

1. the model seeing tokens rather than characters;
2. token-level processing making exact counting awkward;
3. the finite computation available per forward pass.

Using code changes the problem into one the computer can solve exactly.

## 15. Jagged intelligence: the Swiss-cheese model

LLM capability is not smooth or uniformly human-like. It is **jagged**.

A model can be exceptionally strong at a difficult task and strangely weak at a simple one. A useful analogy is Swiss cheese: the model has broad areas of impressive capability, but also holes that may appear unpredictably.

Examples include:

- solving advanced mathematics but making a basic decimal-comparison mistake;
- answering a PhD-level question but failing a character-indexing task;
- writing fluent factual prose that contains an invented claim;
- producing a correct answer through reasoning but making an arithmetic slip in the middle.

### Practical consequence

Do not evaluate a model only by its average intelligence or benchmark score. Evaluate it on the exact task, distribution, tools, and failure costs that matter to your application.

## 16. From supervised fine-tuning to reinforcement learning

### 16.1 Why imitation is not enough

Suppose humans create several valid solutions to a math problem. They may all reach the same final answer, but they may differ in:

- number of tokens;
- intermediate steps;
- hidden assumptions;
- ease for a human reader;
- suitability for the model’s internal computation.

A human labeler knows what is easy for a human, not necessarily what is easy for the model. A solution that looks elegant to a person might contain a sudden conceptual leap that is difficult for the model to reproduce reliably.

SFT is useful because it initializes the model near sensible behavior. But it may not discover the model’s best internal strategy.

### 16.2 RL as guess-and-check

For a prompt with a verifiable answer:

1. Generate many candidate solutions.
2. Check which candidates reach the correct result.
3. Reinforce the token sequences associated with successful solutions.
4. Repeat across many problems.

The successful solutions may have been discovered by the model itself rather than written by a human expert.

This gives the model room to discover strategies that humans did not explicitly prescribe.

####  Example

Suppose the training prompt is:

> *Solve: A store sells notebooks for $4 each and pens for $1.50 each. Maria buys 5 notebooks and 8 pens. How much does she pay in total?*

The system generates, say, four candidate solutions:

| Attempt | Reasoning | Final answer | Correct? |
|---|---|---|---|
| A | 5×4=20, 8×1.5=12, total=32 | $32 | ✅ |
| B | 5×4=20, 8×1.5=8, total=28 | $28 | ❌ |
| C | notebooks=5×4=20, pens=8×1.50=12.00, sum=20+12=32 | $32 | ✅ |
| D | 13×4=52 | $52 | ❌ |

Attempts A and C both reach the correct answer ($32), but through different token sequences. The RL update reinforces both A and C while suppressing B and D. Over many such problems, the model learns which intermediate-step patterns reliably lead to correct outcomes—even if those patterns differ from what a human tutor would have written.

### 16.3 Thinking and chain-of-thought as emergent behavior

With reinforcement learning, solutions tend to become longer. The model begins to:

- reconsider earlier steps;
- check its own work;
- try another approach;
- backtrack;
- compare alternative derivations;
- defer the polished answer until after internal exploration.

This is an emergent cognitive strategy: the training process only needs a way to identify correct outcomes, while the model discovers token sequences that help it reach those outcomes.

## 17. DeepSeek-R1 and reasoning models

DeepSeek-R1 is a reasoning or “thinking” model trained with reinforcement learning on problems with verifiable solutions.

Compared with a straightforward SFT response, a reasoning model may spend many more tokens exploring the problem before presenting a concise final answer. The visible behavior can look like:

1. parse the problem;
2. propose a solution;
3. notice a possible mistake;
4. re-check using another method;
5. settle on the answer;
6. write a cleaner response for the user.

### When to use a thinking model

Use a reasoning model when:

- the problem requires multi-step mathematics;
- code requires nontrivial planning or debugging;
- there are many interacting constraints;
- correctness matters more than response speed.

For simple factual or conversational tasks, the extra reasoning time may be unnecessary. The practical recommendation is to use simpler models for routine work and reach for thinking models on difficult problems.

## 18. AlphaGo as an analogy for reinforcement learning

AlphaGo illustrates the distinction between imitation and discovery.

- **Supervised learning from expert games:** learn to imitate strong human players, but remain bounded by the examples.
- **Reinforcement learning through self-play:** try many move sequences, identify wins, and reinforce successful strategies.

Because the reward is whether the game is won, the system can discover moves that no human expert explicitly demonstrated. The famous “Move 37” is an example of an unexpected but effective strategy—a placement that no professional player would have considered, yet turned out to be decisive.

The analogy is that LLM reasoning may progress similarly when models are placed in sufficiently rich “game environments” made from math, code, science, and other problems with reliable evaluation.

## 19. Verifiable versus unverifiable domains

### 19.1 Verifiable domains

In math or code, a solution can often be checked automatically:

- compare a boxed final answer with the known answer;
- execute code and inspect the result;
- use a formal or programmatic verifier;
- use an evaluator model as an additional judge when appropriate.

This enables large-scale RL without humans inspecting every attempt.

### 19.2 Unverifiable domains

Tasks such as joke writing, poetry, creative writing, and open-ended summarization do not have one obvious correct answer. Human judgment is needed, but evaluating huge numbers of generations directly is expensive and slow.

This is where RLHF is introduced.

## 20. RLHF: reinforcement learning from human feedback

### 20.1 The core trick: learn a reward model

Instead of asking humans to score every generation during RL:

1. Generate several candidate responses to a prompt.
2. Ask humans to rank the candidates from best to worst.
3. Train a separate **reward model** to imitate those human preferences.
4. Use the reward model as an inexpensive, queryable approximation of human judgment.
5. Optimize the language model against that learned reward.

Humans are asked to compare or rank outputs because that is often easier and more reliable than writing a perfect answer from scratch.

#### Concrete example

Prompt: *"Explain what a black hole is in two sentences."*

The model generates three candidates:

> **Response A:** A black hole is a region of spacetime where gravity is so strong that nothing, not even light, can escape once it crosses the event horizon. They form when massive stars collapse at the end of their life cycle.

> **Response B:** Black holes are really cool space things. They suck everything in and nothing gets out, which is pretty wild if you think about it.

> **Response C:** A black hole is defined by the Schwarzschild metric as a solution to Einstein's field equations in which the radius falls below rs = 2GM/c². The resulting singularity is surrounded by an event horizon.

A human ranker might rank these: **A > C > B** (A is clear and accurate; C is correct but too technical for the request; B is vague and informal).

Many such rankings are collected, and a reward model is trained to predict these preferences. During RL, the reward model scores new generations without needing a human in the loop for every output.

### 20.2 The advantage

RLHF makes it possible to optimize behavior in domains where no automatic exact answer exists. It can improve qualities such as helpfulness, style, relevance, and perceived quality.

It also reduces the amount of human effort required: humans provide a smaller set of preference comparisons, and the reward model generalizes those preferences.

### 20.3 The risks

The reward model is itself a neural-network approximation. It can:

- disagree with real human judgment;
- miss important qualities;
- reward superficial patterns;
- be exploited by the optimizing model.

This is a form of reward hacking. The optimizer searches for outputs that receive high reward, not necessarily outputs that humans would genuinely consider good.

The deliberately absurd example is that a reward model might assign a high score to repetitive nonsense. Adding that one example to the reward-model training set does not solve the general problem because many other adversarial inputs may exist.

### 20.4 Why RLHF cannot simply run forever

With a simple, hard-to-game reward—such as whether a program passes tests or a move wins a game—more RL compute can continue to discover useful strategies.

With a learned reward model, optimizing too long can push the language model into regions where the reward model is confidently wrong. Therefore RLHF typically has to be stopped or carefully constrained once the model begins exploiting weaknesses in the reward signal.

The key distinction is:

- **Verifiable RL:** potentially scalable with more compute and more practice.
- **RLHF:** useful preference-based fine-tuning, but limited by the quality and gameability of the reward model.

## 21. Future directions

### 21.1 Multimodality

The same token-sequence framework can be extended beyond text:

- audio can be represented through tokenized signal or spectrogram segments;
- images can be represented through tokenized patches;
- text, image, and audio tokens can be interleaved in a shared context.

The conceptual change is less dramatic than it first appears: create a sequence representation and train a model to model the resulting patterns.

### 21.2 Long-running agents

Current models are good at individual tasks but still struggle to execute coherent, error-correcting workflows over long periods. Future agents may:

- plan and perform multiple actions;
- use tools and computer interfaces;
- report progress;
- recover from errors;
- operate for minutes or hours under human supervision.

Because models remain fallible, the likely operational pattern is supervision rather than complete autonomy. Humans become supervisors of many digital agents.

### 21.3 Computer use and invisible integration

Models are likely to become embedded inside everyday software and may be able to operate keyboards, mice, browsers, and other computer interfaces. The model’s role shifts from answering questions to taking actions on behalf of a user.

### 21.4 Test-time training and long-term adaptation

Deployed models generally have fixed parameters. They can adapt through in-context learning, but they do not automatically update their weights from each experience in the human sense.

For very long-running multimodal tasks, simply making the context window larger may not be enough. Context is finite, expensive, and difficult to manage. New approaches to memory, learning, compression, and test-time adaptation will likely be needed.



## 22. Practical guidance 

### Use an LLM for

- brainstorming and exploration;
- first drafts;
- summarization when the relevant source is provided;
- explaining unfamiliar ideas;
- translating between formats or abstractions;
- coding assistance;
- generating candidate approaches;
- using tools to retrieve or compute information.

### Be cautious when

- the answer concerns obscure facts;
- the information may have changed after training;
- exact arithmetic or counting is required;
- spelling or character-level manipulation matters;
- the response will be used in a high-stakes decision;
- the model is asked to operate without verification;
- a long autonomous workflow can compound small errors.

### A reliable working pattern

1. Give the model the relevant source material instead of relying only on memory.
2. Ask it to state assumptions and identify uncertainty.
3. Request intermediate reasoning or a structured plan for complex tasks.
4. Use web search, retrieval, code, calculators, tests, or domain tools where appropriate.
5. Verify important claims independently.
6. Treat the model as a tool in a toolbox and remain responsible for the final result.

## 23. Key takeaways to remember

1. **LLMs are next-token predictors, but scale makes the behavior rich.**
2. **The training corpus is filtered and curated; data quality strongly shapes behavior.**
3. **Tokens are not words or characters.** This explains cost, context limits, and many quirks.
4. **Pre-training creates a base model, not necessarily a helpful assistant.**
5. **SFT teaches assistant behavior by example.**
6. **RL can let a model discover reasoning strategies that human demonstrations did not specify.**
7. **RL works best when rewards are verifiable and difficult to game.**
8. **RLHF approximates human preferences with a reward model, so it can be exploited.**
9. **Parameters are vague long-term memory; the context window is working memory.**
10. **Tools make the system more reliable by moving retrieval and computation outside the neural network.**
11. **Models need tokens to think: more intermediate tokens can provide more computation.**
12. **Capability is jagged.** A model can be brilliant in one area and unexpectedly weak in another.
13. **The safest posture is productive skepticism:** use the model aggressively for leverage, but verify the work.

## 24. Compact glossary

| Term | Meaning |
|---|---|
| **Token** | A discrete text chunk represented by an integer ID |
| **Tokenizer** | The system that converts text to tokens and tokens back to text |
| **BPE** | Byte Pair Encoding, a common tokenization strategy |
| **Context window** | The maximum token sequence the model can process at once |
| **Embedding** | A learned vector representation of a token |
| **Logit** | An unnormalized score for a possible next token |
| **Inference** | Using fixed model parameters to generate outputs |
| **Pre-training** | Learning next-token patterns from broad data |
| **Base model** | The pre-trained model before assistant-oriented post-training |
| **SFT** | Supervised fine-tuning on curated input/output examples |
| **RL** | Learning by generating attempts, receiving rewards, and reinforcing successful behavior |
| **RLHF** | RL driven by a learned model of human preferences |
| **Reward model** | A model that scores candidate outputs according to learned preferences |
| **Hallucination** | A plausible but unsupported or false generated claim |
| **In-context learning** | Adapting behavior from examples or information in the prompt without changing weights |
| **Tool use** | Calling an external system such as search, code execution, or an API |
| **Reasoning/thinking model** | A model trained or configured to spend more generated tokens on difficult problems |


## 25. Final mental model

When you type into ChatGPT, the system roughly does this:

1. Encode your conversation into a structured token sequence.
2. Feed the sequence into a fixed Transformer network.
3. Use the network’s learned parameters to produce probabilities for the next token.
4. Generate one token, append it to the sequence, and repeat.
5. Potentially call tools when the model has been trained and configured to do so.
6. Present the resulting sequence as an answer.

The answer feels like a conversation because post-training shaped the model to simulate helpful assistant behavior. The model can reason because sequential token generation provides additional computation and because reinforcement learning can discover useful strategies. The answer can still be wrong because the system is a probabilistic, finite, lossy simulation with uneven capabilities.

The best practical stance is therefore neither blind trust nor dismissal: use LLMs as powerful collaborators and tools, give them the context and external tools they need, and verify the parts that matter.

## Source links

- [YouTube video](https://youtu.be/7xTGNNLPyMI)
- [FineWeb dataset](https://huggingface.co/spaces/HuggingFaceFW/blogpost-fineweb-v1)
- [Tiktokenizer](https://tiktokenizer.vercel.app/)
- [Transformer visualizer](https://bbycroft.net/llm)
- [llm.c / GPT-2 reproduction](https://github.com/karpathy/llm.c/discussions/677)
- [Llama 3 paper](https://arxiv.org/abs/2407.21783)
- [InstructGPT paper](https://arxiv.org/abs/2203.02155)
- [DeepSeek-R1 paper](https://arxiv.org/abs/2501.12948)
- [AlphaGo paper](https://discovery.ucl.ac.uk/id/eprint/10045895/1/agz_unformatted_nature.pdf)
- [LM Arena](https://lmarena.ai/)
- [LM Studio](https://lmstudio.ai/)


References:<br>
[Deep Dive into LLMs like ChatGPT by Andrej Karpathy ](https://youtu.be/7xTGNNLPyMI)<br>