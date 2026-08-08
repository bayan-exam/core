# Hikma: Architecture Specification

**This document specifies Hikma, a pipeline architecture for generating practice questions for standardized exams, with an input chain you can prove and difficulty you can measure.** The single most important rule: every input must be derived independently, because one input of unverifiable origin contaminates every question built from it and ruins the licensing position for the whole output set. The document covers terminology, the invariant, the build graph, each of the twelve nodes, the schema, model configuration, the provenance record, what you must supply to adapt it to a specific exam, and the limits of what it can prove.

> **Status: specification.** No implementation lives in this repository. Reference implementations are separate repositories and set their own licenses.

## Contents

- [Terminology](#terminology)
- [The invariant](#the-invariant)
- [Build graph](#build-graph)
- [Nodes](#nodes)
  - [[A] Seed corpus](#a-seed-corpus)
  - [[B] Constraint files](#b-constraint-files)
  - [[C] Prompt templates](#c-prompt-templates)
  - [[D] Generator](#d-generator)
  - [[E] Deterministic validator](#e-deterministic-validator)
  - [[F] Auditor](#f-auditor)
  - [[G] Student gauntlet](#g-student-gauntlet)
  - [[H] Quality judge](#h-quality-judge)
  - [[I] Repair](#i-repair)
  - [[J] Store](#j-store)
  - [[K] Human review](#k-human-review)
  - [[L] Export](#l-export)
- [Schema](#schema)
- [Model configuration](#model-configuration)
  - [Provider and portability](#provider-and-portability)
  - [Notes on the current generation](#notes-on-the-current-generation)
- [The provenance record](#the-provenance-record)
- [Adapting to an exam](#adapting-to-an-exam)
- [Roadmap](#roadmap)
- [Limits](#limits)

## Terminology

The architecture is exam-neutral, so it avoids the terminology of any one exam.

| Term | Meaning |
|------|---------|
| **Band** | One ability level defined by the target exam, from an ordered set. CEFR A1 to C2, a numbered scale, a certification's associate and professional tiers, a scaled score range. |
| **Constraint entry** | One unit that questions are built from, tagged with the band a candidate is expected to have mastered it by. A concept, a formula, an operational scenario, a vocabulary item, a grammar pattern. |
| **Constraint inventory** | The full set of constraint entries for an exam, partitioned by band. |
| **Question type** | One question format the exam actually uses, with its own option count, stem structure, and stimulus requirements. |
| **Stimulus** | Shared source material that several questions reference: a case description, a data table, a diagram, a code listing, a passage, an audio script. Absent for self-contained question types. |
| **Persona** | An examinee simulation pinned to one band, used by the gauntlet to measure difficulty. |
| **Gate** | A stage that returns a structured pass or fail verdict on a candidate question. |
| **Model binding** | The full configuration a stage runs under: the provider endpoint, the model, the reasoning effort level, the thinking mode, and the output schema version. A stage binds to one, the binding is recorded, and changing it counts as a version change rather than a small adjustment. |

## The invariant

**Every input is independently derived.** No constraint entry, seed question, prompt template, or stimulus may originate in an official past paper, a sample question published by an examining body, a commercial question bank, a copyrighted textbook, or any scraped material with an unverified license chain.

This is not a preference. The whole design rests on it. The output can only be redistributed if the input chain is clean, and one contaminated input spreads to every question built from it.

The invariant is enforced in three places, and all three are necessary:

1. **At the input boundary.** Constraint inventories and seed corpora are generated from model knowledge or written directly, then reviewed. Nothing is taken from a third-party compilation. This matters beyond ordinary copyright. A curated list can carry rights in how it was selected and arranged, and in some countries a database right that extraction triggers. Taking nothing from such a list avoids both.
2. **At every call.** Each generation call writes a manifest entry recording the model that ran, the prompt template version, a hash of the assembled prompt, and the commit SHA of the code. The assembled prompt is a record anyone can audit. Without the logs, "no copyrighted material entered the pipeline" is only an assertion, and an assertion is not evidence.
3. **At the audit gate.** Node [F] screens each question for signs that it reproduces something specific rather than being built independently. See [Limits](#limits) for what this stage can and cannot show.

Studying the *format* of an exam does not break the invariant. A format, such as four options with one correct answer, is a method rather than expression, and methods are not copyrightable. Copying real exam content, or rewording it lightly, does break it.

## Build graph

**The letter order [A] to [L] is the build order:** what must exist before what, and in that direction it has no cycles. The diagram below draws that order as the happy path, one pass through every gate. It does not draw what happens after a gate fails, because a single question's actual route branches too many ways to stay readable as arrows. The table underneath lists that routing instead, including the one cycle a question can take at runtime, back to [D] for repair.

```
[A] Seed corpus ──────┐
                      ├──► [C] Prompt templates ──► [D] Generator ──► [E] Deterministic validator
[B] Constraint files ─┘                                                         │
                                                                                 ▼
                                                                         [F] Auditor
                                                                                 │
                                                                                 ▼
                                                                  [G] Student gauntlet
                                                                                 │
                                                                                 ▼
                                                                   [H] Quality judge
                                                                                 │
                                                                                 ▼
                                                                         [J] Store ──► [K] Human review ──► [L] Export
```

**Every gate returns pass or fail, except [F], which returns pass or flag:**

| Gate | On pass | On fail or flag |
|------|---------|------------------|
| [E] Deterministic validator | Continues to [F] | Goes to [I] Repair |
| [F] Auditor | Continues to [G] | **Flag** goes to a terminal quarantined state; still written to [J] |
| [G] Student gauntlet | Continues to [H] | Goes to [I] Repair |
| [H] Quality judge | Written to [J] | Goes to [I] Repair |

**[I] Repair is the one cycle in the runtime route.** It calls [D] again with the failing gate's feedback attached, and the regenerated candidate runs the gate chain again from [E]. The budget is one repair, meaning two attempts in total: a candidate that fails again is abandoned rather than repaired a second time, and abandoned questions are still written to [J].

**The order of the gates is a cost decision, so do not change it without good reason.** [E] is free, so it runs on everything and removes structurally broken output before any paid call. [F] is cheap in most cases and screens the one failure that cannot be repaired, so it runs before any money is spent on a question that may be thrown away. [G] measures difficulty, which is the most common real failure. [H] is the most expensive judgment in the pipeline, so it only sees questions that already passed every cheaper check.

**Every question is written to [J] whatever the outcome**, including quarantined and abandoned ones. Filtering happens at export, not when the question is stored. Keeping the failures is what makes the audit trail complete. Deleting them would destroy the evidence that the pipeline works as described.

## Nodes

### [A] Seed corpus

A small set of example questions per question type and band, given to the generator as few-shot examples.

Seeds are generated from the same clean inputs as everything else, then reviewed by a person for quality and teaching value. They are not copied from an exam. Human review shows quality. It cannot show copyright clearance, and the architecture does not pretend otherwise.

Generation must fail loudly when no seed exists for the requested question type and band. Falling back silently to zero examples produces questions that look plausible and sit at the wrong level.

**Fall back to an easier band, never a harder one.** When the exact cell has no seed, an example from a lower band still teaches the shape of the question type, and it makes the question too easy, which is the error the gauntlet is built to catch. A harder example teaches the generator to aim too high, and no later stage can tell that apart from a question that is at the right level and simply hard. Fail only when the question type has no seed at any band at or below the target.

That fallback also sets the minimum corpus. One seed per question type, at the easiest band where it appears, avoids the hard failure for every cell. Two per cell is the working target, because a single example goes into every call for that cell and the output starts to look like it.

### [B] Constraint files

The constraint inventory, split by band, plus whatever secondary attributes the subject needs for choosing distractors. The secondary attribute is whatever makes two entries easy to confuse: a topic or competency group for content exams, a service category for certification exams, part of speech or functional category for language exams.

The secondary attribute exists so that distractors can come from the same category as the answer. A distractor from a different category can be ruled out from surface features alone, which makes the question test pattern spotting rather than knowledge.

Every entry carries the band a candidate is expected to have mastered it by. This is what makes the deterministic level check at [E] possible.

**Bands generated separately will overlap.** An entry claimed at two bands makes the membership check at [E] contradictory, and it corrupts the gradient at [G], which then measures against a band the entry does not hold on its own. Remove duplicates across the whole inventory before any generation run, resolve each clash to the easier band, and record how you resolved it. Doing this once is cheap. Finding it later, through failed questions, is expensive.

### [C] Prompt templates

One versioned template per question type. Templates take the constraint excerpt, the few-shot examples, the target band, and the output schema as parameters.

Versions are never edited in place. A changed template becomes a new version, so questions generated under an earlier version can still be traced and the manifest keeps its meaning.

### [D] Generator

Produces a candidate question: stem, options, correct answer, an explanation of why the answer is correct, and a justification for each distractor explaining why it fails.

The distractor justifications are internal. Asking for them measurably improves distractor quality, because a model that has to defend each wrong option produces fewer arbitrary ones. They are removed before export.

Three requirements:

- **Constrain the output with a schema**, using the model's schema-constrained output mode or a strict-schema tool, rather than parsing free text. Parsing free text creates a whole group of failures that have nothing to do with question quality, and every current model can enforce the shape at the request level.
- **Store the assembled prompt hash** in the manifest for every call, without exception.
- **Store the model binding that served the call**, read from the response rather than from your configuration. See [Model configuration](#model-configuration) for why those two can differ.

Choose targets with weights rather than evenly, so that common constraint entries appear more often than rare ones, but never exclude an entry. A light weight is enough. Filtering entries out completely means part of the inventory never becomes a question.

Selection is also the only source of variety left. Current models no longer accept the sampling parameters that used to spread output across repeated calls, so variety has to come from what goes into the prompt: which entry is targeted, which seeds are used, and what the template asks for. Expect a generator that sends the same assembled prompt twice to produce nearly the same question twice.

**Not every constraint entry suits every question type**, and a mismatch is a property of the entry rather than a generator failure. Filter out pairs that do not fit at selection time. Letting the generator try one produces a question that fails a deterministic check and spends the repair budget on work that could never have succeeded.

**Question types that share a stimulus are generated in two steps:** the stimulus first, written to the store, then the questions that refer to it. Writing it first is what lets the reference check at [E] pass, and it stops one rejected question from removing the passage the other questions depend on.

### [E] Deterministic validator

Code only. No model, no API cost. Runs on every generator output.

Typical checks: the record matches the schema, the option count is correct for the question type, the options all differ from each other, the stated answer is present among the options, the target constraint entry is actually present in the question, every constraint entry referenced falls within the target band, and, for question types that use one, a referenced stimulus that exists in the store, carries the same band, and is not empty.

The band membership check is the important one, and it is a simple test of set membership. It catches the most common generator error, material from outside the target band, at no cost, before any paid model runs.

A failure here goes to [I].

### [F] Auditor

Screens for copying. It asks whether the item shows signs of being a specific remembered thing rather than something built independently.

Two tiers. A fast model sorts the cases, and only genuinely uncertain ones go up to a more capable model at a higher effort level.

**Triage routes, it does not decide.** Its only decision is whether to spend the escalation, and it never gives a final verdict of its own. That tier is deliberately weak, so reliability has to come from the shape of the task rather than from the model's ability. Make it choose a verdict from a fixed set instead of writing prose, require a one-sentence reason with it, and define the confidently-clean verdict as a list of stated conditions that must all hold. A cheap model applies an explicit checklist reliably, and it judges badly when asked an open question about whether an item seems fine. Keep those conditions in the versioned template, so a change to the threshold shows up as a diff someone can review rather than as an undocumented change in behaviour.

**The escalated verdict must carry a written reason as a field of its structured output.** That reason is the record of diligence, not the model's internal reasoning trace. Current models do not return the raw trace at all, and the summary you can ask for is optional, incomplete, and written after the fact, which makes it a poor basis for a chain of evidence. Ask for the reason as output, store it with the verdict, and treat any summary of the trace as extra material.

Tune the triage prompt so that it clears obviously independent questions with confidence and escalates only real uncertainty. Track the escalation rate, and set a level at which you tune the template before you change the model tier. A rising rate usually means the triage prompt is too cautious rather than that there is a real provenance problem. The two kinds of failure are not equal. Escalating too often costs money. Escalating too rarely defeats the screen and attaches a clean verdict to a suspect question as evidence. So anything that is not clearly clean escalates, and the threshold you may adjust is the confidently-clean one, never the suspicion one. Never tune triage to clear suspicious cases in order to save money.

**A flag here is never repaired.** A suspected provenance problem cannot be reworded away. The question goes to a terminal quarantined state and is kept.

**A refusal is not a flag.** A model's safety classifiers can decline a request, and the decline arrives as a successful response that carries a refusal marker and no usable content, not as an error. That is a failed call, not a verdict about the question. Record it, then retry or escalate it, and never let it become either outcome. Counted as a flag, it quarantines a clean question and raises the escalation rate. Counted as a pass, it puts an unaudited question into the release channel.

Where possible, use a different model family here from the one at [D]. The signal is more useful when the auditor does not share the generator's blind spots, though see [Limits](#limits) for why that independence is only partial.

**Even so, keep the whole per-question chain with one provider.** Both tiers here have to stay comparable for the escalation rate to mean anything, and a second vendor inside the chain turns every stored verdict into a redistribution question as well as a technical one. Real independence between vendors belongs in a separate offline audit over a sample of already published questions, published as its own report. That gives you the outside check without adding a vendor to the evidence chain that travels with every question.

### [G] Student gauntlet

The difficulty measurement itself, and the part of the architecture with no equivalent elsewhere.

One persona per band attempts the question in parallel. Each persona sees the question and its own instructions, nothing else. It does not see the correct answer, the other personas, or any pipeline metadata.

The expected gradient for a question targeted at band L:

| Persona band | Expected | Flag if violated |
|--------------|----------|------------------|
| At L or above | Answers correctly | Question is too hard for its band |
| Exactly one band below L | Either outcome, not scored | n/a |
| Two or more bands below L | Answers incorrectly | Question is too easy or guessable |

The one-band-below case is left unscored on purpose. Real band boundaries are not sharp, and scoring that cell would flag questions that are fairly on the border. Bands at each end of the scale lose one side of the signal: the lowest band has nothing below it to fail the question, and the highest has nothing above it to pass it.

**A persona must be able to abstain, and an abstention counts as a failure.** A forced choice among n options has a 1/n floor from guessing, so a persona well below the target band answers correctly that often by luck alone, and no false-pass rate below that floor can be reached. Abstention is part of the structure for that reason, not a courtesy. Recording it as a failure keeps the gradient a statement about what the persona knew rather than about how many options it saw.

**Personas must think as little as the platform allows.** A real candidate at a given band does not reason at length. A persona that does is too able: it passes questions a real candidate would fail, and the measurement stops meaning anything. This is about keeping the measurement valid, not about saving money.

This used to be one switch, because models did no reasoning by default and turning it off cost nothing. On current frontier models it is not one switch: reasoning is on by default, turning it off is limited to the lower effort levels or not possible at all, and where it is possible it brings problems of its own. Three controls replace the switch, strongest first. Pick the weakest model tier that still reads the question reliably, which is usually the small fast tier and is also the tier most likely to still offer a real non-reasoning mode. Set the lowest effort level the stage offers. Describe the knowledge ceiling concretely in the persona prompt, and allow abstention. Then check the result against the drift metric below instead of trusting the configuration.

**The persona binding is part of the measurement.** A gradient is a statement about one set of personas running under one binding, so gradients recorded under different bindings cannot be compared and must not be combined. Record the binding with every gauntlet result. Changing it invalidates the calibration: measure a new baseline against questions whose difficulty is already known before you trust any new result, and expect the band boundaries to move.

**Constraint feedback.** When a question is abandoned after both attempts failed on band-related flags alone, that is evidence about the constraint entry rather than about the question. Two questions built independently that fail the same way point at the entry. Write the finding to a separate side file. Never change the constraint files themselves, because changing them would break the provenance chain of the original generation run. A flagged entry is held back from selection until a person re-levels it or clears the flag. Holding it back is not deleting it.

Only assign a flag to a specific entry when the question has exactly one constraint target. With several targets, you cannot tell which one is responsible.

**Calibration risk.** Track how often personas two or more bands below the target answer correctly. This is the drift metric for the whole gauntlet: if it rises, the measurement is losing its meaning. It rises for two reasons, and they need different responses. Under one fixed binding, it means the persona prompts allow too much, and the fix is to describe the ceiling more concretely. Across a change of binding, it means the new model is simply more able at the low bands, and no prompt undoes that completely. Measure a new baseline instead of tightening prompts until the number looks right.

**When a provider's cheapest tier is still too able at the lowest bands, replace it with a genuinely smaller model, not with a stricter prompt.** Prompting a model into incompetence is a weak tool, and a model whose floor sits above the lowest band cannot act like a beginner however the persona is worded. This is the one stage where a weaker model is the right model, so it is a decision about ability rather than about cost. Check two things before you swap, in this order. First, test that the failures follow the bands: a weaker model is wrong in different ways rather than worse everywhere, and one that fails because it stops following instructions, rather than because it does not know the material, produces a gradient that looks right and measures nothing. Confirm on questions of known difficulty that its failures track the band. Second, keep every persona on one model, because mixing families across the gradient makes the comparison between bands meaningless. The gradient is published with each question, so the persona model's output becomes redistributed content, and its terms need the same review as any other input.

### [H] Quality judge

Scores the teaching quality that deterministic checks cannot reach: whether the answer is genuinely the only correct one, whether the distractors are plausible enough to tell candidates apart, whether the stated justifications are accurate, whether the explanation is correct, and whether the stem reads naturally.

Runs last among the gates, on the most capable binding in the pipeline, and only on questions that passed every cheaper check. A failure goes to [I].

Score against explicit criteria, and require a verdict for each criterion in the output. Current models follow an instruction literally, so a judge told to be conservative, or to report only serious problems, holds findings back without saying so, and the result reads as a clean pass rather than a filtered one. Let it report everything with a severity, and filter at a threshold you control.

### [I] Repair

The generator called again with the failing gate's structured feedback added, not a separate agent. Same model, same constraints, same prompt assembly, so the provenance record is unchanged.

**The budget is one repair, meaning two attempts in total.** A question that fails again is abandoned. A retry loop with no limit turns one difficult constraint entry into a cost with no limit.

Unlike the first pass, repair benefits from one step up in effort. The first pass generates from constraints and has no feedback to interpret. The second pass has to diagnose: it must read why the question failed and produce a fix that does not bring the same problem back. One level above the generator's setting is enough, because the repair is a small, well-specified edit rather than open-ended work.

**A repaired question keeps its original ID** and adds one to its repair count. It is a second attempt at the same question, not a new question, and giving it a new ID would break the trail between the gate verdict that rejected it and the record that finally ships.

**Repair scope follows the flag.** A flag against a shared stimulus regenerates the stimulus and every question built on it together, because the questions were written against material that is about to change. A flag against a single question repairs that question alone, against the unchanged stimulus.

Pass the failing gate's verdict in as structured feedback, with a template per flag that states what must change and what must not come back. Feedback written as prose invites the same failure in different words.

### [J] Store

Storage written by appending: one file per band, one record per line. Plain text, easy to diff, reviewable in a pull request. At this scale a database gives no benefit and makes auditing harder.

Stimuli are stored separately from questions and referenced by ID, so that editing a shared passage touches one record rather than every question that uses it.

Nothing is ever permanently deleted. Terminal states are kept.

### [K] Human review

An interface presenting one question at a time for accept, reject, or edit. Acceptance is what promotes a question into the reviewed release channel.

Review is a quality gate. It is not, and cannot be, a copyright clearance step. No person can tell by looking that a question is free of unconscious copying. The provenance chain does that work, or nothing does.

**Review decisions are also the only ground truth the gates have.** Every accept or reject is a label on a question the gates already judged, so a labelled sample measures two rates: how often the gates passed a question a person then rejected, and how often they flagged one that was fine. Nothing else in the pipeline can produce those numbers, which is why the channels at [L] treat them as the line between an uncalibrated release and a calibrated one. Measuring them does not need the interface. A labelling pass over an exported sample is enough, spread so that no question type or band goes unmeasured, and it is worth doing before the interface exists rather than making the calibrated channel wait for one.

At scale, review is a problem of managing a queue rather than of designing a form. Nobody can review several thousand questions one by one, so the queue needs filters by band, question type, and gate outcome, plus a way to sample, so that a reviewer can clear one coherent slice instead of an unsorted pile. Show the terminal failures beside it, read-only: a review screen that hides its rejects hides how the pipeline fails. Showing each question's provenance record next to it makes the decision better, and it is the fastest way to notice a gate that is wrong in a consistent way.

### [L] Export

Filters the store by channel and writes a versioned dataset.

**Export must be validated against an allowlist.** Build the exported shape from the stored shape by naming each internal field you leave out, so that any new internal field added later fails loudly at export instead of leaking quietly into a public dataset. This is the most valuable defensive check in the pipeline, because it is the only one whose failure is published.

Quarantined and abandoned questions are always excluded. The per-question provenance record is kept on purpose, because it is the chain of evidence a downstream consumer needs.

**Three channels, each making a stronger claim than the one below it.** Implementations should use these names, so that someone reading two Hikma datasets reads one vocabulary.

| Channel | What it claims | Version |
|---------|----------------|---------|
| `alpha` | The questions cleared every automated gate. How often those gates are wrong has not been measured. | Prerelease |
| `beta` | The questions cleared every automated gate, and the gates have measured error rates. | Prerelease |
| `stable` | Every question was reviewed and accepted by a human at [K]. | Full release |

Both prerelease channels carry a prerelease marker in the version, whether semver-style (`1.2.0-beta.1`) or dated (`beta-2026-08-06`). Never publish an unreviewed set under a stable version number and correct it later.

**A dated version names the batch's generation date, not the day this particular file was produced.** [D] generates a batch on one date, and that date becomes its first tag: `alpha-2026-08-12`. Once enough of that batch clears human review, it is exported again under `beta` or `stable`, and the new version keeps the same date. A `stable-2026-08-12` release means "the batch generated on 2026-08-12, now reviewed," not "produced on 2026-08-12," and it can ship weeks or months later than the date it carries. This is what lets a reader trace a dated release straight back to the run manifest for that generation date, rather than to whichever day someone happened to run the export. Do not fold two generation dates into one version: export them as separate dated releases, so that a date keeps naming exactly one batch.

**A later dated release carries no promise about an earlier one.** Two releases under different dates are two different batches, not a running total. The later one does not have to contain, in whole or in part, whatever the earlier one contained, and an implementation is free to keep publishing disjoint batches indefinitely. A consumer who wants everything reviewed so far has to fetch every dated release and merge them; nothing here guarantees that the newest one is a superset of the rest.

**Recommend that consumers hold a specific release rather than track a dynamic "latest" pointer, on any channel.** Because releases do not accumulate, resolving to whatever is newest can hand a client less content than it had before, not more, with no error to signal the drop. A pinned version is also what makes a data contract reproducible at all: the export schema, the question set, and the provenance behind both stay fixed only as long as the version string does.

**Merging is trivial because a question's ID never changes once assigned.** [I] keeps a repaired question's original ID through every repair, and nothing in the pipeline reissues one when a question is promoted from one channel to the next either. A consumer merging several dated releases can therefore deduplicate on ID alone: the same ID appearing in two releases is the same question, not a collision to resolve.

**A consolidated release is a separate, optional artifact from the dated per-batch releases.** Once enough reviewed material has accumulated across several dated `stable` releases to be worth shipping as one coherent set, an implementation may publish a rollup under a plain version rather than a date, for example `v1`. This is not a fourth channel and carries no schedule of its own; nothing requires one to ever be cut. Version it so it cannot be mistaken for a dated tag, and record in its release notes which dated releases and which question IDs it draws from, so a reader can still trace every question back to its generation batch and run manifest.

**`alpha` and `beta` filter questions in exactly the same way.** What differs is what you can say about the pipeline, never the bar a question had to clear. Letting `alpha` admit questions the gates did not pass would give export a second filtering path and trade an honest claim for a loose one. The channel exists because a weaker statement is the accurate one before the gates have been measured, not because a weaker dataset is acceptable. Write the two as one test that takes the channel as a parameter, not as two branches, so they cannot drift apart.

**Write down what it takes to leave `alpha`, at the time the channel is created**: the measurement that promotes it, the sample size behind it, and where the numbers are recorded. The measurement is the false-negative and false-positive rates of the gates against the human labels described at [K]. Without a stated bar, `alpha` is a `beta` with a disclaimer, and it never moves up.

**No channel gets a lifecycle state of its own.** A question can appear in an `alpha` or `beta` release while it is still waiting for review, and it counts as finally exported only when a `stable` release includes it. A question accepted at [K] between two prerelease builds first appears in the next `stable` release, and it is never moved back down.

The difference between the channels is narrow, so state it exactly. All three make the same claim about provenance, because provenance comes from the build chain at [F] and the manifest, not from review, and an unreviewed question's chain of evidence is complete. What the prerelease channels hold back is the claim about quality: nobody has confirmed that these questions teach well, have one clear answer, or sit at the right level beyond what the gates measured, and on `alpha` nobody has yet measured how often the gates are wrong. Anyone who builds a dataset into a product needs to tell those apart from the version alone, before reading anything else. The dataset metadata records the reviewed and unreviewed counts, and the dataset card says plainly which parts a person has looked at.

**The exported file carries its own claims**, because it gets separated from the page that published it, and a claim that lives only on a download page does not travel with the file. Two metadata fields, kept separate on purpose. One states the maturity claim of the channel and differs per channel. The other states the standing position on how the questions were produced and what the project is not affiliated with, and it is identical in every export. Make both required in the export schema, so that a release cannot ship without either. Do not fold the invariant text into the per-channel string: it would then exist once per channel, and the copies would eventually disagree without anyone noticing.

**The exported file is one object, not a bare list of questions.** It wraps the question array with the two metadata fields above, plus three more: a channel name, the version described above, and an export timestamp for when this particular file was produced. That timestamp is not the version's date: the version names when the batch was generated, and the export timestamp names when this file of it was written, which can be much later. A count of the included questions is worth adding too, so a consumer can check the file against that number without parsing the whole array. Keep this shape identical across channels. Only the values change, never the fields present.

**A prerelease channel must never be reachable as the default download.** Whatever platform hosts a release, mark `alpha` and `beta` through that platform's own prerelease mechanism, so that a client asking for the newest release does not receive an uncalibrated one without asking for it by name. This backs up the pinning recommendation above at the platform level: even a consumer who forgot to pin should not land on an uncalibrated channel by accident. While a channel is `alpha`, treat the export schema itself as free to change without notice, and say so plainly next to the download: a channel that has not measured its own gates has not measured its own stability either.

**Publish the constraint inventory alongside the dataset** once it is stable enough to be a contract. It travels the same provenance chain as the questions, and releasing it lets a reader audit the distractor pool and the band calibration directly instead of trusting the questions.

Option order is exported exactly as generated. Never sort the options or move the correct answer to a fixed position: a regular answer position becomes a pattern anyone can exploit in the published data.

## Schema

Every node speaks one schema, defined once and validated at runtime rather than trusted. The essential shape, independent of any specific exam:

**Constraint entry.** A stable ID derived deterministically from the entry's identifying fields, so that generating the inventory again produces the same IDs. Then the entry itself, its band, a secondary category attribute for choosing distractors, and optionally a frequency or commonality score used to weight selection.

Deterministic IDs matter more than they look. They make an inventory reproducible, they let a question point at the same entry across regenerations, and they separate entries that look the same on the surface.

**Question.** A random-hex ID, the question type, the target band, the stem, the options, the correct answer, an explanation, the internal distractor justifications, references to the constraint entries it was built from, optional stimulus references, the validation state, the flags that produced that state, a human review flag, and the provenance record.

**Stimulus.** A random-hex ID, band, modality, and content. Referenced by ID from questions.

**Validation state.** One of: pending, pass, flag, abandoned, quarantined. Abandoned and quarantined are terminal and excluded from every export. The difference between them matters: abandoned means the pipeline could not produce an acceptable question, while quarantined means someone raised a provenance concern. They are handled differently and must not be merged into one state.

Pending covers a question that passed the deterministic check but never went through the model gates, either because a run sampled questions to control cost or because it came through a manual or single-question path. It is not terminal. A later pass can take pending questions from the store and run the gates over them exactly as if it were their first time. Because export filters on the passed state, pending is excluded without a second code path.

Do not use sequential integers or random UUIDs for constraint entries. Sequential IDs reveal the size and order of the inventory, and UUIDs cannot be reproduced when the inventory is generated again.

## Model configuration

Configure each stage from a single place, and give it a full binding rather than a model name. **Record in the manifest whichever binding actually ran.** The effort level changes a model's behaviour as much as changing the model does, so a record that names only the model is not a record.

The tiering principles, which outlive any particular model:

- **Effort is the control, not reasoning on or off.** Older models did no reasoning by default and let you leave it off at no cost. Current ones reason by default, and turning it off is either unavailable, limited to certain effort levels, or accompanied by problems of its own. Treat the lowest effort level as the equivalent of the old default, and raise it only for stages that really are judgments.
- **Spend capability where an error cannot be caught later.** The constraint inventory is generated once, and everything after it trusts the inventory as ground truth. No later gate catches an error there, so use the most capable model even though the stage does no reasoning.
- **Spend effort where the output is a judgment that has to stand up.** The audit escalation is the clearest case, because its verdict is the record that has to hold.
- **Spend the least on the personas.** As above, thinking at length destroys the measurement.
- **Use a different family for the auditor than for the generator**, so the screen is not simply the generator agreeing with itself.
- **Determinism is not available, so reproducibility comes from records.** Current models reject the sampling parameters that used to make a call repeatable, so no stage can be assumed to produce the same output twice. Nothing may depend on running a call again to reproduce a result: the manifest and the stored output are the reproduction. Where the shape of the output must be stable, fix it with a schema rather than with sampling.
- **Record the binding that served the call, not the one you asked for.** A model other than the one named can serve a request, because a declined request can be sent to a configured fallback inside the same call. Read the model that served it from the response.
- **Treat a refusal as the outcome of a call, not as a verdict.** Any stage can have a request declined by a safety classifier, and the decline arrives as a successful response with no usable content. Check for it before you read the output, record it in the manifest, and route it explicitly. Exams in security-related subjects hit this most often, and a pipeline that counts refusals as gate failures will quietly corrupt both the difficulty data and the audit trail.

Three cost controls are worth building in from the start. **Batch submission**, because nothing in an offline pipeline needs an immediate response and batched calls are billed at a discount. **Prompt caching**, because most stages send a large fixed prefix with a small changing suffix. Caching matches on an exact prefix, so assemble the prefix in a fixed order, schema and instructions and few-shot seeds first, target entry last, and confirm from the reported cache-read counts instead of assuming it worked. **Effort**, the largest of the three: most stages here are recall or pattern matching and run correctly at the lowest setting.

A fourth is not about cost, but at the same scale it stops being optional. A run over thousands of constraint entries will be interrupted, so **batches have to be resumable**: they must restart without generating completed questions again and without leaving the manifest rows they already wrote without an owner. Estimate the cost of a full run from one measured slice before you commit to it, rather than learning the number partway through.

### Provider and portability

**The default is the Claude API, with Sonnet and Opus as the working tiers.** Sonnet runs generation, audit triage, and repair. Opus runs the stages where an error cannot be caught later or the judgment has to stand up: inventory generation, audit escalation, and the quality judge. The persona stage sits below both, following the tiering principles above. That is a binding, not a requirement of the architecture. It is recorded in the manifest like any other, and it is what the current implementations run.

Support for using your own key against other compatible APIs, including open-weight models, is planned. The architecture asks little of a provider, and the short list is the portability contract: schema-constrained output, some control over how much the model reasons, and at least two capability tiers, so that triage and escalation can differ. A provider that offers those three can run every node here. Anything a stage needs beyond them belongs in the binding rather than in the pipeline, and a stage that cannot be written without a provider-specific feature is a design error to fix before anything depends on that feature.

Two things follow once that support arrives. The provider is part of the binding, so the manifest records the endpoint next to the model, and questions generated against different providers cannot be compared without saying so. And a change of provider is a change of persona binding: the gauntlet needs a new baseline before its gradients mean anything, exactly as after a model upgrade.

### Notes on the current generation

These are the concrete facts the principles above were written against, as of August 2026. They will go out of date. The principles should not.

- Reasoning is on by default on the frontier tiers. Turning it off is limited to the lower effort levels or not possible at all, and where it is possible it has faults of its own, so the cheap setting is the lowest effort level rather than a switch.
- Fixed reasoning-token budgets were removed. A request carrying one is rejected. Effort levels replaced them.
- Sampling parameters were removed from the frontier tiers. Smaller and older models still accept them, which is one reason the persona tier and the generator tier now differ in more than cost.
- The raw reasoning trace is never returned. A summary is available on request and empty by default. Any stage whose value depends on a recorded reason has to produce that reason as part of its structured output.
- Token counts are model-specific and have shifted between generations. Recount rather than reusing a budget measured under a different binding.
- Prompt caching has a minimum cacheable prefix that varies by model, and the small fast tier's minimum is the largest of any tier. A short triage prompt can fail to cache without any sign, even though it looks configured for caching.
- Schema-constrained output is a first-class request parameter, and tools can enforce their schemas strictly. Neither needs free-text parsing or a retry loop around a JSON parser.

Two consequences for prompting follow, and both reverse advice that was correct a generation ago. Current models check their own work without being asked, so telling a stage to double-check its output produces repeated work rather than accuracy. The gates are the checking layer, not the prompt text. And prompts written for older models usually say too much about how to work, which now costs output quality: state the constraint, the shape of the output, and what counts as done, then let the model plan the rest.

## The provenance record

Two artifacts, at two scopes.

**Per run**, a manifest recording the commit SHA, every model binding that actually ran, prompt template versions, per-question assembled prompt hashes, gate outcomes including refused calls, counts, and cost. This is the record for the whole batch.

**Per question**, a record held inside the question itself and kept through export: the generating binding, the prompt version, the audit verdict and its written reason, the gauntlet gradient observed and the persona binding it was measured under, the judge scores, and a pointer back to the run manifest.

The per-question record is what lets anyone audit the dataset independently. A reader can trace any single question back to the exact call that produced it, without access to the pipeline.

Reference-data generation writes a manifest of its own, and it is a different record from the generation-batch manifest: a different node, different fields, and only the commit SHA in common. Give it its own schema and its own name rather than widening one shape to cover both. Otherwise the fields each one requires become optional in the other, and neither can be validated.

Assemble the per-question record once, when the question is written to [J], from the verdicts the gates carried with it. Building it up piece by piece means the record is invalid for most of its life, which defeats the point of validating it.

If someone raises a provenance concern, **quarantine the affected material where it is, do not delete it.** The manifests are the evidence that an entry was derived independently, and deleting them destroys that evidence. Once a dispute can reasonably be expected, keeping the records may also become a legal duty, and deleting them at that point can be treated as destroying evidence, not as carelessness.

## Adapting to an exam

Five inputs, listed in build order. Nothing else changes.

1. **Bands.** The ordered ability levels, lowest to highest. The gauntlet at [G] needs at least two bands to run at all, and at least three to produce a full gradient, because the design leaves the band immediately below the target unscored. An exam that defines only one level, a single pass or fail cutoff with nothing above or below it, gives [G] no scale to measure against. See [Limits](#limits).
2. **Constraint inventory.** The units questions are built from, each tagged with a band and a secondary category. Generate or author it; never extract it.
3. **Question types.** The formats the exam actually uses, with option counts and stimulus requirements. Formats are methods and are not copyrightable, so studying real exams to determine them is legitimate. Copying their content is not.
4. **Personas.** One per band, describing a candidate's knowledge ceiling concretely enough that the persona fails what it should fail. Allow abstention.
5. **Seed corpus.** A handful of reviewed examples per question type and band.

One property decides how hard the adaptation is: how cleanly the inventory can be listed, and how clearly band membership is defined. Language exams and certification exams both fit well, the first because the inventory is naturally a list, the second because the examining body publishes a breakdown of scope. Reasoning-heavy exams are the hardest, because their inventory is a set of patterns rather than separate items. The harder the inventory is to list, the weaker the deterministic check at [E] becomes, and the more weight falls on [G].

A second property decides whether the exam fits at all: whether it defines more than one ordered band. Every worked example above has three or more. An exam that certifies competence against a single cutoff, such as a one-tier professional license or a pass or fail bar exam, has no levels for the gauntlet to compare against, and the architecture does not cover it yet.

## Roadmap

Hikma's first target is exams where the majority of question types are multiple choice, and the first question type built for any exam is text only: a stem, an optional short passage, and text options. Reaching that scope needs no stimulus modality beyond plain text, and it is what exercises the deterministic validator at [E] and the gauntlet at [G] at their strongest, since a fixed-option answer is the easiest thing either gate can check.

Three phases widen the stimulus modality without changing the question format. Image-based question types add a diagram, chart, or code listing as the stimulus. Audio-based question types add an audio script or recording as the stimulus. Multi-modal question types reference more than one stimulus in a single question, such as an audio passage paired with an image. All three stay inside the schema already defined at [Schema](#schema), since a stimulus already carries a modality field, and a question that references a non-text stimulus is still one where the examinee picks from a fixed set of options. [E] and [G] apply to these question types unchanged.

Free-response and constructed-answer question types come last, and they are a different kind of change from the three before them. [E] as specified checks that the stated answer is present among a fixed set of options, and the gradient at [G] scores a persona's answer as chosen from that set rather than composed freely. Neither gate can run unmodified against an answer with no fixed option to check against, so this phase needs the gate contracts themselves to change, not only the constraint inventory or the seed corpus.

## Limits

Stating these plainly is part of the design.

**The auditor is a screen for uncertainty, not a detector.** No model can reliably audit its own training data, and there is no ground truth to score it against. It raises uncertainty and records a reason. That reason is the model's own account of its verdict, not a transcript of how the verdict was reached, and current models do not expose the transcript at all. Treat the output as documented diligence, never as proof that a question is clean. Any material that claims otherwise claims too much.

**Using different models across the gates gives only partial independence.** Models from the same provider, and especially from the same family, share training data and therefore share blind spots. Spanning tiers reduces shared error but does not remove it. The gauntlet is the least correlated gate, because it measures behaviour rather than giving a judgment, and shared blind spots are harder to hide in a measurement. The remedy available is the offline audit across vendors described at [F], and it covers a sample from time to time rather than every question.

**The gauntlet measures model behaviour, not human behaviour.** It is a strong proxy for difficulty and a real measurement of something. It is not a substitute for testing questions with real candidates.

**The gauntlet needs a real scale, not a label.** Its signal comes entirely from comparing personas across bands, so an exam with only two ordered bands gives it little room to work with, and an exam with one, a single pass or fail cutoff, gives it none. Single-cutoff exams sit outside what this architecture can calibrate against today, whatever else about them fits the rest of the pipeline.

**Difficulty measurements are tied to one generation of models.** The gauntlet records what one set of personas, under one binding, could do on the day it ran. Models improve, and a persona that fails a question this year may pass it next year, which moves every boundary the gauntlet drew. A dataset's difficulty labels are therefore a statement about the binding recorded beside them, not a permanent property of the questions, and labels produced under different bindings cannot be compared until a new baseline is measured.

**Human review shows quality, not clearance.** No reviewer can detect unconscious copying of material they have never seen.

**The architecture reduces legal risk. It does not remove it.** What it gives you is a claim of independent derivation, supported by records written at the time. That is a much stronger position than having no records. It is not immunity, and nothing here is legal advice.
