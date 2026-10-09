# proton-wine-cachyos

Experimental build pipeline that rebuilds [CachyOS Wine](https://github.com/CachyOS/wine-cachyos) (the Wine behind
[proton-cachyos](https://github.com/CachyOS/proton-cachyos)) for **Android / bionic**, packaged as a Winlator skyNET
Proton `.wcp`.

Stock proton-cachyos is a glibc Linux build and does not start in Winlator (Box64 fails with
`Global Symbol __libc_start_main not found`). Android-ready Wine links against bionic, so it has to be compiled for Android.

**Status: experimental, untested.** Nothing here has run a game yet.

## How it works

- The Android port is [GameNative/proton-wine](https://github.com/GameNative/proton-wine)'s patch series (`android/patches/`),
  their helper libraries (`android/android_sysvshm`, `android/ntsync_android`, `android/shm_utils`) and build scripts
  (`build-scripts/`), all adapted from that repository.
- The CI workflow checks out CachyOS Wine at a pinned commit, copies `android/` and `build-scripts/` into it, applies the
  patches, builds with Android NDK r27d + llvm-mingw, strips the tree and packs
  `profile.json + bin + lib + share + prefixPack.txz` (prefix pack from
  [GameNative/bionic-prefix-files](https://github.com/GameNative/bionic-prefix-files)).
- Run it from the Actions tab (`Build CachyOS Wine for Android (x86_64)`, *Run workflow*). The package is uploaded as an artifact.

## Porting notes (x86_64 series, against wine-cachyos `2ce6b44`)

GameNative's patches were written for Valve's Wine. 40 of 50 applied unchanged; the rest were adjusted:

| Patch | What was done |
| --- | --- |
| `common/server_fsync_c` | ported: `#if defined(__linux__) && !defined(__ANDROID__)` in `fsync_check_support` |
| `common/dlls_winepulse_drv_pulse_c` | ported: Android main-loop branch and the NULL timer-event guard |
| `common/include_winternl_h` | ported: only `MemoryFexStatsShm` was missing |
| `x86_64/dlls_ntdll_unix_virtual_c` | one hunk dropped (`get_unixlib_funcs` already exists); a duplicated `MemoryWineLoadUnixLibByName`/`Unload` switch block removed |
| `common/dlls_ntdll_unix_server_c` | hunk dropped: GameNative-only `dosdevices` setup with a hard-coded `app.gamenative` path |
| `common/dlls_ntdll_unix_unix_private_h`, `common/dlls_rsaenh_rsaenh_c` | conflicting hunk dropped (already upstream) |
| `common/include_wine_unixlib_h`, `common/dlls_wow64_virtual_c` | dropped entirely (already upstream) |
| `x86_64/dlls_ntdll_unix_loader_c` | the added second `load_unixlib_by_name` removed (CachyOS already defines it; a duplicate broke the build) |
| `x86_64/dlls_ntdll_unix_signal_x86_64_c` | dropped: CachyOS no longer has the seccomp/BPF syscall interception it patches |

The ARM64EC series (`build-scripts/build-step-arm64ec.sh`) has not been ported.

## Credits and licences

The patches and scripts derive from [GameNative/proton-wine](https://github.com/GameNative/proton-wine), which in turn adopts work from
[The412Banner/proton-wine](https://github.com/The412Banner/proton-wine) and bylaws' arm64ec patches. Wine is LGPL-2.1-or-later;
the patches modify Wine and are distributed under the same terms. See `NOTICE`.
