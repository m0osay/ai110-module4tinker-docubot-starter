# DocuBot Model Card

This model card is a short reflection on your DocuBot system. Fill it out after you have implemented retrieval and experimented with all three modes:

1. Naive LLM over full docs  
2. Retrieval only  
3. RAG (retrieval plus LLM)

Use clear, honest descriptions. It is fine if your system is imperfect.

---

## 1. System Overview

**What is DocuBot trying to do?**

DocuBot is a documentation assistant that answers developer questions about a codebase using a local set of `.md` and `.txt` files. It supports three modes with increasing sophistication: naive LLM generation, retrieval only, and retrieval-augmented generation (RAG). The goal is to answer questions accurately while refusing when the docs don't support a confident answer.

**What inputs does DocuBot take?**

- A natural language question from the user
- A folder of documentation files (`docs/`) loaded at startup
- An optional `GEMINI_API_KEY` environment variable to enable LLM modes

**What outputs does DocuBot produce?**

- Mode 1 (Naive): A Gemini-generated answer with no grounding in the actual docs
- Mode 2 (Retrieval only): Raw matched document snippets ranked by relevance
- Mode 3 (RAG): A Gemini-generated answer grounded only in the retrieved snippets

---

## 2. Retrieval Design

**How does your retrieval system work?**

- **Indexing:** Each document is tokenized by splitting on whitespace and stripping common punctuation. Each token is mapped to the list of filenames that contain it, building an inverted index.
- **Scoring:** For a given query, each query word is checked for presence in the document text (case-insensitive substring match). The score is the count of matching query words.
- **Selection:** Documents are ranked by score descending. Only documents meeting a minimum threshold are returned (see guardrails). The top `k=3` are passed to the answer function.

**What tradeoffs did you make?**

- **Simplicity over precision:** Word overlap is fast and easy to understand, but it doesn't understand synonyms, context, or sentence meaning. "generate token" and "create auth key" would score 0 against each other even though they mean the same thing.
- **Whole-doc retrieval:** The system returns entire documents rather than paragraphs or sentences. This means relevant answers are often buried inside large irrelevant chunks.

---

## 3. Use of the LLM (Gemini)

**When does DocuBot call the LLM and when does it not?**

- **Naive LLM mode:** Calls Gemini with only the question — the docs are not used at all. The model answers from its own training data.
- **Retrieval only mode:** Never calls the LLM. Returns raw matched doc text directly to the user.
- **RAG mode:** First retrieves relevant snippets using the index, then passes those snippets to Gemini with strict instructions to answer only from the provided context.

**What instructions do you give the LLM to keep it grounded?**

In RAG mode, the prompt tells Gemini to:
- Use only the information in the provided snippets
- Not invent functions, endpoints, or config values
- Reply with "I do not know based on the docs I have." if the snippets are insufficient
- Mention which files it relied on when it does answer

---

## 4. Experiments and Comparisons

Tested with three queries. Identical wording across all modes.

| Query | Naive LLM: helpful or harmful? | Retrieval only: helpful or harmful? | RAG: helpful or harmful? | Notes |
|-------|-------------------------------|--------------------------------------|---------------------------|-------|
| Where is the auth token generated? | Harmful — answered generically about JWT patterns in any app, not this codebase | Harmful — returned SETUP.md (low-quality match) instead of AUTH.md | **Helpful** — correctly cited `generate_access_token` in `auth_utils.py` from AUTH.md | Retrieval scoring failed here; "token" matched SETUP.md more broadly |
| What does the /api/projects/<project_id> route return? | Partially helpful — guessed a plausible JSON shape, but invented field names not in the docs | **Helpful** — returned API_REFERENCE.md which has the exact response schema | **Helpful** — gave a clean, accurate answer citing API_REFERENCE.md with the exact fields | RAG and retrieval both worked well here |
| Is there any mention of payment processing in these docs? | Harmful — asked for more context instead of refusing, did not use the docs at all | Harmful — returned DATABASE.md due to weak word overlap; no actual payment info | **Helpful** — correctly returned "I do not know based on the docs I have." | RAG guardrail worked; retrieval mode failed by returning a false positive |

**What patterns did you notice?**

- **Naive LLM looks impressive but untrustworthy:** For the `/api/projects` question, it guessed a convincing JSON response with reasonable field names — but some were invented. A developer could copy that and be misled. The confidence is the danger.
- **Retrieval only is accurate but hard to interpret:** When it works (like the project endpoint question), it dumps the right file but gives you 800+ words when 5 sentences would do. Users have to read and filter it themselves.
- **RAG balances clarity and evidence:** All three RAG answers were more focused and readable than retrieval, and more grounded than naive LLM. The payment processing refusal was clean and correct.
- **RAG still fails:** The auth token question showed RAG working correctly *because* the right snippet was retrieved. But retrieval found the wrong file first (SETUP.md) and AUTH.md still happened to be in the top 3. If it hadn't been, RAG would have answered from the wrong context.

---

## 5. Failure Cases and Guardrails

**Failure case 1:**

- **Question:** "Where is the auth token generated?"
- **What happened:** Mode 2 (Retrieval only) returned SETUP.md as the top result because "token" and "auth" appear in the setup guide. The actual answer is in AUTH.md.
- **What should have happened:** AUTH.md should have ranked first. The word-overlap scorer treats all matches equally regardless of how central they are to the doc's topic.

**Failure case 2:**

- **Question:** "Is there any mention of payment processing in these docs?"
- **What happened:** Mode 2 returned DATABASE.md. "mention" and "these" matched common words in that doc. No payment content exists — this is a false positive.
- **What should have happened:** The system should have returned "I do not know" since no query-specific evidence was found. The guardrail caught this in Mode 3 but not Mode 2.

**When should DocuBot say "I do not know based on the docs I have"?**

1. When no document scores above the minimum evidence threshold (currently 2 matching query words) — the question is too unrelated to any doc.
2. When the retrieved snippets are returned to the LLM but don't contain information that directly answers the question — the LLM should refuse rather than guess.

**What guardrails did you implement?**

- **Minimum evidence threshold:** `retrieve()` requires a score of at least `MIN_EVIDENCE_SCORE = 2` (both modes 2 and 3 benefit from this). Short queries (fewer words than the threshold) require all their words to match.
- **Score filtering:** Documents with score = 0 are excluded before ranking — no result is better than a wrong result.
- **LLM prompt refusal rule:** The RAG prompt explicitly instructs Gemini to say "I do not know based on the docs I have." when snippets are insufficient, rather than guessing.

---

## 6. Limitations and Future Improvements

**Current limitations**

1. **Whole-document retrieval:** The system returns entire files, not relevant paragraphs. Long docs bury the actual answer in noise.
2. **Word overlap only:** The scorer doesn't understand meaning. Synonyms, paraphrases, and related concepts score 0 if the exact words don't appear in the doc.
3. **No deduplication:** The same concept spread across multiple docs can fill all 3 top slots with redundant information, pushing out genuinely different answers.

**Future improvements**

1. **Chunk documents into paragraphs or sections** instead of returning whole files. This would make retrieval results shorter and more focused.
2. **Use embedding-based similarity** (e.g., sentence-transformers) instead of word overlap. Semantic search would handle synonyms and phrasing variations correctly.
3. **Add a confidence score to RAG output** so the user knows how much evidence was found before the LLM answered.

---

## 7. Responsible Use

**Where could this system cause real world harm if used carelessly?**

- Mode 1 (Naive LLM) can generate plausible-sounding but completely wrong answers about API behavior, auth flows, or database schemas. A developer following that output could introduce security bugs (e.g., incorrect token validation logic).
- Mode 2 (Retrieval only) can return the wrong doc confidently if word overlap gives a false positive — especially dangerous for security-sensitive topics like auth.
- In all modes, the system has no awareness of whether its docs are outdated. Stale documentation leads to confident wrong answers.

**What instructions would you give real developers who want to use DocuBot safely?**

- Always verify answers against the actual source file cited — treat DocuBot as a starting point, not a final authority.
- Do not use Naive LLM mode (Mode 1) for anything security-critical. It has no connection to your actual docs.
- Keep the `docs/` folder up to date. The system is only as reliable as the documentation it indexes.
- If DocuBot says "I do not know," trust it — a refusal is safer than a hallucinated answer.

---
