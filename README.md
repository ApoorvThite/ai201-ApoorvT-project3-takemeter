# TakeMeter — Planning Document

## Community

**Chosen community:** r/nba (Reddit)

r/nba is one of the largest sports discussion communities on the internet, with over 17 million subscribers. It was chosen for three reasons. First, the discourse is genuinely varied in quality — the same subreddit hosts rigorous stat-based arguments, baseless hot takes, open hypotheticals, and pure emotional reactions to games, all mixed together. Second, the community has strong implicit norms about what counts as a "real take" versus noise — regular participants actively distinguish between someone "actually providing analysis" and someone "just yelling their opinion," which means the labels are grounded in real community values rather than invented externally. Third, posts are text-heavy by nature: the rules filter out pure image/video posts, so the remaining content is overwhelmingly written opinion and discussion, which makes it well-suited for text classification.

The discourse is varied enough to be interesting because a single event — say, a team losing a playoff game — generates all four types of response within hours: someone posts stats about fourth-quarter performance (analysis), someone posts "that team is cooked and never wins when it matters" (hot_take), someone asks "do you think they should fire the coach?" (discussion), and someone posts "I can't believe I watched that game until 1am for that ending" (reaction). The same topic, four genuinely different modes of engagement, all in the same thread.

---

## Labels

Four labels were defined to capture the meaningful distinctions in r/nba discourse.

**`analysis`** — The post makes a structured argument supported by specific stats, historical comparisons, or tactical observations, where the evidence is concrete enough that a reader could verify or refute the claim independently.

Example 1: "Jalen Brunson in the playoffs as a Knick: 29.4 PPG, 3.6 RPG, 6.6 APG, 46.0 FG%. SGA in the playoffs on the Thunder: 27.6 PPG, 5.0 RPG, 6.6 APG, 46.7 FG%. Their full career stats are also very similar."

Example 2: "Wemby's conditioning problems in the finals were in large part a roster problem. The Spurs got destroyed in the Kornet minutes, so Mitch had to play Wemby way more than he is used to. Wemby played the most minutes of any player in the finals, even more than Brunson."

**`hot_take`** — The post makes a bold, confident claim without supporting evidence, asserting a conclusion rather than reasoning toward one — the post declares rather than argues.

Example 1: "LeBron James has built an entire career of just running and dunking the ball. Anyone in LeBron's body could do what he does — he is not skilled."

Example 2: "These Knicks were the least opposing champions in a min. They won't win it again."

**`discussion`** — The post poses a genuine open question or hypothetical that invites community input, where the author withholds judgment and treats multiple answers as equally valid.

Example 1: "If Kawhi had stayed with the Raptors in 2019, how many more rings do you think they win? Marc Gasol, Danny Green and Kyle Lowry were getting older, but Pascal Siakam, Fred Vanvleet, Norm Powell and OG Anunoby were all young and improving. Do you think they could win more rings?"

Example 2: "What is your team's biggest need for this offseason? We got the draft coming up in a couple of days so I figured let's get the ball rolling on the discussion."

**`reaction`** — The post expresses an immediate emotional or humorous response to a specific recent event, moment, or news item, with little to no argument — the content is the feeling itself.

Example 1: "10 years ago I reached over LBJ as he was entering the tunnel with his trophies after game 7 in 2016 to touch the Larry O'Brien. 10 years later I still have not found a photograph that captures this moment."

Example 2: "Jalen Brunson's reach is so far that I got notified about his LinkedIn post. Wild shit."

---

## Hard Edge Cases

Three boundaries require explicit decision rules because they generate genuine ambiguity during annotation.

**Boundary 1: `hot_take` vs. `analysis`**

The ambiguous type: posts that cite one or two statistics to support a bold claim. A single stat can be used as genuine evidence in a structured argument, or it can be cherry-picked decoration to make an assertion sound credible.

Example of the ambiguous case: "Trae Young is Fools Gold. His career 3P% is 35.2 and he's one of the worst traffic cones in the NBA." This cites a real number but makes no structural argument around it.

Decision rule: If the evidence would still support the claim even if you stripped out the opinion framing — i.e., the stat is the reasoning, not a garnish — label it `analysis`. If the stat appears alongside hyperbolic framing and no logical structure connecting evidence to conclusion, label it `hot_take`. One cherry-picked number plus an assertion is `hot_take`. A comparison across multiple data points with a traceable conclusion is `analysis`.

**Boundary 2: `discussion` vs. `hot_take`**

The ambiguous type: posts that frame themselves as a question but immediately answer it with a confident conclusion.

Example of the ambiguous case: "Swapping LeBron for Jordan in 2018 Finals — we couldn't decide if the Cavs swept or if the Warriors took a game. Of course the Cavs win no matter what, there's no debate there."

Decision rule: If the post stakes a confident conclusion even while asking a question, label it `hot_take`. A true `discussion` post withholds judgment — the author genuinely does not answer the question they asked. "Of course X wins, no debate" immediately disqualifies a post from `discussion`.

**Boundary 3: `reaction` vs. `hot_take`**

The ambiguous type: emotional posts that also make a claim about a player, team, or the league.

Example of the ambiguous case: "'Y'all, y'all, y'all' — wild to consider there's different people with different opinions in a massive sub and that they aren't all in agreeance with every post."

Decision rule: If the emotional tone is the point — the post is expressing a feeling about a moment without arguing a position — label it `reaction`. If the emotional tone is used as a vehicle for a broader claim about a player, team, or the state of the game, label it `hot_take`. The test is: does removing the emotion leave behind an argument? If yes, it's `hot_take`. If removing the emotion leaves nothing, it's `reaction`.

**Boundary 4: `reaction` vs. `discussion`**

The ambiguous type: hypothetical posts triggered by a specific recent event.

Decision rule: `reaction` requires a specific identifiable recent event as the trigger (a game, a trade, a highlight, an anniversary). Hypotheticals about a player's current situation are `discussion` even if emotionally motivated. If someone posts "do you think a team would sign 48-year-old LeBron for jersey sales?" after LeBron retired, that's `discussion` — no specific event, just a hypothetical. If someone posts "I can't believe I watched that game until 1am" after a specific game, that's `reaction`.

---

## Data Collection Plan

**Source:** r/nba via the Arctic Shift API (`arctic-shift.photon-reddit.com`), which provides access to archived Reddit posts and comments without requiring OAuth authentication.

**Collection method:** Separate API calls to `/api/posts/search` and `/api/comments/search` endpoints, filtered for text-only content (no video or image posts), minimum 80 characters (to exclude one-liners like "facts" or "lol"), maximum 1000 characters for comments (to exclude essay-length walls of text), and excluding deleted or removed content. Pagination uses the `before` parameter on `created_utc` to avoid fetching duplicate batches.

**Target volume:** Collect 350 raw examples (175 posts + 175 comments), anticipating that ~15% will be discarded during cleaning (journalist news dumps with no discussion body, spam, bot posts, help requests). This leaves approximately 300 usable labeled examples, split 70/15/15 into train/validation/test.

**Per-label targets after annotation:**

| Label | Target count |
|---|---|
| `analysis` | ~90 |
| `hot_take` | ~70 |
| `reaction` | ~70 |
| `discussion` | ~70 |

**If a label is underrepresented after 200 examples:** Run a targeted collection pass using the `title` keyword search parameter in the Arctic Shift API. For `discussion`, search for titles containing "do you think", "who would", "what if", "is it weird". For `reaction`, search for titles containing "just saw", "can't believe", anniversary framing ("X years ago"). For `hot_take`, search for titles containing player names followed by strong adjectives ("overrated", "washed", "greatest"). Accept any label falling below 35 examples as underrepresented and collect more before training.

**Annotation process:** Open the collected CSV in Google Sheets. Add a `label` column and fill it manually using the definitions and decision rules above. Track at least 3 genuinely ambiguous examples with notes explaining the final decision, to be included in the README.

---

## Evaluation Metrics

**Primary metric: Overall accuracy** — reported as a baseline comparison between the fine-tuned model and the zero-shot Groq baseline on the same held-out test set.

**Why accuracy alone is insufficient:** The label distribution is imbalanced (`analysis` accounts for ~34% of examples). A model that predicts `analysis` for every input would achieve 34% accuracy without learning anything. Accuracy also hides per-class failures — a model could perform well on the majority class while completely failing on minority classes, and the overall number would look acceptable.

**Additional required metrics:**

Per-class precision, recall, and F1-score for all four labels. Precision matters for this task because a false positive (e.g., labeling a `hot_take` as `analysis`) would cause a community tool to surface low-quality takes as high-quality content — a real harm to users. Recall matters because a false negative (e.g., labeling `analysis` as `hot_take`) would suppress genuinely good content. F1 balances both.

Macro-averaged F1 as the primary single-number comparison between models. Macro F1 weights each class equally regardless of frequency, which is correct here because the goal is a classifier that works across all four discourse types — not just the most common one.

A confusion matrix to identify which specific label pairs the model confuses most frequently. This is essential for understanding failure modes and deciding whether to re-annotate ambiguous examples or adjust label definitions.

**Why these metrics fit this task specifically:** This is a multi-class classification task where all four classes matter equally for a community moderation tool. A classifier that only works on `analysis` is not useful — the tool needs to distinguish all four modes. Macro F1 directly measures whether the model has learned all four, and per-class metrics reveal which labels need more training data or sharper definitions.

---

## Definition of Success

**Minimum threshold for the project:** The fine-tuned DistilBERT model must outperform the zero-shot Groq baseline on overall accuracy. A fine-tuned model that performs worse than a zero-shot prompt would indicate either a data quality problem (noisy labels) or a training problem (hyperparameters, class imbalance).

**Definition of "genuinely useful" for deployment:** A macro F1 score of 0.60 or higher across all four labels, with no individual label F1 below 0.45. The 0.45 floor matters because a label with F1 below 0.45 is barely better than random for a 4-class problem (random baseline ~0.25), and a community tool that fails completely on one discourse type is not trustworthy.

**Definition of "good enough" for deployment in a real community tool:** Macro F1 of 0.65+, with all per-class F1 scores above 0.55. At this level the classifier is making meaningfully fewer errors than a human skimming posts quickly, which is the relevant comparison for a moderation or content surfacing tool. The Groq zero-shot baseline achieves macro F1 of approximately 0.58 on the test set, so 0.65 represents a meaningful improvement over what is achievable without any task-specific training.

**These criteria are objectively checkable:** At the end of the project, run `classification_report` on the held-out test set and compare macro F1 to 0.60 (minimum threshold) and 0.65 (deployment threshold). The answer is binary — either the numbers are above the threshold or they are not.

---

## AI Tool Plan

### Failure Analysis

After training and generating the list of wrong predictions from the fine-tuned model, the following process will be used:

1. Export the full list of wrong predictions as a table: index, true label, predicted label, confidence, text.
2. Give that table to Claude with the prompt: "These are wrong predictions from a 4-class text classifier trained on r/nba posts. Identify the top 3 patterns in the errors — what types of posts is the model consistently confusing, and what linguistic features do the misclassified examples share?"
3. Treat Claude's response as a hypothesis list, not a conclusion. Manually read through the examples for each suggested pattern to verify it independently.
4. A pattern is included in the README only if it can be confirmed by reading the examples, not just because Claude suggested it.

Specific things to look for: whether errors cluster at specific label pair boundaries (e.g., most errors are `hot_take` predicted as `analysis`), whether low-confidence predictions account for a disproportionate share of errors, and whether the errors correlate with post length, presence of statistics, or whether the post is a comment vs. a top-level post. These patterns directly inform the "what the model learned vs. what I intended it to learn" reflection required in the evaluation report.
