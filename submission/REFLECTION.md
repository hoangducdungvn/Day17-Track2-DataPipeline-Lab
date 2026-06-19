# Reflection — Day 17 (≤ 200 words)

Answer briefly, in your own words. This is graded on reasoning, not length.

1. **The flywheel.** Day 13 emitted agent traces; today you turned them into an
   eval set and DPO pairs that Day 22 will train on. Which step in
   `traces → Bronze → datasets` would break most silently in production if you
   got it wrong — and how would you detect it?

2. **Decontamination.** Your run dropped 2 of 3 preference pairs because their
   prompts were in the eval set. What concretely goes wrong if you *skip* this
   step and train on those pairs? How would the lie show up in your metrics?

3. **Point-in-time.** The naive join leaked a future `lifetime_spend` into the
   training row. Describe one feature in a system you know that would be
   dangerous to join without an `ASOF`/point-in-time guard.

4. **Graph vs vector.** From `kg_demo.py`, name one question the knowledge graph
   answers well that flat chunk retrieval (`embed.py`) would struggle with, and
   one where the graph is overkill.

1. **The flywheel:** The decontamination step breaks most silently. If it fails, training data leaks into the eval set without throwing any loud errors. We can detect this by adding a pipeline test that measures n-gram or embedding similarity between the final train and eval sets.
2. **Decontamination:** Skipping this causes the model to memorize the test set. The metrics would "lie" by showing artificially high win-rates or accuracy during evaluation, giving false confidence while real-world performance on unseen user queries remains poor.
3. **Point-in-time:** In a credit card fraud detection system, joining the "current account balance" instead of the balance *exactly at the transaction time* would leak future information (e.g., the balance dropping due to the fraud itself), making the model useless in production.
4. **Graph vs vector:** A knowledge graph excels at multi-hop reasoning across scattered facts, e.g., "Which warehouse stores items that have a 30-day warranty?". However, it is overkill for simple, localized fact lookups like "What is the support email?", where flat vector retrieval is cheaper and faster.
