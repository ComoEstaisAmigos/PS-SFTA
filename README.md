# PS-SFTA (UserChoiceLatest fork)

Set the default application for a file extension or URL protocol on Windows 10/11 from
PowerShell, including on builds where Microsoft's newer **UserChoiceLatest** protection is
active.

A single-file continuation of [DanysysTeam/PS-SFTA](https://github.com/DanysysTeam/PS-SFTA) that
fixes [issue #37](https://github.com/DanysysTeam/PS-SFTA/issues/37): on recent Windows 11 the old
`UserChoice` hash is ignored and associations silently revert.

## Why the original stopped working

Windows 11 (feature *AppDefaultHashRotation*, GA on 24H2/25H2) added a second key,
`UserChoiceLatest`, next to `UserChoice` under
`HKCU\...\Explorer\FileExts\<.ext>` and `HKCU\...\Shell\Associations\UrlAssociations\<proto>`.
Once the per-user flag

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\SystemProtectedUserData\<SID>\AnyoneRead\AppDefaults : HashVersion = 1
```

is set, Explorer reads **only** `UserChoiceLatest`. Its hash is salted differently: the input
mixes the extension, your SID, the ProgId, the key's timestamp, the machine ID and one of three
Microsoft salt strings. `SFTA.ps1` computes it and writes **both** keys, so it works whether or
not the new scheme is active. The salt strings are not shipped; they are read at run time from
the machine's own `windows.storage.dll` / `shell32.dll`.

## Requirements

- Windows 10 or 11
- Windows PowerShell 5.1 or PowerShell 7+
- No elevation for ordinary extensions. Types protected by UCPD (`http`, `https`, `.pdf` on the
  tested builds) need UCPD disabled first, which needs administrator rights once (see below).

## Usage

A stock Windows client has the execution policy set to `Restricted`. Allow scripts for the
current window only, and unblock the file if you downloaded it:

```powershell
Set-ExecutionPolicy -Scope Process Bypass -Force
Unblock-File .\SFTA.ps1        # only for a file downloaded from the internet
```

```powershell
. .\SFTA.ps1

Set-FTA   ChromeHTML .htm                 # file extension -> ProgId
Set-PTA   MSEdgeHTM  https                # URL protocol -> ProgId (https is UCPD-protected, see below)
Set-FTA   'Applications\notepad.exe' .txt # an app's ProgId
Get-FTA   .pdf                            # current default ProgId; Get-FTA alone lists all
Get-PTA   http
Register-FTA 'C:\Tools\SumatraPDF.exe' .pdf -Icon 'shell32.dll,100'
Remove-FTA  'C:\Tools\SumatraPDF.exe' .pdf # undo Register-FTA

Test-UclSelfCheck                         # ok / mismatch / noentries for this machine
Get-UcpdState                             # is the protection driver loaded?
Disable-Ucpd                              # admin; only for protected types; reboot after
```

`Set-FTA` / `Set-PTA` also take `-Icon <path,n>`, `-SkipLegacy` (write only `UserChoiceLatest`)
and `-SkipSelfCheck` (skip the canary below; only if you know the hash is right).

`Set-FTA` and `Remove-FTA` accept a program path instead of a ProgId (an explicit path such as
`C:\...` or `.\...` to an existing file); it then means the `SFTA.<name><.ext>` ProgId
`Register-FTA` created. `Remove-FTA` always clears the choice, but deletes the ProgId's
registration only for those `SFTA.*` ProgIds; another application's ProgId is left alone.

From `cmd.exe`:

```bat
powershell -ExecutionPolicy Bypass -Command "& { . .\SFTA.ps1; Set-FTA 'MSEdgePDF' '.pdf' }"
```

Loading the script straight from GitHub, checked against the SHA256 of this release (it stops
instead of running anything if the check fails, so update the hash together with `SFTA.ps1`):

```bat
powershell -ExecutionPolicy Bypass -Command "& { [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12; $f = Join-Path $env:TEMP 'SFTA.ps1'; (New-Object Net.WebClient).DownloadFile('https://raw.githubusercontent.com/ComoEstaisAmigos/PS-SFTA/master/SFTA.ps1', $f); $h = [BitConverter]::ToString([Security.Cryptography.SHA256]::Create().ComputeHash([IO.File]::ReadAllBytes($f))).Replace('-', ''); if ($h -ne '76251738E4F41A52F0B40C16F88042E07C8B1942D15171C5C7173170A0838264') { throw 'SFTA.ps1 hash mismatch' }; . $f; Set-FTA 'Applications\SumatraPDF.exe' '.pdf' }"
```

Running the file directly forwards two or three arguments to `Set-FTA`
(`.\SFTA.ps1 MSEdgePDF .pdf [Icon]`); anything else prints the usage.

### Choosing a ProgId

The hash only proves the association was not forged. Explorer still requires the ProgId to be a
**registered handler for that extension** (its `OpenWithProgids` entry, a RegisteredApplications
capability, or an AppX package); otherwise it silently ignores a valid entry. Use a ProgId that
already handles the type (`Get-FTA .pdf` or Settings > Default apps shows them, e.g. `MSEdgePDF`,
`AcroExch.Document.DC`), or let **`Register-FTA`** register an executable first:

```powershell
Register-FTA 'C:\Windows\System32\notepad.exe' .pdf
```

`Set-FTA 'txtfile' .pdf` writes a correct hash but is ignored, because `txtfile` is not a
registered `.pdf` handler.

## How it is kept correct

- **Token encoder.** The hard part of the hash runs in Windows' own `OpenWithHost.dll`, found by
  a byte signature and used only if it encodes test inputs exactly like the built-in managed
  encoder. The managed encoder is the fallback where the DLL is missing or no longer agrees.
- **Canary.** Before writing, `Set-FTA` recomputes the `UserChoiceLatest` hashes Windows itself
  wrote. If none reproduces, Windows uses a scheme this script does not implement, and nothing
  is written.
- **Scheme rotation.** Windows already ships a second, unused ("v2") set of salts. If
  `HashVersion` goes above 1 or the v1 salts disappear, the script refuses and says why instead
  of writing a hash Explorer would silently reject.
- **Fallback to the legacy key.** Where `UserChoiceLatest` cannot be computed and `HashVersion`
  is not set (Windows 10, builds without the feature), only the legacy `UserChoice` key is
  written, which is the one Explorer reads there.

### UCPD (UserChoice Protection Driver)

UCPD blocks registry writes for a few associations, depending on the build. Measured with UCPD
loaded on Windows 11 26300 and Windows 10 22H2 (19045, where it is on by default):

| Blocked | Not blocked |
| ------- | ----------- |
| `http`, `https`, `.pdf` | `ftp`, `.htm`, `.html`, `.shtml`, `.xhtml`, `.xht`, `.mht`, `.mhtml`, `.svg`, `.xml`, `.webp`, ordinary extensions |

A blocked write changes nothing and stops with a clear message that includes a ready-to-paste
administrator command. To change a protected type, either use Settings > Apps > Default apps, or
run `Disable-Ucpd` as administrator (it sets the UCPD service to disabled and turns off its
`UCPD velocity` re-enable task) and reboot.

## Tested on

| Windows | PowerShell | UCPD | Key Explorer reads | Result |
| ------- | ---------- | ---- | ------------------ | ------ |
| Windows 11 25H2 (26200) | 5.1, 7.6 | off | `UserChoiceLatest` | works |
| Windows 11 26H2 (26300) | 5.1 | off | `UserChoiceLatest` | works, including `.pdf` |
| Windows 11 26H2 (26300) | 5.1 | on | `UserChoiceLatest` | works; `.pdf`, `http`, `https` refused with the clear message |
| Windows 10 22H2 (19045), as installed | 5.1 | on | legacy `UserChoice` | works; `.pdf`, `http`, `https` refused with the clear message |
| Windows 10 22H2 (19045), after `Disable-Ucpd` and a reboot | 5.1 | off | legacy `UserChoice` | works, including `.pdf` |

"Works" means Windows itself resolves the handler that was set (file extensions and URL
protocols) and rejects a tampered hash; a double-click by hand opened the expected app. With
Defender real-time protection on, the script raised no detections.

## Limitations

- **Run as the user whose defaults you change.** The keys live in `HKCU` and the hash includes
  that user's SID; elevating to a *different* admin account writes to that account instead.
- **Reverse-engineered.** The hash is undocumented and a future Windows update can change it;
  the canary and the scheme check make that visible instead of silent.

## Release History

See [CHANGELOG.md](CHANGELOG.md).

## Credits

- **Danyfirex & Dany3j** (Danysys): the original [PS-SFTA](https://github.com/DanysysTeam/PS-SFTA),
  including the legacy `UserChoice` hash. MIT, Copyright (c) 2022 Danysys. PS-SFTA in turn
  credits **LMongrain** (hash algorithm, PureBasic version) and
  [this thread](https://bbs.pediy.com/thread-213954.htm).
- **cssxn**: [UserChoiceLatestHash](https://github.com/cssxn/UserChoiceLatestHash). The managed
  token encoder, its lookup tables and the mixing routines in `SFTA.ps1` are adapted from it.
  MIT, Copyright (c) 2026 cssxn.
- **araghon007**: [DefaultApps](https://github.com/araghon007/DefaultApps) (`NewHash.cs`) first
  published how `UserChoiceLatest` works: locating Windows' own encoder in `OpenWithHost.dll` by
  signature, the three salt GUIDs and the per-class field order. This fork uses that approach.
  DefaultApps has no license, so no code from it is included.

## License

MIT. `SFTA.ps1` carries the full MIT notice with the copyright lines of Danysys, cssxn and
ComoEstaisAmigos (this fork).
