---
name: code-review
description: "Review configured PHP changes and generate a report."
---

Review PHP changes for actionable defects and applicable coding-standard
violations, then generate a Markdown report. Focus on correctness, security,
reliability, and project standards. Do not report preferences or speculative
risks as findings.

## Project configuration and report

When reviewing a project that contains `.code-review.yml`, read it before
reviewing. The schema is illustrated in `assets/config.yaml`:

- `base_branch`: the branch to compare against when reviewing a local push.
- `review_scope`: `diff` or `full_codebase`, defining which lines in changed
  files are in scope.
- `report_path`: destination for the generated Markdown report.
- `hook`: the Git hook event intended to invoke the review.
- `version`: configuration schema version.

A local `pre-push` hook does not receive the target branch of a future pull or
merge request. For local reviews, use the configured `base_branch` as the diff
base and review the pushed commit's changes against it. Do not claim to know
the remote request's target branch unless that information was explicitly
provided.

Use `assets/report-template.md` for the report structure. Resolve its
placeholders using the available project and review context. If there are no
actionable findings, state that explicitly; do not invent findings or metrics.

## Review scope

First identify the changed files and line ranges from the comparison diff.
Only files changed by the reviewed change are in scope for either mode.

- `diff`: review only added or modified lines shown in the diff. Read nearby
  unchanged code only as context; do not report issues on unchanged lines
  unless the changed lines cause or expose the defect.
- `full_codebase`: review the complete resulting contents of every changed
  file, not the rest of the project. Findings may point to unchanged lines
  within those files when relevant.

For example, if a change modifies lines 10-20 in `src/Example.php`, `diff`
reviews those changed lines, while `full_codebase` reviews all lines in
`src/Example.php`. Keep finding locations precise and tied to changed files.

## Process

### 1. Identify the change

Use the base supplied by the user. For local hook reviews, use `base_branch`
from `.code-review.yml`. If neither is available, ask for the base branch or
commit.

For a local `pre-push` review, use the pushed local commit SHA supplied to the
hook as the comparison endpoint. Capture the diff command once:
`git diff <base-branch>...<pushed-sha>` (three-dot, so the comparison is
against the merge-base). Also note the commits via
`git log <base-branch>..<pushed-sha> --oneline`. For other reviews, use the
user-supplied fixed point and `HEAD`.

Before going further, confirm the base and endpoint resolve and the diff is
non-empty. A bad ref or empty diff should fail here rather than continuing
with an invalid review scope.

### 2. Identify the standards sources

Anything in the repo that documents how code should be written, such as `CODING_STANDARDS.md` or `CONTRIBUTING.md`.

Alongside documented project standards, apply the **smell baseline** below: a
fixed set of Fowler code smells (_Refactoring_, ch. 3) that can help identify
maintainability concerns. Two rules bind it:

- **The repo overrides.** A documented repo standard always wins; where it endorses something the baseline would flag, suppress the smell.
- **Always a judgement call.** Each smell is a labelled heuristic ("possible Feature Envy"), never a hard violation. Like any standard here, skip anything tooling already enforces.

Each smell reads *what it is* → *how to fix*; match it against the active
review scope:

- **Mysterious Name**: a function, variable, or type whose name doesn't reveal what it does or holds. → rename it; if no honest name comes, the design's murky.
- **Duplicated Code**: the same logic shape appears in more than one hunk or file in the change. → extract the shared shape, call it from both.
- **Feature Envy**: a method that reaches into another object's data more than its own. → move the method onto the data it envies.
- **Data Clumps**: the same few fields or params keep travelling together (a type wanting to be born). → bundle them into one type, pass that.
- **Primitive Obsession**: a primitive or string standing in for a domain concept that deserves its own type. → give the concept its own small type.
- **Repeated Switches**: the same `switch`/`if`-cascade on the same type recurs across the change. → replace with polymorphism, or one map both sites share.
- **Shotgun Surgery**: one logical change forces scattered edits across many files in the diff. → gather what changes together into one module.
- **Divergent Change**: one file or module is edited for several unrelated reasons. → split so each module changes for one reason.
- **Speculative Generality**: abstraction, parameters, or hooks added without a demonstrated need. → delete it; inline back until a real need shows.
- **Message Chains**: long `a.b().c().d()` navigation the caller shouldn't depend on. → hide the walk behind one method on the first object.
- **Middle Man**: a class or function that mostly just delegates onward. → cut it, call the real target direct.
- **Refused Bequest**: a subclass or implementer that ignores or overrides most of what it inherits. → drop the inheritance, use composition.

### 3. Review and report

Review only the files and lines allowed by `review_scope`. Check the applicable
standards and look for concrete defects caused by the change. For each
finding, provide a severity, precise file and line location, concise impact,
actionable recommendation, and the specific standard or reasoning that
supports it. Distinguish definite violations from design heuristics, and skip
issues already enforced by project tooling unless the change introduces a
behavioral defect.

Generate one report using `assets/report-template.md` and write it to
`report_path` from `.code-review.yml` (default: `report.md`). If no findings
are identified, say so clearly and include any meaningful review limitations.