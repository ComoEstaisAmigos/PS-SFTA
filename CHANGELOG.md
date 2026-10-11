# Changelog

All notable changes to this fork are documented here. Format based on
[Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## 2.0.0 - 2026-10-11

Single-file continuation of upstream 1.2.0 that adds Windows 11 **UserChoiceLatest** support
(fixes DanysysTeam/PS-SFTA#37). Changes below are relative to upstream 1.2.0.

### Added
- `UserChoiceLatest` hashing: `Set-FTA` / `Set-PTA` write both `UserChoice` (legacy) and
  `UserChoiceLatest`, the key Explorer reads exclusively once the per-user `HashVersion` flag is
  set (Windows 11 24H2 and later). Where the feature is absent (Windows 10, older builds) only
  the legacy key is written, which is the one Explorer reads there.
- Token encoder: Windows' own `OpenWithHost.dll` encoder is located by signature (not a fixed
  offset) and only used if it encodes test inputs exactly like the managed encoder; the
  self-contained managed encoder is the fallback where the DLL is missing or does not agree.
- The salt strings are read at run time from the machine's own `windows.storage.dll` /
  `shell32.dll`, located by GUID; none are shipped in the script.
- `Test-UclSelfCheck`: recomputes the hashes Windows wrote. `Set-FTA` refuses to write when none
  of them reproduce, and when the `HashVersion` flag reports a scheme newer than v1 (Windows
  already ships an unused second salt set), so a hash scheme rotation never leads to a silent
  failure.
- `Get-UcpdState` / `Disable-Ucpd`: detect and (as administrator) disable the UserChoice
  Protection Driver and its `UCPD velocity` re-enable task; a reboot applies it.
- When Windows denies a write (UCPD protects `http`, `https` and `.pdf` on the tested builds),
  `Set-FTA` / `Set-PTA` stop with a clear message, including a ready-to-paste administrator
  command with the script's full path, and change nothing.
- `-SkipLegacy` (write only `UserChoiceLatest`) and `-SkipSelfCheck` (bypass the canary).
- `.\SFTA.ps1 <ProgId> <Extension> [Icon]` forwards to `Set-FTA`; any other direct run prints
  the usage and exits with code 1.
- `Get-FTA` / `Get-PTA` report the `UserChoiceLatest` ProgId first, falling back to
  `UserChoice`.

### Changed
- `Remove-FTA` clears `UserChoiceLatest` as well as `UserChoice`.
- Write order: `UserChoiceLatest` first, then the legacy key, then the toast entries, so a
  refused write leaves nothing behind.
- `Set-FTA` throws, writing nothing, when `UserChoiceLatest` cannot be computed but the
  `HashVersion` flag is set (a legacy-only write would be ignored there).
- `Register-FTA` requires the program path to be an existing file (a folder was accepted).

### Fixed
- Locale bug: the legacy `UserChoice` base string was lowercased with culture-aware
  `.ToLower()`, which corrupts the hash on some locales. It now uses invariant-culture
  lowercasing (the new `UserChoiceLatest` input is lowercased ASCII-only, matching the OS).
- Legacy `UserChoice` race: the hash used the minute of "now" taken before the write, so a write
  landing in the next minute (e.g. called at hh:mm:59.7) produced a hash Windows rejected, and
  Windows deleted the key. Where the legacy key is authoritative (Windows 10) the default then
  silently stayed unchanged. The key's real write time is now read back and the hash redone when
  the minute moved.
- `Remove-FTA` deleted `HKCU\Software\Classes\<ProgId>` and the `OpenWithProgids` entry for any
  ProgId, so clearing a choice for another application's ProgId (a per-user `ChromeHTML`, an
  `AppX...` ProgId) wiped that application's registration. It now deletes the registration only
  for `SFTA.*` ProgIds that `Register-FTA` creates, and otherwise just clears the choice.
- `Set-FTA` / `Remove-FTA` ran `Test-Path` on the ProgId, so a ProgId was silently rewritten to
  `SFTA.<name><ext>` whenever the current directory held a file or folder of that name, or a
  wildcard in it matched. Only an explicit path to an existing file now counts as a program.

### Notes
- Tested Windows and PowerShell versions are listed in the README ("Tested on").
- The `UserChoiceLatest` hash is reverse-engineered and undocumented; a future Windows change can
  alter it. The self-check canary is the early warning. A second "v2" salt/order set found in the
  DLL is documented in the source but unused, as no shipping build activates it.

## 1.2.0 - 2022-04-17 (upstream DanysysTeam/PS-SFTA)
- Refresh ApplicationAssociationToasts to avoid OpenWith.exe on first default selection.

## 1.1.0 - 2020-09-14 (upstream)
- Hash algorithm as raw PowerShell code.

## 1.0.0 - 2020-09-08 (upstream)
- First release.
