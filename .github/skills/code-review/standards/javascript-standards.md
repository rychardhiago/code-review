# JavaScript Standards

Keep reviews compatible with the JavaScript and browser versions documented by
the project. Do not assume a specific JavaScript edition, browser target,
framework, or library version. Prefer existing project patterns and report
only concrete defects.

## Basic checks

- Treat data from users, URLs, storage, and network responses as untrusted.
- Do not insert untrusted data as HTML or executable code. Use text-safe DOM
  APIs, or an appropriate sanitizer when rendering intentional HTML.
- Validate URLs and avoid passing untrusted input to dynamic script loaders,
  `eval`, or equivalent code-execution APIs.
- Keep authentication and authorization decisions on the server; client-side
  checks are not access controls.
- Handle failed asynchronous requests when ignoring failures can cause
  incorrect behavior or expose sensitive information.
- Check that new APIs and syntax are supported by the project's declared
  browsers/runtime or are covered by its transpilation/polyfill setup.

Do not flag a library API as unavailable until its actual version has been
confirmed from project assets or dependency configuration. For legacy
libraries such as jQuery, follow the version actually used by the project;
avoid recommending newer APIs as replacements without checking compatibility.
