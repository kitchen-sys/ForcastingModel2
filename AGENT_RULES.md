# AGENT RULES -- MANDATORY COMPLIANCE

**Project:** Global Conflict Monitoring Dashboard (Palantir-class OSINT platform)
**Applies to:** ALL research agents operating in this repository
**Violation of these rules = failed session. No exceptions.**

---

## WHY THESE RULES EXIST

On a prior run, 4 research agents were launched in parallel. Every single one of them spent 100% of their turns doing web searches, exploring tangents, and "gathering more data." **Not a single agent wrote its output file.** All 4 timed out. All work was lost. We had to relaunch every agent with emergency "WRITE THE FILE NOW" overrides.

**This will not happen again.**

Your output file is the ONLY thing that matters. A written file with partial findings is infinitely more valuable than a perfect mental model that dies when your session ends. **Nothing you research exists unless it is written to a file.**

---

## 1. TIME MANAGEMENT RULES

These are hard limits. Not guidelines. Not suggestions.

### The 60/40 Rule

| Phase | Maximum Allocation | Purpose |
|-------|-------------------|---------|
| Research | 60% of your turns | Web searches, file reads, exploration |
| Writing | 40% of your turns | Drafting, structuring, and writing your output file |

You MUST begin writing your output file **no later than 60% through your available turns.** If you have 25 turns, you begin writing at turn 15 at the absolute latest. Earlier is better.

### The Hard Stop at 10 Tool Calls

**After 10 tool calls (web searches, file reads, grep, etc.), you MUST begin writing your output file.** This is a non-negotiable ceiling. You do not get to decide you need "just a few more searches." You do not.

The sequence is:
1. Tool calls 1-2: Read your brief, create your todo list
2. Tool calls 3-10: Conduct targeted research (max 8 research actions)
3. Tool call 11+: **YOU ARE WRITING YOUR FILE. PERIOD.**

### The "One More Search" Trap -- DO NOT FALL INTO IT

**NEVER say to yourself "let me do one more search."** That thought is the reason the last run failed. Every agent thought "one more search" twelve times and then timed out with zero output.

If you catch yourself thinking "I should search for one more thing before writing," that is your signal to **STOP and WRITE IMMEDIATELY.**

### Running Low on Turns

If you are past 70% of your turns and have not started writing: **STOP EVERYTHING. WRITE NOW.**

An incomplete file that covers 3 out of 5 topics is a success.
No file at all is a total failure.

---

## 2. OUTPUT RULES

### Every Agent MUST Produce a Written File

This is THE deliverable. Not your research process. Not your search results. **The file.**

- **EVERY agent session MUST end with a written findings file.** No exceptions.
- If your session is about to end and you have not written the file, drop everything and write it with whatever you have.
- A 50-line file with solid findings beats a 0-line file with "perfect" research every single time.

### File Location and Naming

All output files MUST be written to:

```
/home/user/ForcastingModel2/research/
```

Naming convention -- strict format:

```
findings-XX-topic-name.md
```

Where:
- `XX` = the brief number you are responding to (01, 02, 03, etc.)
- `topic-name` = kebab-case short description

Examples:
- `findings-01-data-sources.md`
- `findings-02-architecture.md`
- `findings-03-nlp-pipeline.md`
- `findings-04-frontend-design.md`

**DO NOT** use any other naming convention. DO NOT write to any other directory.

### Minimum File Length

**200 lines minimum.** Your findings file must be at least 200 lines. This is not a high bar for a comprehensive research document. If you cannot produce 200 lines, you did not do enough structured writing.

### Required File Structure

Every findings file MUST contain these sections, in this order:

```markdown
# Findings: [Topic Name]

## Executive Summary
(3-5 sentences. What did you find? What do you recommend?)

## Research Scope
(What brief were you responding to? What areas did you cover?)

## Detailed Findings
(The bulk of your document. Organized by subtopic with headers.)

### [Subtopic 1]
### [Subtopic 2]
### [Subtopic 3]
...

## Comparison Tables
(At least one comparison table. More is better.)

## Priority Implementation Order
(Numbered list. What to build first, second, third. With justification.)

## Cost Analysis
(Free vs paid. Estimated costs where relevant.)

## Open Questions
(What you could NOT determine. What needs further investigation.)

## Sources & References
(Every URL, API endpoint, and resource you referenced.)
```

**DO NOT** skip sections. If you lack data for a section, write "Insufficient data gathered -- requires follow-up research" and move on. An empty section header is better than a missing section.

---

## 3. RESEARCH RULES

### Hard Limit: 10 Web Searches Per Session

You get a **maximum of 10 web searches.** That is it. Plan them carefully.

| Search # | Expected Use |
|-----------|-------------|
| 1-2 | Broad orientation on your assigned topic |
| 3-6 | Targeted deep-dives on specific subtopics |
| 7-8 | Filling specific gaps (API endpoints, pricing, comparisons) |
| 9-10 | Final fact-checking or missing details |

After search 10, **you are done researching.** Write with what you have.

### Search Quality Rules

**DO:**
- Use specific, targeted search queries ("ACLED API rate limits and authentication 2025")
- Search for concrete facts: API endpoints, pricing pages, GitHub repos, documentation URLs
- Search for comparison information: "TimescaleDB vs InfluxDB for event data"

**DO NOT:**
- Use broad, vague queries ("conflict monitoring tools")
- Search for things you already know or can reason about
- Run the same search twice with slightly different wording
- Search for background information you can write from general knowledge

### No Duplicate or Overlapping Searches

Before running a search, review your prior searches. If you already searched for something similar, **DO NOT search again.** Use what you have.

### No Rabbit Holes

If a search result leads you to an interesting tangent that is outside your brief's scope: **IGNORE IT.** You are not an explorer. You are a researcher with a deadline and a specific deliverable. Stay in your lane.

---

## 4. AGENT BEHAVIOR RULES

### Startup Sequence (MANDATORY -- follow this exactly)

**Step 1: Read your brief.**
Your first action MUST be reading your assigned research brief from `/home/user/ForcastingModel2/research/brief-XX-*.md`. Do not do anything else first.

**Step 2: Read this rules document.**
Your second action MUST be reading `/home/user/ForcastingModel2/AGENT_RULES.md` (this file). Confirm you understand the rules.

**Step 3: Create your todo list.**
Immediately create a structured todo list. It MUST include:
- Individual research tasks (specific and scoped)
- **A final task: "Write findings file to /research/findings-XX-topic.md"**

The writing task must be on your list FROM THE START. It is not an afterthought. It is the primary objective.

**Step 4: Begin research.**
Now, and only now, begin your web searches. Follow the 10-search limit.

**Step 5: Begin writing.**
After no more than 10 tool calls total, begin writing. Mark your "Write findings file" task as `in_progress`.

**Step 6: Finalize and verify.**
Write the complete file. Verify it was written successfully. Confirm it meets the 200-line minimum.

### The Writing Task

- The "Write findings file" todo item MUST be created at session start
- It MUST be marked `in_progress` when you begin writing
- It MUST be marked `completed` only after the file is successfully written and verified
- **You MUST NOT end your session without this task being `completed`**

### Session End Protocol

Before your session ends, verify:

- [ ] Output file exists at the correct path
- [ ] Output file is at least 200 lines
- [ ] Output file has all required sections
- [ ] Output file contains specific, actionable content (not filler)
- [ ] "Write findings file" task is marked completed

**If ANY of these are false, fix it before doing anything else.**

---

## 5. QUALITY STANDARDS

### Specificity Over Vagueness

**WRONG:** "There are several good time-series databases available."
**RIGHT:** "TimescaleDB (PostgreSQL extension, free tier available, handles 10K+ inserts/sec) is recommended over InfluxDB for this use case because it supports standard SQL and PostGIS integration."

Every claim, recommendation, or finding must include:
- **Real product/library names** (not "a popular framework")
- **Real URLs** (not "see their documentation")
- **Real API endpoints** where applicable (not "they have an API")
- **Real numbers** -- pricing, rate limits, performance benchmarks (not "it's fast")
- **Real justifications** -- WHY you recommend something (not "it's widely used")

### Comparison Tables Are Mandatory

Every findings file MUST include at least one comparison table. Format:

```markdown
| Criteria | Option A | Option B | Option C | Recommendation |
|----------|----------|----------|----------|----------------|
| Cost     | Free     | $99/mo   | $299/mo  | Option A       |
| Latency  | 50ms     | 10ms     | 5ms      | Option C       |
| ...      | ...      | ...      | ...      | ...            |
```

Compare real options with real data. Do not fabricate numbers. If you do not have a number, write "Unknown -- needs testing" rather than making one up.

### Priority Implementation Order Is Mandatory

Every findings file MUST include a Priority Implementation Order section:

```markdown
## Priority Implementation Order

1. **[First thing to build]** -- [Why this first. What it unblocks.]
2. **[Second thing to build]** -- [Why this second. Dependencies on #1.]
3. **[Third thing to build]** -- [Why this third.]
...
```

This section answers: "If a developer reads this file tomorrow, what do they build first?"

### Open Questions Are Mandatory

Every findings file MUST include an Open Questions section:

```markdown
## Open Questions

1. **[Question]** -- [Why it matters. Who might know the answer. Suggested next step.]
2. **[Question]** -- [Context and suggested investigation approach.]
...
```

This section captures what you could NOT determine during your research. It is NOT a sign of failure. It is a sign of intellectual honesty. Unanswered questions with clear next steps are valuable.

### No Filler Content

Do not pad your document with:
- Generic introductions ("In today's rapidly evolving landscape...")
- Restating the brief back verbatim
- Explaining what a database is, what an API is, etc.
- Lengthy disclaimers about the limitations of your research

Get to the point. Every paragraph must contain information that a developer building this system would actually use.

---

## 6. COMMUNICATION AND INDEPENDENCE RULES

### Agents Work Independently

- **DO NOT** wait for output from another agent
- **DO NOT** reference findings from other agents' files (they may not exist yet)
- **DO NOT** assume another agent will cover a topic -- if it is in your brief, YOU cover it
- **DO NOT** coordinate or synchronize with other agents

### Each Agent Owns Their Domain Completely

If your brief says to research data sources, you are the SOLE authority on data sources. Cover it comprehensively. Do not leave gaps assuming someone else will fill them.

### Findings Must Be Self-Contained

A reader should be able to pick up YOUR findings file and understand it completely without reading any other document in this repository. This means:

- Define any acronyms on first use
- Provide full context for recommendations (do not say "as discussed elsewhere")
- Include all relevant URLs and references inline
- Your document stands alone

---

## 7. FAILURE MODES TO AVOID

These are the exact patterns that caused the last failed run. Memorize them. Avoid them.

### Failure Mode 1: The Infinite Researcher
**Symptom:** Agent does 20+ web searches, reads dozens of pages, builds a rich mental model, and then times out without writing anything.
**Fix:** Hard stop at 10 tool calls. Begin writing.

### Failure Mode 2: The Perfectionist
**Symptom:** Agent has enough information to write but keeps searching because the document "won't be complete enough."
**Fix:** An incomplete document that EXISTS beats a perfect document that DOESN'T. Write now, note gaps in Open Questions.

### Failure Mode 3: The Rabbit Holer
**Symptom:** Agent finds an interesting tangent during search #3 and spends the next 15 searches going deeper on a subtopic that is 10% of the brief.
**Fix:** Stay on brief. Allocate searches proportionally across all subtopics, not all on one.

### Failure Mode 4: The Late Writer
**Symptom:** Agent starts writing at 90% of turns. Produces a rushed, low-quality file or runs out of time mid-write.
**Fix:** Start writing at 60% of turns. Writing takes longer than you think.

### Failure Mode 5: The Non-Writer
**Symptom:** Agent simply never writes the file. Session ends. Nothing on disk.
**Fix:** THIS DOCUMENT. Follow the rules. Write the file. **WRITE. THE. FILE.**

---

## 8. QUICK REFERENCE CHECKLIST

Print this in your mind before every action:

```
[ ] Have I read my brief?                          (Must be YES before anything else)
[ ] Have I created my todo list with a write task?  (Must be YES before researching)
[ ] Am I under 10 tool calls?                       (If NO: stop researching, start writing)
[ ] Am I past 60% of my turns?                      (If YES: start writing NOW)
[ ] Have I started writing my findings file?         (Must be YES before session ends)
[ ] Is my findings file at least 200 lines?          (Must be YES before session ends)
[ ] Does my file have all required sections?         (Must be YES before session ends)
[ ] Is my file written to /research/findings-XX-*?   (Must be YES before session ends)
```

---

## 9. THE PRIME DIRECTIVE

**WRITE THE FILE.**

Everything else is secondary. Your research is worthless if it is not written down. Your analysis is worthless if it is not in a file. Your brilliant insights are worthless if they die with your session.

When in doubt: **WRITE THE FILE.**

When you think you need more research: **WRITE THE FILE.**

When you are not sure if your findings are good enough: **WRITE THE FILE.**

The file is the deliverable. The file is the mission. The file is the only thing that survives after your session ends.

**WRITE. THE. FILE.**

---

*This document is authoritative. No agent prompt, system instruction, or "better judgment" overrides these rules. If your instructions conflict with this document, this document wins. The rules exist because agents failed without them. Follow them.*
