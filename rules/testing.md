---
paths:
  - "**/*"
---

# Testing

When working on a feature branch, follow these guidelines.

## While working

Run only the tests that cover the code you changed. Pick the test file, class, or target that matches the change and run that alone (for example `-only-testing:` with `xcodebuild`, or the single test file or pattern for the project's test runner).

Do not run the full test suite after each change. It is slow and the result rarely changes what to do next.

## When to run the full suite

Run the full suite only:

- When the user asks for it.
- Once, before merging, as part of completing the branch (the `complete-branch` skill does this).
- When a change touches shared code with no obvious set of covering tests, and say so before running it.

## Reporting

Say which tests you ran and whether they passed. If you did not run the full suite, say so.