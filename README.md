# SC Lite firmware with the sclite agent

Builds an SD-card burn image from Goldshell's stock SC Lite image, with the sclite agent built in
and Goldshell's extras removed.

```
python3 build.py --stock burn-Goldshell-SCLITE_MCB_V4_3-2_2_0.img \
                 --agent sclite-agent-arm \
                 --out burn-Goldshell-SCLITE_MCB_V4_3-2_2_0-sclite.img --xz
```

Build the agent first (from `agent/`, Go 1.24+, no downloads needed):

```
CGO_ENABLED=0 GOOS=linux GOARCH=arm GOARM=7 go build -trimpath -buildvcs=false -ldflags="-s -w" -o sclite-agent-arm .
```

The build is reproducible: file times and the build date are pinned (override with `SOURCE_DATE_EPOCH`),
so the same stock image and agent give the same image byte for byte.

Needs python3 and e2fsprogs. The three small ARM helpers (the port library, the intminer wrapper and the
chip-trim library) come prebuilt and tested in `files/*.bin`; `--rebuild-helpers` compiles them from `files/*.c` with clang + ld.lld (clang 18 gives the same bytes). Tested on stock 2.2.0
(sha256 `27051af0…df0bfe`). On any other version every edit checks that what it replaces is there,
and the build stops instead of producing a half-modified image.

Before an image is written, the build also checks the result:
- **Room:** all removals are applied first, then it checks the new files fit, then writes them. (debugfs
  doesn't fail on a full filesystem: it leaves the rest of the file as zeros and exits 0. That broke
  v1.4.5 and v1.4.6, whose agent was written before the removals and didn't fit.) Any error debugfs
  prints stops the build.
- **Read back:** every written file is read back from the filesystem and compared (sha256).
- **Startup:** the agent is started from the finished filesystem 5 times under qemu-arm, in a chroot of
  the image's own files; each run must serve the panel and not crash. Needs root, `unshare` and a static
  qemu-arm (`qemu-user-static`, or `QEMU_ARM=path`); `--skip-startup-check` skips it (not for releases).

## Flashing

Write the `.img` (or `.img.xz`) to a microSD card with balenaEtcher and use Goldshell's normal SD
burn procedure. The image keeps Goldshell's burn format byte for byte except inside the system
filesystem. That format has no signature, only a CRC32 over the container header, which is
unchanged. To go back, flash the stock image the same way.

After it boots: open `http://<miner-ip>/` (or `:8080`), sign in with `admin` / `admin` and set a new
password. Each miner gets its own password, and the same username and password sign in over SSH
(`ssh admin@<miner-ip>` before the first change also asks for a new password). The agent finds Goldshell's internal web password on its own
(the factory `123456789`, or `admin` on older units). Then add a pool in the Pools tab: on a fresh unit
the miner only starts once a pool exists.

To flash: write the image to a microSD card, power the miner off, insert the card, power on. Red and
green blink while it writes (about 2 minutes); when only green blinks it's done. Pull the card and the
miner reboots into the new firmware.

## What changes

| | stock | this build |
|---|---|---|
| Control panel | Goldshell web UI on :80 | sclite agent on **:8080**, also on :80 (the LAN address) and https on :443; :8080 and :80 take both http and https. SSH on :22 (same account) |
| Goldshell web API (minerd) | :80 on the LAN | 127.0.0.1:80, only the agent can reach it |
| cgminer API (intminer) | :4028 on the LAN (`A:0/0`) | 127.0.0.1:4028 |
| Goldshell cloud | MQTT to hub.goldshell.com, reports to find.goldshell.com | both hostnames resolve to 127.0.0.1 |
| Wi-Fi | wifi daemon, hostapd, wpa_supplicant, dnsmasq | removed (no Wi-Fi chip on this board) |
| mDNS | avahi-daemon on the LAN | removed |
| Time zone | fixed to China (UTC+8) in `/etc/localtime` | `/etc/localtime` links to `/usr/config/sclite/localtime`, which the panel writes (the browser's zone on first sign-in, or Settings → Time); Goldshell's file is kept as `/etc/localtime.stock` |
| NTP | 4 clients, 2 on hardcoded servers in China | 1 loop (`/usr/libexec/sclite/ntp`): the servers set in Settings → Time, then the ones the router hands out by DHCP (option 42, which `dhcpcd` already requests; `/libexec/dhcpcd-hooks/40-sclite-ntp` saves them); pool.ntp.org only when those give no time (it can be switched off in Settings → Time). Names are passed to `ntpd -q` only once a name resolves (BusyBox 1.21 ntpd gives up for good when any server name doesn't). Re-synced hourly; after 6 failures in a row, retried every 5 min; works on a LAN without internet |
| Per-chip clocks | the board clock for every chip | optional chip trims (Controls & fan → Chip trims, off by default): a second copy of intminer, `/usr/libexec/sclite/hook/intminer`, with a hook at the end of its clock-change function (`files/patch_intminer.py`, 4 + 30 bytes, every original byte checked; the cave sits in the never-called DTFS code), and `/lib/sclite_chip.so`, which sets trimmed chips with the miner's own per-chip clock function and writes per-chip counters to `/tmp/sclite-chipstats.json`. The same copy carries two guards (`patch_divzero.py`: a board that answers 0 chips no longer crashes the miner; `patch_poolinput.py`: a tiny pool difficulty or an endless pool line no longer does) and is the miner the wrapper runs: with the trim library while `/usr/config/sclite/chiphook` exists, without it otherwise. After 3 starts in a row that never get mining going, the wrapper runs the stock miner (`/usr/libexec/sclite/intminer`). Save report downloads the trims, counters, their 48-hour history, the hook log, auto-trim and the strong-chip test as one JSON file. (The self-tests that proved a trim reaches its chip and the chip pairs ran from v1.4.2 to v1.5.20 and were removed in v1.5.21.) |
| Verbose share log | config key the firmware drops | `-D -P` added by the intminer wrapper while the agent's flag is set (the miner prints its per-share lines only at debug level); survives restarts and reboots |
| SSH | none (no SSH server; shared host keys left in the image) | built into the agent on **:22**: signs in with the panel username and password, forced password change on first sign-in, shell + exec + SFTP, host key made per unit; on/off and port in Settings. The leftover shared host keys are removed |
| MAC address | `maccfg` sets eth0 from the u-boot environment (`/dev/env`); on boards where that area is empty it fails and the network chip starts with a different address at every boot | `/etc/sclite-mac`, run by `S40network` right after `maccfg`: keeps the address in `/usr/config/sclite/mac` (saved from eth0 on the first boot), or a new random one each boot, or one typed in Settings → Network & SSH. Only a valid unicast address is ever set; nothing MAC-related is in the image, so every unit keeps its own |
| Internet check | xminerd kills/restarts the miner unless a pool host or www.baidu.com / www.google.com:80 answers, checked every 3 s, pools first (a pool named by host name was looked up in DNS, forwards and backwards, every 3 s) | both fallback hosts are replaced in the binary by 127.0.0.2:80, where the agent answers, and that is tried first; the pools only if it fails. No outside host, no DNS: mining doesn't depend on the internet |
| Fan check (xminerd) | the first reading at a new fan level becomes that level's reference, even from a fan that is already dead or still spinning down | kept, but a reading is stored only when the fan passed (`patch_xminerd.py` p1, 10 bytes, original bytes checked) |
| Pools | whatever the config says | the miner always starts with `--no-pool-redirect` (a pool can't move it to another server); the agent's pool guard removes any pool or account that isn't in the Pools tab from the running miner (Idle if it keeps coming back); the factory Litecoin config `/root/settings/qomo.json` (someone else's pools, unused) is removed |
| Serial console (UART) root | published Goldshell factory password (`root123`) — the same on every stock unit | replaced with a port recovery password: sign in as **`root`** / **`sclite-recover`**. Serves as the owner's physical way back in (the UART is inside the miner; no network service reads `/etc/shadow`). Separate from the panel/SSH password — changing that does not change this, and this does not work over SSH or the web panel |
| `default` / `goldshell` accounts | `default` has an empty password, `goldshell` = "goldshell" (serial console) | both locked |
| SD / USB auto-mount | normal | `nosuid,nodev` (the SD upgrade path still works) |

How the ports are closed: the kernel has no netfilter, so there's no firewall to use. Instead
`files/sclite_bind.c` (a 2 KB library with no libc of its own) is loaded into minerd (from inittab)
and intminer (by the wrapper). It turns their TCP listening sockets into loopback ones. UDP and
outgoing connections are untouched.

`files/intminer_wrap.c` replaces `/usr/bin/intminer`. The stock miner moves to
`/usr/libexec/sclite/intminer`, so the running process keeps its name, and the wrapper keeps
argv[0] as `/usr/bin/intminer`, because minerd recognises the miner by its command line.

The agent stores its settings on the config partition (`/usr/config/sclite`). The SD card is only
for flashing.

## Update packages (Settings → Firmware update)

`build.py --version 1.4.8 --update-out sclite-update-v1.4.8.tar.gz` also writes an update package: a
gzipped tar of `manifest.json` and `files/<path>` for every file the build adds or changes against stock,
plus the stock paths it removes. The panel installs it over any earlier sclite build without an SD card:

- **Checked first:** every file against its sha256 and size, nothing extra, no path on the config
  partition or in /proc, /sys, /dev, /tmp; and a base check (the stock miner every sclite build keeps
  at `/usr/libexec/sclite/intminer`), so a package for another miner or Goldshell version is refused.
  Packages are not signed: anyone can build and install their own.
- **Writable for the install:** the system partition is mounted read-only (Goldshell's inittab), so the
  panel remounts `/` read-write for an install or a roll back and read-only again after; S48sclite does
  the same for a boot-time roll back.
- **Backed up:** the current version of every path it touches goes to `/usr/config/sclite/update/backup.tgz`
  (paths relative to /, so BusyBox tar can restore them), with `created` and `created_dirs` listing what
  it adds. Then a `pending` marker, then each file is written beside its target and renamed over it,
  or, when the partition hasn't room for both copies, written after the old one is removed; each is read
  back and checked. Any error restores the backup.
- **Room for the agent:** the agent is ~10 MB and the system partition has ~4 MB free, and a running
  program's file keeps its blocks until it ends. So when a package replaces the agent, the panel copies
  itself to /tmp (RAM) and execs that copy with `-install-update`, which installs and reboots (or, on
  failure, restores the backup and exits; S48sclite's loop starts the old panel again).
- **Confirmed or rolled back:** after the reboot, the new panel confirms once it has run for 90 s. Until
  then `S48sclite` restores the backup and reboots if the panel isn't up within 5 minutes, or on the 4th
  boot. *Roll back* in the panel leaves a `rollback` marker and reboots; S48sclite restores the backup
  before the panel starts.
- `tools/test-update.sh A.img update-B.tar.gz agent-A agent-B` runs the whole flow on a loop-mounted copy
  of image A's system partition (its real size), with RAM /tmp, the image's own boot script and its own
  network namespace: install from A's panel, boot B, B confirms, roll back from B's panel, boot A, fsck.
- Installing the package over v1.0.0 through v1.4.4 gives exactly the same system files as flashing the
  image, and rolling back gives exactly the files from before (checked file by file).

The panel's https certificate is made on the miner's first start (`/usr/config/sclite/tls.crt`), unique to
each unit. It's self-signed, so browsers warn once. The agent binds port 80 on each LAN address rather
than on all addresses, because minerd holds 127.0.0.1:80, and follows DHCP changes.

## Not changed (and why)

- **Kernel (Linux 3.10, InfoTM vendor build) and C library (uClibc 0.9.33.2):** Goldshell's
  closed-source binaries (minerd, intminer, xminerd, fanctrl) are built against them, and there is
  no vendor kernel source.
- **OpenSSL 1.0.1e, curl 7.33, BusyBox 1.21:** old. With the cloud cut off and only the agent
  listening on the LAN, nothing reachable from the network uses them. Updating them is a possible
  second phase.
- **minerd** stays: the agent uses its `/mcb` and `/dbg` API to set clocks, voltages, fans and
  pools.
