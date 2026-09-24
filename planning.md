# TakeMeter Planning

## Project Goal

TakeMeter will classify the **evidence grounding** of comments in Reddit's r/AmItheAsshole (AITA). The classifier will judge how well a comment's reasoning is supported by the information available in the original post and, for replies, the relevant parent-comment context.

The classification unit is one Reddit comment. For depth-0 comments, the context is the original post. For depth-1 and depth-2 comments, the context also includes the comment chain above the target comment.

---

## 1. Community

I chose **r/AmItheAsshole (AITA)** because it is an active, text-heavy community built around people presenting a situation and other users evaluating it. This makes it a strong fit for a discourse-quality classification task because commenters regularly make claims about what happened, why people behaved the way they did, and what should happen next.

The discourse is varied enough to make evidence grounding nontrivial. AITA comments include direct references to details in the post, reasonable inferences, personal anecdotes, general knowledge, moral judgments, legal claims, predictions, assumptions about motives, jokes, and very short verdicts such as "NTA." Two comments can reach the same verdict while differing greatly in how well their reasoning is grounded in the evidence that was actually provided. That variation is what TakeMeter is designed to capture.

My current dataset uses comments from two AITA threads:

1. **Post `ocx94s`** — a dispute about a father putting a lock on his daughter's door after her cousins repeatedly took her belongings.
2. **Post `gr8bp3`** — a dispute about a girlfriend having a 1967 Impala restoration project removed while the owner was away.

I am using comments at **depths 0, 1, and 2**:

- **Depth 0:** a top-level comment that replies directly to the original Reddit post.
- **Depth 1:** a direct reply to a depth-0 comment.
- **Depth 2:** a direct reply to a depth-1 comment.

This keeps the dataset focused on the main discussion and the first two layers of conversation below it, rather than much deeper reply chains that may drift away from the original situation. Moderator comments and comments whose text is `[deleted]` or `[removed]` are excluded.

---

## 2. Labels

### Grounded

A comment is **Grounded** when its main reasoning is supported by explicit details from the post or conversation, a reasonable inference from those details, or relevant general knowledge without inventing case-specific facts.

Examples:

- "Taking Zoey's makeup without permission and ruining it is not normal borrowing, so giving her a lock is a reasonable response."
- "She arranged for his car and parts to be removed while he was away, so seeking repayment for the loss is supported by what happened."

### Partially Grounded

A comment is **Partially Grounded** when it has a substantial evidence-based core but also includes one or more unsupported assumptions, predictions, motives, or extrapolations that materially contribute to the argument.

Examples:

- "The father is right to protect Zoey, and she will probably remember when she is older that he was the parent who supported her."
- "The girlfriend clearly ignored his property rights, and if they had eventually married she probably would have become even more controlling."

### Speculative

A comment is **Speculative** when its central argument depends primarily on information that the post or conversation does not establish, especially invented motives, hidden history, personality claims, causal explanations, or confident predictions.

Examples:

- "Sammy's attitude toward boundaries is probably the reason his marriage ended."
- "The scrapyard workers definitely knew the car was stolen and planned from the beginning to resell the parts."

### Not Assessable

A comment is **Not Assessable** when it does not contain enough substantive reasoning to evaluate evidence grounding, such as a bare verdict, joke, short reaction, or simple agreement.

Examples:

- "NTA."
- "Exactly!!"

### What the labels do not measure

Evidence grounding is **not** the same as whether I agree with the comment, whether the AITA verdict is correct, whether the comment is polite, or whether the comment is long. A comment can make a debatable moral judgment and still be Grounded if the reasoning is tied to the available evidence.

Relevant general knowledge also does not automatically make a comment speculative. The main concern is **unsupported case-specific claims**.

---

## 3. Hard Edge Cases

The hardest boundary is **Grounded vs. Partially Grounded**. A comment may closely use the post's evidence but add a prediction or assumption that the post does not establish.

For example, a comment might correctly say that Zoey's father defended her privacy and then add that Zoey will remember this for the rest of her life. The first part is grounded; the second is a prediction. I would label the full comment Partially Grounded because the unsupported prediction contributes to the overall argument.

The second difficult boundary is **Partially Grounded vs. Speculative**. I will use a removal test:

- If I remove the unsupported assumption and the main evidence-based argument still substantially stands, the label is **Partially Grounded**.
- If I remove the unsupported assumption and the main argument mostly collapses, the label is **Speculative**.

I will also distinguish **Not Assessable** from weakly grounded comments. A short comment is not automatically Not Assessable. If it makes a substantive claim that can be checked against the context, it can still receive one of the grounding labels. Not Assessable is reserved for comments with too little reasoning to evaluate.

For replies, I will read the relevant conversation chain before labeling:

- **Depth 0:** the target comment is replying directly to the original post, so I evaluate it using the original post as context.
- **Depth 1:** the target comment is replying to a depth-0 comment, so I evaluate it using the original post plus that parent comment.
- **Depth 2:** the target comment is replying to a depth-1 comment, so I evaluate it using the original post, the depth-0 comment, and the depth-1 parent comment.

If a comment still feels genuinely ambiguous after applying these rules, I will flag it for a second annotation pass rather than changing the label definitions ad hoc. I will keep a small review log of these borderline examples and resolve them using the same decision rules across the dataset.

---

## 4. Data Collection Plan

The data comes from public comments on r/AmItheAsshole. I am collecting the original post context plus comments from depths 0–2.

Each comment record contains:

- `post_id`
- `post_title`
- `comment_id`
- `comment_depth`
- `parent_comment_id`
- `comment_text`
- `grounding_label`

The full original post text can be stored once per thread rather than repeated for every comment.

### Current dataset

I currently have **223 usable comments**:

| Label | Count |
| --- | ---: |
| Grounded | 104 |
| Partially Grounded | 60 |
| Speculative | 14 |
| Not Assessable | 45 |
| **Total** | **223** |

I do not want to force the dataset into an artificial 25/25/25/25 balance because the natural distribution of discourse is part of the problem. However, I want enough examples of every class for the model and evaluation to be meaningful. My working target is **at least 30 examples per label** when feasible.

The current Speculative class is underrepresented. If a class remains underrepresented after 200 examples, I will first collect additional depth-0/1/2 comments from another AITA thread rather than relabeling examples or manufacturing balance. If the class remains naturally rare, I will preserve the real distribution and address the imbalance during training with class-aware methods such as class weighting, while reporting per-class metrics and macro-averaged metrics.

### Split strategy

I will avoid putting a comment and its direct parent/child replies on opposite sides of the train/test split because those examples share too much language and context. I will group comments by conversation branch when splitting.

A stronger final evaluation would hold out entire AITA threads so the model is tested on stories it has never seen. Because the current dataset has only two source posts, I will either add at least one additional thread before final evaluation or clearly state the limitation if I must use branch-grouped splits within the current two threads.

---

## 5. Evaluation Metrics

### Primary metric: Macro F1

**Macro F1** will be my primary metric because the four classes are not evenly distributed. It computes F1 separately for each class and then gives every class equal weight. This prevents the large Grounded class from dominating the evaluation and hiding poor performance on the smaller Speculative class.

### Per-class precision, recall, and F1

I will report **precision, recall, and F1 for each label**.

These are especially important for this task because different mistakes have different meanings. For example, low precision on Speculative would mean the classifier frequently accuses comments of relying on unsupported assumptions when they are actually grounded. Low recall on Speculative would mean it misses many comments whose arguments depend on unsupported claims.

### Confusion matrix

I will use a **confusion matrix** to identify which boundaries the model struggles with. I expect the most meaningful confusions to be:

- Grounded ↔ Partially Grounded
- Partially Grounded ↔ Speculative
- Not Assessable ↔ short substantive comments

This will directly support the failure analysis.

### Accuracy

I will still report **accuracy**, but only as a secondary metric. Because Grounded is the largest class, a model could achieve a respectable-looking accuracy while performing poorly on the rarer labels.

### Balanced accuracy

I will also report **balanced accuracy** as a secondary summary because it gives equal importance to recall on each class and provides another check against majority-class bias.

---

## 6. Definition of Success

I want the success criteria to be specific enough that I can objectively decide whether the project worked.

### Successful course-project classifier

I will consider the classifier successful for this project if it achieves all of the following on held-out data:

- **Macro F1 ≥ 0.75**
- **No individual class F1 below 0.60**
- **Speculative precision ≥ 0.75**
- The confusion matrix shows that errors are concentrated mostly at adjacent boundaries such as Grounded vs. Partially Grounded rather than systematic collapse into one majority class.

These thresholds matter more than raw accuracy because the goal is to distinguish reasoning quality across all four categories.

### Good enough for a real community tool

For a real community-facing tool, I would use the classifier as an **assistive signal**, not an automatic moderation decision. I would want stronger performance before surfacing labels to users:

- **Macro F1 ≥ 0.80**
- **Every class F1 ≥ 0.70**
- **Speculative precision ≥ 0.85**

The higher precision requirement for Speculative is intentional. A false Speculative label could unfairly suggest that a user's argument is based on invented or unsupported information, so I would rather leave some uncertain comments unflagged than confidently apply that label incorrectly.

If the model does not meet these thresholds, the useful outcome of the project will instead be identifying which kinds of discourse cannot yet be reliably separated with the current label definitions, context representation, or amount of training data.

---

## 7. AI Tool Plan

### Label stress-testing

Before treating the label definitions as final, I will give ChatGPT the four definitions and the edge-case rules and ask it to generate **5–10 synthetic AITA-style comments that sit near label boundaries**.

I will specifically request examples near:

- Grounded vs. Partially Grounded
- Partially Grounded vs. Speculative
- Partially Grounded vs. Not Assessable

I will classify the generated examples using my written rules. If I cannot explain why an example belongs to one label rather than another, I will revise the definitions or decision rules before using them as the final annotation guide.

The purpose of this step is not to add synthetic examples to the training dataset. It is only to stress-test the label definitions.

### Annotation assistance

The annotation process will be **jointly completed by AI and me**. ChatGPT will review each example using the label definitions and conversation context, and I will independently review the same examples rather than treating the AI output as automatic ground truth.

For depth-1 and depth-2 replies, both the AI and I will consider the relevant parent-comment chain before assigning a label. When my label and the AI label agree, I will keep that annotation. When we disagree, I will revisit the example using the written decision rules—especially the Grounded vs. Partially Grounded and Partially Grounded vs. Speculative boundaries—and make the final annotation myself.

I will disclose that AI was used as a co-annotator in the project write-up. I will also keep track of disagreements or changed labels so I can describe how AI assistance affected the annotation process.

The model-training input will not include Reddit scores, usernames, or other popularity/user metadata that could allow the classifier to learn shortcuts unrelated to evidence grounding.

### Failure analysis

After evaluation, I will give ChatGPT the set of incorrect predictions along with the true label, predicted label, target comment, and necessary context. I will ask it to identify recurring error patterns.

I will specifically look for patterns involving:

- unsupported motive attribution
- future predictions
- personal anecdotes
- legal or technical claims
- sarcasm and jokes
- very short comments
- long comments containing both grounded and speculative claims
- failures caused by missing parent-comment context
- confusion between Partially Grounded and Speculative

I will not accept the AI's pattern summary automatically. I will manually inspect examples from each proposed pattern, compare them with the confusion matrix and annotation rules, and only report a pattern if I can verify it in the actual errors.
