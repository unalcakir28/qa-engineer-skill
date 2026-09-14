# CLI / binary / filesystem-tool checklist

Merge the applicable items into the Phase 1 matrix. A command-line tool has no
UI to click and no endpoint to curl, so the categories land differently: its
contract is **exit code + stdout + stderr + what it did to the disk**, and most
of its hostile input space is the filesystem rather than a request body.

Two framing notes before the list:

- **Category 7 (permissions/tenancy) usually does not apply**, and category 13's
  injection surface is different. Say so once in the report with the reason,
  instead of generating empty cases — but read "Privilege and trust boundaries"
  below before concluding the tool has no security surface at all.
- **The oracle is rarely the tool itself.** A tool that reports a number needs an
  independent way to know the number is right (a second implementation, a
  hand-computed fixture, a system utility, an invariant that must hold). See
  `oracles.md`; without one, every case degrades into "it printed something".

## The contract: exit code, stdout, stderr

- Exit code `0` on success and **non-zero on every failure path** — including the
  paths that print an error and then fall through. A tool that prints `error:`
  and exits `0` breaks every script that wraps it.
- Distinct codes for distinct failure classes if the tool documents them (usage
  error vs runtime failure vs partial success vs interrupted); the documented
  table and the real behaviour must agree.
- **stdout is the payload, stderr is the commentary.** Progress, warnings,
  prompts and diagnostics go to stderr; the parseable result to stdout. Verify by
  piping stdout alone and confirming the stream is still valid.
- Machine-readable modes (`--json`, `--format`, `--porcelain`) emit *only* the
  document on stdout — no banner, no progress line, no trailing summary. Parse
  the output with a real parser in the test, not a regex.
- The machine-readable schema is a contract: field names, types, null vs missing,
  number precision, key order if documented. A field removed or renamed between
  versions is a breaking change (category 19).
- `--help` and `--version` exit `0`, go to stdout, and work with no config, no
  network and in an empty directory. Every flag in `--help` actually exists;
  every flag that exists is in `--help`.
- Error messages name the offending input (which path, which flag, which value)
  and say what was expected. "Invalid argument" with no referent is a finding.

## Argument and flag parsing

- Missing required argument, unknown flag, misspelled flag, flag without its
  value, value that looks like a flag (`--out --verbose`), duplicated flag
  (last-wins or error — pick one and be consistent).
- `--` terminator: everything after it is positional, even if it starts with `-`.
- A file literally named `-foo`, `--`, `-`, or an empty string as an argument.
- Mutually exclusive flags passed together; dependent flags passed alone
  (`--format=json` without the subcommand that produces output).
- Boundary values on every numeric flag: `0`, `-1`, `1`, max, max+1, a float
  where an integer is expected, a value with a unit suffix if suffixes are
  supported (`10`, `10k`, `10KiB`, `10 KB`).
- Abbreviated/prefix flag matching, if the parser does it: does `--ver` mean
  `--verbose` or `--version`? Ambiguity must error, not guess.
- Subcommands: unknown subcommand, no subcommand, a subcommand's flags applied
  to another subcommand, global flags before *and* after the subcommand.

## Filesystem input — the real hostile surface

This is where a filesystem tool's bugs live. Build the fixtures; they are cheap
and they find things no argument matrix reaches.

- **Path shapes:** spaces, newlines, tabs, quotes, `*`, `?`, `[`, backslashes,
  leading `-`, trailing spaces/dots, non-UTF-8 bytes in a name, a name at the
  filesystem's maximum length, a path near the OS path limit, deeply nested
  directories.
- **Unicode:** Turkish characters and the `i`/`İ` case trap, emoji, RTL marks,
  and — specifically — **NFC vs NFD normalisation**: the same visible name stored
  two ways is two different names on some filesystems and one on others.
- **Link types:** symlink to a file, to a directory, to a nonexistent target
  (broken), to an ancestor (**cycle**), relative vs absolute symlink, hardlink
  (counted once or twice — whichever the tool promises, assert it), and on
  platforms that have them, junctions/reparse points.
- **Special files:** empty file, sparse file (logical size ≫ blocks used),
  zero-length directory, FIFO, socket, device node, a file that is a mount point,
  a file being written while the tool reads it.
- **Permission and access errors:** unreadable file, unreadable directory
  (traversal denied mid-walk), file deleted between listing and stat, directory
  replaced by a file mid-run. The rule to assert: the tool **counts and reports
  the failure and keeps going** — a single unreadable path must not abort a run
  or silently vanish from the totals.
- **Mounts and remote filesystems:** a network mount that stops responding, a
  read-only mount, crossing a filesystem boundary when the tool claims it does
  not, a mount appearing or disappearing mid-run.
- **Destination side:** output path already exists (overwrite, refuse, or
  `--force`), output directory missing, output path not writable, disk full,
  output written to the same file being read.

## Output correctness and idempotency

- Run the tool twice on unchanged input: identical result (or a documented,
  explained difference). Non-determinism that is real — parallel traversal,
  hash-map iteration, timestamps — must be stated in the docs and must not change
  the *answers*, only incidental ordering.
- Run it with different parallelism/thread settings: the answers must not change.
  This is one of the highest-value cases for any concurrent tool and it is cheap.
- Compare against an independent oracle on a fixture whose correct answer is
  known by construction (a directory you built to contain exactly N bytes in M
  files). Assert the number, not that a number appeared.
- Totals reconcile: the sum of the parts equals the reported whole, at every
  level of any hierarchy the tool prints.
- Empty input: empty directory, zero matches, empty stdin. The tool succeeds with
  an empty result — it does not error, and its machine-readable mode emits a
  valid empty document (`[]`, not nothing).

## Streams, pipes and the terminal

- Piping stdout into `head` closes the pipe early: the tool must exit cleanly on
  `EPIPE`, not print a panic or a broken-pipe traceback.
- **TTY vs non-TTY:** colour codes, progress bars, spinners and interactive
  prompts are suppressed when stdout is not a terminal. Redirect to a file and
  grep for escape sequences.
- `NO_COLOR` / `--no-color` / `TERM=dumb` honoured.
- Reading from stdin when stdin is a terminal, a pipe, a file, and `/dev/null`.
- Very wide and very narrow terminal widths; output that must stay aligned.
- Output flushed and complete when the process exits — a buffered last line lost
  on exit is a real and easily missed bug.

## Signals, cancellation and long runs

- `SIGINT` (Ctrl-C) mid-run: the exit code is the documented interrupted code,
  partial output is **not** presented as a complete result, and any partially
  written output file is removed or clearly marked. Writing a truncated artefact
  that looks valid is the worst outcome here.
- `SIGTERM` and, where relevant, `SIGHUP`; a second signal during cleanup.
- Cancellation while holding a lock, a transaction, or a half-written file.
- Long-running phases emit progress that actually moves. A phase with no counter
  is indistinguishable from a hang — walk the run and confirm every phase
  advances something observable.
- Timeouts: an operation that never returns (dead mount, unresponsive dependency)
  is bounded, and exceeding the bound is reported as a per-item failure rather
  than a whole-run abort.

## State, config and environment

- Config precedence, asserted explicitly: defaults < config file < environment
  variable < flag. Test each adjacent pair, not just the extremes.
- Missing config file, unreadable config file, malformed config, unknown key,
  a key with the wrong type, an empty config file.
- Environment variables: unset, empty string, whitespace, an unexpected value.
  `HOME`/`XDG_*`/`TMPDIR` unset or pointing somewhere unwritable.
- First run with no state directory; upgrade from an older state/schema version;
  a state file written by a *newer* version (must refuse clearly, not corrupt);
  a corrupt or truncated state file.
- Two instances of the tool running at once on the same state: locked, queued, or
  cleanly refused — never silent interleaved corruption. Kill one mid-write and
  verify the other's view stays consistent.
- Cleanup: temp files removed on success, on error, and on interrupt.

## Privilege and trust boundaries

Even without auth, a filesystem tool has a security surface:

- Input paths that escape the intended root (`../`) when the tool takes a base
  directory; symlinks that redirect a write outside the destination.
- Untrusted input files: a crafted/corrupt input that the tool parses (imported
  snapshots, caches, archives, state files) must fail with an error, not a panic,
  an unbounded allocation, or an infinite loop. Fuzz this surface if it exists
  (`automation-toolbox.md`).
- Sensitive data in output: absolute paths, usernames, tokens or file contents
  leaked into logs, error messages, crash dumps or telemetry.
- File permissions on anything the tool creates — a state file or export that
  contains sensitive data must not be world-readable.
- If the tool runs with elevated privileges or as a service: what it does when
  the caller is not the owner of the target, and whether it follows symlinks it
  should not.

## Cross-platform

- Path separators, case-insensitive-but-case-preserving filesystems, reserved
  names and characters on Windows (`CON`, `NUL`, `:`, trailing dot/space).
- Line endings in output and in parsed input files.
- Locale-dependent formatting: decimal comma, thousands separator, date format,
  and case conversion — the `i`/`İ` trap breaks uppercase comparisons under a
  Turkish locale specifically.
- Behaviour claimed for a platform that cannot be exercised locally is
  `NOT RUN (platform)` with the environment where it *can* run named — never a
  guessed PASS. If CI is the only place it runs, say so in the report.

## Packaging and distribution

Relevant at release-gate depth (category 19):

- The built artefact runs on a machine without the toolchain and without the
  source tree.
- `--version` reports the version that was actually built, including commit and
  build channel if the tool embeds them.
- Checksums match the published files; the archive extracts to the expected
  layout; the binary is executable after extraction.
- Install and upgrade paths: install over an existing older version, and run the
  new binary against the old version's state.
