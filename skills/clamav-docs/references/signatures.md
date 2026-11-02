# Signatures: files, testing, hashes, allow lists, body format (ClamAV 1.5)

## Contents

- Where databases go
- Extensions
- Signature names
- Testing a signature
- Normalization and unpacking
- sigtool helpers
- Hash signatures
- Allow lists and ignore lists
- Hex body format
- Extended signatures (.ndb)

## Where databases go

- Put custom files in the database directory (`/var/lib/clamav` or `/usr/local/share/clamav`), next to the `.cvd` files. They load after the official CVDs and need no signing. For a quick test, pass a file with `clamscan -d`. https://docs.clamav.net/manual/Signatures.html#database-formats https://docs.clamav.net/faq/faq-cvd.html
- Unpack a CVD or CLD with `sigtool -u <file>`, read its header with `sigtool --info`. https://docs.clamav.net/manual/Signatures.html#inspecting-signatures-inside-a-cvd-file

## Extensions

| Extension | Content |
| --------- | ------- |
| `.hdb` `.hsb` | File hash (MD5, SHA1/SHA256) |
| `.mdb` `.msb` | PE section hash |
| `.imp` | PE import table hash |
| `.ndb` | Extended body signatures |
| `.ldb` `.idb` | Logical signatures, icons |
| `.ftm` | File type magic |
| `.cdb` | Container metadata |
| `.cbc` | Bytecode |
| `.pdb` `.gdb` `.wdb` | Phishing URLs |
| `.yar` `.yara` | YARA rules |
| `.fp` `.sfp` | File allow lists |
| `.ign2` | Signature ignore list |
| `.cat` `.crb` | Trusted and revoked PE certificates |
| `.pwdb` | Encrypted archive passwords |
| `.cfg` | Dynamic config |

https://docs.clamav.net/manual/Signatures.html#settings-databases https://docs.clamav.net/manual/Signatures/HashSignatures.html

- A `u` at the end (`.ndu`, `.ldu`, `.hdu`, `.hsu`, `.mdu`, `.msu`) marks a PUA database, loaded only when PUA detection is on (off by default). https://docs.clamav.net/manual/Signatures.html#signature-databases

## Signature names

- Use only letters, digits, `-`, `.` and `_`. Never spaces, apostrophes, colons, semicolons or quotes. https://docs.clamav.net/manual/Signatures.html#signature-names
- Official format: `{platform}.{category}.{name}-{signature id}-{revision}`, for example `Win.Trojan.Zusy-9935890-0`. https://docs.clamav.net/manual/Signatures/SignatureNames.html
- No brand names or living people, reuse the family name other vendors use, `Agent` as the fallback. https://docs.clamav.net/manual/Signatures/SignatureNames.html

## Testing a signature

- Put the signature on its own line in a file with the matching extension, then scan with it. https://docs.clamav.net/manual/Signatures.html#testing-rules-with-clamscan

```bash
clamscan -d test.ldb --debug --verbose sample.bin
```

- Usual format errors: wrong extension, a blank line in the file, a trailing semicolon in an `.ldb`. https://docs.clamav.net/manual/Signatures.html#testing-rules-with-clamscan
- `-d` loads only the files given. Bytecode unpackers and Authenticode trust (`.crb`) are off unless the CVDs are added with more `-d` flags. https://docs.clamav.net/manual/Signatures.html#testing-rules-with-clamscan
- No match may come from default limits. Raise them while testing, `--max-filesize=2000M --max-scansize=2000M --max-recursion=2000000` and the other `--max-*` and `--pcre-*` limits. https://docs.clamav.net/manual/Signatures.html#testing-rules-with-clamscan
- `sigtool --test-sigs` runs a signature on a sample and shows the match offset. https://docs.clamav.net/manual/Signatures.html#using-sigtool

## Normalization and unpacking

- `clamscan --debug --leave-temps` shows how a file is unpacked and keeps the temp files. Signatures on the innermost unpacked file are more generic and survive other packers. https://docs.clamav.net/manual/Signatures.html#debug-information-from-libclamav
- Packed PE (UPX, FSG, Petite and others) is unpacked before matching. https://docs.clamav.net/manual/Signatures.html#compressed-portable-executable-files
- HTML is lower-cased, stripped of comments and extra whitespace, with character references and JScript.encode decoded. Match the output of `sigtool --html-normalise` (target type 3). https://docs.clamav.net/manual/Signatures.html#html
- ASCII text is lower-cased with whitespace and control characters removed. Match `sigtool --ascii-normalise` output (target type 7). https://docs.clamav.net/manual/Signatures.html#text-files
- Target types: 0 any, 1 PE, 2 OLE2, 3 HTML, 4 mail, 5 graphics, 6 ELF, 7 ASCII, 9 Mach-O, 10 PDF, 11 Flash, 12 Java. Type 0 runs on every file, avoid it when a type fits. https://docs.clamav.net/appendix/FileTypes.html

## sigtool helpers

- `--md5`, `--sha1`, `--sha256`: ready `.hdb` or `.hsb` line. `--mdb`: section hashes. https://docs.clamav.net/manual/Signatures.html#using-sigtool
- `--imp`: import table hash, shown in the debug log output. https://docs.clamav.net/manual/Signatures/HashSignatures.html#pe-import-table-hash-signatures-func-level-90
- `--hex-dump` (`echo -n "Match on this" | sigtool --hex-dump`), `--decode` for an existing signature, `--vba` for macros, `--print-certs` for Authenticode. https://docs.clamav.net/manual/Signatures.html#using-sigtool

## Hash signatures

- Only for static files: a single changed byte breaks the match. Never hash text or HTML, which are normalized before matching. https://docs.clamav.net/manual/Signatures/HashSignatures.html
- Format `HashString:FileSize:MalwareName`. Put SHA1 and SHA256 in `.hsb`. https://docs.clamav.net/manual/Signatures/HashSignatures.html#sha1-and-sha256-hash-based-signatures

```bash
sigtool --sha256 bad.exe >> local.hsb
```

https://docs.clamav.net/manual/Signatures/HashSignatures.html

- PE section hashes reverse the order: `PESectionSize:PESectionHash:MalwareName` in `.mdb` or `.msb`. https://docs.clamav.net/manual/Signatures/HashSignatures.html#pe-section-based-hash-signatures
- As of 0.105 (the version the page names), section and import hashes never alert on 64-bit PE (PE32+). https://docs.clamav.net/manual/Signatures/HashSignatures.html#pe-section-based-hash-signatures
- Import hash `.imp`: `PEImportTableHash:PEImportTableSize:MalwareName`, MD5, from `sigtool --imp`. A `*` size was broken in every release up through 0.105 (it only alerts in all-match mode). https://docs.clamav.net/manual/Signatures/HashSignatures.html#pe-import-table-hash-signatures-func-level-90
- Unknown size `*` needs FLEVEL 73: `HashString:*:MalwareName:73`. It is much slower than a fixed size. https://docs.clamav.net/manual/Signatures/HashSignatures.html#hash-signatures-with-unknown-size

## Allow lists and ignore lists

- Allow a file: MD5 lines in `.fp`, SHA1 or SHA256 lines in `.sfp`. https://docs.clamav.net/manual/Signatures/AllowLists.html#file-allow-lists

```bash
sigtool --sha256 /path/to/false/positive/file >> /var/lib/clamav/false-positives.sfp
```

- Ignore a signature everywhere: its name, one per line, in an `.ign2` file in the database directory. Append the MD5 of the database entry to stop ignoring once the signature is fixed upstream. https://docs.clamav.net/manual/Signatures/AllowLists.html#signature-ignore-lists

```
Eicar-Test-Signature:bc356bae4c42f19a3de16e333ba3569c
```

- `.ign` is the historical ignore format, still read but replaced by `.ign2`. https://docs.clamav.net/manual/Signatures/AllowLists.html#signature-ignore-lists
- Phishing heuristic false positives go in `.wdb`, see the advanced signatures reference. https://docs.clamav.net/manual/Signatures/AllowLists.html#phishing-heuristic-allow-lists

## Hex body format

- Wildcards: `??` any byte, `a?` and `?a` nibbles, `*` any length, `{n}`, `{-n}`, `{n-}`, `{n-m}`. https://docs.clamav.net/manual/Signatures/BodySignatureFormat.html#wildcards
- `*` and `{}` split the signature into parts, each with at least 2 static bytes. `{n}` under 128 becomes `??` repeats and does not split. https://docs.clamav.net/manual/Signatures/BodySignatureFormat.html#wildcards
- `[x-y]` (y at most 32) binds a single byte to a sequence of at least 2 bytes: `64[4-4]61616161`. https://docs.clamav.net/manual/Signatures/BodySignatureFormat.html#wildcards
- Classes: `(B)` word boundary, `(L)` line end, `(W)` non-alphanumeric. https://docs.clamav.net/manual/Signatures/BodySignatureFormat.html#character-classes
- Alternates: `(aa|bb)` and `!(aa|bb)` single byte, `(aaaa|bbbb)` fixed length with negation, generic `(alt1|alt2)` of any length without negation. https://docs.clamav.net/manual/Signatures/BodySignatureFormat.html#alternate-strings
- Hex negation `~HH`, `~H?`, `~?H` needs ClamAV 1.6, and the database must declare FLEVEL 240: https://docs.clamav.net/manual/Signatures/BodySignatureFormat.html#minimum-functionality-level-requirement

```
MySignature;Engine:240-255,Target:0;0;41~0042
MySignature:0:*:41~0042:240
```

## Extended signatures (.ndb)

- Format `MalwareName:TargetType:Offset:HexSignature[:min_flevel[:max_flevel]]`. `max_flevel` needs `min_flevel`. https://docs.clamav.net/manual/Signatures/ExtendedSignatures.html
- Offsets: `*`, `n`, `EOF-n`. For PE, ELF and Mach-O also `EP+n`, `EP-n`, `Sx+n`, `SEx`, `SL+n`. `n,MaxShift` floats (`10,5` covers 10 to 15). https://docs.clamav.net/manual/Signatures/ExtendedSignatures.html
