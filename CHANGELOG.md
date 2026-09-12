# Changelog

## 0.0.1 — interface

The interface, published before anything is implemented: every `pub fn`
body is a `todo()`, and the signatures, the effect rows and the tests are
the design.

- Six modules, `[]` throughout: `gitobj` (the object model and ids under
  either hash), `gitpack` (the pack index and delta resolution as a
  reader over a caller's byte ranges), `gitref` (ref names, loose refs
  and `packed-refs`), `gitindex` (the index file and its `stat` cache),
  `gitignore` (the ignore grammar) and `gitdiff` (the blob diff over
  diff-nv).
- 78 public functions and four `Error` implementations, every body a
  `todo("git-core-nv.<module>.<fn>")`.
- Five test suites, 44 tests, red on purpose against git's own constants
  and its own `t/` cases.
- The split git-nv 0.0.1's README named: the `[]` half of that package,
  with every type and function name unchanged.  Three module paths move
  — `parse_gitignore`, `gitignore_matches` and `gitignore_reason` into
  `gitignore`, `diff_blobs` and `similarity` into `gitdiff`, and
  `paths_to_rehash` into `gitindex` beside `needs_rehash`.
