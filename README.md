# Hikma

**Hikma is an architecture for generating practice questions for standardized exams, where the origin of every question can be proved and its difficulty is measured rather than asserted.** The single most important idea: every input to the pipeline is derived independently, never copied from copyrighted exam material, so the questions that come out can be published under an open license. This document explains the problem, who the project is for, the two ideas the architecture rests on, the pipeline itself, and how to adapt it to a specific exam. The full specification is in [ARCHITECTURE.md](ARCHITECTURE.md).

> **Status: specification.** This repository contains the architecture and a reference design system ([DESIGN.md](DESIGN.md)). It contains no code. Reference implementations live in separate repositories (see [Repositories](#repositories)).

## Contents

- [The problem](#the-problem)
- [Who it is for](#who-it-is-for)
- [Two ideas](#two-ideas)
- [The pipeline](#the-pipeline)
- [Adapting it to an exam](#adapting-it-to-an-exam)
- [Repositories](#repositories)
- [Contributing](#contributing)
- [Licensing](#licensing)
- [Trademarks](#trademarks)

## The problem

Practice material for standardized exams comes from three places, and you cannot publish freely from any of them.

Official past papers and sample questions are copyrighted by the examining body. Commercial question banks are copyrighted by their publishers. Scraped and crowd-sourced material has a license chain nobody can verify, so you cannot prove it is clean even when it is.

The result is that free, openly licensed, high-quality practice material barely exists for most exams. Learners who cannot pay are left with material of unknown quality and unknown legality. Anyone who wants to build a study tool has no dataset they can safely redistribute.

Generating questions with a language model looks like the answer, but it brings two new problems. First, you cannot prove that the model did not reproduce copyrighted exam content it saw during training. Second, a generated question that claims to sit at a given difficulty level is only making a claim. Nothing checks it.

Hikma is an architecture that addresses both.

## Who it is for

Two audiences, and the same output serves both.

**People preparing for an exam.** Practice material that costs nothing, states which ability level each question targets, and carries the evidence behind that claim rather than a bare label.

**People building learning software.** Generating a question bank is the expensive part of building a study app. It costs model spend, calibration work, and review time before the first user sees a screen, and today every team that wants a dataset generates one from scratch. Nobody outside that team can check the quality or the origin of the result. Hikma exists so that the work happens once, in public. A published dataset ships with its provenance record and its difficulty evidence attached, so you can judge the material instead of trusting a claim, and the intended release license, CC BY 4.0, asks only for attribution.

That describes what this project publishes and how it is meant to be used. It is not legal advice and carries no warranty. Anyone who redistributes a dataset should read the license and the provenance record shipped with that release and decide for themselves.

## Two ideas

### Provenance is a record, not a promise

Every input to the pipeline is derived independently. The vocabulary, grammar patterns, formulas, or competencies that questions are built from are generated from model knowledge or written directly. They are never taken from a copyrighted list, a textbook, or a past paper.

That rule is worth nothing unless someone can check it. So every generation call writes a manifest that records the model configuration that ran, the prompt template version, a hash of the assembled prompt, and the commit the code was at. Every exported question carries a provenance record that points back to that manifest.

The claim "no copyrighted material entered this pipeline" can therefore be proved from logs. Without the logs it is an assertion, and an assertion proves nothing.

A dedicated audit stage adds a second layer. It checks each question for signs that it reproduces something specific rather than being built independently, and it sends uncertain cases to a more capable model that has to record a written reason for its verdict. This stage is a screen for uncertainty, not a detector. No model can reliably audit its own training data. The architecture treats it as documented diligence, not as proof of cleanliness, and says so.

### Difficulty is a measurement, not a label

A question tagged "intermediate" is a claim by whoever tagged it. Hikma replaces the claim with an experiment.

Every candidate question is answered by a panel of model personas, one per ability band of the target exam. Each persona sees the question and its own instructions, nothing else. The pipeline records which personas answered correctly, together with the persona configuration the question was measured under, because two results are only comparable when they were measured the same way.

A question at the right level produces the expected pattern. Personas at or above the target band answer it. Personas well below the target band do not. When the pattern comes out the other way, the question is too easy, too hard, or answerable from surface clues, and it goes back for repair.

This turns difficulty calibration into something you can show someone. It also produces evidence about the underlying constraint data: when two independently generated questions built on the same entry both come out the wrong way, the entry itself is probably at the wrong level, and the pipeline writes that finding back.

## The pipeline

Twelve nodes, labelled [A] to [L]. Reference data is built first, then generation, then a chain of gates, then storage, review, and export.

```
[A] Seed corpus ──┐
                  ├──► [C] Prompt templates ──► [D] Generator ──► [E] Deterministic validator
[B] Constraints ──┘                                  ▲                        │
                                                     │                        ▼
                                              [I] Repair ◄──────────── [F] Auditor
                                                     ▲                        │
                                                     │                        ▼
                                                     └───────────── [G] Student gauntlet
                                                     ▲                        │
                                                     │                        ▼
                                                     └───────────── [H] Quality judge
                                                                              │
                                                                              ▼
                                              [J] Store ──► [K] Human review ──► [L] Export
```

The order of the gates is deliberate. [E] costs nothing to run, so it runs on everything and rejects structurally broken output before any paid call. [F] is cheap in most cases and screens provenance, the one failure that cannot be repaired. [G] measures difficulty, the failure most likely to occur. [H] is the most expensive judgment, so it runs last, only on questions that already survived the cheaper checks.

A failure at [E], [G], or [H] goes to [I], which generates the question again with the gate's feedback attached. The budget is one repair. A second failure ends the question. A failure at [F] is never repaired, because a suspected provenance problem cannot be reworded away. Those questions are quarantined and kept as evidence.

Node behaviour, the schema every node speaks, model configuration, and the provenance record are specified in [ARCHITECTURE.md](ARCHITECTURE.md).

Implementations run on the Claude API for now, with Sonnet and Opus as the working tiers. Nothing in the architecture depends on that. It asks only three things of a model provider, so support for other compatible APIs, including open-weight models, with your own key, is planned.

## Adapting it to an exam

The architecture does not depend on any one exam. To set it up for a specific exam, you supply five things.

| What you supply | What it is |
|-----------------|------------|
| Ability bands | The ordered difficulty levels the exam defines, from lowest to highest |
| Constraint inventory | The units questions are built from, each tagged with a band |
| Question types | The formats the exam actually uses, with their option counts and structure |
| Personas | One examinee persona per band, used by the gauntlet |
| Seed corpus | A small set of reviewed example questions per question type |

Nothing else in the pipeline changes. The gates, the repair loop, the provenance chain, and the export contract are the same for every subject.

Some worked examples of what those five inputs look like:

| Exam | Bands | Constraint inventory |
|------|-------|----------------------|
| Cloud and IT certifications | Associate, professional, expert | Services, concepts, operational scenarios |
| DELF, Goethe-Zertifikat | A1 to C2 | CEFR-tagged vocabulary and structures |
| GMAT, GRE, SAT | Scaled score ranges | Concepts, formulas, reasoning patterns |
| HSK | 1 to 9 | Characters, vocabulary, grammar patterns |
| JLPT | N5 to N1 | Vocabulary entries, grammar patterns |
| TOEIC | Score bands | Business vocabulary, register, functional patterns |
| TOPIK | 1 to 6 | Vocabulary, grammar patterns |

The fit depends on one property: how cleanly you can list the constraint inventory and tag each entry with a band. Language exams score well here, because their inventory is naturally a list. Certification exams score well, because the examining body publishes its scope as a breakdown of services and domains. Reasoning-heavy exams are the hardest case, because their inventory is a set of patterns rather than items. That weakens the deterministic check at [E] and puts more weight on the gauntlet at [G].

## Repositories

| Repository | Contents | License |
|------------|----------|---------|
| `hikma-exam/core` | This architecture specification | MIT |
| `hikma-exam/hikma-<exam>` | Working pipelines for specific exams, one repository each | Each sets its own license; **not** covered by this repository's MIT grant |
| Published datasets | Generated question sets | Each sets its own license; the intent is CC BY 4.0 |

Implementations are separate repositories in the same organisation. The MIT grant here does not extend to them.

## Contributing

**Name an implementation repository `hikma-<exam>`, never the exam name on its own.** The JLPT implementation is `hikma-jlpt`, not `jlpt`, and the same pattern applies to any exam added later.

The prefix is the point. A repository called `jlpt` reads as though it were the exam itself, or an official product of the body that owns the name. `hikma-jlpt` reads as what it is: the Hikma pipeline set up for that exam. That keeps the exam name in its descriptive role, saying which exam the pipeline targets, instead of a role that suggests affiliation, endorsement, or a claim on the mark itself. It also removes the ambiguity for anyone reading a repository list, a package name, or a URL out of context.

Apply the same rule to anything a user sees that inherits the repository name: package names, module paths, dataset filenames, and deployed subdomains. Keep the disclaimer in [Trademarks](#trademarks) in every implementation repository's README as well, because that is where people arrive first.

**[DESIGN.md](DESIGN.md) is a reference, not a requirement.** It is a complete design system, published so that an implementation starts from a working default rather than a blank page, and an implementation may adopt it, fork it, or design something unrelated. One rule survives a redesign: an implementation must not imitate the visual identity of the exam it targets, meaning its logo, palette, logotype, or the layout of its official surfaces. That is the visual half of the naming rule above, and it exists for the same reason. Using an exam name descriptively says which exam a pipeline targets. Looking like the examining body's own product implies an affiliation this project does not have and does not claim. [Section 2 of DESIGN.md](DESIGN.md#2-do-not-adopt-the-examining-bodys-visual-identity) states it in full.

**A dataset published before human review ships on a prerelease channel**, versioned with a prerelease marker and labelled as such. There are three channels, and each makes a stronger claim than the one below it: `alpha`, where the questions cleared every automated gate; `beta`, where the same questions ship once those gates have measured error rates; and `stable`, which contains only questions a human accepted one by one. Provenance does not depend on review, so all three make the same claim about origin. What the prerelease channels withhold is the claim about quality. The rule, including the exit criterion an uncalibrated channel has to state up front so that it cannot become permanent, is in [ARCHITECTURE.md](ARCHITECTURE.md), under node [L].

## Licensing

This repository is MIT licensed. See [LICENSE](LICENSE).

The MIT license covers the text of these documents. It does not stop anyone from implementing the architecture, because an architecture is an idea and ideas are not copyrightable. That is intentional. The point of publishing this is for people to build it.

If you do build on it, attribution is welcome but not required beyond what the MIT license already asks for.

## Trademarks

Exam names used in this document, including JLPT, TOEIC, HSK, TOPIK, SAT, GRE, and GMAT, are trademarks of their respective owners. They appear here only to describe the kinds of exams this architecture applies to. This project is independent and is not affiliated with, endorsed by, or sponsored by any examining body.

No logo, wordmark, palette, or official document design belonging to an examining body appears in this repository or in any implementation of it, and none may be added. Implementations carry this disclaimer on every public surface, in the interface rather than only in a README.
