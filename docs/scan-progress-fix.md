# Scan progress and speed patch for v1.1.0

## Reproduced problem

The published v1.1.0 scanner collects every directory result in memory and
only writes after finishing an entire watched root. Watching `C:\` therefore
shows `Scanning C:\ | DB: 0 entries` throughout the recursive walk. USN
`Rescan queued` messages come from a separate watcher; they prove change
detection is alive, not that the scheduler has finished or committed a scan.

`IOThrottle::Checkpoint()` is documented in directories, but v1.1.0 calls it
after every file too. At Low priority that adds 10 ms per 50 files; Very Low
adds 50 ms per 10 files. A million files can incur several minutes of sleeps
even on a fast SSD.

The deterministic production-engine test pauses `FindFirstFileExW` at the
550th enumeration in a 600-folder fixture. An independent read-only database
connection sees **0 rows** before this fix and **500 rows** afterwards. The
pause and failure hooks exist only in the test executable, using the project's
existing Detours dependency. They do not change the installed binaries.

## Resulting behavior

- Commit each batch of 500 completed folders during recursion, or flush a
  smaller pending batch after about a second of active enumeration.
- Log files seen, directories completed, entries published and current path
  about once a second. Counts are activity counters, not a percentage.
- Throttle once per completed directory instead of once per file.
- Notify Explorer about committed folder entries.
- Keep completed child results when interrupted. Do not write incomplete
  parent totals or label interrupted/failed scans successful.
- Report database commit and enumeration errors instead of silently treating
  truncated results as complete.

The implementation still uses Windows' large-fetch directory enumeration.
It does not implement an NTFS MFT scanner. A raw MFT implementation can be
faster for whole NTFS volumes, but needs separate handling for hard links,
reparse points, permissions and non-NTFS/network volumes. The small benchmark
below does not establish parity with WinDirStat or whole-drive performance.

## Verification

On this Windows 11 x64 machine, a 5,000-file single-directory fixture at Low
priority took **1,555 ms and 100 artificial sleeps** before the patch, versus
**24 ms and zero artificial sleeps** after it (a later regression run: 21 ms).
File creation is outside the timed scan. The result is a warm local microbenchmark.

Tests cover an independent reader observing results before root completion,
cancellation, time-based flushing below 500 folders, empty folders, injected
mid-enumeration failure preserving cached root totals, and file-heavy totals
and throttle calls. Existing database and scanner tests also pass with
assertions explicitly enabled in Release. The legacy IPC test is excluded
from the local run because it uses the same named pipe as the installed app.

## Build and test

Requires Visual Studio 2022 or newer with the C++ workload, CMake and vcpkg.
Set `VCPKG_ROOT` to the vcpkg directory. Dependencies and the CRT are static.

```powershell
cmake -S . -B build -A x64 `
  "-DCMAKE_TOOLCHAIN_FILE=$env:VCPKG_ROOT/scripts/buildsystems/vcpkg.cmake" `
  -DVCPKG_TARGET_TRIPLET=x64-windows-static
cmake --build build --config Release --parallel 4
ctest --test-dir build -C Release `
  -R 'DatabaseTests|ScannerTests|ScanProgressTests' --output-on-failure
build/tests/Release/test_scan_progress.exe --benchmark
```

For an MSI, install WiX 6.0.2 and its Util/UI extensions. Stage these files in
`build/installer-stage`: `DirSizeSvc.exe`, `DirSizeTray.exe`,
`DirSizeShellExt.dll`, and `DirSizeTotalSize.propdesc` from the respective
Release build folders.

```powershell
wix build installer/Product.wxs installer/Components.wxs `
  -d BuildDir=build/installer-stage -o build/DirSizeForExplorer-1.1.1.msi `
  -arch x64 -ext WixToolset.Util.wixext/6.0.2 -ext WixToolset.UI.wixext/6.0.2
```

The MSI's version is 1.1.1.0 with the existing upgrade code, so it targets an
upgrade from 1.1.0. Packaging and contents have been checked; the installer has
not been run against the user's live installation. This patch retains the
old service architecture and its existing permissions, cache and watcher
limitations. Unreadable/truncated subtrees leave ancestor totals incomplete;
completed accessible subtrees are still published.
