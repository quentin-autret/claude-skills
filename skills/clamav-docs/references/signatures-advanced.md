# Advanced signatures: YARA, phishing, containers, certificates (ClamAV 1.5)

## Contents

- YARA rules
- Phishing signatures
- Container metadata (.cdb)
- Encrypted archive passwords (.pwdb)
- Trusted and revoked certificates (.crb)
- Bytecode signatures

## YARA rules

- Files ending `.yar` or `.yara` in the database directory are parsed as YARA. https://docs.clamav.net/manual/Signatures/YaraRules.html
- Not supported: modules (`import`), `global` rules, external variables (`contains`, `matches`), rules compiled with `yarac`. https://docs.clamav.net/manual/Signatures/YaraRules.html
- Limits: at most 64 strings per rule, at least one literal, hex or regex string, and every part between wildcards at least 2 bytes. https://docs.clamav.net/manual/Signatures/YaraRules.html
- Conditions run only when a string matches. A condition without strings, such as `filesize` alone, needs an always-true string clause: https://docs.clamav.net/manual/Signatures/YaraRules.html

```
rule CheckFileSize
{
  strings:
    $abc = "abc"
  condition:
    ($abc or not $abc) and filesize < 200KB
}
```

- Rules also run on extracted and decompressed inner files. https://docs.clamav.net/manual/Signatures/YaraRules.html
- HTML, JavaScript and ASCII text are normalized first, and rules see the normalized text. Inspect it with `clamscan --leave-temps --tempdir=mytempdir`, or turn normalization off with `--normalize=no` (clamd.conf `Normalize`). https://docs.clamav.net/manual/Signatures/YaraRules.html
- Hex string negation `{ 41 ~00 42 }` needs ClamAV 1.6, FLEVEL 240. It differs from the bitwise `~` in conditions. `~~00`, `~??`, `~( AA | BB )` and `~[1-2]` are parse errors. https://docs.clamav.net/manual/Signatures/YaraRules.html

## Phishing signatures

- ClamAV checks only listed targets: `.pdb` names the domains to watch, `.wdb` allows known-good pairs. `--phish-scan-alldomains` checks everything and produces false positives. https://docs.clamav.net/manual/Signatures/PhishSigs.html#how-to-create-and-maintain-the-domain-list-dailypdb
- Before a heuristic alert, clamscan and clamd print "Display URL" and "Real URL". `clamscan --debug undetected.eml` shows which URLs are checked. https://docs.clamav.net/manual/Signatures/PhishSigs.html
- Every line needs its colons, no trailing spaces. A single malformed line aborts loading with a Malformed Database Error. https://docs.clamav.net/manual/Signatures/PhishSigs.html#hints

### .pdb

- `R:DisplayedURL[:FuncLevelSpec]` (regex) or `H:DisplayedHostname[:FuncLevelSpec]` (literal, matches the host or a subdomain). https://docs.clamav.net/manual/Signatures/PhishSigs.html
- The last 3 characters of an `R` regex must be literal. FuncLevelSpec is `min` or `min-max`. https://docs.clamav.net/manual/Signatures/PhishSigs.html

```
H:amazon.co.uk:20-
R:.+\.amazon\.(com|co\.uk)([/?].*)?
```

### .wdb

- `X:RealURL:DisplayedURL[:FuncLevelSpec]` matches the regex, anchored, on `RealURL:DisplayedURL` as one string, with a trailing `/` added. `M:RealHostname:DisplayedHostname` matches hosts and subdomains. `Y:RealURL` (any displayed URL) needs ClamAV 1.6. https://docs.clamav.net/manual/Signatures/PhishSigs.html#wdb-format
- End regexes with `([/?].*)?` so `evilurl.example.com/amazon.co.uk/` does not pass. Allow only pairs owned by the same entity. https://docs.clamav.net/manual/Signatures/PhishSigs.html#wdb-format

```
X:.+\.amazon\.(at|ca|co\.uk|co\.jp|de|fr)([/?].*)?:.+\.amazon\.com([/?].*)?:17-
```

### .gdb

- `S:` and `S2:` lines carry Safe Browsing hashes and are reserved to that use. `S1:` blocks phishing sites as `Phishing.URL.Blocked`. A `:F:` full hash needs its `:P:` prefix entry. https://docs.clamav.net/manual/Signatures/PhishSigs.html#gdb-format
- Allow a hash locally with `S:W:<HASH>` in `local.gdb`. https://docs.clamav.net/manual/Signatures/PhishSigs.html#gdb-format

### Regex and flags

- POSIX regex, implicitly `^...$`. Keep unsupported tokens (`+`, `*`, `{}`) out of the first characters to cut load time. https://docs.clamav.net/manual/Signatures/PhishSigs.html#regular-expressions
- Default flags: `CLEANUP_URL | CHECK_SSL | CHECK_CLOAKING | CHECK_IMG_URL`. The SSL check applies to `a` tags only. https://docs.clamav.net/manual/Signatures/PhishSigs.html#flags
- URLs come from `a` (href real, text displayed), `form` action, `img` and `area` src, `iframe` src. https://docs.clamav.net/manual/Signatures/PhishSigs.html

## Container metadata (.cdb)

```
VirusName:ContainerType:ContainerSize:FileNameREGEX:FileSizeInContainer:FileSizeReal:IsEncrypted:FilePos:Res1:Res2[:MinFL[:MaxFL]]
```

- `ContainerType` is `CL_TYPE_ZIP`, `CL_TYPE_RAR`, `CL_TYPE_7Z`, `CL_TYPE_MAIL` and others, or `*`. Sizes are bytes or `x-y`. `IsEncrypted` is 1, 0 or `*`. `FilePos` counts from 1. `Res1` is a CRC for ZIP and RAR only, `Res2` unused. https://docs.clamav.net/manual/Signatures/ContainerMetadata.html

## Encrypted archive passwords (.pwdb)

- Experimental, traditional PKWARE ZIP encryption only. Passwords are tried in file order, then normal parsing resumes. https://docs.clamav.net/manual/Signatures/EncryptedArchives.html
- `SignatureName;Engine:X-Y,Container:CL_TYPE_*;PWStorageType;Password`, storage 0 cleartext, 1 hex. https://docs.clamav.net/manual/Signatures/EncryptedArchives.html

## Trusted and revoked certificates (.crb)

```
Name;Trusted;Subject;Serial;Pubkey;Exponent;CodeSign;TimeSign;CertSign;NotBefore;Comment[;minFL[;maxFL]]
```

- `Trusted` 1 trusts, 0 revokes. `Subject` is the SHA1 of the subject in hex, `Serial` as `clamscan --debug --verbose` prints it. `Exponent` is ignored. https://docs.clamav.net/manual/Signatures/AuthenticodeRules.html
- `.crb` rules can block malicious executables and match leaf certificates. https://docs.clamav.net/appendix/Authenticode.html
- `sigtool --print-certs` shows embedded signatures without a prior detection. A `.cat` file loads with `-d` and needs a trusted cert in its chain. https://docs.clamav.net/appendix/Authenticode.html
- Authenticode trust does not work under `clamscan -d` alone, because `-d` stops ClamAV loading the freshclam databases and the check relies on `.crb` rules. Load the CVD files with extra `-d` flags if needed. https://docs.clamav.net/manual/Signatures.html#testing-rules-with-clamscan

## Bytecode signatures

- C compiled to bytecode in `.cbc` files, shipped in `bytecode.cvd` (or `.cld`), and ClamAV interprets the bytecode once the database is loaded. One function may report several names. Writing them is covered by the ClamAV Bytecode Compiler project, not the docs. Test with `clambc`. https://docs.clamav.net/manual/Signatures/BytecodeSignatures.html https://docs.clamav.net/manual/Usage/SignatureManagement.html#clambc
- UPX unpacking for ELF and Mach-O runs as bytecode, so keep the official bytecode database loaded. https://docs.clamav.net/manual/Development/libclamav.html#executables
