# OverTheWire: Bandit Wargame — Progress Notes

[OverTheWire Bandit](https://overthewire.org/wargames/bandit/) is a well-known, legal, beginner-friendly wargame for practicing Linux command-line and basic security concepts over SSH. Doing this alongside my main lab roadmap as extra hands-on practice — not part of the structured lab plan, but directly reinforces the same Linux fundamentals a SOC Analyst relies on daily.

## Progress Log

| Level | Status | Technique / Notes |
|---|---|---|
| 0 → 1 | ✅ Completed | Basic SSH login and fundamental Linux commands (`ls`, `cd`, `cat`) — reviewed from prior coursework, exact commands not re-documented here. |
| 1 → 2 | ✅ Completed | Reading a file with a dash-prefixed name (`./-filename` or `--` trick). Reviewed from prior coursework. |
| 2 → 3 | ✅ Completed | Handling filenames with spaces (quoting or escaping). Reviewed from prior coursework. |
| 3 → 4 | ✅ Completed | Locating a hidden file (`ls -a`). Reviewed from prior coursework. |
| 4 → 5 | ✅ Completed, fully verified | Ten dash-prefixed files (`-file00` to `-file09`) in a directory, only one human-readable. Used `file ./-file* \| grep ASCII` to filter all files by type in a single command instead of `cat`-ing each one individually, then `cat`'d only the matching file. |
| 5 → 6 | ✅ Completed, fully verified | Password stored somewhere under `inhere/`, described as human-readable, exactly 1033 bytes, not executable. Multiple nested decoy directories and files, including hidden dotfiles. Solved by checking each clue independently instead of jumping straight to a combined command — see below. |
| 6 → 7 | ✅ Completed, fully verified | Password file identified purely by metadata: owned by `bandit7`, group `bandit6`, exactly 33 bytes — located anywhere on the entire filesystem, not restricted to a specific folder. Required searching from filesystem root and filtering out permission errors. |
| 7 → 8 | ✅ Completed, fully verified | Password stored in `data.txt`, on the line next to the word `millionth`. Solved with a single `grep` call once the tool's argument syntax was properly understood. |

## Key Technique Learned (Level 4)

Instead of checking each file one by one:
```bash
cat ./-file00
cat ./-file01
# ...repeated 10 times
```

A much faster approach:
```bash
file ./-file* | grep ASCII
```

This runs `file` (which identifies content type) against all matching files at once, then filters the output to only the lines mentioning ASCII text — immediately pointing to the one human-readable file among the noise.

**Why this matters for SOC work:** this is the same pattern used to triage large sets of unknown files during forensics or incident response — quickly separating readable text/log files from binary or unrelated data before investigating further.

## Key Technique Learned (Level 5) — Verifying Multiple Clues Independently

The level gave three independent clues about the target file: human-readable, exactly 1033 bytes, not executable. Rather than jumping straight to a single combined `find` command, each clue was tested separately first, to *prove* the answer rather than guess it:

```bash
find . -type f -size 1033c            # size clue alone
find . -type f ! -executable          # permission clue alone
find . -type f -exec file {} \; | grep ASCII   # content-type clue alone
```

In this case, the size clue alone (`-size 1033c`) was already enough to return exactly one match (`./.file2`) out of nine candidate files in the directory, so no further disambiguation was needed. Cross-checked with `ls -l` to confirm the file's permissions had no execute bit set.

**Mistake made along the way:** initially read the wrong file (`-file1`, a dash-prefixed decoy of a very different size) because a wildcard search like `file ./file*` silently skips hidden dotfiles (`.file1`, `.file2`, ...) — bash wildcards don't match filenames starting with a dot unless told to. This is intentional shell behavior (it also protects `.` and `..` from being accidentally matched by destructive commands like `rm -rf *`), but it means hidden files require an explicit check (`ls -a` or a dot-prefixed pattern).

**Why this matters for SOC work:** attackers sometimes name malicious files with a leading dot specifically because casual wildcard-based scripts and manual `ls` checks silently skip them. Always verify hidden files exist with `ls -a` or `find -name ".*"` rather than assuming a folder listing is complete.

## Key Technique Learned (Level 6) — Searching the Entire Filesystem by Metadata

This level introduced two new `find` predicates: `-user <name>` (filter by file owner) and `-group <name>` (filter by group owner), combined with the already-known `-size 33c`.

```bash
find / -user bandit7 -group bandit6 -size 33c 2>/dev/null
```

Two things had to go right that didn't on the first few tries:

1. **Search scope:** the level clue said the file could be "somewhere on the server," not restricted to a specific folder. An initial attempt searched only `..` (one directory up, i.e. `/home`), which only covers other users' home directories — not the whole system. The fix was starting the search from `/`, the filesystem root, which covers every mounted directory.
2. **Redirecting noisy errors:** searching from `/` as a non-root user generates a large number of `Permission denied` messages (for directories the current user isn't allowed to read). These are expected and don't indicate a broken command — `2>/dev/null` redirects that error stream away so only valid results are visible.

**Mistake made along the way:** after finding the file's path, tried `cd` into it — but `cd` only works on directories, not files. Also initially tried a relative path (`./var/lib/...`) on a result that was actually an *absolute* path (`/var/lib/...`) from filesystem root, causing "No such file or directory." Corrected to `cat /var/lib/dpkg/info/bandit7.password` — matching the absolute path exactly as `find` reported it, and using `cat` (read a file) instead of `cd` (enter a directory).

**Why this matters for SOC work:** this is essentially file-integrity/ownership triage — identifying a specific artifact on a live system purely by its metadata (owner, group, size) when its name and exact location are unknown, which is a common technique when hunting for planted files during an investigation.

## Key Technique Learned (Level 7) — Correct `grep` Argument Order

```bash
grep millionth data.txt
```

**Mistake made along the way:** first tried `cat data.txt | grep strings millionth`, which failed. This exposed a misunderstanding of `grep`'s argument order: `grep` takes exactly one search pattern as its first argument, and any arguments after that are treated as filenames to search — not additional patterns. Supplying two words after `grep` made it search for `strings` inside a (nonexistent) file called `millionth`. Piping from `cat` was also unnecessary here — `grep <pattern> <file>` reads the file directly and doesn't need `cat` unless chaining through another command (e.g., `sort`) first.

## Notes on Documentation

- Passwords/credentials obtained at each level are intentionally **not included** in this file or in any screenshots committed to this repo, even though Bandit passwords are level-specific practice credentials with no real-world value. This is deliberate practice of not screenshotting or logging credentials — a habit worth having before doing this with real, sensitive systems.
- Levels 0–3 are marked completed based on my own confirmation; I did not re-verify the exact commands used for those levels in this session, so no specific command claims are made for them beyond what's general knowledge from prior coursework.

## Reference

- [OverTheWire Bandit — official wargame](https://overthewire.org/wargames/bandit/)
