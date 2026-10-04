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

The skill provides review instructions; it does not install Git hooks by
itself. The hook must be installed in each local clone. The example
configuration is
[`config.yaml`](.github/skills/code-review/assets/config.yaml); copy it to the
reviewed project's root as `.code-review.yml` and set `base_branch` to the
branch against which changes should be reviewed. The hook installer and
runtime are planned follow-up work. As with other local hooks, users can
bypass it, so it is a convenience and feedback mechanism rather than a
centrally enforced check.

## Review scope

Set `review_scope` in `.code-review.yml` to control which lines are reviewed.
Both modes are limited to files changed by the reviewed commit:

- `diff`: review only added or modified lines in the diff, using surrounding
  unchanged code only as context.
- `full_codebase`: review the complete contents of each changed file, including
  unchanged lines, without scanning unrelated project files.

## Review standards

Reviews target PHP applications. The default standards are PSR-1 and PSR-12;
other PHP-FIG recommendations, such as PSR-4 and PSR-3, apply when relevant to
the changed code. Project-specific documentation, PHP version constraints,
and formatter or linter configuration should also be considered. See
[`CODING_STANDARDS.md`](CODING_STANDARDS.md) for review guidance and links to
the official specifications.

## Output

The default output is a Markdown report named `report.md`, configurable with
`report_path`. See the
[`report-template.md`](.github/skills/code-review/assets/report-template.md)
for its structure.
