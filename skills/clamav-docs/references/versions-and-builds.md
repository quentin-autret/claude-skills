# Versions and source builds (ClamAV 1.5)

## Contents

- Building from source
- Version support
- Functionality levels

## Building from source

- CMake 3.17+ and a Rust toolchain are required. Install Rust with rustup: distro packages are likely too old. Never run Cargo yourself, CMake drives it. https://docs.clamav.net/faq/faq-troubleshoot.html https://docs.clamav.net/manual/Installing/Installing-from-source-Unix.html#install-rust-toolchain https://docs.clamav.net/faq/faq-rust.html
- The clamav.net release tarball vendors the Rust crates and builds offline. A Git checkout needs internet. https://docs.clamav.net/faq/faq-rust.html
- libcurl is needed for freshclam, clamsubmit and clamonacc, ncurses for clamdtop. https://docs.clamav.net/manual/Installing/Installing-from-source-Unix.html
- Distro-style layout: https://docs.clamav.net/manual/Installing/Installing-from-source-Unix.html

```bash
cmake .. -D CMAKE_INSTALL_PREFIX=/usr -D CMAKE_INSTALL_LIBDIR=lib \
  -D APP_CONFIG_DIRECTORY=/etc/clamav -D DATABASE_DIRECTORY=/var/lib/clamav \
  -D ENABLE_JSON_SHARED=OFF
cmake --build . && ctest && sudo cmake --build . --target install
```

- `ENABLE_JSON_SHARED=OFF` (json-c 0.15+) avoids crashes in apps that load `libclamav.so` beside another JSON library. https://docs.clamav.net/manual/Installing/Installing-from-source-Unix.html
- 1.5.0+ may fail to build on RHEL 8 or AlmaLinux 8 (old json-c, `json_object_new_uint64` undefined). Build json-c yourself, use the project-provided RPM, or upgrade the OS. https://docs.clamav.net/faq/faq-troubleshoot.html
- Uninstall with `xargs rm < install_manifest.txt` from the build directory. Directories, databases and configs stay. https://docs.clamav.net/faq/faq-uninstall.html
- Upgrading a source install: back up config and databases, uninstall, install, restore. Restoring the database avoids freshclam's `not locked` error. https://docs.clamav.net/faq/faq-upgrade.html
- Still "OUTDATED" after an upgrade: check for a second install (`whereis freshclam`) and stale libraries (`ldd $(which freshclam)`). https://docs.clamav.net/faq/faq-upgrade.html
- Windows builds use Visual Studio 2015+, CMake, Rust, and Mussels (preferred) or vcpkg for dependencies. Run `msl` only in a small subdirectory, it searches recipes recursively. https://docs.clamav.net/manual/Installing/Installing-from-source-Windows.html

## Version support

- Critical systems run the latest stable release, betas only on non-critical ones. https://docs.clamav.net/faq/faq-whichversion.html
- 1.4 is the current LTS. An LTS gets at least 3 years of support plus 1 more year of signature downloads. https://docs.clamav.net/faq/faq-eol.html
- A non-LTS release gets critical patches for at least 4 months after the next feature release is published, or until the one after that is published. https://docs.clamav.net/faq/faq-eol.html
- Older patch versions can be blocked 4 months after a newer patch ships. 0.105 and older are blocked. https://docs.clamav.net/faq/faq-eol.html#version-support-matrix
- The ABI is stable across patches of a feature release: libclamav 12.1.0 and libfreshclam 4.0.0 for 1.5.x. https://docs.clamav.net/faq/faq-eol.html https://docs.clamav.net/appendix/FunctionalityLevels.html
- "Malformed hexstring: This ClamAV version has reached End of Life" means the EOL block applies. https://docs.clamav.net/faq/faq-upgrade.html

## Functionality levels

- Each engine has an integer FLEVEL. Signatures declare the range they need. Choosing the right FLEVEL matters most for features added in the last 3 to 4 major releases. https://docs.clamav.net/appendix/FunctionalityLevels.html

| Version | FLEVEL | Notable additions |
| ------- | ------ | ----------------- |
| 1.6.0 | 240 | Hex byte and nibble negation in body and YARA hex strings |
| 1.5.4 | 234 | |
| 1.5.0 | 230 | `cl_scan*_ex()`, scan callbacks, `.cvd.sign`, `CL_TYPE_AI_MODEL`, encrypted OLE2 detection |
| 1.4.0 | 210 | ALZ and LHA extraction |
| 1.3.0 | 200 | OneNote, `.pyc` type |
| 1.0.0 | 160 | `cl_cvdunpack()`, file inspection callback |
| 0.105.0 | 150 | Fuzzy image hash subsignatures |
| 0.104.0 | 140 | CMake stabilized (autotools removed) |
| 0.101.0 | 100 | API break, RAR5, byte compare |
| 0.100.0 | 90 | Intermediates |
| 0.99.0 | 81 | YARA, PCRE |
| 0.98.0 | 74 | allmatch, `{}` wildcards |
| 0.96 | 51 | Bytecode, `.cdb` |

https://docs.clamav.net/appendix/FunctionalityLevels.html

- "Current functionality level = 1, required = 2": the database needs a newer engine. Updates still download, but the engine cannot use all signatures. https://docs.clamav.net/faq/faq-freshclam.html
