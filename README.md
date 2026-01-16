# DnD-AI-Assistant
Project for building an AI assistant for transcribed DnD sessions in Discord. The aim is to provide story summaries in Discord, structure extracted information, provide a RAG query bot for discord, and more

---

## Current Plan

### Phase 0 — Foundations and repo setup (0.5–1 day)

**Goal:** make everything reproducible + automatable.

**Deliverables**

* Git repo with a clear structure:

  * `data/raw_audio/<session_id>/`
  * `data/transcripts/<session_id>/`
  * `data/outputs/<session_id>/`
  * `src/` for pipeline modules
  * `prompts/` for LLM prompts
* `session_id` convention (e.g., `2026-01-16_s01`)
* Speaker mapping config (Discord user → canonical name)

**Acceptance criteria**

* One command can run a “dry run” on a small sample audio file and create output folders.

---

### Phase 1 — Recording MVP (Craig multi-track) (1 session to validate)

**Goal:** reliable **per-speaker audio tracks** every week.

**Tasks**

* Add Craig to server; test join/leave workflow.
* Decide who is responsible for starting/stopping recording.
* Standardise how you store the downloaded audio files.

**Deliverables**

* A documented “how to record” checklist for the party.
* Raw multi-track audio saved under the session folder.

**Acceptance criteria**

* After a real session, you have **one audio file per speaker** + a combined mix.

---

### Phase 2 — Local transcription MVP (overnight batch) (1–2 days)

**Goal:** fully local transcription on your 3060 Ti with timestamps.

**Suggested method**

* Use **faster-whisper (CTranslate2)** on GPU for each speaker track.
* Process tracks **sequentially** (safe VRAM usage), optionally 2-at-a-time later.

**Deliverables**

* `turns.jsonl` canonical transcript format:

  * `{start, end, speaker, text}`
* `transcript.md` readable format:

  * timestamped dialogue blocks
* Basic CLI:

  * `python transcribe.py --session <id>`

**Acceptance criteria**

* For a 2–4h session, transcription completes overnight and attribution is correct (because tracks are per-speaker).

---

### Phase 3 — Post-processing outputs (no wiki yet) (1–3 days)

**Goal:** generate consistent outputs from the transcript (summary, outcomes, chapter) and store them locally.

**Outputs you generate**

1. **Strict session summary** (`summary.md`)
2. **Structured outcomes** (`outcomes.json`)

   * NPC changes, quests, loot, decisions, open threads, locations, timeline events
3. **Book chapter draft** (`chapter.md`)

**Key design choice**

* Make outcomes **structured JSON** with schema validation (Pydantic).
* Keep “chapter writing” separate so you can swap models later.

**Acceptance criteria**

* One command produces all three outputs for a session and passes JSON validation:

  * `python process_session.py --session <id>`

---

### Phase 4 — Discord publishing (low complexity) (0.5–1 day)

**Goal:** automatically post the summary to your Discord channel when processing is complete.

**Simplest method**

* Use a **Discord webhook** (no bot hosting required).
* Your pipeline POSTs:

  * short summary
  * key outcomes bullets
  * “open threads”
  * link placeholder for wiki (added later)

**Deliverables**

* `publish_to_discord.py`
* Message template you like

**Acceptance criteria**

* Running the pipeline ends with a clean Discord post in the right channel.

---

### Phase 5 — Notion wiki MVP (structure first, then automation) (2–4 days)

**Goal:** party-accessible wiki in Notion with minimal friction.

**Step A: manual structure**
Create these top-level pages/databases:

* Sessions (session log pages)
* NPCs
* Locations
* Quests
* Timeline
* Items/Loot (optional)
* “Book” (chapters)

**Step B: automated updates**
From `outcomes.json`, generate “wiki ops” (upserts):

* create/update NPC rows
* create/update quest rows
* append timeline events
* create a Session page with:

  * transcript link / attachment (optional)
  * summary
  * chapter

**Deliverables**

* `notion_client.py` + `apply_wiki_ops.py`
* Mapping file: schema fields → Notion properties

**Acceptance criteria**

* After processing a session, Notion updates correctly:

  * new session page created
  * NPC/Quest/Timeline updated

---

### Phase 6 — RAG groundwork (indexing + retrieval locally) (2–5 days)

**Goal:** build a reliable retrieval layer before you build the Discord Q&A experience.

**Knowledge sources**

* Transcripts (chunked)
* Session summaries + outcomes JSON (high-signal)
* Notion exports (or your local “wiki state” JSON)

**Chunking strategy (works well for D&D)**

* Chunk transcripts by time windows (e.g., 3–5 minutes) or scene blocks.
* Store metadata: `{session_id, start, end, speaker(s), tags}`

**Deliverables**

* `indexer.py` that builds a local vector store (FAISS/Chroma)
* `retriever.py` that returns top-k chunks + citations (session/time)

**Acceptance criteria**

* Given “What happened to X?” you can retrieve relevant passages with timestamps.

---

### Phase 7 — Discord Q&A bot (RAG + guardrails) (3–7 days)

**Goal:** party can ask questions and get grounded answers.

**Minimum commands**

* `/ask <question>` → RAG answer with citations
* `/recap last` → latest summary
* `/threads` → open plot hooks
* `/npc <name>` → pulls from Notion/wiki state

**Guardrails (strongly recommended)**

* Require citations for factual answers (“Session 08 @ 01:12:34…”)
* If retrieval confidence is low, bot says it doesn’t know.
* Keep responses concise by default.

**Acceptance criteria**

* Bot answers correctly for common questions and doesn’t invent canon.

---

### Phase 8 — Polish and reliability (ongoing)

**Goal:** make it “set and forget”.

Add:

* automatic glossary injection (NPC names, locations) to improve transcription + LLM
* retry logic and logging
* HTML/PDF “session pack” export
* optional: re-transcribe with a better model later

---

## Cost control under £30/month

* Transcription: **£0** (local GPU)
* Notion: likely **£0** (depending how you set up members/guests)
* OpenAI API: typically **single digits to teens** monthly if you:

  * use efficient models for summary/outcomes/wiki ops
  * optionally use a better model only for the chapter



