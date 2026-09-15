# git-core-nv

Git stores a project's history as objects named by the hash of their own
contents, packed into files, pointed at by references, and staged through an
index. This package reads and writes those formats in novo-lang, and it opens no
file to do it. The formats are the ones git documents:
[gitformat-pack](https://git-scm.com/docs/gitformat-pack),
[gitformat-index](https://git-scm.com/docs/gitformat-index),
[gitignore](https://git-scm.com/docs/gitignore),
[git-check-ref-format](https://git-scm.com/docs/git-check-ref-format) and
[gitrepository-layout](https://git-scm.com/docs/gitrepository-layout).
[git-nv](https://novo-lang.org/packages/git-nv) is built on this package. It
adds the filesystem, the working tree and the smart HTTP client.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it is
implemented. Version 0.1.0 will be the first working release.

## What it is

An **object** is the unit git stores. There are four kinds: a **blob** is a
file's contents, a **tree** is a directory listing, a **commit** is one revision
of the whole project, and a **tag** is an annotated name for another object. An
object's **id** is the hash of the string `<kind> <length>\0` followed by the
object's contents. Computing an id therefore needs nothing but the bytes. Git
names objects under SHA-1 today and is moving to SHA-256, and the two give
different ids for the same contents.

A **loose object** is one object stored in its own file, compressed with zlib. A
**packfile** holds many objects in one file, and most of them are stored as
**deltas**: a set of copy and insert instructions against another object in the
same pack. Following a delta to the object it is against, and that one to the
next, is walking a **delta chain**. A packfile is paired with an **index file**,
the `.idx`, which says at what offset in the pack each id lives.

A **reference**, or ref, is a name for an object: `refs/heads/main` is a branch,
`refs/tags/v1.0.0` is a tag. A ref is stored either as its own small file under
`.git/refs`, called a **loose ref**, or as a line in `.git/packed-refs`. A
**symbolic ref** names another ref instead of an object, which is what `HEAD` is
on a branch.

The **index** is the file git stages changes in, `.git/index`. It is also a
cache. Each entry records the `stat` data the filesystem reported for the file
when it was last hashed, so `git status` can skip files whose size, mode and
timestamps are unchanged.

A **`.gitignore`** file lists patterns for paths git should not track. The
grammar is not a glob. Order matters, a leading `!` re-includes, and a pattern
containing a slash means something different from one that does not.

Every function in this package takes the bytes as an argument and returns a
value. Nothing is read, nothing is written, no clock is consulted and no
randomness is drawn.

## Install

```
novo pkg add git-core-nv
```

## Example

```novo
use std.bytes
use std.list
use gitobj
use gitref
use gitignore

fn main() [io]
    // The id of an empty file, computed from its bytes with no store present.
    let blob = gitobj.oid_of(GitKindBlob, bytes.zeros(0), GitSha1)
    println(gitobj.oid_hex(blob))

    // Every rule this ref name breaks. An empty list means git would take it.
    println("${list.len(gitref.check_ref_format("refs/heads/main"))}")

    // One .gitignore file, parsed. The empty string is the directory it governs.
    let root = gitignore.parse_gitignore("", "build/\n!build/keep.txt\n")

    // Whether git would ignore this path, given the ignore files above it.
    println("${gitignore.gitignore_matches([root], "build/keep.txt", false)}")
```

The first line prints `e69de29bb2d1d6434b8b29ae775ad8c2e48c5391`, the id git
gives an empty blob in every repository.

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails on
purpose: every test reaches a `not implemented` panic.

## What the package contains

| Module | Contents |
| --- | --- |
| `gitobj` | Blobs, trees, commits and tags, to and from bytes, and object ids under either hash. |
| `gitpack` | The pack index, the pack entry headers, and delta resolution as a reader that asks its caller for byte ranges. |
| `gitref` | Ref names and the rules they must follow, loose refs, `packed-refs`, and resolving a short name to a full one. |
| `gitindex` | The index file, its `stat` cache, its merge stages and its extensions. |
| `gitignore` | The `.gitignore` grammar, the match, and which rule decided. |
| `gitdiff` | Two blobs compared, their similarity as a percentage, and whether git would call a blob binary. |

## How to choose an entry point

**A program that already holds an object's bytes starts at `gitobj`.** Call
`oid_of` for the id, `unframe` for the kind and payload, and `parse_tree`,
`parse_commit` or `parse_tag` for the structure.

**A program reading a packfile starts at `gitpack`.** Call `read_at` with the
pack offset, then `step` with each range of bytes the reader asks for, until it
answers a finished object. See rule 4 below.

**A program that only wants to know where an object is starts at
`gitpack.offset_of`.** It searches the `.idx` and reads no pack bytes at all.

**A program building a `status` starts at `gitindex.paths_to_rehash`.** It takes
the index and one `stat` block per path, and answers only the paths that have to
be read.

`gitobj.oid_of` and `gitpack.step` need the bytes in hand. `gitpack.parse_index`
and `gitindex.parse` keep ranges into the buffer you pass and never copy it, so
that buffer must outlive the parsed value.

## The rules a user needs

1. **An object's id covers a header you must not forget.** The hash is over
   `<kind> <length>\0` and then the payload. `gitobj.oid_of` prepends it.
   gitrepository-layout, "Object storage format".
2. **Tree order is not lexicographic.** A directory compares as though its name
   ended in `/`, so `foo.txt` sorts before `foo/bar`. A tree in any other order
   is an object git reads whose id differs from the one git would have written.
   `gitobj.entries_sorted` is the rule, and `gitobj.serialise_tree` refuses when
   it does not hold.
3. **The two hashes are separate namespaces.** `gitobj.oid_eq` answers `false`
   for a SHA-1 id and a SHA-256 id whatever their bytes. A comparison that
   ignored the algorithm reports every object missing across a conversion.
4. **The pack reader asks and the caller answers.** `gitpack.read_at` and
   `gitpack.step` return a byte range to read. Nothing in `gitpack` opens a
   file, so a pack in memory, in a mapped region or behind an HTTP range request
   all work the same way. gitformat-pack, "Pack file format".
5. **The offset-delta base is not LEB128.** Bytes are big-endian, seven bits
   each, and each continuation adds one to the accumulated value before
   shifting. `0x80 0x00` is 128, not 0. A reader that reuses an ordinary varint
   routine is correct below 128 and wrong above it.
   `gitpack.parse_ofs_delta_base` is that encoding on its own. gitformat-pack,
   "Deltified representation".
6. **A delta declares its output size before anything is allocated.** Set
   `max_object_bytes` on `GitPackLimits` for any pack from a stranger.
   `gitpack.delta_sizes` reads the claim, and `GitPackTooLarge` is the refusal.
7. **A delta chain is bounded at 50 by default, and a cycle is a fault.**
   `GitPackTooDeep` and `GitPackCycle` are answers, not hangs. That default is
   git's own `pack.depth`.
8. **A thin pack is missing its bases.** Every fetch sends one. `step` answers
   `GitPackMissingBase`, and the caller looks the base up in its own store and
   hands it to `gitpack.supply_base`. gitformat-pack, "Thin pack".
9. **A loose ref beats a packed one, and an empty loose file is a deletion.**
   Git writes an empty loose file as a tombstone over a packed ref. Merging the
   two sources by concatenation reports a branch at the commit before its last.
   `gitref.merge_sources` is the rule. gitrepository-layout, "packed-refs".
10. **A ref name is a file path, and a fetch creates refs from names a stranger
    chose.** Check every name with `gitref.check_ref_format`, which answers
    every rule broken rather than the first. git-check-ref-format.
11. **A short name resolves in a fixed order, and a tag beats a branch.** The
    order is the name itself, then `refs/`, `refs/tags/`, `refs/heads/`,
    `refs/remotes/` and `refs/remotes/<name>/HEAD`. `gitref.expand` follows it.
12. **`HEAD` may resolve to nothing.** A fresh repository names a branch whose
    file does not exist yet. `gitref.resolve` answers `None`, and a program that
    assumed `HEAD` always resolves is broken on an empty repository.
13. **The index is keyed by path and stage together.** Stage 0 is an ordinary
    file. Stages 1, 2 and 3 are the base, ours and theirs of an unresolved
    conflict. `gitindex.conflicts` is the query, and a lookup by path alone
    answers the wrong one of three. gitformat-index, "Index entry".
14. **A file written in the index's own second must be hashed anyway.** Its
    mtime cannot distinguish "unchanged" from "changed within the second".
    `gitindex.is_racy` is that test, and skipping it reports a modified file as
    clean, once, unreproducibly.
15. **`needs_rehash` takes the two configuration flags as arguments.** `ctime`
    is compared only under `core.trustCtime`, and `dev`, `ino`, `uid` and `gid`
    only under `core.checkStat = default`. A checkout on a network filesystem
    has all four wrong, and every file would otherwise be reported modified.
16. **Index version 4 prefix-compresses its paths.** `gitindex.parse` undoes
    that, so a caller never sees the difference. `gitindex.serialise` writes
    version 2 unless an entry needs 3, which is git's own rule.
    gitformat-index, "Extensions".
17. **A `.gitignore` pattern with no slash matches at any depth, and one with a
    slash is anchored.** `build/` and `*/build` therefore mean different things.
    gitignore, "Pattern format".
18. **The last matching rule in the deepest file wins.** A search that stopped
    at the first match gets every negation backwards.
    `gitignore.gitignore_matches` takes the whole stack in root-to-deepest
    order, and each file carries the directory it governs.
19. **A directory exclusion prunes the walk.** Git never descends into an
    excluded directory, so a negation written inside it can re-include nothing.
    `build/` plus `!build/keep.txt` keeps nothing.
    `gitignore.prunes_directory` is that rule. gitignore, "Pattern format".
20. **Git calls a blob binary when a NUL byte appears in its first 8000.** That
    is the whole test, and it decides whether a diff is hunks or one line.
    `gitdiff.is_binary`.
21. **Tokenise both blobs in one call.** `gitdiff.tokenise_blobs` interns the
    two sides into one table. Token ids from two separate calls are unrelated
    and must not be compared.
22. **Ranges point into the buffer you passed.** `GitTreeEntry.name_at`,
    `GitCommit.message_at` and `GitIndexEntry.path_at` are offsets, not copies.
    `gitobj.span_str` and `gitindex.path_of` turn one into a string.

## What is not included

- **Anything that opens a file.** Repository discovery, the object store,
  reading and writing refs and reading the index off disk are
  [git-nv](https://novo-lang.org/packages/git-nv)'s.
- **The smart HTTP protocol.** The pkt-line framing and the fetch conversation
  are git-nv's `gitwire`.
- **The working tree.** `status`, `staged_changes`, `untracked`, `diff_trees`
  and `blame` need a filesystem, and they are git-nv's.
- **Push, merge and rebase.** This package reads and writes the formats. Writing
  a repository is a larger job than reading one.
- **`.git/config`.** That is a config file grammar, and
  [config-core-nv](https://novo-lang.org/packages/config-core-nv) is the package
  for it. The two settings this package needs, `core.trustCtime` and
  `core.checkStat`, arrive as arguments.
- **SHA-1DC.** Git uses a collision-detecting SHA-1 that refuses the known
  crafted collisions. This package uses plain SHA-1. For every object anyone has
  committed the two agree. For a crafted collision this package produces an id
  and git refuses.
- **Rename detection across a whole diff.** `gitdiff.similarity` is the
  percentage a rename carries. Pairing added and deleted paths by it is the
  caller's.
- **Running on a microcontroller.** `Bytes`, `Str` and `Result` are used
  throughout, and a packfile is megabytes. There is no device this package is
  meant for, and it makes no claim.

## Related packages

- [git-nv](https://novo-lang.org/packages/git-nv) is the half that performs.
  Repository discovery, the object store, the working tree and the smart HTTP
  client, all built on this package. Every type and function name here is the
  one git-nv publishes.
- [crypto-nv](https://novo-lang.org/packages/crypto-nv) supplies SHA-1 and
  SHA-256. An object's id is a hash, so this dependency is not optional.
- [flate-nv](https://novo-lang.org/packages/flate-nv) supplies zlib. Every loose
  object is a zlib stream, and so is every packed object's payload.
- [diff-nv](https://novo-lang.org/packages/diff-nv) supplies the diff itself.
  `gitdiff.diff_blobs` tokenises two blobs and hands them to `diffscript.diff`,
  so the hunks a caller renders are diff-nv's.
- `std.fs` and `std.process` in the standard library are how a program reaches a
  repository on disk or shells out to `git`. Neither knows anything about the
  formats this package reads.

## Tests

```bash
novo test tests                          # every suite
novo test tests/gitobj_tests.nv          # the ids, and the tree order
novo test tests/gitpack_tests.nv         # the pack numbers, and the offset delta
novo test tests/gitref_tests.nv          # the ref-name rules
novo test tests/gitindex_tests.nv        # the stat cache, and the racy second
novo test tests/gitignore_tests.nv       # the ignore grammar, and the blob diff
```

| Suite | Tests |
| --- | --- |
| `gitobj_tests.nv` | 12 |
| `gitpack_tests.nv` | 8 |
| `gitref_tests.nv` | 8 |
| `gitindex_tests.nv` | 8 |
| `gitignore_tests.nv` | 8 |

`novo test` fails on purpose today. Every assertion reaches a `not implemented:
git-core-nv.<module>.<fn>` panic, because every body is a `todo()`. The tests
are the specification the implementation will have to satisfy.

The reference data is git's own. The empty-blob id `e69de29b…` and the empty-tree
id `4b825dc6…` are the constants `EMPTY_BLOB_SHA1_HEX` and `EMPTY_TREE_SHA1_HEX`
in git's `hash.c`. The pack numbers are `Documentation/technical/pack-format.txt`
and `packfile.c`: the `PACK` signature, versions 2 and 3, the reserved type 5,
the depth of 50, and the offset-delta continuation that adds one before
shifting. The ref names are the accepted and refused cases of git's
`t/t1402-check-ref-format.sh`. The index facts are
`Documentation/technical/index-format.txt` and `read-cache.c`. The ignore pattern
pairs are the shapes in git's `t/t0008-ignores.sh`, and the binary threshold is
git's `buffer_is_binary`.

libgit2 is the reference implementation the fixtures are checked against, and
gitoxide's `gix-object`, `gix-pack`, `gix-ref` and `gix-index` are the crates
this package's module split follows.

## Implementation status

| Item | Implemented |
| --- | --- |
| `gitobj`'s eight types, from `GitHashAlgo` to `GitFramed`, and `impl Error for GitObjFault` | declared |
| `gitobj`'s 24 functions, from `oid_of` to `loose_path` | no |
| `gitpack`'s eight types, from `GitPackKind` to `GitPackEntry`, and `impl Error for GitPackFault` | declared |
| `gitpack`'s 16 functions, from `default_limits` to `depth_of` | no |
| `gitref`'s four types, from `GitRefTarget` to `GitPackedRefs`, and `impl Error for GitRefNameFault` | declared |
| `gitref`'s 15 functions, from `check_ref_format` to `head_name` | no |
| `gitindex`'s six types, from `GitStat` to `GitTreeLevel`, and `impl Error for GitIndexFault` | declared |
| `gitindex`'s 14 functions, from `parse` to `paths_to_rehash` | no |
| `gitignore`'s three types, from `GitIgnoreFile` to `GitIgnoreMatch` | declared |
| `gitignore`'s five functions, from `parse_gitignore` to `parse_rule` | no |
| `gitdiff`'s four functions: `diff_blobs`, `similarity`, `tokenise_blobs`, `is_binary` | no |

78 public functions and four `Error` implementations, every body a `todo()`.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
