# Building with AI agents

I built Logbook alone, and most of the code went through **Claude Code**, which I've used nearly every day since it came out. The interesting part isn't "AI wrote code." It's what you need around an agent so a 270k-line production codebase stays correct after more than 1,000 commits.

## 1. Project memory that doesn't rot

Agents start every session cold. The repo carries the context they need:

- **`CLAUDE.md` plus instructions** hold only things that change slowly: architecture, conventions, mandatory workflows and "known fixes, don't regress."
- **A single living `Status.md`** holds what's shipped, what's open, launch blockers, and **what was deliberately deferred and why**. Status used to be duplicated in several files, it went stale, and agents acted on stale facts. Now there's one source of truth, and instructions point to it instead of copying it.
- **Operational gotchas** are written down as soon as they're discovered: git hanging in iCloud-synced folders, unit tests that can write to production when a simulator is signed in, the SwiftLint version CI pins. Each one cost hours once and should never cost hours again.

## 2. Guardrails as code, not hopes

Rules an agent *might* follow become checks that **fail the build**:

- **Design-token lint:** a script fails CI if raw font sizes, spacing or hex colours appear where a token exists. A per-file baseline means counts can only go down.
- **Product facts:** `product-facts.json` (limits, trial length, supported platforms) is checked by unit tests against the app *and* by the marketing website's build. The site can't claim something the app doesn't do.
- **A pinned lint in CI** that runs on a clean `git archive` export, not the working tree.

## 3. Rules learned from failures

Each recurring agent mistake became a written rule with the *why* attached:

- **Adding an enum case?** Grep every `switch` over that enum and handle the new case in the same commit. (One new case once caused three separate rounds of build errors.)
- **Stage files by explicit path**, never `git add -A`.
- **Never claim it works without proof.** Gate on `BUILD SUCCEEDED` *and* a zero exit code, because `grep` for the string exits 0 even on failure.
- **Run tests on a fresh, signed-out simulator.**

## 4. Make the agent prove it

"Done" means evidence: a build log, a test count, a screenshot from the simulator, a diff against the live Firestore rules. Status entries record what was verified, and what *wasn't*, with the same precision. "Merged, 117 unit tests pass, not yet run against Firestore" is more useful than "fixed."

## 5. Humans own the irreversible

Agents work on branches and open PRs. Merges to `main` start Xcode Cloud builds, and production data reads, rule deploys and App Store Connect actions all need my explicit sign-off.

## Takeaway

Agents make one person ship like a small team, **if** you engineer the environment: durable context, automated guardrails, and a culture of verification. That's the same discipline it takes to run AI agents in production for a business.
