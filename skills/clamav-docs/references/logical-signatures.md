# Logical signatures and file type magic (ClamAV 1.5)

## Contents

- Format
- Logical expression
- Target description block
- Subsignature modifiers
- PCRE subsignatures
- Byte compare subsignatures
- Macro subsignatures
- Image fuzzy hash
- PE version information and icons
- File type magic (.ftm)

## Format

- `SignatureName;TargetDescriptionBlock;LogicalExpression;Subsig0;Subsig1;...`, up to 64 subsignatures, in an `.ldb` file. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html
- No semicolon at the end of the file (a common load error), and no blank lines in the file. A literal `;` inside a PCRE is written `\x3B`. https://docs.clamav.net/manual/Signatures.html#testing-rules-with-clamscan https://docs.clamav.net/manual/Signatures/LogicalSignatures.html#pcre-subsignatures

## Logical expression

- Subsignatures are numbered from 0. Operators: `&`, `|`, `A=X`, `A>X`, `A<X`, and on a block `A=X,Y` where X counts matches and Y is the minimum number of distinct subsignatures. `A=0` negates. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html

```
Sig3;Target:0;((0|1|2|3)=2)&(4|1);6b6f74656b;616c61;7a6f6c77;73746566616e;deadbeef
```

https://docs.clamav.net/manual/Signatures/LogicalSignatures.html

## Target description block

- Comma-separated `Key:Value`. `Engine:X-Y` must come first when present. Others: `Target`, `FileSize:X-Y`, `EntryPoint`, `NumberOfSections`, `Container:CL_TYPE_*`, `Intermediates:CL_TYPE_*>CL_TYPE_*` (up to 16 layers, `CL_TYPE_ANY` allowed), `IconGroup1`, `IconGroup2`, `HandlerType:CL_TYPE_*`. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html
- Declare `Engine` for any feature newer than the oldest engine you target: modifiers and PCRE need `Engine:81-255`, macros `51-255`, fuzzy image hash `150-255`. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html https://docs.clamav.net/appendix/FunctionalityLevels.html
- `Container:CL_TYPE_ANY` matches root objects only: the same file inside an archive no longer alerts. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html
- `HandlerType` does not alert. It rescans the file as another type, so the file is scanned twice. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html

```
Filetype.PDF;Engine:54-255,Target:0,HandlerType:CL_TYPE_PDF;(0|1)&2&3;0:255044462d??2e;0:257064662d??2e;737461727478726566;2525454f46
```

https://docs.clamav.net/manual/Signatures/LogicalSignatures.html

## Subsignature modifiers

- Suffix `::` then flags: `i` case-insensitive (patterns are case-sensitive by default), `w` wide (NULs interleaved, not true UTF-16), `f` full word, `a` ASCII (with `w` for both). Requires `Engine:81-255`. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html#subsignature-modifiers

```
clamav-wide-C0;Engine:81-255,Target:0;0&1;414141;68656c6c6f::iwfa
```

## PCRE subsignatures

- `Trigger/PCRE/[Flags]`, `Engine:81-255`. The trigger is a logical expression over earlier subsignatures, never itself. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html#pcre-subsignatures
- No empty regex, no `(?UTF*)` sequences. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html#pcre-subsignatures
- PCREs are auto-anchored to the first offset by default. `r` makes them rolling, `g` finds all matches (slower), `e` encompasses with a maxshift (faster), plus `i s m x A E U`. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html#pcre-subsignatures

```
Find.All.ClamAV;Engine:81-255,Target:0;1;6265676c6164697427736e6f7462797465636f6465;0/clamav/g
```

https://docs.clamav.net/manual/Signatures/LogicalSignatures.html#pcre-subsignatures

## Byte compare subsignatures

- `subsigid_trigger(offset#byte_options#comparisons)`, evaluated after all other subsignatures except PCRE. The trigger is one plain subsignature. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html#byte-compare-subsignatures
- Offset `>>n` or `<<n` from the trigger match. Options `[h|d|a|i][l|b][e]num_bytes`: hex, decimal, auto or raw binary, little or big endian, exact. Decimal is big-endian, `d` with `l` is a malformed database. Raw binary takes 1, 2, 4 or 8 bytes only. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html#byte-compare-subsignatures
- One or two comparisons with `<`, `>`, `=`, separated by a comma. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html#byte-compare-subsignatures

## Macro subsignatures

- `${min-max}MACROID$` with `Engine:51-255`, groups defined in `.ndb` as `D1:0:$12:626262`, 32 groups at most. The offset is relative to the preceding subsignature, so a macro cannot come first. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html#macro-subsignatures

```
TestMacro;Engine:51-255,Target:0;0&1;616161;${6-7}12$
```

## Image fuzzy hash

- `fuzzy_img#<hash>#0` with `Engine:150-255`. A Hamming distance above 0 is not supported. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html#image-fuzzy-hash-subsignatures
- The hash is close to but not equal to Python imagehash `phash()`. Generate it with ClamAV: `clamscan --gen-json --debug /path/to/file`, field `ImageFuzzyHash`. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html#image-fuzzy-hash-subsignatures

```
logo.png;Engine:150-255,Target:0;0;fuzzy_img#af2ad01ed42993c7#0
```

- Combine with `Container:CL_TYPE_HTML` or `CL_TYPE_MAIL` to target images in pages or mail. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html#image-fuzzy-hash-subsignatures

## PE version information and icons

- `VI` offset anchors on PE version info pairs, written as UTF-16 hex of key and value, only `??` and `(aa|bb)` wildcards. `clamscan --debug` prints each pair's `VI:` hex. Decode with `echo hex | xxd -r -p | strings -el`. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html#signatures-for-version-information-vi-metadata-in-pe-files
- Icon signatures in `.idb`, `ICONNAME:GROUP1:GROUP2:ICON_HASH`, fire only through `IconGroup1` or `IconGroup2` in a logical signature. The hash comes from the `ICO SIGNATURE:` debug line. https://docs.clamav.net/manual/Signatures/LogicalSignatures.html#icon-signatures-for-pe-files

## File type magic (.ftm)

- `magictype:offset:magicbytes:name:rtype:type[:min_flevel[:max_flevel]]`. magictype 0 is a raw comparison, 1 the body hex format, 4 a partition type. `*` offset only with 1. `rtype` is usually `CL_TYPE_ANY`. https://docs.clamav.net/manual/Signatures/FileTypeMagic.html
- A `.ftm` file can override the built-in magic. Magictype 0 is likely insufficient to identify a file, so type 1 may be necessary. If type 1 is also insufficient, `HandlerType` in an `.ldb` is an option, but it scans the file twice. https://docs.clamav.net/manual/Signatures/FileTypeMagic.html
