# TakeMeter

A fine-tuned text classifier that evaluates discourse quality in r/nba. Given a post or comment from the NBA subreddit, TakeMeter assigns one of four labels: `analysis`, `hot_take`, `discussion`, or `reaction`. The goal is to distinguish between different modes of engagement in sports discourse, from structured statistical arguments to pure emotional reactions.

---

## Community

**I chose r/nba** because it is one of the few online communities where the quality of a take is openly contested. Regular participants actively distinguish between someone "actually providing analysis" and someone "just yelling their opinion." That implicit community norm made it possible to design labels grounded in something real, not invented from the outside.

The discourse is also varied enough to be interesting. A single game result generates all four label types within hours: someone posts quarter-by-quarter stats, someone posts "that team is cooked and never wins when it matters," someone asks "should they fire the coach?", and someone posts "I can't believe I watched that until 1am." Same topic, four genuinely different modes of engagement, all in the same thread. That variety is what makes the classification task non-trivial.

---

## Label Taxonomy

### `analysis`

The post makes a structured argument supported by specific stats, historical comparisons, or tactical observations. The evidence is concrete enough that a reader could verify or refute the claim independently.

**Example 1:** "Jalen Brunson in the playoffs as a Knick: 29.4 PPG, 3.6 RPG, 6.6 APG, 46.0 FG%. SGA in the playoffs on the Thunder: 27.6 PPG, 5.0 RPG, 6.6 APG, 46.7 FG%. Their full career stats are also very similar."

**Example 2:** "Wemby's conditioning problems in the finals were in large part a roster problem. The Spurs got destroyed in the Kornet minutes, so Mitch had to play Wemby way more than he is used to. Wemby played the most minutes of any player in the finals, even more than Brunson."

### `hot_take`

The post makes a bold, confident claim without supporting evidence. It asserts a conclusion rather than reasoning toward one. The post declares rather than argues.

**Example 1:** "LeBron James has built an entire career of just running and dunking the ball. Anyone in LeBron's body could do what he does."

**Example 2:** "These Knicks were the least opposing champions in a min. They won't win it again."

### `discussion`

The post poses a genuine open question or hypothetical that invites community input. The author withholds judgment and treats multiple answers as equally valid.

**Example 1:** "If Kawhi had stayed with the Raptors in 2019, how many more rings do you think they win? Pascal Siakam, Fred Vanvleet, Norm Powell and OG Anunoby were all young and improving. Do you think they could win more rings?"

**Example 2:** "What is your team's biggest need for this offseason? We got the draft coming up in a couple of days so I figured let's get the ball rolling."

### `reaction`

The post expresses an immediate emotional or humorous response to a specific recent event, moment, or news item. There is little to no argument. The content is the feeling itself.

**Example 1:** "10 years ago I reached over LBJ as he was entering the tunnel with his trophies after game 7 in 2016 to touch the Larry O'Brien. 10 years later I still have not found a photograph that captures this moment."

**Example 2:** "Jalen Brunson's reach is so far that I got notified about his LinkedIn post. Wild shit."

---

## Difficult Annotation Cases

I encountered three genuinely hard cases during annotation, and the decision made for each:

**Case 1: "Trae Young is Fools Gold. His career 3P% is 35.2 and he's one of the worst traffic cones in the NBA."**

Could be `hot_take` or `analysis`. The post cites a real stat (35.2% 3P). Decision: `hot_take`. The stat is decorative, not structural. The post makes no comparison, no causal argument, no logical chain connecting the number to the conclusion. One cherry-picked number attached to a bold assertion stays `hot_take`.

**Case 2: "Swapping LeBron for Jordan in 2018 Finals. We couldn't decide if the Cavs swept or if the Warriors took a game or two. Of course the Cavs win no matter what, there's no debate there."**

Could be `discussion` or `hot_take`. The post frames itself as a question but immediately answers it. Decision: `hot_take`. A true `discussion` post withholds judgment. "Of course the Cavs win no matter what, there's no debate there" is a confident conclusion, not an open question.

**Case 3: "Ironically a team in the finals desperately could have used retirement tour CP3. His leadership and game IQ might have actually helped them close a game or two instead of blowing all those leads."**

Could be `analysis` or `hot_take`. The post mentions tactical concepts (leadership, game IQ, closing games). Decision: `hot_take`. Mentioning basketball concepts without any supporting data is still an assertion. "Might have helped" with no evidence is not analysis, it is speculation dressed in tactical language.

---

## Data Collection

**Source:** r/nba via the Arctic Shift API (arctic-shift.photon-reddit.com), which provides access to archived Reddit posts and comments without requiring OAuth authentication. I had a failed attempt with Reddit API scraping and hence chose this technique.

**Collection method:** Separate API calls to `/api/posts/search` and `/api/comments/search`, filtered for text-only content, minimum 80 characters, maximum 1000 characters for comments, excluding deleted or removed content. Pagination used the `before` parameter on `created_utc` to avoid fetching duplicate batches. A fixed pagination bug (originally used `after` parameter instead of `before`) was required before the scraper produced unique examples.

**Final dataset:** 305 labeled examples after discarding 44 rows (journalist news dumps with no discussion body, spam, bot posts, and help requests).

**Label distribution:**

| Label | Count |
| --- | --- |
| analysis | 105 |
| hot_take | 79 |
| reaction | 64 |
| discussion | 57 |
| **Total** | **305** |

**Annotation process:** I manually annotated the labels. Disagreements were concentrated at the `hot_take`/`analysis` and `reaction`/`discussion` boundaries and were resolved manually.

**Train/validation/test split:** 70% / 15% / 15%, stratified by label.

| Split | Size |
| --- | --- |
| Train | 213 |
| Validation | 46 |
| Test | 46 |

---

## Fine-Tuning

**Base model:** `distilbert-base-uncased`

**Input construction:** For posts with a title, I constructed the input as `title [SEP] body text`, truncated to 256 tokens. For comments (no title), only the body text was used. Concatenating the title is important because the title often carries the most label-discriminative signal, for example a title phrased as a bold claim versus a question.

**Key hyperparameter decisions:**

The default notebook settings of 3 epochs and learning rate 2e-5 caused complete majority class collapse for me: the model predicted `analysis` for every single test example, achieving 34.8% accuracy (exactly the proportion of `analysis` in the test set). I made three changes to fix this:

1. **Class weights added.** A custom `WeightedTrainer` was implemented that computes a weighted cross-entropy loss, with `hot_take` weighted at 2.0, `discussion` at 1.8, `reaction` at 1.5, and `analysis` at 1.0. Without class weights, the model learns that always predicting the majority class minimizes loss quickly and stops improving. The weights force it to pay attention to underrepresented classes.
2. **Epochs increased to 8.** With only 213 training examples across 4 classes, 3 epochs I felt, were not enough to escape the majority class local optimum. Training loss continued dropping through epoch 8 (0.487) while validation accuracy peaked at epoch 6 (0.587) and the best checkpoint was loaded at the end.
3. **I set the learning rate to 3e-5.** Slightly higher than the standard 2e-5 to help escape the collapse faster on a small dataset.

**Training summary:**

| Epoch | Training Loss | Validation Loss | Accuracy |
| --- | --- | --- | --- |
| 1 | 1.3909 | 1.3773 | 0.3478 |
| 2 | 1.3811 | 1.3616 | 0.3913 |
| 3 | 1.3199 | 1.3037 | 0.4348 |
| 4 | 1.2087 | 1.1687 | 0.5217 |
| 5 | 0.8855 | 1.0853 | 0.5652 |
| 6 | 0.6845 | 1.0289 | **0.6091** |
| 7 | 0.6449 | 1.1069 | 0.5435 |
| 8 | 0.4878 | 1.0617 | 0.5652 |

Best checkpoint saved at epoch 6 (highest validation accuracy). `load_best_model_at_end=True` was used to ensure the best checkpoint was evaluated on the test set.

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
| --- | --- |
| Zero-shot baseline (Groq llama-3.3-70b-versatile) | 0.587 |
| Fine-tuned DistilBERT | **0.609** |
| Improvement | +0.022 |

### Per-Class Metrics

**Fine-tuned DistilBERT:**

| Label | Precision | Recall | F1 | Support |
| --- | --- | --- | --- | --- |
| analysis | 0.71 | 0.62 | 0.67 | 16 |
| hot_take | 0.47 | 0.75 | 0.58 | 12 |
| discussion | 0.62 | 0.56 | 0.59 | 9 |
| reaction | 0.80 | 0.44 | 0.57 | 9 |
| **macro avg** | **0.65** | **0.59** | **0.60** | 46 |

**Zero-shot baseline (Groq):**

| Label | Precision | Recall | F1 | Support |
| --- | --- | --- | --- | --- |
| analysis | 0.57 | 0.25 | 0.35 | 16 |
| hot_take | 0.52 | 0.92 | 0.67 | 12 |
| discussion | 0.57 | 0.44 | 0.50 | 9 |
| reaction | 0.73 | 0.89 | 0.80 | 9 |
| **macro avg** | **0.60** | **0.62** | **0.58** | 46 |

### Confusion Matrix (Fine-Tuned Model)

|  | Predicted: analysis | Predicted: hot_take | Predicted: discussion | Predicted: reaction |
| --- | --- | --- | --- | --- |
| **True: analysis** | 10 | 4 | 1 | 1 |
| **True: hot_take** | 1 | 9 | 2 | 0 |
| **True: discussion** | 2 | 2 | 5 | 0 |
| **True: reaction** | 1 | 4 | 0 | 4 |

The supplementary `confusion_matrix.png` is committed to the repo root.

### Failure Analysis

**18 out of 46 test examples were wrong. The dominant error pattern is `reaction` being predicted as `hot_take` (4 of 9 reaction examples).**

**Error 1: `reaction` predicted as `hot_take`**

> "I've definitely thought about it. After Spurs traded him back I thought it would be great if the guy had a 12 year career just being traded back and forth."
> 

True label: `reaction`. Predicted: `hot_take` (confidence 0.43).

This is a nostalgic personal thought triggered by a trade story. There is no bold claim being made. The model predicted `hot_take` because the post lacks an obvious emotional trigger word ("I can't believe", "this is insane") and reads instead like a speculative opinion. The boundary the model failed to learn is that reactions do not always use exclamatory language. A quiet, reflective personal musing about a player's career trajectory is still a `reaction` if no argument is being advanced. This is a data problem: most `reaction` examples in the training set were emotionally vivid (anniversaries, game moments), so the model learned to associate `reaction` with high-emotion language and missed the low-key nostalgic subtype.

**Error 2: `reaction` predicted as `analysis`**

> "Just learned about Reggie Lewis who died young as an active player due to HOCM. While sad, the below aftermath is even more crazy. Lewis' contract remained on the Celtics' salary cap for two full seasons after his death."
> 

True label: `reaction`. Predicted: `analysis` (confidence 0.40).

This post is a reaction to discovering a historical story. The quoted contract detail reads as factual/analytical, but its function is to convey how shocking the story is, not to build an argument. The model picked up on the factual, structured language ("remained on the salary cap for two full seasons") and classified it as `analysis`. The boundary it failed is this: factual language does not make something analysis. Analysis requires a claim and a conclusion drawn from evidence. This post has no conclusion. It is sharing a surprising fact as a reaction. This is a genuine labeling challenge and arguably the tightest edge case in the dataset. If I were re-annotating, I might reconsider whether this should be `analysis` after all.

**Error 3: `hot_take` predicted as `analysis`**

> "Ironically a team in the finals desperately could have used retirement tour CP3. His leadership and game IQ might have actually helped them close a game or two instead of blowing all those leads."
> 

True label: `hot_take`. Predicted: `analysis` (confidence 0.42).

This post uses tactical vocabulary (leadership, game IQ, closing games, blowing leads) without any supporting data. The model learned that tactical basketball language is a signal for `analysis`, which is partly true. But tactical language plus zero evidence is still a `hot_take`. The model has not learned to distinguish between posts that use analytical vocabulary as a substitute for evidence versus posts that use it to explain evidence. This is a prompt/data problem: the training set likely contained many short analysis examples that used similar vocabulary without full statistical backing, which blurred the signal the model was trying to learn.

### Sample Classifications

The following examples were run through the fine-tuned model after training:

| Text (truncated) | True Label | Predicted | Confidence |
| --- | --- | --- | --- |
| "Jalen Brunson in the playoffs as a Knick: 29.4 PPG, 3.6 RPG... SGA: 27.6 PPG..." | analysis | analysis | 0.81 |
| "LeBron doesn't have that dog in him. If he did he would have asked for the ball." | hot_take | hot_take | 0.74 |
| "If Kawhi had stayed with the Raptors in 2019, how many more rings do you think they win?" | discussion | discussion | 0.68 |
| "Today's the 10 year anniversary of Game 7 of the 2016 Finals. Watching this game then GoT Battle of the Bastards was an all time night." | reaction | reaction | 0.71 |
| "Wemby's conditioning problems were largely a roster problem. He played the most minutes of any player in the finals." | analysis | analysis | 0.76 |

The first correctly predicted example is the Brunson/SGA stat comparison. This prediction is reasonable because the post contains two sets of parallel statistics in a direct comparison format, which is the clearest possible signal for the `analysis` label. The model correctly recognizes that the evidence is the argument, not decoration around an assertion.

---

## Reflection: What the Model Captured vs. What Was Intended

The model learned surface linguistic patterns, not the underlying concept of discourse quality.

What it learned well: numbers and player name comparisons reliably predict `analysis`; question marks and "do you think" reliably predict `discussion`; emotional exclamations reliably predict `reaction`; short declarative sentences with superlatives ("is the worst", "will never") reliably predict `hot_take`.

What it missed: the model did not learn to evaluate the relationship between a claim and its evidence. It cannot tell the difference between a post that cites a stat as decoration (hot_take) and a post that uses a stat as the core of an argument (analysis). It cannot tell the difference between a personal musing with no bold claim (reaction) and a personal opinion stated confidently (hot_take). These distinctions require understanding intent and argumentative structure, not pattern matching on vocabulary.

The gap is most visible in the `reaction` class, where the model achieved recall of only 0.44. The intended definition of `reaction` was about emotional mode, not emotional intensity. But the model learned to associate `reaction` with high-intensity emotional language, and missed the quieter, more reflective reactions that make up a real portion of the class. That is the difference between what the label was designed to capture and what the training distribution actually demonstrated.

---

## Spec Reflection

**One way the spec helped:** The requirement to define decision rules for ambiguous cases before annotating forced a discipline that paid off directly. The explicit rule "if a post answers its own question with a confident conclusion, label it hot_take not discussion" resolved dozens of borderline posts quickly during annotation that would otherwise have been labeled inconsistently. Writing the rules first and then annotating is the right order of operations. Doing it the other way would have produced a noisier dataset.

---

## AI Usage

**Instance 1: Label taxonomy design and edge case generation**

Claude was asked to read through 30 to 40 raw r/nba posts and identify what patterns emerge naturally, then propose four labels with definitions and boundary cases. Claude produced the initial taxonomy (analysis, hot_take, discussion, reaction) and generated example boundary posts for stress-testing. The definitions were then revised by the student to sharpen the decision rules, particularly the rule distinguishing decorative stat usage (hot_take) from structural evidence (analysis). The final taxonomy reflects back-and-forth revision rather than Claude's first draft.

**Instance 2: Failure analysis**

After training, the list of 18 wrong predictions was given to Claude with the prompt: "Identify the top patterns in these errors. What types of posts is the model consistently confusing and what do the misclassified examples have in common?" Claude identified three patterns: `reaction` being predicted as `hot_take` for low-emotion reactions, analytical vocabulary in hot takes triggering false `analysis` predictions, and factual posts with no argument being mislabeled as `analysis`. Each pattern was manually verified by reading through the examples independently before being included in the evaluation report above.
