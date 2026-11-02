# Configuration and signature updates (ClamAV 1.5)

## Contents

- Creating the config files
- clamd.conf essentials
- freshclam.conf essentials
- Running freshclam
- freshclam errors
- Private mirrors
- Custom signatures in the database directory
- SELinux
- Windows specifics
- DCONF

## Creating the config files

- A from-source install needs `freshclam.conf` before freshclam, `clamd.conf` before clamd, `clamav-milter.conf` before the milter. Generate them with clamconf, or copy the `.example` files without the extension. https://docs.clamav.net/manual/Usage/Configuration.html#first-time-set-up

```bash
clamconf -g freshclam.conf > freshclam.conf
clamconf -g clamd.conf > clamd.conf
```

https://docs.clamav.net/manual/Usage/Configuration.html#unix

- clamd treats a config that still has the `Example` line as invalid and refuses to run. Comment it out or remove it. https://docs.clamav.net/manual/Usage/Configuration.html#clamdconf
- Sample options are commented out, enable one by deleting its `#`. https://docs.clamav.net/manual/Usage/Configuration.html#other-freshclamconf-settings
- `clamconf` prints the effective config, its validity, the engine, database and build details. https://docs.clamav.net/manual/Usage/Configuration.html#clamconf

## clamd.conf essentials

- `LocalSocket` has no default, you set it. `LocalSocketMode` defaults to disabled, which leaves the socket world accessible. https://docs.clamav.net/manual/Usage/Configuration.html#clamdconf

```ini
LocalSocket /tmp/clamd.socket
LocalSocketMode 660
```

- Set `User`: otherwise clamd started as root keeps running as root. https://docs.clamav.net/manual/Installing/Add-clamav-user.html#about-how-the-service-accounts-are-used
- Options worth reviewing: `LogTime`, `LogClean`, `LogRotate`, `User`, and the on-access settings in the second half of the sample. https://docs.clamav.net/manual/Usage/Configuration.html#other-clamdconf-settings
- Protocol switches: `EnableReloadCommand`, `EnableSelfCheckCommand`, `EnableShutdownCommand` gate their commands. `EnableVersionCommand`, `EnableStatsCommand`, `AllowAllMatchScan` can disable theirs. `StreamMaxLength` caps `INSTREAM`. https://docs.clamav.net/manual/Usage/ClamdProtocol.html
- `ConcurrentDatabaseReload no` minimizes clamd RAM usage, at the cost that clamd blocks any new scans until the reload is complete. https://docs.clamav.net/manual/Installing/Docker.html
- `ArchiveMaxCompressionRatio` triggers `Oversized.zip` on archive bombs, raise it if legitimate files trip it. https://docs.clamav.net/faq/faq-misc.html
- `DatabaseDirectory` must be the same in clamd.conf and freshclam.conf. https://docs.clamav.net/faq/faq-cvd.html
- clamav-milter needs a running clamd, its own config and logs, and a build with `-D ENABLE_MILTER=ON` plus libmilter. https://docs.clamav.net/manual/Usage/Configuration.html#clamav-milterconf

## freshclam.conf essentials

- `DatabaseOwner` must own the database directory. https://docs.clamav.net/manual/Usage/Configuration.html#users-and-on-user-privileges
- Options worth enabling: `LogTime`, `LogRotate`, `NotifyClamd`, `DatabaseOwner`. https://docs.clamav.net/manual/Usage/Configuration.html#other-freshclamconf-settings
- Proxies are set only in the config file. With `HTTPProxyPassword`, freshclam requires strict permissions on the file. https://docs.clamav.net/manual/Usage/Configuration.html#freshclamconf

```ini
HTTPProxyServer myproxyserver.com
HTTPProxyPort 1234
HTTPProxyUsername myusername
HTTPProxyPassword mypass
```

- Create the log file owned by the freshclam user, then point `UpdateLogFile` at it. https://docs.clamav.net/manual/Usage/Configuration.html#freshclamconf

```bash
touch /var/log/freshclam.log
chmod 600 /var/log/freshclam.log
chown clamav /var/log/freshclam.log
```

- `TestDatabases no` saves memory, at the risk of keeping a broken download that clamd then fails to load. https://docs.clamav.net/manual/Installing/Docker.html

## Running freshclam

- Install the CVDs before starting clamd or clamscan. https://docs.clamav.net/manual/Usage/SignatureManagement.html#freshclam
- freshclam downloads everything into an empty directory, otherwise patches existing databases with CDIFFs. A corrupted database is replaced by a fresh download. https://docs.clamav.net/manual/Usage/SignatureManagement.html#freshclam
- Daemon mode: `freshclam -d`. Cron alternative, once an hour, with N between 3 and 57 and never a multiple of 10: https://docs.clamav.net/manual/Usage/Configuration.html#freshclamconf

```
N * * * *   /usr/local/bin/freshclam --quiet
```

- Databases update once or twice a day. You can check as often as 4 times an hour with `DNSDatabaseInfo current.cvd.clamav.net` and `DatabaseMirror database.clamav.net` set. https://docs.clamav.net/faq/faq-cvd.html https://docs.clamav.net/faq/faq-troubleshoot.html
- Only freshclam and cvdupdate are supported download paths. Other download methods can lead to 403, 503, 1020 or 429 errors (blocked or rate limited), so move to freshclam or cvdupdate. https://docs.clamav.net/faq/faq-freshclam.html
- `freshclam -v` shows HTTP request and response details. https://docs.clamav.net/faq/faq-freshclam.html

## freshclam errors

| Message | Cause and fix |
| ------- | ------------- |
| `Failed to get information about user "clamav"` | Root run without a `clamav` user. Create it or set `DatabaseOwner`. |
| `Can't create freshclam.dat in ...` | Database directory not owned by the user freshclam runs as. Change its ownership to that user. |
| `Download failed (77) ... SSL CA cert` | Install `ca-certificates` or set `CURL_CA_BUNDLE` to the bundle path. |
| `Invalid DNS reply. Falling back to HTTP mode` | DNS cannot resolve the TXT record. Test `host -t txt current.cvd.clamav.net`. |
| `DNS record is older than 3 hours` | Falls back to HTTPS. Check the clock if constant. |
| `Connection with ??? failed` | DNS broken or port 53/tcp blocked (answers over 512 bytes). |
| `Updating too frequently with an outdated version` | FLEVEL gap over 3: limited to 6 checks a day. Upgrade. |
| `Ignoring mirror <IP> (has connected too many times ...)` | Stop the daemon, delete `mirrors.dat` and `daily.cvd`, restart. |
| `Your ClamAV installation is OUTDATED` | A newer engine exists. Upgrade it, the database update did not fail. |
| HTTP 403, 503, 1020 | Blocked IP, EOL version or unsupported download method. |
| HTTP 429 | Rate limited, common on shared cloud IPs. Slow down or change IP. |
| `libclamav.so.7: cannot open shared object file` | Run `sudo ldconfig`. |

https://docs.clamav.net/faq/faq-freshclam.html https://docs.clamav.net/faq/faq-troubleshoot.html https://docs.clamav.net/manual/Usage/SignatureManagement.html#freshclam

## Private mirrors

- A caching HTTP proxy (squid) plus `HTTPProxyServer` is easy to implement and bandwidth efficient. https://docs.clamav.net/appendix/CvdPrivateMirror.html#use-an-http-proxy
- cvdupdate (`pip3 install cvdupdate`, Python 3.6+) serves CVDs and CDIFFs. Run `cvd update` at least daily. `cvd serve` is for tests only. Do not use it unless you host a mirror: it downloads more than freshclam. https://docs.clamav.net/appendix/CvdPrivateMirror.html
- The mirror web server must support HTTP Range requests, because if clients cannot use DNS for the version check, it may serve the entire CVD on every check. Python's simple HTTP server does not. https://docs.clamav.net/appendix/CvdPrivateMirror.html
- Clients of a cvdupdate mirror: https://docs.clamav.net/appendix/CvdPrivateMirror.html

```ini
DatabaseMirror http://mirror.mylan:8000
```

- Clients still query clamav.net DNS first. Offline clients set `DNSDatabaseInfo no` and fall back to a Range request. https://docs.clamav.net/appendix/CvdPrivateMirror.html
- A freshclam-fed mirror serves whole CVD or CLD files only. Clients use `PrivateMirror`, which accepts CLDs and checks with a Range request instead of DNS. https://docs.clamav.net/appendix/CvdPrivateMirror.html#use-freshclam-to-serve-only-whole-database-files-from-a-private-mirror

```ini
# Mirror side
DatabaseMirror database.clamav.net
DatabaseDirectory /your/server/www
CompressLocalDatabase yes

# Client side
PrivateMirror http://mirror.mylan:8000
ScriptedUpdates no
```

## Custom signatures in the database directory

- Save local signatures as text files with the right extension beside the `.cvd` files. They load after the official ones and need no signing. https://docs.clamav.net/faq/faq-cvd.html
- The `.info` file inside a CVD lists `name:size:sha256` and cannot be loaded on its own. https://docs.clamav.net/manual/Signatures/DatabaseInfo.html

## SELinux

- Red Hat variants use the boolean `antivirus_can_scan_system`, which libclamav does not set. Symptom: `RWX mapping denied` while freshclam tests databases. https://docs.clamav.net/manual/Usage/Configuration.html#configure-selinux-for-clamav

```bash
setsebool -P antivirus_can_scan_system 1
```

- A new freshclam log file needs its context, `restorecon /var/log/freshclam.log`. https://docs.clamav.net/manual/Usage/Configuration.html#freshclamconf

## Windows specifics

- Copy `conf_examples/freshclam.conf.sample` and `clamd.conf.sample` in the install directory, then delete the `Example` line. https://docs.clamav.net/manual/Usage/Configuration.html#windows
- Config lookup: registry `HKEY_LOCAL_MACHINE/Software/ClamAV/ConfDir`, then the `libclamav.dll` directory, then `C:/ClamAV`. Databases: `DataDir` key, then `database` beside the DLL, then `C:/ClamAV/db`. https://docs.clamav.net/manual/Usage/Configuration.html#additional-notes-about-the-config-files-and-database-directories
- Services: `clamd --install-service`, `net start clamd`, `net stop clamd`, `clamd --uninstall-service`. https://docs.clamav.net/manual/Usage/Services.html

## DCONF

- `daily.cfg` lets the ClamAV team disable defective features per engine version, format `Category:Flags:StartFlevel:EndFlevel`. The settings are largely vestigial and target versions that should no longer run. https://docs.clamav.net/manual/Signatures/DynamicConfig.html
