# git-core-nv

The git format, with nothing that performs: objects, packfiles, refs,
the index, the ignore grammar and the blob diff — `[]` throughout, so a
program can read what git wrote without taking a filesystem with it.

**Status: NOT IMPLEMENTED — interface only.**  Every `pub fn` body is a
`todo()`, so the signatures, the effect rows and the tests are published
and nothing is implemented.  The first implementation is the `0.1.0`
published over this.

## What this is

This is the half [git-nv](https://github.com/novolang/git-nv) named in
its own README when it was staged: the four modules that declare nothing,
plus the `[]` functions that were sitting in its working-tree module
because that is where their consumers were.  git-nv keeps what performs
— the filesystem, the working tree and the smart HTTP client — and
depends on this.

| module | holds | rows |
| --- | --- | --- |
| `gitobj` | blob, tree, commit, tag; ids under either hash | `[]` |
| `gitpack` | the pack index, and delta resolution over ranges | `[]` |
| `gitref` | ref names, loose refs, `packed-refs` | `[]` |
| `gitindex` | the index file, and the `stat` cache `status` runs on | `[]` |
| `gitignore` | the `.gitignore` grammar, which is not a glob | `[]` |
| `gitdiff` | two blobs, compared, over diff-nv | `[]` |

Six modules, 78 public functions and four trait methods, every body a
`todo()`.

## The load-bearing interface

```novo norun:pseudo
pub enum GitPackStep
    GitPackNeeds(read: GitPackRead, reader: GitPackReader)
    GitPackDone(kind: GitObjKind, payload: Bytes)
    GitPackFollows(read: GitPackRead, reader: GitPackReader)
```

**The core asks and the host answers, and a packfile leaves no other
choice.**  Reading one object out of a pack means seeking to an offset
the `.idx` gave you, inflating a header, discovering the object is a
delta against another offset, seeking there, and repeating until the
chain bottoms out.  `docs/publishing.md` § How a `core` package takes
bytes from its host calls this the third shape; it is the right one here
for the reason it is right for btree-nv's pages — a reader that streamed
the pack would read a gigabyte to answer one object.

So `GitPackRead` is a byte range, `gitpack.step` answers either another
range or a finished object, and **nothing in `gitpack` opens a file**.  A
caller with the pack in memory, in a mapped region, or behind an HTTP
range request uses the same reader.

`GitPackStep` is an enum and not a `Result` because "I need these bytes"
is neither a success nor a failure, and a reader that could not say it
would have to own the file.  It is also what makes this package `core` at
all: the one surface in git's format that genuinely needs random access
is the one that asks for it by value.

## The one example that will work

```novo
use std.bytes
use gitobj

// The name a blob has in every clone of every repository that ever
// existed — computed from the bytes, with no store in the room.
fn name_of(payload: Bytes) -> Str
    gitobj.oid_hex(gitobj.oid_of(gitobj.GitKindBlob, payload, gitobj.GitSha1))

fn main() [io]
    println(name_of(bytes.zeros(0)))
```

That prints `e69de29bb2d1d6434b8b29ae775ad8c2e48c5391`, which is the id
git hard-codes for the empty blob.

## Adding it, and checking it

```console
$ novo pkg add git-core-nv
$ novo pkg build
$ novo test --isolate tests/gitobj_tests.nv
```

The suites are **red on purpose**: every body is a `todo()`, so every
assertion reaches `not implemented: git-core-nv.<module>.<fn>`.  Forty-
four tests across five suites, all red, every failure that message.

`novo pkg add` says `NOT IMPLEMENTED — interface only` on the way in,
because an interface resolves, downloads and builds exactly like an
implemented package and the difference only shows the first time
something calls it.

## The layer, and why

`core`, from the plan — and the line that puts it there is the line
between **what the format says** and **where the bytes came from**.

Everything here is arithmetic over bytes the caller already holds.
Nothing is read, nothing is written, no clock is consulted and no
randomness is drawn.  That is what a registry verifying a tag's commit
wants, and a build cache keyed by a tree id, and a signer that reads a
commit's bytes and never touches a work tree — three consumers that
today would have to take a filesystem to get a hash.

The three dependencies are all `core` too, which is what lets this
package be `core` at all: a dependency's layer may not be wider than its
consumer's.

**No device claim.**  `Bytes`, `Str` and `Result` are throughout and a
packfile is megabytes; a microcontroller has no use for either.  There is
no `tests/embedded_probe.nv`, and the audit's `core-embedded` row passes
on a package that makes no claim.

## What moved out of git-nv, and what stayed

| git-nv 0.0.1 | git-core-nv 0.0.1 | why |
| --- | --- | --- |
| `gitobj` (whole module) | `gitobj` | the object model; every row already `[]` |
| `gitpack` (whole module) | `gitpack` | the pack reader asks for ranges; it opens nothing |
| `gitref` (whole module) | `gitref` | the ref grammar and `packed-refs` are text |
| `gitindex` (whole module) | `gitindex` | the index file format and its `stat` cache |
| `gitwork.parse_gitignore` | `gitignore.parse_gitignore` | the grammar, with its own module |
| `gitwork.gitignore_matches` | `gitignore.gitignore_matches` | — |
| `gitwork.gitignore_reason` | `gitignore.gitignore_reason` | — |
| `gitwork.diff_blobs` | `gitdiff.diff_blobs` | the diff-nv boundary, with its own module |
| `gitwork.similarity` | `gitdiff.similarity` | — |
| `gitwork.paths_to_rehash` | `gitindex.paths_to_rehash` | it is the batch form of `needs_rehash`, and it belongs beside it |

**Every public type and every function keeps the name git-nv published**,
so nothing downstream renames.  Three module paths change, because the
six `[]` functions that lived in `gitwork` needed somewhere to live that
was not a working-tree module: `gitwork.parse_gitignore` is
`gitignore.parse_gitignore`, `gitwork.diff_blobs` is
`gitdiff.diff_blobs`, and `gitwork.paths_to_rehash` is
`gitindex.paths_to_rehash`.  No consumer holds any of them today — git-nv
is an interface release and nothing calls it yet.

**What stayed in git-nv**: `gitrepo` (discovery, the object store, refs
and history — `[fs]` throughout), `gitwire` (the smart HTTP read half),
and the rest of `gitwork` — `status`, `staged_changes`, `is_clean`,
`untracked`, `diff_trees` and `blame`, all `[fs]`.

**git-nv's manifest change at its next version** is one line in
`[dependencies]`:

```toml
git-core-nv = "^0.0.1"
```

and four `src/*.nv` files deleted.  Its `layer` stays `host`, because
what is left of it reads the disk and talks to a server.

### What did not move, and why

`default_rename_options`, `change_code` and `porcelain_line` are `[]`
today and did NOT move, and the third is the interesting one.  All three
take or return `GitRenameOptions`, `GitChange` and `GitStatusEntry` —
types that only a `[fs]` function can produce.  A `core` package that
declared a type nothing in it can construct would be publishing a shape
whose only purpose is to serve the half that stayed behind; the ignore
rules and the `stat` blocks are different, because `parse_gitignore` and
`stat_of` build theirs from a caller's own bytes and numbers.  That is the
test this split used, and it is the one the milestone review should argue
with if it disagrees.

`gitwire`'s pkt-line codec — `pkt_line`, `flush_pkt`, `delim_pkt`,
`read_pkt`, `sideband_of` and the three URL builders — is `[]` as well and
also stayed, because the plan row puts the smart HTTP client in the host
half and a framing layer with one caller is not a package boundary.  If a
second consumer appears (a git server, a proxy, a bundle reader), those
eight functions are the next thing to move and they move without a
signature change.

## Five places the format bites, and where each one is

Every item here is a rule a hand-written git tool gets wrong, and each is
published as its own function so a test can name it:

- **Tree order is not lexicographic.**  A directory sorts as though its
  name ended in `/`, so `foo.txt` comes before `foo/bar`.  A tree in the
  wrong order is an object git reads and whose id differs from what git
  would have written — `gitobj.entries_sorted`.
- **The offset-delta base is not LEB128.**  Big-endian base-128 where
  each continuation adds one before shifting, so `0x80 0x00` is 128
  rather than 0.  A reader that reused a varint routine works on a test
  pack and fails on a real one — `gitpack.parse_ofs_delta_base`.
- **A loose ref beats a packed one, and an empty loose file is a
  deletion.**  A reader that concatenated the two sources reports a
  branch at the commit before its last — `gitref.merge_sources`.
- **A ref name is a file path.**  `..` in one walks out of the
  repository, and a fetch creates refs from names a remote chose —
  `gitref.check_ref_format`, answering every rule broken rather than the
  first.
- **The index is a `stat` cache and `status` runs on it.**  A model
  without the `stat` block produces a `status` that is correct and forty
  times slower; the racy-clean rule is why an entry written in the
  index's own second must be hashed anyway — `gitindex.needs_rehash` and
  `gitindex.is_racy`.

And a sixth this package adds, because the split gave it a home:
**`.gitignore` is not a glob.**  A pattern with no `/` matches at any
depth, one with a `/` is anchored, and a directory exclusion prunes the
walk so a negation inside it can never re-include anything —
`gitignore.prunes_directory` is that last rule, and it is the answer to
"why is my file still ignored".

## The three dependencies

**crypto-nv**, and it is not optional: an object's id is the hash of its
bytes, and this package has both of git's — SHA-1 for the format git has
and SHA-256 for the one it is moving to.  That is why the hash is a
parameter on every call that makes an id rather than a constant, and why
`GitOid` carries which algorithm made it: comparing ids across a
conversion is then a type-level mistake rather than a silent "object
missing".

The SHA-1 is the **plain** one, not the collision-detecting SHA-1DC git
itself now uses.  For every object anybody has committed the two agree;
for the handful of crafted collisions this package would hash them and
git would refuse.

**flate-nv**, because every loose object is a zlib stream and so is every
packed object's payload.  `inflate.feed` is exactly the feed-and-drain
shape a pack reader over a caller's bytes needs — no file, no allocation
the caller did not ask for.

**diff-nv**, because git's own diff is Myers with the same patience and
histogram variants diff-nv already publishes.  `gitdiff.diff_blobs`
tokenises two blobs and hands them to `diffscript.diff`; the hunks a
caller renders are diff-nv's, so a tool that already renders a diff
renders this one.  The one thing git adds — rename detection by
similarity — is arithmetic over the scripts rather than a different
algorithm, and `gitdiff.similarity` is that number on its own.

## What is deliberately absent

- **Anything that opens a file.**  Discovery, the object store, writing
  refs and reading the index off disk are git-nv's, and that is the whole
  point of the split.
- **The smart HTTP protocol.**  git-nv's `gitwire`, for the same reason.
- **Push, merge, rebase.**  Reading a repository and writing one are
  different amounts of work, and git-nv is the reading half; this is the
  format under it.
- **`.git/config`.**  That is config-nv's grammar, and the flags this
  package needs — `core.trustCtime`, `core.checkStat` — arrive as
  arguments, which is what keeps `needs_rehash` testable.
- **SHA-1DC**, as above.

## Reference

gitoxide's `gix-object`, `gix-pack`, `gix-ref` and `gix-index` are the
ports this package's module split follows; libgit2 is the reference
implementation and its test fixtures are the oracle; git's own
`Documentation/technical/` — the pack format, the index format — is the
specification the module headers transcribe.  The vectors in `tests/` are
git's own constants and its own `t/` cases, retyped: the empty-blob and
empty-tree ids from `hash.c`, the ref-name rules from
`t1402-check-ref-format`, the ignore pairs from `t0008-ignores`.

## Licence

Apache-2.0.
