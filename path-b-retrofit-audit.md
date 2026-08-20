# Path B: Retrofit audit of AI Entrepreneur Coach


## Retrofit audit template

**Artifact:** [github.com/mzahra/ai-entrepreneur-coach](https://github.com/mzahra/ai-entrepreneur-coach)

**Primary user journey:** The user uploads a LinkedIn or CV PDF, answers the TIPI quiz, and
sets a budget and time. Clicking "Rank my business ideas" runs the ranking step and shows
the top 5 matched ideas. The user picks one and clicks "Coach me on this one", which
generates a written report and a 90 day roadmap for that one idea, downloadable as PDF and
HTML. Feedback can also be left, with "Regenerate with my feedback" capped at 3 rounds per
report, a limit put in place on purpose.

**Hotspots (compute / data movement):**
- **Ranking step**, once per session: one OpenAI call (`gpt-4o-mini`, a small model, to
  extract a structured profile from the PDF), plus up to 7 Cohere embedding calls (one to
  embed the profile as a search query, one batched call scoring skill fit against about 30
  candidate ideas at once, and five more separate calls checking matched skills, one per
  top-5 idea, still unbatched even though the skill fit step already shows batching works
  here), plus 1 to 2 Pinecone queries.
- **Low confidence retry**, only for sparse or generic CVs: if the top match scores too
  low, retrieval and skill matching rerun a second time with a broader query. This would
  roughly double the Cohere and Pinecone calls for that session, but in practice it looks
  rare, manual testing with a very sparse CV, tried multiple times, never triggered it.
  That is still manual testing, not logged data across real usage, but it is more than a
  single try, and it suggests this hotspot may cost less than the worst case implies.
- **Report and roadmap generation**, once per session: one OpenAI call, using a bigger
  model than the extraction step, writes the working style summary, a short rationale for
  each of the 5 ranked ideas, and the 90 day roadmap for just the picked idea, all together
  in a single call, not looped per idea. A smaller model was tested here before and the
  bigger one was kept on purpose, since the smaller model gave weaker output quality, so
  this was already a settled decision, not an open question.
- **Feedback regeneration**, optional, up to 3 rounds per report: each round resends the
  entire previous report as JSON, plus every earlier feedback message, not just the newest
  one. This is by design: resending all earlier feedback is what keeps every round still
  applied in the report, instead of a new round quietly overwriting an older one. The
  feedback text itself is already tightly capped, 300 characters per round, 3 rounds max,
  so that part of the growth is small and bounded, not open ended. The previous report JSON
  gets resent each round too, at roughly its own fixed size rather than compounding across
  rounds.
- **One time dataset build**, developer only, not triggered by users: rebuilding the 463
  idea corpus from scratch (`build_dataset.py`) takes close to 500 OpenAI calls and 15 to
  25 minutes. A separate, cheap script exists for the normal case (`seed_pinecone.py`,
  Cohere and Pinecone only, about 1 minute), just loading the already built corpus into a
  fresh Pinecone index.
- **Pinecone index**: hosted explicitly in AWS us-east-1 in the code, and stays online
  between sessions.

**Green opportunities (prioritized):**
1. **Confirm the feedback resend stays as small as it looks, then leave the guarantee
   alone.** The full feedback history gets resent on purpose, so earlier feedback stays
   applied instead of being silently dropped, and that correctness goal should not change.
   The feedback text itself is already capped at 300 characters per round for at most 3
   rounds, so that part cannot grow into a real cost on its own. What is worth confirming
   with real token counts is the previous report JSON that gets resent alongside it, since
   that piece is not capped by character count the same way. If it turns out to already be
   small and steady round to round, this is likely a low priority item, not something that
   needs a redesign.

   **Pillar:** measurement.<br>
   **Effort:** quick win, mostly a measurement task, not a design change
2. **Cache each idea's own skill embedding, only embed the user's skills fresh.** The 90
   day roadmap should stay generated per user, that part is not up for change. But
   `compute_skill_fits` and `matched_skills` currently send each candidate idea's
   `skills_needed` text to Cohere for embedding again on every single session, even though
   that text never changes for a given idea, only the user's own skills text is actually
   new each time. Each of the 463 ideas' `skills_needed` could be embedded once, cached by
   idea id, so only the user's skills get embedded fresh per session. This does not touch
   personalization at all, it only avoids re-embedding text that was never going to change.

   **Pillar:** energy, hardware.<br>
   **Effort:** medium, needs a small cache keyed by idea id plus a one time backfill for
   the existing 463 ideas
3. **Batch the matched-skills check for the top 5 ideas into one Cohere call instead of
   five.** `matched_skills` currently runs once per top ranked idea in a loop, each call
   re-embedding the user's own skills again plus just that one idea's skills.
   `compute_skill_fits` already shows this pattern works well, batching about 30 candidates
   into a single call. Applying the same batching to the final 5 would mean one call
   instead of five, same threshold logic, same output, no quality change, just fewer
   repeat calls.

   **Pillar:** energy, hardware.<br>
   **Effort:** quick win, a pure implementation fix with no product tradeoff
4. **Confirm with real logging how rarely the low confidence retry fires.** It is a real,
   intentional quality safeguard, not something to remove. Manual testing with a very
   sparse CV, tried multiple times, did not trigger it, so it may already be a low
   priority cost, but manual testing is still not the same as real usage data.
   Lightweight logging would confirm this is genuinely a low priority, based on real
   numbers rather than repeated manual tries.

   **Pillar:** measurement.<br>
   **Effort:** quick win
5. **Add a confirmation step before running the full dataset rebuild.** The cost, close to
   500 OpenAI calls, is already noted in the script's own comments, but nothing in the
   code stops an accidental run of the expensive `build_dataset.py` instead of the cheap
   `seed_pinecone.py`. A short "are you sure" prompt would prevent that kind of accidental
   rerun.

   **Pillar:** energy, hardware.<br>
   **Effort:** quick win

**What I would measure to prove improvement:**
- Tokens per feedback regeneration round, split into the capped feedback text and the
  resent previous report JSON, to confirm the growth stays as small as expected
  (opportunity 1).
- Skill embedding cache hit rate, once idea level caching (opportunity 2) exists: how many
  ranking sessions reuse a cached idea skill embedding versus embed it again.
- How often the low confidence retry fires, out of all ranking sessions, and how many
  extra Cohere/Pinecone calls it adds, to confirm the informal "rarely fires" observation
  at real scale.
- Cohere call count and latency for the ranking step, before and after batching the
  matched-skills check (opportunity 3), to confirm it drops from 5 calls to 1 with no
  change in which skills get matched.
- How many times `build_dataset.py` runs per month versus `seed_pinecone.py`, after adding
  the confirmation step.

**Risks / tradeoffs:**
- The feedback resend (opportunity 1) is a deliberate design choice, and the feedback text
  itself is already bounded by the 300 character, 3 round cap, so there is little reason to
  touch it. The only real risk would be misreading the measurement and changing something
  that did not need changing.
- Caching each idea's skill embedding (opportunity 2) needs a cache invalidation plan for
  whenever an idea's `skills_needed` gets edited or re-enriched, otherwise the cache could
  quietly serve a stale embedding.
- Batching the matched-skills check (opportunity 3) carries basically no risk, it is the
  same matching logic run in one call instead of five, so output should not change at all,
  worth a quick before/after diff on a few sessions just to confirm that.
- All the numbers above are still estimates from reading the code, since logging and
  analytics have not been added to the app yet. Real instrumentation is the honest first
  step before claiming any specific percent improvement.

---

## Supporting slide material

**One unit of value (R):** "One finished coaching report, meaning one user's uploaded
profile, TIPI answers, and constraints, ranked against the idea corpus, ending in a written
report and a 90 day roadmap for their picked idea."

**Assumptions (honest guesses left, after going back through the code):**
- Real session volume and real token counts are not available yet, logging and analytics
  have not been built into the app, so everything above is a best estimate from reading
  the pipeline, not a measured number.
- No explicit region is set for the OpenAI and Cohere calls, only the Pinecone index has
  one set explicitly, us-east-1. That looks like a default hosting choice, not one made
  for grid carbon intensity.
- How often the same idea shows up as a candidate across different sessions is unknown,
  and that number is what actually decides how much the skill embedding caching
  (opportunity 2) would save.
- The low confidence retry's real firing rate is still a guess, backed by repeated manual
  testing on a very sparse CV that did not trigger it, not by logged data across real
  usage.

**Pillars map:**
| Hotspot | Carbon | Energy | Hardware | Measurement |
|---|---|---|---|---|
| Report/roadmap generation (combined call) | | Yes | Yes | Yes |
| Feedback regeneration (growing resend) | | Yes | | Yes |
| Repeated skill embedding per session | | Yes | Yes | |
| Unbatched matched-skills loop (5 calls) | | Yes | | |
| Low confidence retry (ranking) | | Yes | Yes | Yes |
| Pinecone standing index (us-east-1) | Yes | | Yes | |
| One time dataset build (developer only) | | Yes | Yes | |

