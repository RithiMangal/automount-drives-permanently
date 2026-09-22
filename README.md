# Automount

<p align="center">
  <img src="icon.png" width="128" alt="Automount icon">
</p>

**Permanently automount all data drives at boot — on any Linux distro.**

Automount is a click-only, dependency-light utility that scans every disk
partition and registers the data volumes in `/etc/fstab`, so they mount
automatically on every boot. It ships as a single window — tick drives,
press **Activate**, done — and as a portable **AppImage** with the icon
bundled inside.

## Highlights

- **GUI-only, zero terminal** — launching the binary opens the window and
  nothing else: a pre-ticked drive checklist, an **Activate Permanent
  Automount** button, **Undo Last Change**, **Close**, and a live log panel.
  Privilege elevation goes through a graphical `pkexec` prompt; the
  privileged backend is the same binary re-executed headlessly.
- **No Linux drive left behind** — ext2/3/4, **btrfs volumes *and* hidden
  subvolumes** (each subvolume gets its own `subvol=` entry via a
  read-only top-level probe mount), xfs, f2fs, NTFS (`ntfs-3g`
  auto-detected), exFAT, VFAT, LVM logical volumes, md arrays, unlocked
  dm-crypt volumes. Multi-device btrfs members invisible to `lsblk` are
  caught via `btrfs filesystem show`.
- **Force mount, not hope-mount** — after writing the table, every entry
  is mounted immediately: plain attempt first, then harder retries
  (`btrfs device scan` + `degraded` mode for btrfs). Each drive reports
  `MOUNTED` or `FAILED` with the kernel's reason.
- **Boot can never hang** — every generated line carries
  `nofail,x-systemd.device-timeout=15`. A missing drive is a skipped
  mount, never a blocked boot.
- **Safe by construction** — timestamped `/etc/fstab` backup before any
  change, idempotent planning (present UUIDs/LABELs are never duplicated),
  `findmnt --verify` with automatic backup-restore on failure, one-click
  **Undo**, and system partitions (`/`, `/boot*`, `/efi`, swap,
  snap/loop) are never touched.
- **Universal method** — plain `fstab`, honoured by every distro
  (Debian/Ubuntu/Parrot OS, Arch, Fedora, …). No DE daemons, no
  desktop-specific tricks, no background services.
- **Tiny footprint** — ~850 KB binary / ~1.3 MB AppImage, pure Rust,
  no runtime. Build deps are `serde`, `serde_json`, `libc`, `gtk` only;
  runtime needs system GTK3 + polkit, present on every full desktop.

## Run

```bash
./dist/automount                        # opens the window
./dist/Automount-x86_64.AppImage        # same, portable (double-clickable)
```

1. Tick the drives to mount at every boot (sensible defaults pre-ticked).
2. Press **Activate Permanent Automount**, approve the root prompt.
3. Watch each drive report `MOUNTED` in the log. Reboot once — everything
   is already where it should be, on every boot after.

> Encrypted (`crypto_LUKS`) volumes need a key at boot and are listed
> with a note instead of an entry. On KDE Plasma the desktop shortcut
> handling differs — the tool still works, it just manages `fstab`.

## Build from source

```bash
./build.sh            # cargo build --release, strip → dist/automount
./build-appimage.sh   # + AppDir assembly → dist/Automount-x86_64.AppImage
cargo test            # planner unit tests (inside automount-rs/)
```

Build dependencies: a Rust toolchain, `libgtk-3-dev`, and `appimagetool`
for the AppImage step.

## How it works

```text
launch → GTK window (the only interface)
  │  checklist = lsblk scan − system/swap/mounted volumes
  │              (+ btrfs subvolumes when elevated)
  ▼
[Activate] → pkexec self --internal-apply --only <uuids>
  │  1. backup /etc/fstab (timestamped)
  │  2. append managed block (UUID=/LABEL=, nofail, subvol= where needed)
  │  3. mkdir mount points (/mnt/<Label> or /mnt/disk-<uuid8>)
  │  4. findmnt --verify (failure → restore backup, abort)
  │  5. daemon-reload + per-entry force-mount with report
[Undo]     → pkexec self --internal-undo (newest backup restored)
```

- **Discovery** — `lsblk -J` (normalised for multi-mount output) plus
  `btrfs filesystem show` as a second source; unmounted btrfs volumes
  are probe-mounted read-only at `subvolid=5` purely to list subvolumes,
  then released.
- **Identity** — `UUID=` preferred, `LABEL=` (space-escaped) fallback;
  volumes with neither are skipped with a reason instead of guessed.
- **Elevation without a terminal** — the GUI never runs as root itself;
  it re-executes its own path (`/proc/self/exe`-equivalent) under
  `pkexec`, streaming backend stdout live into the log view.

## Project layout

```text
automount-drives/
├── dist/
│   ├── automount                  # standalone binary
│   └── Automount-x86_64.AppImage  # portable bundle (icon included)
├── Automount.AppDir/              # AppImage source (desktop + icon + binary)
├── automount-rs/                  # Rust source (main.rs + gui.rs)
├── icon.png                       # app icon (512×512 PNG)
├── icon-gen.py                    # reproducible icon generator (Pillow)
├── build.sh / build-appimage.sh
└── README.md                      # this file
```

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| List is missing btrfs subvolumes | Subvolume probing needs root — they appear automatically during Activate |
| A drive reports FAILED | Read the kernel reason in the log; the entry stays with `nofail` and retries at boot |
| No pkexec prompt | Install polkit, or run the backend manually with sudo |
| Want the old table back | **Undo Last Change**, or restore `/etc/fstab.automount-bak-*` yourself |
