---
name: clamav-docs
description: Deploys, configures and integrates ClamAV 1.5 from the official ClamAV documentation. Use when installing ClamAV or running the clamav/clamav Docker image, writing clamd.conf or freshclam.conf, keeping signature databases up to date with freshclam or a private mirror, scanning with clamscan or clamdscan, writing a client for the clamd socket protocol (INSTREAM, FILDES, IDSESSION, PING), setting up on-access scanning with clamonacc, writing custom signatures (.hdb, .hsb, .ndb, .ldb, .ftm, .wdb, .pdb, .cdb, .crb, YARA rules), handling a false positive with an allow list or an .ign2 ignore list, or linking against libclamav. Triggers on ClamAV, clamd, clamscan, clamdscan, freshclam, clamd.conf, freshclam.conf, clamonacc, on-access, INSTREAM, StreamMaxLength, sigtool, logical signature, YARA, clamav/clamav, _base image, cvdupdate, functionality level, FLEVEL, libclamav, cl_scanfile, false positive, PUA.
metadata:
  version: "1.5"
  source: https://docs.clamav.net/
  read: "2026-11-02"
---

# ClamAV 1.5

## Overview

This skill holds the ClamAV rules that are easy to get wrong: a clamd TCP socket that anyone can reach, a container killed once a day by the database reload, a world-accessible Unix socket, freshclam traffic that gets blocked, signatures that never fire because of a missing functionality level. Every claim carries the docs.clamav.net page it comes from. The reference files hold the detail per domain.

ClamAV is a malware detection toolkit designed especially for mail gateways, not an endpoint security suite. It has no quick scan mode. https://docs.clamav.net/Introduction.html https://docs.clamav.net/manual/Usage/Scanning.html#disclaimers

## Version covered

The docs are unversioned and track the latest stable release, the 1.5.x feature release (1.5.4, functionality level 234). Hex byte and nibble negation (`~HH`) in body signatures needs ClamAV 1.6, functionality level 240. https://docs.clamav.net/appendix/FunctionalityLevels.html https://docs.clamav.net/manual/Signatures/BodySignatureFormat.html

Stay on the latest patch of a supported feature release: versions 0.105 and older are blocked from downloading updates, and false positive testing of new signatures runs only on the latest stable release. https://docs.clamav.net/faq/faq-eol.html#version-support-matrix https://docs.clamav.net/faq/faq-cvd.html

## Quick start

- Run the official image with persisted databases. Pin the feature release tag, take `_base` only with a volume on `/var/lib/clamav`, give the container 4 GiB of RAM. https://docs.clamav.net/manual/Installing/Docker.html#image-selection-recommendations

```bash
docker volume create clam_db
docker run --detach --name clamav --memory 4g \
  --mount source=clam_db,target=/var/lib/clamav \
  clamav/clamav:1.5_base
```

https://docs.clamav.net/manual/Installing/Docker.html

- A default install from source places only example configs. Generate the files with clamconf or copy the examples, then comment out or remove the `Example` line. https://docs.clamav.net/manual/Usage/Configuration.html#unix

```bash
clamconf -g freshclam.conf > freshclam.conf
clamconf -g clamd.conf > clamd.conf
```

https://docs.clamav.net/manual/Usage/Configuration.html#unix

- Set `User` in clamd.conf for a service: without it, clamd started as root keeps running as root. freshclam always drops to `DatabaseOwner`. https://docs.clamav.net/manual/Installing/Add-clamav-user.html#about-how-the-service-accounts-are-used
- Prefer the Unix socket and set `LocalSocketMode`: its default leaves the socket world accessible. https://docs.clamav.net/manual/Usage/Configuration.html#clamdconf

```ini
LocalSocket /tmp/clamd.socket
LocalSocketMode 660
User clamav
```

https://docs.clamav.net/manual/Usage/Configuration.html#clamdconf

- Send content to clamd with `INSTREAM`: NUL-framed command, 4-byte big-endian length chunks, a zero-length chunk to finish. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#instream

```python
sock.sendall(b"zINSTREAM\0")
for chunk in chunks:
    sock.sendall(struct.pack(">I", len(chunk)) + chunk)
sock.sendall(struct.pack(">I", 0))
# read until EOF: b"stream: OK\0" or b"stream: Eicar-Signature FOUND\0"
```

https://docs.clamav.net/manual/Usage/ClamdProtocol.html#instream-example

- Check a daemon from the shell. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#basic-checks

```bash
printf 'zPING\0' | nc -U /run/clamav/clamd.sock | tr '\0' '\n'
```

- Test a custom signature alone with `clamscan -d test.ldb file`, adding `--debug --verbose` for format errors. https://docs.clamav.net/manual/Signatures.html#testing-rules-with-clamscan

## Top pitfalls

### Daemon and network

- clamd neither encrypts nor authenticates its TCP socket and accepts commands from anyone who can reach it. Never expose port 3310 beyond a trusted network. https://docs.clamav.net/manual/Usage/Scanning.html#clamd https://docs.clamav.net/manual/Installing/Docker.html
- Start `clamd` plainly, never with `&` or a background wrapper: it loads the database, then daemonizes itself. https://docs.clamav.net/manual/Usage/Scanning.html#clamd
- `INSTREAM` beyond `StreamMaxLength` fails with `INSTREAM size limit exceeded`. Size that option for the largest upload you accept. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#instream
- Path commands (`SCAN`, `CONTSCAN`, `MULTISCAN`) are resolved by clamd, so the daemon must see the path and have read access. Across containers or users, use `INSTREAM`, or `FILDES` on a Unix socket. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#socket-types
- The legacy `STREAM` command is gone, use `INSTREAM`. Frame new commands with `z...\0`, since unprefixed commands are legacy. https://docs.clamav.net/manual/Usage/ClamdProtocol.html#legacy-stream

### Memory

- clamd builds the new engine before freeing the old one on each daily reload, using upwards of 2.4 GiB for a short period. Short of RAM, the OS kills clamd and ClamAV fails or hangs once a day. https://docs.clamav.net/manual/Installing/Docker.html#why-is-this-much-ram-required
- `ConcurrentDatabaseReload no` saves that memory but blocks scans during the reload. https://docs.clamav.net/manual/Installing/Docker.html

### Updates

- Download signatures only with freshclam, or cvdupdate for a private mirror. Scripted `wget` or `curl` downloads are explicitly denied and get blocked. https://docs.clamav.net/faq/faq-freshclam.html
- A cron freshclam runs at minute N between 3 and 57, never a multiple of 10. https://docs.clamav.net/manual/Usage/Configuration.html#freshclamconf
- "Your ClamAV installation is OUTDATED" means a newer engine exists, not that the update failed. Upgrade the engine as well as the database. https://docs.clamav.net/faq/faq-freshclam.html

### Scanning

- `clamscan .` does not recurse. Add `--recursive`. https://docs.clamav.net/manual/Usage/Scanning.html#some-basic-scans
- Files over `--max-filesize` are skipped and assumed clean. https://docs.clamav.net/manual/Usage/Scanning.html#clamscan
- clamdscan silently ignores most clamscan options: the engine settings come from clamd.conf. https://docs.clamav.net/manual/Usage/Scanning.html#clamdscan
- `--remove` is "generally a terrible idea" because of false positives. Use `--move` or `--copy`, or a `VirusEvent` script. https://docs.clamav.net/manual/Usage/Scanning.html#disclaimers

### Signatures

- `clamscan -d` loads only the databases given, not the downloaded ones, so bytecode unpackers and Authenticode trust are off. Add the CVDs with more `-d` flags. https://docs.clamav.net/manual/Signatures.html#testing-rules-with-clamscan
- A blank line, a wrong extension, or a trailing semicolon in an `.ldb` breaks the database. https://docs.clamav.net/manual/Signatures.html#testing-rules-with-clamscan
- Databases ending in `u` (`.ndu`, `.ldu`, `.hdu`) load only when PUA detection is on, which is off by default. https://docs.clamav.net/manual/Signatures.html#signature-databases
- Allow a false positive file with an `.sfp` hash entry, or ignore a signature name with `.ign2`. Do not delete flagged files out of hand. https://docs.clamav.net/manual/Signatures/AllowLists.html https://docs.clamav.net/faq/faq-scan-alerts.html

```bash
sigtool --sha256 /path/to/false/positive/file >> /var/lib/clamav/false-positives.sfp
```

https://docs.clamav.net/manual/Signatures/AllowLists.html#file-allow-lists

## Reference files

| File | Read it for |
| ---- | ----------- |
| [references/install-and-docker.md](references/install-and-docker.md) | Install methods and paths, Docker tags, volumes, sockets, env vars, RAM, service users |
| [references/versions-and-builds.md](references/versions-and-builds.md) | Building from source, upgrades, LTS and EOL policy, functionality level (FLEVEL) table |
| [references/configuration-and-updates.md](references/configuration-and-updates.md) | clamd.conf and freshclam.conf setup, freshclam errors, rate limits, private mirrors, SELinux, DCONF |
| [references/scanning.md](references/scanning.md) | clamscan, clamdscan transport modes, on-access with clamonacc, PUA, false positive investigation, troubleshooting |
| [references/clamd-protocol.md](references/clamd-protocol.md) | Socket commands, framing, INSTREAM, FILDES, IDSESSION, replies, limits |
| [references/signatures.md](references/signatures.md) | Database extensions, names, testing, hash signatures, allow and ignore lists, hex body format, `.ndb` |
| [references/logical-signatures.md](references/logical-signatures.md) | `.ldb` expressions, target block, modifiers, PCRE, byte compare, fuzzy image hash, `.ftm` |
| [references/signatures-advanced.md](references/signatures-advanced.md) | YARA rules, phishing `.pdb` `.wdb` `.gdb`, `.cdb`, `.pwdb`, `.crb` Authenticode, bytecode |
| [references/libclamav.md](references/libclamav.md) | C API: init, load, compile, scan, options flags, reload checks, licensing |

## Old patterns

| Old usage | Replacement | Source |
| --------- | ----------- | ------ |
| clamd `STREAM` command | `INSTREAM` | https://docs.clamav.net/manual/Usage/ClamdProtocol.html#legacy-stream |
| Unprefixed clamd commands (`PING`) | `zPING\0` framing | https://docs.clamav.net/manual/Usage/ClamdProtocol.html#command-framing |
| `ScanOnAccess yes` in clamd (0.101 and older) | `clamonacc` client beside clamd | https://docs.clamav.net/manual/Usage/Scanning.html#clamonacc-v0102 |
| `.ign` ignore list | `.ign2` | https://docs.clamav.net/manual/Signatures/AllowLists.html#signature-ignore-lists |
| `.db` body signatures | `.ndb` | https://docs.clamav.net/manual/Signatures/ExtendedSignatures.html |
| `clamscan --scan-html=no` (before 0.99.2) | `--normalize=no` | https://docs.clamav.net/manual/Signatures/YaraRules.html |
| `SafeBrowsing yes` in freshclam.conf | No official database, separate clamav-safebrowsing project with your own Google API key | https://docs.clamav.net/faq/faq-safebrowsing.html |
| Downloading CVDs with wget or curl | freshclam, or cvdupdate for a mirror | https://docs.clamav.net/faq/faq-freshclam.html |
| Autotools build (removed in 0.104) | CMake 3.17+ with a Rust toolchain | https://docs.clamav.net/appendix/FunctionalityLevels.html https://docs.clamav.net/faq/faq-troubleshoot.html https://docs.clamav.net/faq/faq-rust.html |
| `R ...` phishing lines with spaces | `R:` and `X:` colon formats | https://docs.clamav.net/manual/Signatures/PhishSigs.html |
