# SodaOS — Technical Plan (Draft)

> Target: Ubuntu **24.04 LTS** base, Pop!_OS-style boot with **systemd-boot + UKI (ukify)**, **Btrfs-by-default**, **Full-Disk Encryption (LUKS2)**, **Flatpak-first** (no Snap), **Firefox (deb)**, and **two images**: AMD/Intel and NVIDIA. Secure Boot out of scope for v0.x.

---

## 1) Goals & Scope

- Ship a minimal, fast Ubuntu-based desktop with:
  - Vanilla **GNOME** (no COSMIC).
  - **Btrfs** root with sensible subvolumes and snapshot tooling readiness.
  - **FDE**: Encrypt everything except the ESP (UEFI requirement).
  - **Modern boot**: `systemd-boot` + **UKI** via **ukify** + `kernel-install`.
  - **No Snap**: Flatpak + Flathub preconfigured. Firefox from Mozilla **apt** repo.
  - **Two ISOs**: **AMD/Intel** (Mesa) and **NVIDIA** (proprietary driver).
- Keep the distro delta **tiny** (metapackages + installer profile + defaults).

---

## 2) Base & Editions

- **Base**: Ubuntu **24.04 LTS**.
- **Editions**:
  - **SodaOS (AMD/Intel)**: Open drivers (Mesa/i915/amdgpu).
  - **SodaOS (NVIDIA)**: Proprietary NVIDIA driver preinstalled; assumes **Secure Boot disabled**.

---

## 3) Disk & Filesystem Layout

**Partitions**
- `ESP` (FAT32, 512 MiB) → mount at **`/efi`** (un­encrypted; firmware requirement).
- **LUKS2 container** → holds everything else (including what would be `/boot`).
  - **Option A (recommended initially)**: LUKS2 → **LVM** → Btrfs LV (`lv_root`) + swap LV (`lv_swap`).
  - **Option B (later)**: LUKS2 → **Btrfs partition** + Btrfs swapfile (simpler, but test hibernate early).

**Btrfs subvolumes**
```
@            → /
@home        → /home
@log         → /var/log
@cache       → /var/cache
@snapshots   → /.snapshots
```

**Mount options**
```
compress=zstd:3,ssd,space_cache=v2,noatime,discard=async
```
*(If you prefer periodic TRIM, drop `discard=async` and enable `fstrim.timer`.)*

**Swap**
- **ZRAM default**: enabled on all installs via `zram-tools` using `zstd`, target size **200% of RAM** (kernel will cap appropriately), swap **priority 100**.
- **Disk-backed swap (for hibernate)**: if the installer option **Enable hibernation** is selected, create an LVM swap LV equal to RAM. Keep ZRAM at higher priority so it services runtime swapping; disk swap exists primarily for hibernate.
- **No hibernate** → no disk swap LV/file; ZRAM-only.

**LUKS2 defaults**
- Cipher `aes-xts-plain64` (keysize 512), PBKDF `argon2id` (iter-time ~2000ms).
- `/etc/crypttab`: `luks[,discard]` (make `discard` configurable).

---

## 4) Boot: systemd-boot + UKI (ukify)

**Overview**
- Use `systemd-boot` (installed via `bootctl`).
- Build **Unified Kernel Images** (kernel + initramfs + cmdline) with `ukify` triggered by `kernel-install`.
- Store UKIs at `\EFI\Linux\` on the ESP; loader entries under `\loader\entries\`.

**Key files**
- `/etc/kernel/cmdline`  
  - **AMD/Intel** baseline:
    ```
    root=UUID=<ROOT_UUID> rw rd.luks.name=<LUKS_UUID>=cryptroot rootflags=subvol=@ quiet splash
    ```
  - **NVIDIA** add:
    ```
    nvidia-drm.modeset=1 modprobe.blacklist=nouveau
    ```
  - Add `resume=UUID=<SWAP_UUID>` if hibernation enabled.
- `/etc/kernel/install.conf`
  ```ini
  layout=uki
  default=auto
  ESPPath=/efi
  ```
- `/etc/kernel/install.d/90-ukify-install.sh` (exec, builds UKI & loader entry on kernel updates).

**Postinstall boot steps**
1. Ensure `initramfs` exists: `update-initramfs -u`.
2. Seed first UKI:
   ```
   kernelver="$(uname -r)"
   kernel-install add "$kernelver" "/boot/vmlinuz-$kernelver"
   ```
3. Install bootloader:
   ```
   bootctl --esp-path=/efi install
   ```

---

## 5) Package Policy

**No Snap**
- Purge and pin `snapd`; remove `/snap`, `/var/{snap,lib/snapd,cache/snapd}`.
- Avoid Ubuntu meta packages that drag Snap (use SodaOS metas).

**Flatpak-first**
- Install `flatpak`, `gnome-software`, `gnome-software-plugin-flatpak`.
- Add Flathub system-wide:
  ```
  flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
  ```

**Firefox (deb)**
- Add Mozilla apt repo key & source in `/etc/apt/sources.list.d/mozilla-firefox.list`.
- Pin `origin packages.mozilla.org` high.
- `apt install firefox`.

---

## 6) Meta-packages (minimal surface)

```
packages/
  sodaos-core/                 # firmware, microcode, cryptsetup-initramfs, lvm2, btrfs-progs, zram-tools, ukify/kernel-install, flatpak, bootctl deps
  sodaos-gnome/                # minimal GNOME session (Files, Terminal, Settings, Software)
  sodaos-desktop-defaults/     # dconf defaults, theming, fstrim.timer, Firefox deb setup, Flathub bootstrap
  sodaos-desktop-amd/          # mesa-vulkan-drivers, vulkan-tools, firmware; optional linux-generic-hwe-24.04
  sodaos-desktop-nvidia/       # nvidia-driver-###, nvidia-dkms-###, nvidia-prime, nvidia-settings, dkms, linux-headers-generic
```

---

## 7) Installer (Calamares) Plan

**Automatic install (recommended)**
- **Partitioning**:
  - Create `ESP` (vfat, 512 MiB) → mount `/efi`.
  - Create LUKS2 on rest; prompt passphrase.
  - Inside LUKS:
    - Create LVM VG `vg_soda` with `lv_root` + `lv_swap`.
    - Format `lv_root` as **Btrfs**, create subvols, mount with options.
    - Format `lv_swap` as swap (size per hibernate toggle).
- **Users**: standard user setup.
- **fstab**: write Btrfs subvol mounts + swap UUID.
- **bootloader**: skip GRUB; run `bootctl` in postinstall.
- **Postinstall (shared)**:
  - Purge & pin Snap, add Flatpak+Flathub.
  - Add Mozilla repo + install Firefox.
  - Enable `fstrim.timer`.
  - Drop `/etc/kernel/cmdline`, `/etc/kernel/install.conf`, and `90-ukify-install.sh`.
  - Seed UKI and install systemd-boot.
- **Postinstall (NVIDIA)**:
  - Ensure NVIDIA stack is present.
  - Append NVIDIA flags to cmdline and rebuild the UKI once more.

**Manual install**
- Offer advanced/custom partitioning while keeping UKI & boot steps identical.

**Installer options**
- **Encryption**: passphrase input with strength meter.
- **Hibernate**: checkbox. If selected → create disk swap LV = RAM and add `resume=UUID=...`; if not selected → ZRAM-only (no disk swap).

---

## 8) Image Variants

- **SodaOS-24.04-btrfs-amd64-amd.iso**
- **SodaOS-24.04-btrfs-amd64-nvidia.iso**

**Differences**
- NVIDIA ISO includes proprietary driver stack and NVIDIA cmdline baked into UKI.
- Both share core packages, GNOME minimal, Flatpak policy, Firefox deb.

**Secure Boot**
- Explicitly **unsupported** in v0.x (document requirement to disable SB in firmware for NVIDIA ISO).

---

## 9) Repo Structure (proposed)

```
sodaos/
  packages/
    sodaos-core/
    sodaos-gnome/
    sodaos-desktop-defaults/
    sodaos-desktop-amd/
    sodaos-desktop-nvidia/
  image/
    calamares/
      settings.conf
      modules.d/
        partition.conf
        users.conf
        fstab.conf
        bootloader.conf
        postinstall.sh
        postinstall-nvidia.sh
    branding/sodaos/
      branding.desc
      artwork/...
    profiles/
      amd/profile.yaml
      nvidia/profile.yaml
  tools/
    build_iso.sh
    build_profile.sh
    sign.sh
  .devcontainer/
    devcontainer.json
  docs/
    INSTALL.md
    RELEASE.md
    ARCHITECTURE.md
```

---

## 10) Defaults & Tweaks

- Enable `fstrim.timer`.
- Sensible GNOME dconf (night light on, disable crash reports by default, etc.).
- `vm.swappiness=10` on laptops (via `/etc/sysctl.d/99-sodaos.conf`).
- Initramfs modules: ensure `btrfs`, `dm-crypt`, `dm-mod`, `nvme` included.
- For PRIME laptops (NVIDIA ISO), enable `nvidia-persistenced`.

---

## 11) Testing Matrix (MVP)

**AMD ISO**
- Desktop (UEFI, NVMe) with encryption + hibernate on/off.
- Intel iGPU laptop sanity.
- Snapshot creation/rollback sanity (timeshift/snapper later).

**NVIDIA ISO**
- Desktop dGPU (UEFI, SB off): boot, modeset, `nvidia-smi`.
- Hybrid laptop: PRIME on Wayland, suspend/resume, hibernate if enabled.

**Both**
- Kernel update → UKI regenerate → system boots new entry.
- Flatpak visible in GNOME Software; Firefox deb updates via `apt`.

---

## 12) CI/Release

- Build inside **KVM snapshot VM** (privileged, clean).
- Artifacts:
  - ISOs: AMD + NVIDIA
  - `SHA256SUMS` + `SHA256SUMS.gpg`
  - Release notes (SB off, Flatpak policy, known issues).
- Two channels (later):
  - **Stable**: tracks 24.04.x + security updates.
  - **Edge**: optional HWE kernel / newer Mesa via backports.

---

## 13) Roadmap

- **v0.1 (Initial)**: LUKS2+LVM+Btrfs, ukify UKI boot, Flatpak/Flathub, Firefox deb, AMD & NVIDIA ISOs (SB off).
- **v0.2**: Optional Btrfs swapfile path (no LVM), timeshift/snapper integration UX.
- **v0.3**: TPM2 auto-unlock option (`systemd-cryptenroll`).
- **v1.0**: Hardening, more hardware QA, docs polish; consider Secure Boot/signing later or keep out of scope.

---


---

## 15) Memory Compression (ZRAM)

**Policy**
- Enable **ZRAM swap by default** on all installs for faster, SSD-friendly swapping.
- Algorithm: **zstd**; target size: **min(200% of physical RAM, 16 GiB)**; swap **priority 100** so ZRAM is preferred over any disk swap.

**Packages**
- Include `zram-tools` in `sodaos-core` (unit disabled). Use a **custom oneshot** to control size cap reliably across kernels.

**Custom unit**
Create `/usr/local/sbin/sodaos-zram-setup`:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Compute size: min(2 * RAM, 16GiB)
ram_bytes=$(awk '/MemTotal/ {print $2 * 1024}' /proc/meminfo)
target=$(( ram_bytes * 2 ))
max=$(( 16 * 1024 * 1024 * 1024 ))
size=$(( target < max ? target : max ))

# Load module if needed
modprobe zram || true

# Create a single zram device
dev=/dev/zram0
# Reset if already configured
if [ -e /sys/class/zram-control/hot_remove ]; then
  echo 0 > /sys/class/zram-control/hot_remove || true
fi

echo zstd > /sys/block/zram0/comp_algorithm
echo $size > /sys/block/zram0/disksize

mkswap $dev
# Priority 100 so it is preferred over any disk swap
swapon -p 100 $dev
```

Make it executable:
```bash
chmod +x /usr/local/sbin/sodaos-zram-setup
```

Create the unit `/etc/systemd/system/sodaos-zram.service`:
```ini
[Unit]
Description=SodaOS ZRAM swap (capped at 16GiB)
DefaultDependencies=no
After=local-fs.target
Before=swap.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/sodaos-zram-setup
RemainAfterExit=yes

[Install]
WantedBy=swap.target
```

Disable the distro service and enable ours:
```bash
systemctl disable --now zramswap.service || true
systemctl enable --now sodaos-zram.service
```

**Hibernate interaction**
- If the user enables **Hibernate** in the installer, create a disk swap LV equal to RAM and add `resume=UUID=<swap>` to the kernel cmdline.
- Keep ZRAM at **higher priority**; disk swap exists primarily to support hibernation.


## 14) Open Questions (to finalize before build)

- **LVM vs swapfile** for v0.1? (Plan uses LVM; swapfile is a nice simplification later.)
- Default **kernel** track: LTS generic vs enable **HWE** by default?
- Preinstall any **developer tools** (e.g., `git`, `build-essential`) or keep base minimal?
- Ship any **preinstalled Flatpaks** (none vs browser only—already using Firefox deb)?
