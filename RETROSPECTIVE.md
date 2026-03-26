<div align="center">

# 🧠 Daily Interview Digest
### Project Retrospective — Google Apps Script + Claude API
*March 2026 · Built together*

</div>

---

## 1. Approach & Reasoning

The goal was simple: send yourself 40 interview questions every day at 6AM IST, fully automated, zero manual effort. The architecture mirrors how production scheduled jobs work — a time-based trigger fires a function, which calls an external API, builds an HTML payload, delivers via email, and saves to Drive.

The approach was deliberately layered: topic banks were hard-coded (giving full control over content quality), seeded randomness was used to make daily selection reproducible (same seed = same questions for a given date), and Claude was called separately for concept questions vs. coding problems because the two require very different prompt structures.

---

## 2. Rejected Alternatives

- **Google Sheet as a question bank** — rejected because structured arrays in code are easier to version, diff, and extend than spreadsheet rows.
- **Single Claude prompt for all 40 questions** — rejected because mixing concept + coding in one prompt degrades quality. The two system prompts are meaningfully different.
- **`DocumentApp.create()` + `setText(body)`** — rejected after discovering `setText()` writes HTML tags as raw text. Switched to `Utilities.newBlob()` with `text/html` MIME type, which saves as a proper `.html` file that renders correctly in the browser.
- **Monthly auto-reset of the serial counter** — rejected after discussion. `resetCounter()` is kept as a manual-only utility for test runs, not scheduled.

---

## 3. How the Parts Connect

The data flow is linear and clean:

```
getTodaySeed_()
  → selectTopicsForToday_()
  → generateConceptQuestions_() ×3 + generateCodingProblems_()
  → buildQuestionEmail_()
  → GmailApp.sendEmail() + createDigestAndAnswerFiles()
```

- **`seededSample_()`** uses a Linear Congruential Generator (LCG) — a deterministic PRNG seeded by the date integer (e.g. `20260326`). Different offsets (`+0`, `+1`, `+2`, `+3`) for each topic bank ensure independent shuffles.
- **`callClaude_()`** is the single API boundary — all generation goes through it. It handles key retrieval, payload construction, HTTP call, error checking, and text extraction.
- **`buildQuestionEmail_()`** contains two nested functions: `conceptSection()` and `codingSection()`. These are inner functions — pure string builders, they close over nothing.
- **`createDigestAndAnswerFiles()`** is wired into `sendQuestions()` *before* `GmailApp.sendEmail()` — Drive save happens first so a Drive failure doesn't silently swallow the error.

---

## 4. Tools Used & Why

| Tool | Why |
|---|---|
| **Google Apps Script** | Zero infrastructure. Runs entirely in Google's cloud. No server, no cron job, no deployment. |
| **`ScriptApp.newTrigger()`** | Time-based triggers are the Apps Script equivalent of cron — fire-and-forget, IST timezone aware. |
| **`PropertiesService`** | Key-value store for secrets (API key, email) and state (serial counter). Never hardcode secrets. |
| **`UrlFetchApp.fetch()`** | The only way to make outbound HTTP calls from Apps Script. Used for the Anthropic API call. |
| **`Utilities.newBlob()`** | Creates an in-memory file blob with MIME type. The correct way to save HTML to Drive (not `DocumentApp`). |
| **`claude-sonnet-4-6`** | Best balance of quality and speed for question generation. `max_tokens: 8192` gives room for long coding problems. |
| **LCG (`seededSample_()`)** | Deterministic shuffle — same date always yields same topics. Useful for debugging and reproducibility. |

---

## 5. Tradeoffs

- **HTML email vs plain text** — HTML looks great but some email clients strip inline styles. Plain text is more reliable but unreadable for 40 questions. HTML won — the formatting is central to usability.
- **Separate Claude calls per section vs one big call** — 4 API calls instead of 1 means higher latency (~2 minutes total) and cost, but far better quality. Mixing concept + coding in one prompt is a known quality degradation pattern.
- **`.html` file in Drive vs Google Doc** — `.html` renders the digest beautifully when opened in a browser. A Google Doc would require parsing and re-formatting the entire HTML structure — not worth it.
- **Seeded randomness vs truly random** — The LCG ensures today's digest is the same if you run `testDigest()` twice. Tradeoff: if you want true variety on re-runs during the same day, it's a limitation.
- **Hard-coded folder ID vs dynamic lookup** — Hardcoding is fragile (breaks if you move the folder) but simple. A lookup by folder name would be more robust but adds a `DriveApp` call on every run.

---

## 6. Mistakes & Fixes

| # | Mistake | Why It Happened | Fix |
|---|---|---|---|
| 1 | `folder.createFile()` with `MimeType.GOOGLE_DOCS` | Assumed MimeType controls conversion, not just labeling | Switched to `Utilities.newBlob()` — creates a true `.html` file |
| 2 | `createDigestAndAnswerFiles()` never called | Implemented but forgot to wire into `sendQuestions()` | Added call before `GmailApp.sendEmail()` |
| 3 | `GmailApp.sendEmail()` with 5 args | Copy-paste error — subject string duplicated as 3rd arg | Removed extra argument, used `subject` variable |
| 4 | DriveApp permissions error | Apps Script didn't auto-detect `auth/drive` scope | Added explicit `oauthScopes` in `appsscript.json` manifest |
| 5 | `DocumentApp.create()` + `setText(html)` | Assumed `setText()` would render HTML | Switched to `newBlob()` — `.html` file renders in browser |
| 6 | `codingSection()` showing no problems | Regex relied on blank line before problem 1 (none exists) | Prepend `\n` to `rawText` before split; added fallback splitter |

---

## 7. Future Pitfalls

- **Apps Script execution timeout is 6 minutes.** With 4 Claude calls (each potentially 30–60s), you're close to the limit. If any call is slow, the script times out and Drive/email never runs. Mitigation: add per-call timing logs; consider splitting into two triggers if needed.
- **The Anthropic API key is stored in Script Properties** — this is correct. Never hardcode it in the script body. If you share the project, Script Properties are not shared with collaborators.
- **The folder ID is hardcoded.** If you move or delete the Drive folder, `createDigestAndAnswerFiles()` will throw and the email won't send either (Drive save is before email). Add a `try/catch` around the Drive block if you want email-resilience independent of Drive.
- **The LCG seed is date-based.** Two runs on the same day produce identical question sets. This is a feature during testing but means `testDigest()` is not a true "fresh run" simulator for production.
- **`codingSection()` has a dual-fallback now**, but if Claude returns a completely unexpected format (e.g. markdown headers instead of numbered problems), the raw text fallback will dump HTML-unescaped content. Add a basic HTML escape for the fallback path.

---

## 8. Expert vs Beginner Perspective

### What a beginner sees
A script that sends an email every day. The hard part was the permissions error.

### What an expert sees

- **The LCG is a deliberate design choice** — stateless, deterministic, no external dependencies. `Math.random()` would break reproducibility.
- **Separating system prompts per task type** (`CONCEPT_SYSTEM` vs `CODING_SYSTEM`) is prompt engineering best practice — task-specific system prompts significantly outperform generic ones.
- **The Drive save-before-email ordering is intentional fault isolation** — if Drive fails, you see the error; if email fails after Drive succeeds, you still have the questions saved.
- **The `appsscript.json` manifest override is the correct production pattern** for Apps Script auth — relying on auto-detection is fine for demos, but explicit scopes are required for reliability.
- **The fallback chain in `codingSection()`** (primary regex → blank-line split → raw text dump) is defensive programming — never show a blank section when you can degrade gracefully.

---

## 9. Transferable Lessons

- **Permissions are architecture.** Before writing a single line of Apps Script that touches Drive, Gmail, or external APIs, declare your `oauthScopes` in `appsscript.json`. Debugging auth errors mid-run is painful.

- **MIME types label, they don't convert.** `MimeType.GOOGLE_DOCS` on a `createFile()` call doesn't create a Google Doc — it labels a text blob. Use the right API for the right job (`DocumentApp` for Docs, `newBlob` for raw files).

- **Seeded randomness is underused.** Whenever you need "different every day but reproducible for a given day", a simple LCG seeded by date is the right tool. No database, no state, no side effects.

- **Prompt separation is a quality multiplier.** One big prompt for 40 mixed questions is worse than 4 targeted prompts. The system prompt is the most important part of any Claude API call — treat it with the same care as production code.

- **Regex on LLM output needs defensive fallbacks.** LLMs don't guarantee output format consistency. Any parser for LLM output should have at least one fallback — a looser parse, or a raw text dump — so the UI never shows a blank section.

- **Wire things together explicitly.** `createDigestAndAnswerFiles()` was implemented but never called. In modular code, the integration step (calling function A from function B) is its own task and must be deliberately done — it doesn't happen automatically just because both functions exist.

---

<div align="center">

*Built together. March 2026. Stay sharp. 🚀*

</div>
