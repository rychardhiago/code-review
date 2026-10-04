# Code Review

A tool to review changes in PHP codebases and generate a `report.md` file.

## Review inputs

- A pull request, merge request, or commit.
- The diff associated with the change.
- Optional context about the change.

## Intended workflow

- Developers can start a review locally on demand.
- A local Git `pre-push` hook can review changes before they are pushed,
  independent of the hosting platform.
- The configured base branch determines the diff reviewed by the hook. Git
  does not expose the target branch of a future pull/merge request to a local
  hook.

On its first run in a project, the skill creates a managed `pre-push` hook if
one does not already exist and `review_command` is configured. It will not
overwrite a hook installed by another tool. The hook must be installed
separately in each local clone. The skill first looks for `code-review.yaml`
in the reviewed project's root. If it is absent, it uses the bundled default
[`code-review.yaml`](.github/skills/code-review/assets/code-review.yaml).
Copy that file to the project's root to customize it. On first invocation,
the skill adds `/code-review.yaml` to the root `.gitignore` if the rule is
missing, keeping project-local settings out of version control.

Set `base_branch` to the branch against which changes should be reviewed and
`review_command` to a non-interactive command and argument list that runs the
review and writes its report. The example uses GitHub Copilot CLI in
programmatic mode (`copilot -p`), which must be installed and authenticated
locally; replace the command list to use another AI runner. The hook supplies
the base branch, pushed commit SHA, review scope, and report path through
environment variables. It also sets `CODE_REVIEW_TRIGGER=pre-push`, so the
skill can distinguish hook runs from direct user invocations. Reports record
how the review was started. The hook prints a message before review and the
report path on success. If the review command fails or does not create the
expected report, it warns and permits the push to continue. As with other
local hooks, users can bypass it, so it is a convenience and feedback
mechanism rather than a centrally enforced check.

## Review scope

Set `review_scope` in the project's root `code-review.yaml` to control which
lines are reviewed. Both modes are limited to files changed by the reviewed
commit:

- `diff`: review only added or modified lines in the diff, using surrounding
  unchanged code only as context.
- `full_codebase`: review the complete contents of each changed file, including
  unchanged lines, without scanning unrelated project files.

## Review standards

Reviews target PHP applications. The default standards are PSR-1 and PSR-12;
other PHP-FIG recommendations, such as PSR-4 and PSR-3, apply when relevant to
the changed code. Project-specific documentation, PHP version constraints,
and formatter or linter configuration should also be considered. See
[`php-standards.md`](.github/skills/code-review/standards/php-standards.md)
for review guidance and links to the official specifications.

## Output

The default output is a Markdown report named `report-{commit}.md`, where
`{commit}` is the reviewed commit's short hash. The filename pattern is
configurable with `report_path`. See the
[`report-template.md`](.github/skills/code-review/assets/report-template.md)
for its structure. After a successful review started directly by a user, the
skill asks whether to remove older reports matching
`report-<7-character-commit-hash>.md`; it preserves the report from the
current run. Hook-triggered reviews do not prompt for cleanup.

On its first invocation in a project, the skill adds a specific pattern for
generated reports to the root `.gitignore` (for the default location:
`/report-???????.md`). It preserves existing ignore rules and avoids duplicate
entries.
