# Security & code review: search-sessions

Reviewer: Piotr (fork audit before adoption)
Upstream: sinzin91/search-sessions @ 058b682 (v0.3.0)
Date: 2026-06-06

## Scope

Audited before trusting the tool to read every Claude Code transcript on this
machine (transcripts contain secrets). Three questions:

1. Does it shell out unsafely (command injection)?
2. Can input escape the intended `~/.claude/projects` tree (path traversal)?
3. Does it execute anything, or only print resume commands?

## Verdict

Safe to adopt. No network egress, no self-updater, no second at-rest copy of
transcript data. Memory-safe Rust, 2453 LOC, six well-known local crates
(`clap`, `serde`, `serde_json`, `dirs`, `glob`, `chrono`). One low-severity
hardening item, self-inflicted only.

## Findings

### 1. Subprocess execution — PASS
All four `rg` call sites (src/main.rs ~750, ~1041, ~1172, ~1336) build arguments
with `.args([...])` arrays and pass the search path via a separate `.arg()`.
No shell, no string concatenation, so no command injection from file contents
or paths. The only spawned binary is `rg`; availability is probed once and a
pure-Rust fallback runs if it is missing.

### 2. Path handling — PASS
Base directories come only from `dirs::home_dir()` joined with literal segments
(`.claude/projects`, `.openclaw/...`). `resolve_search_path` (src/main.rs:515)
matches `--project` against names of directories that already exist under base
via `.contains()` and returns that real child path. It never joins raw user
input onto a path, so `../` traversal is not possible.

### 3. Resume behaviour — PASS
The resume command is printed for copy-paste only
(`Resume: cd {} && claude -r {}`, src/main.rs:1535 and 1595). The tool never
executes `claude` or `cd`. README claim holds.

## Hardening (optional, low severity)

The `rg` query is passed positionally before the path without a preceding
`-e`/`--regexp` or `--` separator. A query that starts with `-` (e.g.
`--pre=...`) would be parsed by ripgrep as a flag. Exploitable only by the user
typing their own query against their own files, so not a third-party vector, but
passing the query as `-e <query>` (or inserting `--` before it) removes the
foot-gun. Tracked as a follow-up; not a blocker.

## Conclusion

Adopt. Build from this fork, keep `upstream` wired for updates, re-diff before
pulling new releases.
