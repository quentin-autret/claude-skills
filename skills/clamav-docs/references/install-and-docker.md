# Install and Docker (ClamAV 1.5)

## Contents

- Choosing an install method
- Paths by install method
- Docker image and tags
- Docker volumes, sockets and ports
- Docker environment variables
- Docker RAM
- Service users and permissions

## Choosing an install method

- Package managers are the quickest way and ease upgrades. Third-party packages may differ from upstream: paths, preconfiguration, extra patches, split components. https://docs.clamav.net/manual/Installing.html https://docs.clamav.net/manual/Installing/Packages.html
- Project installers (deb, rpm, macOS pkg) have their external library dependencies statically compiled in, but do not presently include a preconfigured `freshclam.conf` or `clamd.conf`, nor a database directory. The deb and rpm also ship no `clamav` user accounts. You create the configs, the `clamav` user (if needed) and the database directory permissions yourself. https://docs.clamav.net/manual/Installing.html#installing-with-an-installer
- The project deb and rpm do not include `clamav-milter`. The rpm needs glibc 2.28+, the deb glibc 2.27+, both install to `/usr/local`. https://docs.clamav.net/manual/Installing.html
- The macOS pkg installs to `/usr/local/clamav`, which is not in `PATH`. https://docs.clamav.net/manual/Installing.html

```bash
export PATH=/usr/local/clamav/bin:/usr/local/clamav/sbin:$PATH
```

- Debian: `apt install clamav clamav-daemon`. RAR support is a separate package, `libclamunrar9`, from the `non-free` archive on Debian (no `non-free` needed on Ubuntu). https://docs.clamav.net/manual/Installing/Packages.html#debian
- EPEL (AlmaLinux, RHEL, Fedora): `dnf install -y clamav clamd clamav-update`. https://docs.clamav.net/manual/Installing/Packages.html
- Homebrew and MacPorts: copy `freshclam.conf.sample` to `freshclam.conf`, remove the `Example` line, then run `freshclam`. Same for `clamd.conf`, with Unix socket settings preferred over TCP. https://docs.clamav.net/manual/Installing/Packages.html
- Verify downloads with GPG, the package and its `.sig` in the same directory: `gpg --import cisco-talos.gpg` then `gpg --verify clamav-X.XX.tar.gz.sig`. https://docs.clamav.net/manual/Installing.html#verifying-clamavnet-downloads-with-gpg

## Paths by install method

| Install | Configs | Databases |
| ------- | ------- | --------- |
| From source (default) | `/usr/local/etc/` | `/usr/local/share/clamav/` |
| Linux distro package (probable) | `/etc/clamav` | `/var/lib/clamav` |
| Homebrew | `/usr/local/etc/clamav` | |
| MacPorts | `/opt/local/etc` | |
| Docker image | `/etc/clamav` | `/var/lib/clamav` |

https://docs.clamav.net/manual/Installing/Packages.html https://docs.clamav.net/manual/Installing/Docker.html

## Docker image and tags

- Images: `clamav/clamav` (Alpine) and `clamav/clamav-debian` (Debian). https://docs.clamav.net/manual/Installing.html
- `<version>` ships preloaded databases. `<version>_base` ships none: use it only with a volume on `/var/lib/clamav`, or every new container downloads the full set. https://docs.clamav.net/manual/Installing/Docker.html
- Tags: `MAJOR.MINOR.PATCH[_base]` (a non-base tag only gets newer signatures, a CVE rebuild adds a suffix such as `1.0.9-2_base` and moves the plain tag), `MAJOR.MINOR[_base]` (latest patch), `stable[_base]` and `latest[_base]` (latest stable patch version image, `latest` being the same as `stable`), `unstable[_base]` (nightly `main`). https://docs.clamav.net/manual/Installing/Docker.html
- Default: pin the feature release tag, `MAJOR.MINOR[_base]` (the page's examples use `clamav/clamav:1.4_base`, written before 1.5, and it says `stable` tracks `1.5` and `1.5_base` once 1.5 is released, so `1.5_base` for this version), with a persisted volume. Use `stable` or `latest` only if you accept a new feature release without evaluating it. https://docs.clamav.net/manual/Installing/Docker.html#image-selection-recommendations
- clamd in every image checks the database volume periodically and reloads newer signatures. https://docs.clamav.net/manual/Installing/Docker.html#image-selection-recommendations
- Do not use `--pull always` with the larger images, and do not download the full database set or the database images regularly. https://docs.clamav.net/manual/Installing/Docker.html
- End-of-life images stop being updated and may lose signature downloads. https://docs.clamav.net/manual/Installing/Docker.html

## Docker volumes, sockets and ports

- Databases are per container unless persisted. https://docs.clamav.net/manual/Installing/Docker.html

```bash
docker volume create clam_db
docker run --mount source=clam_db,target=/var/lib/clamav clamav/clamav:1.5_base
```

- A bind mount on `/var/lib/clamav` makes the entrypoint chown the host directory to its `clamav` user. https://docs.clamav.net/manual/Installing/Docker.html
- Run unprivileged with `--user "clamav" --entrypoint /init-unprivileged`. https://docs.clamav.net/manual/Installing/Docker.html
- Override configs by mounting files or the whole directory on `/etc/clamav`. https://docs.clamav.net/manual/Installing/Docker.html
- Host files are invisible to the container. Bind mount them (`--mount type=bind,source=/path/to/scan,target=/scandir`) or send content with `clamdscan --stream`. https://docs.clamav.net/manual/Installing/Docker.html
- The default socket is `/tmp/clamd.sock`. Share it by mounting a host directory on `/tmp/`. Wrong permissions on that directory open clamd to anyone on the host. A custom `LocalSocket` moves the socket. https://docs.clamav.net/manual/Installing/Docker.html
- clamd listens on 3310, clamav-milter on 7357, neither published by default. TCP traffic is unencrypted and unprotected. https://docs.clamav.net/manual/Installing/Docker.html
- The healthcheck sends `ping` to clamd and expects `pong`. https://docs.clamav.net/manual/Installing/Docker.html
- Keep freshclam in the same container as clamd unless the clamd socket is shared, otherwise freshclam cannot notify clamd. For separate clamd and clamdscan containers, share the socket directory. https://docs.clamav.net/manual/Installing/Docker.html
- Containers sharing one database volume run competing freshclam daemons. To reload all at once, disable their freshclam daemons and drive one update. Start the first container and wait for its download before starting others. https://docs.clamav.net/manual/Installing/Docker.html

```bash
docker exec -it clam_container_01 freshclam --on-update-execute=EXIT_1 || if [ $? == 1 ]; then docker exec -it clam_container_01 clamdscan --reload; docker exec -it clam_container_02 clamdscan --reload; fi
```

## Docker environment variables

| Variable | Default | Effect |
| -------- | ------- | ------ |
| `CLAMAV_NO_CLAMD` | `false` | `true` skips clamd |
| `CLAMAV_NO_FRESHCLAMD` | `false` | `true` skips the freshclam daemon |
| `CLAMAV_NO_MILTERD` | `true` | milter off unless set to `false` |
| `CLAMD_STARTUP_TIMEOUT` | `1800` | seconds to wait for clamd |
| `FRESHCLAM_CHECKS` | once a day | freshclam checks per day |

https://docs.clamav.net/manual/Installing/Docker.html#controlling-the-container

- With the freshclam daemon disabled, freshclam still runs once at startup when no database exists. https://docs.clamav.net/manual/Installing/Docker.html

## Docker RAM

- Minimum 3 GiB, preferred 4 GiB. 2 GB may be insufficient. https://docs.clamav.net/manual/Installing/Docker.html
- The engine takes over 1.2 GiB. The daily reload builds a second engine before unloading the first, over 2.4 GiB for a moment. freshclam load-tests new databases too. https://docs.clamav.net/manual/Installing/Docker.html#why-is-this-much-ram-required
- Symptom of too little RAM: clamd killed, unresponsive or crashing once a day. https://docs.clamav.net/faq/faq-troubleshoot.html#clamav-becomes-unresponsive
- Escape hatches: `ConcurrentDatabaseReload no` in clamd.conf (scans block during reload), `TestDatabases no` in freshclam.conf (a broken download may be kept and fail the clamd reload). https://docs.clamav.net/manual/Installing/Docker.html

## Service users and permissions

- Run as root, freshclam always switches to `DatabaseOwner` (default `clamav`). clamd switches only if `User` is set in clamd.conf, else stays root. Always set `User` for a service. https://docs.clamav.net/manual/Installing/Add-clamav-user.html#about-how-the-service-accounts-are-used
- `DatabaseOwner` owns the database directory. clamd, clamdscan and clamscan users need only read access. https://docs.clamav.net/manual/Installing/Add-clamav-user.html

```bash
groupadd clamav
useradd -g clamav -s /bin/false -c "Clam Antivirus" clamav
sudo chown -R clamav:clamav /usr/local/share/clamav
```

https://docs.clamav.net/manual/Installing/Add-clamav-user.html
