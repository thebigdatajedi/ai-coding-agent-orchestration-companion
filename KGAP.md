# KGAP — v1.0 (2026-10-08)

Questions Gabe asked while working through Ducktate's [Orchestrator Course](https://ducktate.com/learn/orchestrator/), with the answers Claude gave. Source: the Claude chat "AI orchestrator compaction and state management" (2026-10-06). The answers are condensed but faithful to what was said.

Context for questions 1 to 5: the "Why I compact so often" section of [Claude Code Orchestrates, Codex and OpenCode Build](https://ducktate.com/prompts/orchestrator-stack/#:~:text=Why%20I%20compact%20so%20often), and the NEXT.md template it provides.

---

1. **Q: Is the orchestrator on a "small plan" because it runs a smaller model?** *(premise check, not asked directly)*

   **A:** No. "Small plan" means a small subscription tier, not a small model. The orchestrator runs the most capable model (Claude Fable 5.1) on a cheap plan. Every turn re-sends the whole conversation to the model, so a bloated context makes every turn cost more of the quota, even a one-line "merge it." Keeping the context small is what stretches a small plan across a full day. `/compact` is a quota tool before it is anything else.

2. **Q: How does git play a role in preserving or protecting state? Through the commit log, or through the commit itself?**

   **A:** Both, plus two more things. Git holds state in four forms:
   - **Commits (content):** the code that shipped is a fact on disk. No conversation event can unmake it.
   - **The log (history):** `git log --oneline -5` is a session-start read. It lets a fresh session cross-check the ledger, which is why entries record a commit SHA.
   - **Branches and worktrees (in-flight state):** every lane runs in its own worktree on a branch named for the work, like `lane/slash-menu`. The names encode what is in progress. In module 6 the author rebuilt lost state from worktree branch names alone after compaction lost a background task handle.
   - **`git status` (uncommitted reality):** what is dirty right now.

   NEXT.md is itself a committed file, so git protects the ledger too ("Commit the tree first"). Compaction only rewrites the conversation; it never touches the filesystem.

3. **Q: Does a properly filled-out NEXT.md provide enough state data for recovery?**

   **A:** No, deliberately. It is half of a two-part design. When the ledger and git disagree, git wins: commits are facts, the ledger is a claim about facts. Fix the ledger before doing anything else. The session-start recovery sequence is: the repo's `CLAUDE.md`, the repo map, `NEXT.md`, then `git status`, `git log --oneline -5`, and the test command.

   *Nitpick:* the template from the article is the older one. The course's module 6 ledger drops "Done this wave" and "Landmines," adds a **State** section (branch, test status, open PR numbers, background task handles), and puts the exact next command under "Now." That revision came from the lost task handle: the old template had nowhere to put one. The real rule: the moment a fact exists only in context, write it to the ledger. The 150-line cap is enforced by a test, and old entries roll verbatim into a dated archive file.

4. **Q: Is "scrollback" the word for the event where the agent's context is set to empty?**

   **A:** No. Scrollback is a place, not an event: the terminal term for lines that scrolled off the top of the screen. Here it means the conversation history itself. "A fact that only exists in the scrollback" is a fact said in chat but never written to a file or committed. The event is compaction, and `/compact` does not empty the context, it **summarizes** it. `/clear` is what empties it. Compaction is lossy and you don't control what survives, hence "one `/compact` from gone": treat the fact as already lost.

   *Nitpick:* Kafka log compaction keeps the latest value per key, so it is lossless for current state. Claude Code compaction is summarization, lossy by nature. Same word, opposite guarantee.

5. **Q: Why a merge "wave" instead of simply a merge? Is it because the sub-agents code and verify, then report items ready to merge back in waves?**

   **A:** Close, but the direction is reversed. The orchestrator **dispatches** in waves; sub-agents don't report back in waves. A wave is a batch of lanes sent out in parallel, capped at four builders plus one validator. (Eight at once blew the session limit and pushed machine load past 900.) Within a wave each lane returns on its own and is verified and merged individually, which gives two compaction points: every time a lane returns, and after every merged wave (batch landed, worktrees torn down).

   Correction: builders only self-check. Real verification belongs to the orchestrator and the Opus validator. A returned report is a claim, not a fact; the orchestrator re-runs the tests and reads the changed-file list before the diff.

   Spark mapping:
   - **Lane = task:** one isolated unit of work, run in parallel with its siblings.
   - **Wave = stage:** tasks inside it don't depend on each other.
   - **Merged wave = stage boundary:** the next wave can depend on everything that landed.
   - **NEXT.md + git = checkpointing to durable storage:** compaction is like losing driver memory; you recover from what was persisted.

   Lexicon:
   - **Lane:** one dispatched unit of work, on one backend, in its own worktree, on its own branch.
   - **Lane pack:** the brief handed to a lane. It must carry everything, because the external model has none of the orchestrator's context.
   - **Kick back:** a failed lane gets a fresh dispatch with a new pack, never a resumed conversation. Two misses means split the unit smaller, never escalate to a bigger model.

6. **Q: Is the "ledger" a log, or a construct that persists in collaboration with the context?**

   **A:** Neither. The ledger is just the NEXT.md file; "ledger" is the author's name for it.
   - **Not a log:** a log is append-only history. NEXT.md is rewritten in place; when "Now" changes, the old "Now" is replaced. It describes current state. The dated archive file that old entries roll into is the log.
   - **Not in collaboration with the context:** it persists *independently* of the context. It is a committed file; compaction, `/clear`, or a crash don't touch it. At session start the orchestrator reads it into context; when a fact appears only in context, the orchestrator writes it out. The context is a cache, NEXT.md is the source of truth, so anything that matters is written through.

   *Nitpick:* in bookkeeping, the **journal** records every transaction in order (history) and the **ledger** holds each account's current balance. Journal = `git log`; ledger = NEXT.md. That is why git wins a disagreement: you fix the ledger from the journal, never the reverse. In Delta Lake terms, `_delta_log` is the journal and the table snapshot is the ledger.

---

## Sources

- [The Orchestrator Course overview](https://ducktate.com/learn/orchestrator/)
- [Module 6: The daily loop](https://ducktate.com/learn/orchestrator/the-daily-loop/)
- [Claude Code Orchestrates, Codex and OpenCode Build](https://ducktate.com/prompts/orchestrator-stack/)
