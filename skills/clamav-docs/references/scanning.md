# Scanning (ClamAV 1.5)

## Contents

- clamscan or clamd
- clamd
- clamscan
- clamdscan transport modes
- Quarantine actions
- On-access scanning
- PUA detection
- Alerts and false positives
- Troubleshooting
- Windows scanning

## clamscan or clamd

- clamscan loads the signature set on each run and does not use a config file. clamd is a service you start and leave up, and clamd.conf must be set before it starts. clamdscan, clamav-milter and clamonacc send scan requests to it. https://docs.clamav.net/faq/faq-misc.html

## clamd

- Run freshclam before the first clamd start, it needs a database. https://docs.clamav.net/manual/Usage/Scanning.html#clamd
- Start `clamd` plainly: it loads the database and daemonizes itself. Do not use `&` or a background tool. https://docs.clamav.net/manual/Usage/Scanning.html#clamd
- Signals: `SIGTERM` exits cleanly, `SIGHUP` reopens the log, `SIGUSR2` reloads the database. https://docs.clamav.net/manual/Usage/Scanning.html#clamd
- clamd does not protect or authenticate traffic on the TCP socket: it accepts commands from any source that can reach it. Keep it off the Internet. https://docs.clamav.net/manual/Usage/Scanning.html#clamd
- `clamdtop` monitors one or several clamd instances. https://docs.clamav.net/manual/Usage/Scanning.html#clamdtop

## clamscan

- `clamscan .` scans only the current directory. Use `clamscan --recursive .`. https://docs.clamav.net/manual/Usage/Scanning.html#some-basic-scans
- `--max-filesize` skips larger files and assumes them clean. `--max-scansize` caps the data scanned per container. https://docs.clamav.net/manual/Usage/Scanning.html#clamscan
- Defaults worth knowing: `--heuristic-alerts=yes`, `--alert-encrypted=no`, `--official-db-only=no`, `--cross-fs=yes`. https://docs.clamav.net/manual/Usage/Scanning.html#clamscan
- `--memory` scans running process memory (Administrator only). `--kill` and `--unload` act on infected modules. https://docs.clamav.net/manual/Usage/Scanning.html#process-memory-scanning
- In a container, clamscan uses the image's databases, possibly stale. Mount an up-to-date database volume. https://docs.clamav.net/manual/Installing/Docker.html

## clamdscan transport modes

- clamdscan takes its engine settings from clamd.conf. It accepts many clamscan options and silently ignores most. https://docs.clamav.net/manual/Usage/Scanning.html#clamdscan
- Default on a Unix socket: clamd opens the paths itself, so it must see them and have read access. https://docs.clamav.net/manual/Usage/Scanning.html#clamdscan-mode-and-option-interactions
- `--fdpass` passes open file descriptors, for a daemon running as another user or with another filesystem view. Unix socket only. https://docs.clamav.net/manual/Usage/Scanning.html#clamdscan-mode-and-option-interactions
- `--stream` sends file contents. A remote TCP daemon always gets contents. https://docs.clamav.net/manual/Usage/Scanning.html#clamdscan-mode-and-option-interactions
- Pick one transport override per scan, `--stream` or `--fdpass`. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#transport-override-options

| Mode | Command sent |
| ---- | ------------ |
| Local path | `SCAN` family |
| Unix socket with `--fdpass` | `FILDES` |
| Remote TCP or `--stream` | `INSTREAM` |
| stdin | `FILDES` when possible, else `INSTREAM` |
| `--multiscan`, local path | server-side `MULTISCAN` |
| `--multiscan` with `--stream` or remote | `IDSESSION` plus `INSTREAM` per file |
| `--multiscan` with `--fdpass` | `IDSESSION` plus `FILDES` per file |

https://docs.clamav.net/manual/Usage/ClamdProtocol.html#mode-selection

- `--allmatch` works only in local path mode, without `--multiscan`, `--stream` or `--fdpass`. https://docs.clamav.net/manual/Usage/Scanning.html#clamdscan-mode-and-option-interactions

## Quarantine actions

- Choose one of `--move`, `--copy`, `--remove`. clamdscan performs them, not clamd. https://docs.clamav.net/manual/Usage/Scanning.html#clamdscan-mode-and-option-interactions
- `--remove` is "generally a terrible idea" outside upload and download directories, since false positives happen. Default to `--move` or `--copy`, or a clamd `VirusEvent` script. https://docs.clamav.net/manual/Usage/Scanning.html#disclaimers
- ClamAV cannot disinfect files. https://docs.clamav.net/faq/faq-misc.html

## On-access scanning

- Linux only, kernel 3.8+ with fanotify, libcurl 7.45+ (else build with `-D ENABLE_CLAMONACC=OFF`). https://docs.clamav.net/manual/OnAccess.html#requirements
- `clamonacc` is a separate client of clamd and must run as root. It checks its config and the clamd connection, then forks to the background. https://docs.clamav.net/manual/Usage/Scanning.html#clamonacc-v0102
- Exclude clamd's own user with `OnAccessExcludeUname` or `OnAccessExcludeUID`, or clamonacc loops on clamd's own file accesses. https://docs.clamav.net/manual/Usage/Scanning.html#clamonacc-v0102
- With `LocalSocket`, clamd must run as a user allowed to read every watched file. https://docs.clamav.net/manual/OnAccess.html#general-use
- Default is notify only. `OnAccessPrevention yes` blocks access until scanned and seriously hurts performance on busy directories. Restart clamd and clamonacc after changes. https://docs.clamav.net/manual/Usage/Scanning.html#clamonacc-v0102 https://docs.clamav.net/manual/OnAccess.html#general-use

```
OnAccessIncludePath /home/user/Downloads
OnAccessExcludeUname clamav
OnAccessPrevention yes
OnAccessDisableDDD yes
```

https://docs.clamav.net/manual/OnAccess.html#configuration-and-recipes

- Run clamonacc several times with different configs to mix prevention and notify-only directories. https://docs.clamav.net/manual/Usage/Scanning.html#clamonacc-v0102
- `OnAccessIncludePath` rejects `/`. List several paths instead. https://docs.clamav.net/manual/OnAccess.html#troubleshooting
- `OnAccessMountPath` watches a whole mount without the inotify limit, but cannot combine with `OnAccessIncludePath`, `OnAccessExcludePath`, `OnAccessExtraScanning` or `OnAccessPrevention`. https://docs.clamav.net/manual/OnAccess.html#troubleshooting
- Blocking needs `CONFIG_FANOTIFY_ACCESS_PERMISSIONS=y`, check with `grep FANOTIFY /boot/config-$(uname -r)`. Without it, notify only. https://docs.clamav.net/manual/OnAccess.html#troubleshooting
- Recursive watching (DDD) uses inotify, 8192 watches by default. https://docs.clamav.net/manual/OnAccess.html#troubleshooting

```bash
echo 524288 | sudo tee -a /proc/sys/fs/inotify/max_user_watches
```

- clamonacc is silent without `--log=FILE`. `--foreground` keeps it attached. https://docs.clamav.net/manual/OnAccess.html#command-line-options
- Raise `MaxQueue`, `MaxThreads`, `OnAccessMaxThreads` for throughput. https://docs.clamav.net/manual/OnAccess.html#general-use

## PUA detection

- PUA detection must be switched on explicitly (`--detect-pua`, `DetectPUA yes`) and is not recommended in production: less curated, more false positives. https://docs.clamav.net/faq/faq-misc.html https://docs.clamav.net/faq/faq-pua.html
- Enable with `DetectPUA yes` (clamd) or `--detect-pua`. Filter with `ExcludePUA` and `IncludePUA` (`--exclude-pua`, `--include-pua`) on the category, the second token of `PUA.category.subcategory.description`. https://docs.clamav.net/faq/faq-pua.html
- Excluding `Win` does not exclude all Windows PUA (`Packed`, `Tool`, `Spy` and others remain). https://docs.clamav.net/faq/faq-pua.html

## Alerts and false positives

- Do not delete a flagged file out of hand. https://docs.clamav.net/faq/faq-scan-alerts.html
- Find the rule behind a detection: https://docs.clamav.net/manual/Usage/SignatureManagement.html#sigtool

```bash
sigtool --find="Win.Test.EICAR"
```

- Or unpack and decode: `sigtool -u /usr/local/share/clamav/daily.cvd` (path varies by install), `grep -r <SigName>`, `sigtool --decode-sigs`. https://docs.clamav.net/faq/faq-scan-alerts.html#technical-investigation
- Local fix: an allow list entry (`.fp` for MD5, `.sfp` for SHA1 or SHA256) or an ignore entry (`.ign2`), see the signatures reference. Report at https://www.clamav.net/reports/fp or with `clamsubmit`. Changes take at least 48 hours. https://docs.clamav.net/manual/Signatures/AllowLists.html https://docs.clamav.net/faq/faq-ignore.html https://docs.clamav.net/faq/faq-malware-fp-reports.html
- False positives are tested only against the latest stable release, so stay current. https://docs.clamav.net/faq/faq-cvd.html
- Clean locally but known upstream: rerun with `--alert-broken`, and check that freshclam and the scanner use the same database directory. https://docs.clamav.net/faq/faq-cvd.html
- `Phishing.Heuristics.Email.SpoofedDomain` on legitimate mail: a link text naming a protected domain while the href points elsewhere. Allow it in `.wdb`. https://docs.clamav.net/faq/faq-misc.html https://docs.clamav.net/manual/Signatures/AllowLists.html#phishing-heuristic-allow-lists
- `Oversized.zip`: the archive exceeded `ArchiveMaxCompressionRatio`. https://docs.clamav.net/faq/faq-misc.html
- Find which mbox message is infected with `clamscan --debug`, line "Deal with email number". https://docs.clamav.net/faq/faq-misc.html

## Troubleshooting

- Unresponsive clamd, stopping after about 24 hours, or crashing once a day: not enough RAM for the daily reload. https://docs.clamav.net/faq/faq-troubleshoot.html#clamav-becomes-unresponsive

## Windows scanning

- No shell wildcard expansion: ClamAV emulates `*` and `?` only. UNC paths work. https://docs.clamav.net/manual/Usage/Scanning.html#windows-specific-issues
- `cl_scanfile()` and clamd socket commands take UTF-8 and reply in UTF-8. Console output is OEM encoded, even redirected. https://docs.clamav.net/manual/Usage/Scanning.html#socket-and-libclamav-api-input
