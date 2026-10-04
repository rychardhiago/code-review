---
name: code-review
description: "Review PHP changes, generate a commit-specific report, and set up the configured Git pre-push hook on first use."
---

Review PHP changes for actionable defects and applicable coding-standard
violations, then generate a Markdown report. Focus on correctness, security,
reliability, and project standards. Do not report preferences or speculative
risks as findings.

## Project configuration and report

At the start of every review, look for `code-review.yaml` in the reviewed
project's root. Read and use it when present. If it is absent, read the
bundled default at `assets/code-review.yaml`. Do not look for or prefer
`.code-review.yml`. Keep the bundled default unchanged when project-specific
settings are needed; copy it to the project root and edit that copy.

- `base_branch`: the branch to compare against when reviewing a local push.
- `review_scope`: `diff` or `full_codebase`, defining which lines in changed
  files are in scope.
- `report_path`: destination/path pattern for the generated Markdown report.
- `review_command`: argument list for a non-interactive review runner.
- `hook`: the Git hook event intended to invoke the review.
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

Use the base supplied by the user. For local hook reviews, use `base_branch` from the selected configuration.
If neither is available, ask for the base branch or commit.

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

Find project guidance such as `CONTRIBUTING.md` and coding-standard
documentation. This skill bundles default PHP guidance in
`standards/php-standards.md`; read it when applicable. For other projects,
prefer their own coding standards and configuration, using the bundled
guidance only when appropriate.

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

Generate one report using `assets/report-template.md`. Name it
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
must exit successfully even if report generation fails; report generation
failures must not block a push.