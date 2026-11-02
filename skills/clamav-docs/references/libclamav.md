# libclamav C API (ClamAV 1.5)

## Contents

- License
- Lifecycle
- Loading databases
- Scanning
- Scan options
- Keeping the database current
- Engine limits
- Forking and threads
- Building
- Format coverage

## License

- libclamav is GPL v2: closed-source commercial software must not link against it. Software using it must be GPL compliant. https://docs.clamav.net/manual/Development/libclamav.html#license

## Lifecycle

- Order: `cl_init(CL_INIT_DEFAULT)`, `cl_engine_new()`, `cl_load()`, `cl_engine_compile()`, then scan, then `cl_engine_free()`. https://docs.clamav.net/manual/Development/libclamav.html#engine-structure
- `cl_engine_new()` returns NULL when out of memory. https://docs.clamav.net/manual/Development/libclamav.html#initialization
- Call `cl_engine_free()` even after a failed `cl_engine_compile()`. https://docs.clamav.net/manual/Development/libclamav.html#engine-structure
- `cl_strerror()` returns a static string, never free it. https://docs.clamav.net/manual/Development/libclamav.html#error-handling

```c
#include "clamav.h"

struct cl_engine *engine;
unsigned int sigs = 0;
int ret;

if ((ret = cl_init(CL_INIT_DEFAULT)) != CL_SUCCESS) {
    printf("cl_init() error: %s\n", cl_strerror(ret));
    return 1;
}
if (!(engine = cl_engine_new())) {
    printf("Can't create new engine\n");
    return 1;
}
ret = cl_load(cl_retdbdir(), engine, &sigs, CL_DB_STDOPT);
if (ret != CL_SUCCESS) {
    printf("cl_load() error: %s\n", cl_strerror(ret));
    cl_engine_free(engine);
    return 1;
}
```

https://docs.clamav.net/manual/Development/libclamav.html#database-loading

## Loading databases

- `cl_load(path, engine, &signo, options)` loads a file or a whole directory. `cl_retdbdir()` gives the built-in default directory. https://docs.clamav.net/manual/Development/libclamav.html#database-loading
- `signo` is added to, not set: start it at 0. https://docs.clamav.net/manual/Development/libclamav.html#database-loading
- `cl_countsigs()` also adds to its counter, which should start at 0. https://docs.clamav.net/manual/Development/libclamav.html#database-checks
- Default options: `CL_DB_STDOPT`. Extras: `CL_DB_PHISHING`, `CL_DB_PHISHING_URLS` (loads `.wdb` and `.pdb`), `CL_DB_PUA`, `CL_DB_OFFICIAL_ONLY` (signed databases only), `CL_DB_BYTECODE`. https://docs.clamav.net/manual/Development/libclamav.html#database-loading
- `cl_load()` returns `CL_SUCCESS` for a non-database file too. Validate a database with `cl_cvdverify(file)`, which load-tests it and can use a lot of RAM. https://docs.clamav.net/manual/Development/libclamav.html#database-verification

## Scanning

- `cl_scanfile(filename, &virname, &scanned, engine, &options)` and `cl_scandesc(fd, filename, ...)`. https://docs.clamav.net/manual/Development/libclamav.html#data-scan-functions
- Returns `CL_CLEAN`, `CL_VIRUS`, or an error code for `cl_strerror()`. https://docs.clamav.net/manual/Development/libclamav.html#data-scan-functions
- `virname` points into the engine, never free it. `scanned`, if not NULL, grows by the scanned size in `CL_COUNT_PRECISION` units. https://docs.clamav.net/manual/Development/libclamav.html#data-scan-functions

```c
const char *virname;
struct cl_scan_options options;
memset(&options, 0, sizeof(options));
options.parse = ~0;
options.general |= CL_SCAN_GENERAL_HEURISTICS;

if ((ret = cl_scanfile("/tmp/test.exe", &virname, NULL, engine, &options)) == CL_VIRUS) {
    printf("Virus detected: %s\n", virname);
} else if (ret != CL_CLEAN) {
    printf("Error: %s\n", cl_strerror(ret));
}
```

https://docs.clamav.net/manual/Development/libclamav.html#data-scan-functions

## Scan options

- `struct cl_scan_options` has bit fields `general`, `parse`, `alert`, `heuristic_alert`, `mail`, `dev`. `memset()` it to zero first. https://docs.clamav.net/manual/Development/libclamav.html#data-scan-functions
- `parse = ~0` enables every parser (archive, ELF, PDF, SWF, HWP, XML docs, mail, OLE2, HTML, PE). https://docs.clamav.net/manual/Development/libclamav.html#data-scan-functions
- `CL_SCAN_GENERAL_HEURISTICS` is required for any heuristic alert to work. `CL_SCAN_GENERAL_ALLMATCHES` is all-match mode, `CL_SCAN_GENERAL_COLLECT_METADATA` the `--gen-json` metadata. https://docs.clamav.net/manual/Development/libclamav.html#data-scan-functions
- `CL_SCAN_GENERAL_HEURISTIC_PRECEDENCE` stops at the first heuristic hit, recommended to save CPU. Without it, heuristics are reported at the end and real malware wins. https://docs.clamav.net/manual/Development/libclamav.html#data-scan-functions
- Heuristics: `CL_SCAN_HEURISTIC_ENCRYPTED_ARCHIVE`, `_ENCRYPTED_DOC`, `_BROKEN`, `_EXCEEDS_MAX` (alert past size or recursion limits), `_PHISHING_SSL_MISMATCH`, `_PHISHING_CLOAK`, `_MACROS`, `_PARTITION_INTXN`, `_STRUCTURED` (credit cards and SSNs). https://docs.clamav.net/manual/Development/libclamav.html#data-scan-functions
- `CL_SCAN_MAIL_PARTIAL_MESSAGE` scans split messages and leaves files in `$TemporaryDirectory/clamav-partial` to clean up periodically. https://docs.clamav.net/manual/Development/libclamav.html#data-scan-functions
- The struct names a field `heuristic_alert` while the flag list heading says `heuristic`. No flags are listed for `alert`. https://docs.clamav.net/manual/Development/libclamav.html#data-scan-functions

## Keeping the database current

- Watch the directory with `cl_statinidir()`. `cl_statchkdir()` returns 1 on change: reload, then reset the watcher. https://docs.clamav.net/manual/Development/libclamav.html#database-checks

```c
struct cl_stat dbstat;
memset(&dbstat, 0, sizeof(struct cl_stat));
cl_statinidir(dbdir, &dbstat);

if (cl_statchkdir(&dbstat) == 1) {
    reload_database...;
    cl_statfree(&dbstat);
    cl_statinidir(cl_retdbdir(), &dbstat);
}
```

## Engine limits

- A new engine has the recommended defaults. Change them with `cl_engine_set_num()` and `cl_engine_set_str()` only when needed, following clamscan and clamd sources. https://docs.clamav.net/manual/Development/libclamav.html#limits

## Forking and threads

- The library is thread-safe. https://docs.clamav.net/manual/Development/libclamav.html#libclamav
- A forking daemon calls `srand()` in each child before any libclamav call, or temp file names collide. Threaded daemons need nothing. https://docs.clamav.net/manual/Development/libclamav.html#forking-daemons

## Building

- `clamav-config --cflags` and `--libs` print the flags. Link with `-lclamav`. Build the bundled examples with `-D ENABLE_EXAMPLES=ON`. https://docs.clamav.net/manual/Development/libclamav.html#example
- Building ClamAV with `-D ENABLE_JSON_SHARED=OFF` is preferred. It needs json-c 0.15 or newer unless you build json-c yourself with custom options. With the shared json-c library, downstream apps using `libclamav.so` may crash if they use a different JSON library. https://docs.clamav.net/manual/Installing/Installing-from-source-Unix.html

## Format coverage

- Office, RTF and PDF: only embedded objects are extracted, the text is not decoded. Text normalization covers HTML only. https://docs.clamav.net/manual/Development/libclamav.html#documents
- UPX for ELF and Mach-O works through bytecode signatures, PE UPX is built in. https://docs.clamav.net/manual/Development/libclamav.html#executables
