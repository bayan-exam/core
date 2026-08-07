# Hikma core: agent instructions

**This file tells any coding agent working in `hikma-exam/core` what this repository is and what not to do to it.** The single most important rule: this repository is a specification and contains no code, so a change that adds an implementation belongs in an implementation repository instead. It covers scope, how the specification is written, the naming rule, and commit conventions.

`CLAUDE.md` is a symlink to this file. Edit this one.

## Scope

- **Specification only.** `ARCHITECTURE.md` is the specification and `README.md` is its public summary. No pipeline code, no scripts, no dependencies, no lockfiles.
- **`DESIGN.md` is a reference design system, not part of the specification.** It is a default for implementations to adopt, fork, or ignore, and it ships no stylesheet, font, or asset files, because those would be code. Do not add them here, and do not rewrite the document as though it bound anyone. Its section 2 is the exception: an implementation must never imitate the visual identity of the exam it targets, and that rule does not get softened, shortened, or moved into a footnote.
- **Implementations live elsewhere**, one repository per exam, each setting its own license. The MIT grant here does not extend to them.
- **Public and MIT licensed.** Assume everything committed is read by people outside the project.

## Writing the specification

- **`ARCHITECTURE.md` is the source of truth.** `README.md` summarises it and must never contradict it. A change to node behaviour, the schema, model configuration, or the provenance record lands in `ARCHITECTURE.md` first, and the README follows in the same commit.
- **Keep the specification exam-neutral and provider-neutral.** Named exams belong in examples. Named models, providers, and API details belong in the two subsections under Model configuration, which exist so that perishable facts rot in one visible place. Do not scatter them back through the nodes.
- **Write every reader-facing document in plain English.** Most of the audience, learners and developers alike, reads English as a second language. Keep sentences short and to one idea, put the subject and verb near the front, and prefer the common word to the precise-but-rare one. Cut idiom, metaphor, and phrasal verbs where a plain verb exists. This constrains the prose, never the content: the specification stays exact, a defined term keeps its name, and no rule is softened to make it read more easily.
- **Keep both tables of contents in sync** with the headings above them.
- **Never hard-wrap prose.** One line per paragraph, one line per bullet, and let the renderer wrap.
- **Never use the em-dash character.** Use a comma, a colon, parentheses, or a separate sentence.

## Naming

**The project name is Ḥikma, with a dot below the H.** Plain **Hikma** is also correct. Either spelling is fine in prose, so use plain `Hikma` when the diacritic is hard to type or would not survive the text around it, and never change one spelling into the other in text that someone else wrote.

**Never use the diacritic in a name that a machine reads.** URLs, domains, repository names, directory and file names, package names, dataset filenames, and identifiers are always plain ASCII `hikma`. Ḥ is not an ASCII character, so a URL escapes it, a domain encodes it, and different filesystems store it in more than one way. One name then becomes several. If the dot cannot be used, drop it. Never replace it with a dot above the letter or with a different letter.

The monogram is a different thing and always keeps its dot. See `DESIGN.md` section 4.

Implementation repositories are named `hikma-<exam>`, never the bare exam name, and the same prefix applies to package names, dataset filenames, and subdomains. The reasoning is in `README.md` under Contributing. Do not simplify it back.

## Commits and pull requests

**Never add AI attribution.** No `Co-Authored-By` trailer for Claude or any other AI tool, and no "Generated with" footer, in commit messages, pull request bodies, issue comments, or release notes. This holds however much of the change an agent wrote.

Commit directly to `main`. This project is pre-production and does not use feature branches.
