---
name: reflect
description: Use when a VibeYoga participant wants to critically evaluate their hackathon project or harness idea — surveys prior art and context first, then walks through novelty, acknowledgment, uniqueness, human benefit, MVP scope, validation tests, and the skill-vs-program choice in brainstorming style. Trigger phrases include "reflect on my project", "is my idea worth building", "what should my MVP cover", "help me validate this", "should this be a skill or a program".
---

# Reflect

A guided reflection for a VibeYoga hackathon project. The builder leaves with: a prior-art map, an acknowledgment block, a defensible uniqueness claim, a concrete user + value story, a trimmed MVP, a validation test set, and a justified skill-vs-program decision.

One project per session. Re-invoke for a new project.

## Operating rules

**Survey before asking.** Before each phase's questions, do a cheap survey of what's already knowable — read the repo, check commits, web search for prior art, look at related skills. Come back with findings, then ask questions that build on them. Never ask the builder something the survey already answered.

**Brainstorming style — one question at a time.**
- Prefer multiple-choice (`AskUserQuestion`) over open-ended where options exist.
- One question per message. If a topic needs more, break it into separate questions.
- Where helpful, propose 2-3 options with trade-offs and your recommendation, then ask which fits.
- Wait for the answer before moving on.

**Skip what doesn't fit.** If the survey or an earlier answer makes a phase irrelevant (e.g., the project is a pure research artifact with no "user," or prior-art search returns nothing and the builder can confirm it's a genuinely new niche), say so and skip. Note the skip in the final summary.

**Push back on vagueness.** "It helps researchers" is not an answer. "It saves my lab 2 hours per paper submission because X" is. Ask again.

**Don't implement.** No code, no scaffolding, no file edits for the project under reflection. The output is clarity, not artifacts.

## Phase 0 — Context survey (always)

Before anything else, silently gather:

1. **Repo state** — if the builder is in a project directory: `git log --oneline -20`, top-level `ls`, any `README.md`, `CLAUDE.md`, `docs/`, recent issues/PRs (`gh issue list`, `gh pr list`).
2. **The project's own description** — look for a one-liner in the README, a pitch in an issue, or a draft in the conversation history.
3. **Nothing found?** Ask the builder once: "Give me a one-sentence pitch and a link/path to anywhere you've written about this." Then proceed.

Present a 2-4 line summary of what you found before moving on. This tells the builder you're grounded, and catches misunderstandings before they propagate.

## Phase 1 — Novelty & prior art

**Survey first.** Do 2-3 web searches for obvious keyword combinations around the project's core claim. Also search for relevant skills in the plugin ecosystem and obvious GitHub/PyPI/Julia/npm packages. Spend ~2 minutes, not 20.

Present findings as a short table: name, one-line description, link, how close it is (exact overlap / adjacent / loosely related).

Then ask **one** question:
- If overlap found → `AskUserQuestion`: "For [closest match], are you (a) building on it, (b) competing with it, (c) replacing it, or (d) unaware of it until now?"
- If nothing found → `AskUserQuestion`: "Search turned up nothing close. Is this because (a) the niche is genuinely new, (b) the search keywords are wrong — give me better ones, or (c) the problem might not be one people have?"

**Output:** a prior-art table and the builder's stance on it. If (b) or (c) happens, loop once with better keywords or a sharper problem statement.

## Phase 2 — References & acknowledgment

**Skip if Phase 1 found no prior art AND the builder confirmed the niche is genuinely new** — note the skip in the summary.

Otherwise, for each prior-art item the builder is building on or competing with, ask **one** `AskUserQuestion`:
- "How will you acknowledge [X]? (a) README credit line, (b) formal citation, (c) inspired-by note, (d) fork lineage, (e) don't plan to — change my mind."

**Output:** a draft `## Acknowledgments` block the builder can paste into their README. Generate it yourself from the answers; don't make the builder write it.

## Phase 3 — Why new & what's unique

**Survey first.** From Phase 1, identify the 1-2 closest prior-art items. Read their README/docs briefly so the uniqueness question isn't hypothetical.

Ask **one** question at a time, in order:

1. `AskUserQuestion`: "Given [closest prior art], the strongest reason to build a new one is usually: (a) different target user, (b) different workflow assumption, (c) different deployment/licensing constraint, (d) the existing one is abandoned or broken, (e) fundamentally different approach. Which fits yours?"
2. Open-ended: "In one sentence, what is unique about yours that no existing tool has?"
3. Stress-test (only if the uniqueness claim sounds cosmetic): "If the maintainer of [X] added [your unique feature] tomorrow, would your project still be worth building? Why?"

If the stress test fails, flag it: "Your differentiator is cosmetic — [X] could absorb it in a week. Can we find a deeper one, or is the honest answer that this is a learning project?" A learning project is a fine answer; pretending it's not isn't.

## Phase 4 — Human benefit & value

**Survey first.** Look at the project description for who it's aimed at. If the repo has issues or discussions, skim for user personas.

Ask, one at a time:

1. Open-ended: "Who specifically does this help? Name one person (a real labmate, a participant, yourself last month) and the exact task they do."
2. Open-ended: "What do they do today without your tool? How long does it take? What goes wrong?"
3. `AskUserQuestion`: "Is this (a) creating new value — something that wasn't possible before, (b) reducing friction — something that worked but was painful, or (c) both?"

**Skip if the project is explicitly a learning/playground project** with no target user — note the skip.

**Red flag to surface:** if the answer is "AI can now do X 10x faster," ask whether the speedup actually changes the user's workflow, or is just a novelty. Speed without a workflow change is often not valuable.

## Phase 5 — MVP feature set

**Survey first.** Look at the repo's README, open issues, TODOs in code, any `docs/` folder — extract the feature list the builder already has in mind. Don't ask them to re-enumerate.

Present the extracted feature list back to the builder:
> "From your repo I see features A, B, C, D, E. Did I miss any?"

Then, for each feature, use `AskUserQuestion`:
- "Feature A: (a) Core — without this, the MVP doesn't prove the unique value, (b) Nice-to-have — improves UX but not load-bearing, (c) Cut — defer or drop, (d) Let's discuss."

Or batch the easy ones and only ask individually for the ambiguous ones.

**Rule of thumb:** a hackathon MVP usually has 2-4 core features. If the builder tags 6+ as Core, push back: "That's likely too wide for the hackathon window. Which two are the ones a demo can't survive without?"

**Output:** a bulleted MVP feature list tagged Core / Nice-to-have / Cut.

## Phase 6 — Validation test set

**Survey first.** Check whether the repo has any tests, fixtures, or example inputs. If so, reuse them as seed cases.

Propose 4-6 validation tests, drawing from these shapes — present them as a concrete list and ask the builder to confirm/adjust one at a time or as a batch:

- **Golden-path scenario**: a realistic task end-to-end. Specific input, specific expected artifact.
- **Edge case**: an input the builder already knows is hard (empty, huge, ambiguous, missing dependency).
- **Comparison baseline**: the same task without the harness, to see whether the harness actually helped.
- **Regression case**: once something works, freeze the input/output pair as a test.
- **User smoke test**: a real human (labmate, other participant) tries it cold. Does the skill trigger from the description alone? Do they understand the output?

Each test must be:
- **Concrete** — specific input, not vague "it should work."
- **Falsifiable** — a clear outcome that counts as failure.
- **Cheap** — runnable within the hackathon window; no user study.

**Output:** a numbered test list. For each: input, expected outcome, what counts as failure.

## Phase 7 — Skill or program?

**Survey first.** Look at what the builder has already started. If there's a `SKILL.md`, they've committed to skill. If there's a Python package structure, they've committed to program. Note which and ask whether that choice was intentional.

Then present the signal table and ask the builder to mark the row that matches their project most:

| Signal | Points toward skill | Points toward program |
|---|---|---|
| Task requires judgment on each input | ✓ | |
| Inputs are unstructured text / user intent | ✓ | |
| Steps are deterministic and well-defined | | ✓ |
| Output must be reproducible bit-for-bit | | ✓ |
| Needs to run unattended / in CI | | ✓ |
| The "hard part" is orchestration across tools | ✓ | |
| The "hard part" is a specific algorithm | | ✓ |
| Users will invoke it from a conversation | ✓ | |
| Users will invoke it from a script | | ✓ |

`AskUserQuestion`: "Based on the signals, your project leans [skill/program/mixed]. Does that match your intent? (a) Yes — keep as [current form], (b) No — I should switch form, (c) It's mixed — some parts should be code the skill calls, (d) Let's talk it through."

If signals point to *program* but the builder is building a skill, name it: the LLM is overhead, not value, and a plain script would be faster, cheaper, and more reliable. Vice versa for the other direction.

If the answer is "mixed" (usually correct for non-trivial harnesses), help them split: what's deterministic goes in code the skill calls; what needs judgment stays in the skill.

**Output:** a one-line justification the builder can defend. Example: "This is a skill because the core work is classifying unstructured issue descriptions into topic groups, which needs judgment. The GitHub API calls and output formatting are a plain script the skill invokes."

## Wrap-up: reflection summary

Produce a compact summary in the conversation (not a file, unless the builder asks) with:

1. **What it is** — one sentence.
2. **Prior art & acknowledgment** — table + draft README block. Or "(skipped: genuinely new niche)".
3. **What's unique** — one defensible sentence.
4. **Who it helps & how** — specific person + task + value type. Or "(skipped: learning project)".
5. **MVP features** — Core list only.
6. **Validation tests** — numbered list.
7. **Skill or program, and why** — the one-line justification.
8. **Skipped phases** — which ones and why.

Close with one honest question:
> "Looking at this summary — if you had to cut the project in half, what would you cut?"

The answer often reveals what the builder actually cares about.

## Notes

- If the builder resists a phase ("I already know this"), ask them to state the answer out loud anyway. Reflection only works if it's spoken.
- Do not start implementing during the reflection. The goal is clarity before code.
- If answers to later phases change earlier phases (common in Phase 5 or 7), go back and update — don't pretend the earlier answers still hold.
- Keep your own survey work silent unless it found something. Dumping raw search output wastes the builder's attention.
