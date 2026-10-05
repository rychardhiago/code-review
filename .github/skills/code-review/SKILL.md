---
name: code-review
description: "Review PHP and JavaScript changes, generate a commit-specific report, and set up the configured Git pre-push hook on first use."
---

Review enabled languages in changed files for actionable defects and
applicable coding-standard violations, then generate a Markdown report. Focus
on correctness, security, reliability, and project standards. Do not report
preferences or speculative risks as findings.

## Project configuration and report

At the start of every review, look for `code-review.yaml` in the reviewed
project's root. Read and use it when present. If it is absent, read the bundled
default at [`./assets/code-review.yaml`](./assets/code-review.yaml). Do not
look for or prefer `.code-review.yml`. Keep the bundled default unchanged when
project-specific settings are needed; copy it to the project root and edit
that copy.

- `base_branch`: the branch to compare against when reviewing a local push.
- `review_scope`: `diff` or `full_codebase`, defining which lines in changed
  files are in scope.
- `languages`: optional per-language `enabled` flags. Set `false` explicitly
  to skip a language; otherwise, an available standard enables its review.
- `report_path`: destination/path pattern for the generated Markdown report.
- `review_command`: argument list for a non-interactive review runner.
- `hook`: the Git hook event intended to invoke the review.
- `push_policy`: currently only `report_only` is supported; never block pushes.
- `security_validation_overrides`: optional severity or `off` overrides for
  security rule IDs in `standards/php-standards.md`.
- `version`: configuration schema version.

## Invocation source

At the beginning of each run, determine whether the skill was invoked
directly by a user or launched by the Git hook:

- If `CODE_REVIEW_TRIGGER=pre-push` is present in the environment, treat this
  as a hook-triggered, non-interactive review.
- If that variable is absent, treat this as a direct user invocation.

Record the source as `Git pre-push hook` or `User invocation` in the report.
Never ask questions during a hook-triggered run.

A local `pre-push` hook does not receive the target branch of a future pull or
merge request. For local reviews, use the configured `base_branch` as the diff
base and review the pushed commit's changes against it. Do not claim to know
the remote request's target branch unless that information was explicitly
provided.

Use [`./assets/report-template.md`](./assets/report-template.md) for the report
structure. Resolve its placeholders using the available project and review
context. If there are no actionable findings, state that explicitly; do not
invent findings or metrics.

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

## Language detection and selection

Identify languages in changed files from their extensions and inspect
mixed-language files for embedded languages, such as JavaScript in HTML or
PHP templates. For each detected language, look for a matching
`<language>-standards.md` file in `standards/` (for example,
`javascript-standards.md`). If no standard exists, skip review of that
language. Do not infer support merely from a configuration entry.

After finding a standard, check `languages.<language>.enabled` in the selected
configuration. Skip the language only when this value is explicitly `false`;
otherwise review it using the standard. Adding a standard file therefore
enables review by default. A project-root `code-review.yaml`, when present, is
the selected configuration; use the bundled default only when the root file
does not exist. For mixed-language files, apply enabled standards only to
their corresponding code segments and review cross-language interactions when
they are in scope.

Bundled standards:

- PHP: [`./standards/php-standards.md`](./standards/php-standards.md)
- JavaScript: [`./standards/javascript-standards.md`](./standards/javascript-standards.md)

## Process

### 1. Identify the change

Choose the comparison endpoint in this order:

1. For a local `pre-push` review, use the pushed local commit SHA supplied to
   the hook.
2. Otherwise, use a commit or fixed point explicitly supplied by the user.
3. If no commit is identifiable, compare the current branch and tracked
   working-tree changes against `base_branch` from the selected configuration.
   Use `HEAD` as the endpoint; combine `git diff <base-branch>...HEAD` (the
   branch changes since the merge-base) with `git diff HEAD` (staged and
   unstaged tracked changes). Include untracked files as added files when
   inspecting the working-tree state.

For local hook reviews, use `base_branch` from the selected configuration.
For direct reviews, use an explicit base supplied by the user when present;
otherwise use the configured `base_branch`. If no base is available, ask for
the base branch or commit.

Validate that the base and endpoint resolve before reviewing. Capture the
comparison diff once and use it to identify changed files and line ranges.
When comparing the current state against the base, an empty combined diff
means the branch and tracked working tree have no changes to review: report
that there is nothing to review and do not generate a findings report. If the
current branch is equal to the base but tracked or untracked working-tree
changes exist, review those changes instead. For an explicitly requested
commit comparison or hook invocation, an empty diff is also not a review and
must not produce a report. Do not continue with a bad ref or invalid scope.

### 2. Identify the standards sources

Find project guidance such as `CONTRIBUTING.md` and coding-standard
documentation. Read the bundled standards for each enabled language present
in changed files. For other projects, prefer their own coding standards and
configuration, using bundled guidance only when appropriate.

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

Apply the configurable PHP security validation catalog in
[`./standards/php-standards.md`](./standards/php-standards.md), honoring any
`security_validation_overrides` in the selected configuration. Treat defaults
as review guidance, not proof of vulnerability; confirm applicable context and
existing controls before reporting. Apply the JavaScript security guidance in
its standards document when JavaScript review is enabled.

Generate one report using `assets/report-template.md`. Set `{COMMIT}` in its
heading to the short hash of the reviewed head commit. Name the file
`report-{commit}.md`, where `{commit}` is the short hash of the reviewed commit
(7 characters by default). For a review containing multiple commits, use the
short hash of the reviewed head commit. If the selected configuration sets
`report_path`, use it as the directory or filename pattern, replacing
`{commit}` if that placeholder appears. If the configured path has no
`{commit}` placeholder, append `-{commit}` to the filename stem so reports
from different commits do not overwrite one another. By default, create the
report in the project root as `report-{commit}.md`.

If no findings are identified, say so clearly and include any meaningful
review limitations.

After successfully writing the report, if this was a direct user invocation,
use the available user-question mechanism to ask whether to remove older
reports generated by this skill. Offer **Yes, remove old reports** and
**No, keep them**. Do not ask this for hook-triggered reviews. If the user
confirms, remove only older report files matching
`report-<7-character-commit-hash>.md` in the configured report directory;
preserve the report generated in the current run. Do not remove other
Markdown files or reports that do not match this exact pattern. If
`report_path` uses a different filename pattern, only remove files that
unambiguously match that configured pattern and are reports generated by this
skill; otherwise explain that cleanup was skipped.

## First-invocation ignore setup

On the first invocation in a project, ensure the root `code-review.yaml` is
ignored by Git. Inspect the root `.gitignore` and, if the exact
`/code-review.yaml` rule is absent, append it under a clearly marked
`# Code Review local configuration` section. Create `.gitignore` if needed,
preserve its existing content, and do not duplicate the marker or rule. Do not
alter `.gitignore` if the root config file is not inside the project or cannot
safely be ignored. This only prevents future untracked config files from being
added; it does not untrack a config already committed to Git.

Also ensure generated commit reports are ignored by Git:

1. Resolve the configured `report_path` (or the default `report-{commit}.md`)
   relative to the project root. Replace `{commit}` with seven `?` glob
   characters to form the Git ignore pattern (for example,
   `/report-???????.md`).
2. If the report path is inside the project, inspect the root `.gitignore`.
   If the exact pattern is not already present, append a clearly marked
   `# Code Review generated reports` section and the pattern. Create
   `.gitignore` if it does not exist. Preserve all existing content and do not
   duplicate the marker or pattern.
3. Do not modify `.gitignore` if the report path is outside the project or
   cannot be safely represented by a specific ignore pattern; explain that
   reports will not be ignored automatically.

These checks are idempotent: check the ignore rules on later invocations but
only edit `.gitignore` when a rule is missing. Never replace or rewrite
unrelated ignore rules.

## First-run Git hook setup

When this skill is invoked for a project, check whether its Git
`pre-push` hook already exists. Resolve the hook path with
`git rev-parse --git-path hooks/pre-push`; this also handles worktrees where
`.git` is a file rather than a directory.

- If no hook exists, create a managed `pre-push` hook using the settings in
  the selected configuration file.
- If the existing hook contains this skill's managed marker, update it to
  match the current configuration.
- Never overwrite an existing unmanaged hook. Explain the conflict and ask
  before integrating with or replacing it.
- Do not install a hook if `review_command` is missing or empty. Ask the user
  to configure it first. It must be a non-interactive command and argument
  list that runs the review and writes the report. Execute the list directly
  without shell interpolation or `eval`; do not join it into a shell command.
- Do not make hook installation conditional on the runner being available in
  the current assistant process. The skill may run in a different environment
  from Git (for example, Windows versus WSL), so check runner availability in
  the hook's runtime environment. Install the hook when the command is
  configured, even if it cannot be found during setup.
- At runtime, check whether the configured executable is available. If it is
  missing, print a clear warning naming the executable and explaining that the
  review was skipped; continue the push successfully. If it is present but
  fails, or does not produce the expected report, also warn and continue.
- The bundled example uses GitHub Copilot CLI (`copilot -p`), which must be
  installed and authenticated in the environment where Git runs for reports
  to be generated. If it is unavailable there, tell the user to install and
  authenticate it or configure another runner; do not leave the hook
  uninstalled solely for that reason.

The managed hook must process every non-deletion ref received on pre-push
stdin. For each pushed commit, resolve its short hash and report path, then
export these variables while running `review_command` from the project root:

- `CODE_REVIEW_BASE_BRANCH`
- `CODE_REVIEW_HEAD_SHA`
- `CODE_REVIEW_REVIEW_SCOPE`
- `CODE_REVIEW_REPORT_PATH`
- `CODE_REVIEW_TRIGGER=pre-push`

Before invoking the runner, print to stderr a message that a code review
report is being generated for the pushed ref and commit. On success, verify
that the expected report file exists and print its path to stderr. If the
runner executable is unavailable, the command fails, or the report is
missing, print a warning to stderr and allow the push to continue. The hook
is report-only: findings, including Critical findings, must never reject a
push. Always exit successfully, including when the runner is unavailable,
report generation fails, or the report is missing. Do not add a blocking mode
or configurable push gate unless the user explicitly requests it after
approval from the project leaders.