# Windows implementation candidate — validation record

Date: 2026-10-27. This is an AI-assisted implementation candidate, not a claim
of maintainer acceptance or a completed bounty payment.

The supplied archive was based on 11d87caa345fcda41849ed5a562fcf357de76857.
The changes were applied cleanly to canonical xoriors/rencfs main at
60c232777006793636165cf6724059d116d3150b (version 0.14.11). The package version
is unchanged. Existing source licenses are preserved.

## Initial archive checks

Toolchain: nightly-2025-03-01, rustc 1.87.0-nightly. Execution host: Linux x86-64.

| Check | Result |
|---|---|
| `cargo test --locked --lib -- --test-threads=1` | 113 passed, 0 failed |
| `cargo test --locked --test cli_passwd_itest` | 1 passed, 0 failed |
| `cargo check --locked --all-targets` | passed |
| `cargo build --locked --all-targets` | passed |
| `cargo check --locked --target x86_64-pc-windows-gnu --all-targets` | passed |
| release Clippy, all Linux targets | passed, no warnings |
| release Clippy, all Windows GNU targets | passed, no warnings |
| root and Java bridge rustfmt checks | passed |
| Java bridge `cargo check --locked --all-targets` | passed |
| Java bridge `cargo build --locked` | passed |
| `git diff --check HEAD` | passed |

Clippy used the repository's existing allowlist: similar_names,
too_many_arguments, significant_drop_tightening, redundant_closure,
missing_errors_doc and type_complexity. Windows cross-checks used Rust's
Windows GNU standard library and Zig as the C compiler for ring. They establish
Rust type correctness for that target, not a native MSVC link or driver result.

The 8 new core regressions cover portable filename/dot records, legacy reads,
read-only enumeration/release, failed writer-open cleanup, and concurrent writers.
The total 113 library tests includes these regressions.

## Not executed when the initial archive was prepared

- 11 Windows-only callback tests: compiled by the cross-target checks, not run.
- Real WinFsp mounted-drive integration test: not run.
- Native Windows/MSVC build and driver initialization: not run.
- Updated GitHub Actions jobs: not run.
- Linux FUSE integration tests: not run; this host has no /dev/fuse.
- macOS and Android builds: not run.

Run from native Windows Developer PowerShell with Rust/MSVC and the WinFsp 2.1
runtime and Developer files installed:

```powershell
.\scripts\test-windows.ps1 -Drive R: -TranscriptPath windows-test.log
```

Use an unused D:-Z: drive; the script refuses an occupied letter. It uses only a
temporary encrypted store. Preserve and inspect its transcript before marking
the PR ready or claiming the reward. Runtime defects found by this run need to
be corrected, and the script rerun, before describing the mount as tested.

## Native session and follow-up fixes

The native Windows/MSVC build completed successfully on the user's machine.
After the live-state fix, all 12 Windows callback tests and all 3 focused
`test_live_handle` core tests passed there. The live-state fix also passed all
116 library tests on Linux and release Clippy for Linux and Windows GNU.

The native mounted-drive smoke test then failed with OS error 32 while reopening
the truncated file for append. Its backtrace identified
`write_and_check`, at the append-open operation. The old adapter retained the
core's exclusive writer until the final WinFsp Close, which Windows can defer
after Cleanup.

The sharing-lifetime follow-up replaces persistent writers with transient
writer leases per synchronous callback. It keeps readers through ordinary
Cleanup, releases all cleaned contexts before deletion, and rejects subsequent
operations through removed contexts. Four added callback regressions cover
reopen before Close, late paging I/O, multiple writable contexts, write-error
lease release, and deletion with cached contexts. The native smoke test now
adds operation names to its errors.

The sharing follow-up passes Windows GNU all-target checks and release Clippy.
Its 16 Windows callback tests then passed on native Windows. The mounted-drive
test advanced past append but returned an empty vector when reading the file.
The WinFsp trace showed a read of 4096 bytes at offset zero returning 8 bytes,
followed by a second read at offset zero through the same context returning
zero, despite metadata still reporting a size of 8.

The EOF follow-up reproduces that exact failure on Linux before the fix. The
encrypted reader now tracks logical position independently of its decrypted
buffer, preserves position at EOF, and reloads the encrypted block when a seek
cannot use cached plaintext. Empty reads leave reader state intact. Ciphertext
block overhead uses ceiling division so seeking to the end of an exact full
block returns its true length. The shared decryption macro and writer are
unchanged.

Three new crypto regressions and the filesystem reproduction fail before the
fix. All 120 Linux library tests pass with the fix, including all 86 crypto
tests and the filesystem reproduction covering both ciphers. Release Clippy
passes for all Linux and Windows GNU targets. Two additional
Windows callback tests cover repeated reads after EOF and internal reads on
write-only contexts. The adapter now services those observed internal requests;
the native smoke test additionally checks that a user read through the
append-only handle still fails with PermissionDenied.

After this EOF follow-up, the native Windows x86-64/MSVC session passed both
checks below with nightly-2025-03-01. The library suite included all 18 Windows
callback tests.

| Native check | Result |
|---|---|
| `cargo +nightly-2025-03-01 test --locked --lib -- --nocapture --test-threads=1` | 136 passed, 0 failed; 51.06 s |
| `cargo +nightly-2025-03-01 test --locked --features winfsp_wrs/debug --test rencfs_windows_itest -- --ignored --nocapture --test-threads=1` | 1 passed, 0 failed; 4.24 s |

These native results use Cargo's debug/test profile. The mounted-drive run used
an unused `R:` drive and enabled the binding's diagnostic feature. It completed
Unicode file operations, truncate/append, repeated reads, rename and deletion,
graceful unmount, encrypted backing inspection, wrong-password rejection,
read-only remount checks, persistence and final deletion after a writable
remount. The append-only user-handle read was denied as expected.

The release-profile PowerShell runner and updated GitHub Actions jobs have not
been executed in this native session. Their release behavior remains for CI to
verify; these debug-profile results do not claim that exact runner passed.

## Scope and compatibility

See docs/readme/Windows.md. This initial backend intentionally excludes rename
replacement, arbitrary NTFS metadata/ACL mutation, whole-volume flush, and
unlink with other active user handles. Native writable contexts share transient
core writer leases under the WinFsp coarse operation guard. File data
flush is implemented; Unix directory-fsync crash guarantees are not claimed on
Windows. New storage names are readable by this reader, including on Linux;
older releases cannot read the new encoding. Existing Unix backing stores with
Win32-invalid names require a separate migration before copying to Windows.

The Opire listing and upstream issue point at different repository identities.
The existing upstream PR #327 reports a rejected claim caused by that mismatch.
Neither the code nor these local results establish that payment is available.
