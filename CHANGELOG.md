# Changelog

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

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

### Design notes

- `default_rename_options`, `change_code` and `porcelain_line` are `[]`
  today and stayed in git-nv. All three take or return
  `GitRenameOptions`, `GitChange` or `GitStatusEntry`, types that only
  a `[fs]` function can construct. A `core` package declaring a type
  nothing in it can build would publish a shape whose only purpose is
  to serve the half that stayed behind. The ignore rules and the `stat`
  blocks are different: `parse_gitignore` and `stat_of` build theirs
  from a caller's own bytes and numbers.
- `gitwire`'s pkt-line codec — `pkt_line`, `flush_pkt`, `delim_pkt`,
  `read_pkt`, `sideband_of` and the three URL builders — is `[]` as
  well and also stayed in git-nv, because a framing layer with one
  caller is not a package boundary. If a second consumer appears (a git
  server, a proxy, a bundle reader) those eight functions move without
  a signature change.
- git-nv's manifest change when it adopts this package is one line in
  `[dependencies]`, `git-core-nv = "^0.0.1"`, and four `src/*.nv` files
  deleted. Its `layer` stays `host`, because what is left of it reads
  the disk and talks to a server.
