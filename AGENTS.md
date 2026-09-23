# AGENTS.md - System Architecture & Engineering Memory

This file serves as persistent memory and architectural documentation for AI agents working on the `lanparty` codebase.

---

## 1. System Overview & Dual-Drive Storage Architecture

The system provides high-performance, diskless netbooting for LAN party client nodes over iSCSI using LIO (Linux-IO / `targetcli`) and LVM on Linux.

### Storage Layout:
- **Base OS Image (`BASE_IMAGE="main"`)**:
  - Located on a fast SAS SSD (250GB).
  - Loopback cached via `/dev/loop7` (`CACHE_LOOP_DEVICE`) with page caching enabled.
  - Client COW overlays are thin-provisioned LVs inside `scratchpool` (default virtual quota: `OS_COW_QUOTA=150G`).
  - Device mapper target: `0 <sectors> snapshot /dev/loop7 /dev/$VGROUP/$MACHINE-cow N 128`.
  - Exposed via iSCSI as **LUN 1** (matching initiator DHCP `root-path` `:::1:`).
- **Base Games Image (`GAMES_IMAGE="master"`)**:
  - Located on a 4TB RAID 0 HDD array (`vg1/master`).
  - Client COW overlays are **external-origin thin snapshots** inside `scratchpool` (default quota: `GAMES_SNAPSHOT_QUOTA=350G`).
  - Exposed via iSCSI as **LUN 2** (`GAMES_LUN=2`).
- **Scratch Pool (`THIN_POOL="scratchpool"`)**:
  - LVM Thin Pool residing on a dedicated NVMe scratch SSD (`700GB`).
  - Hosts both the OS thin COW overlays (`$MACHINE-cow`) and the games thin snapshots (`$MACHINE-games`).
- **Optional Games Mode**:
  - If `GAMES_IMAGE=""` or is commented out in `/etc/lanparty.conf`, the system transparently runs in single-drive OS-only mode (skipping all games logic and LUN 2 exports).

---

## 2. Hard-Won Lessons & Critical Quirks

### A. LVM External Origin Lifecycle & Permissions
- **Requirement**: In LVM, an external origin logical volume (like `master`) **MUST be both read-only (`lvchange -p r`) and inactive (`lvchange -an`)** before creating thin snapshots from it (`lvcreate -s -n ... --thinpool ...`).
- **Exit Code 5 Trap**: If a volume is already in the target permission or activation state, `lvchange` prints a notice (e.g. `is already read only`) and returns **exit code 5**. Because `lanparty` runs under `set -euo pipefail`, this terminates the entire script prematurely!
  - **Rule**: Always inspect attributes using `lvs --noheadings -o lv_attr` (`${ATTRS:1:1}` for `r` vs `w`, `${ATTRS:4:1}` for `a` vs `-`) before calling `lvchange`, and append `|| true`.
- **Teardown Restoration (`lanparty destroy`)**:
  - When all client snapshots are destroyed, `master` is no longer serving as an external origin.
  - `lanparty destroy` checks if any dependent snapshots remain (`lvs --noheadings -o origin`). If none remain, it automatically restores `master` to active read-write: `lvchange -p rw` and `lvchange -ay`.

### B. SCSI & iSCSI Protocol Realities
- **`iscsiadm -m discovery -t sendtargets`**:
  - Outputs entries like `10.67.27.1:3260,1 iqn.2019-12.com.example.server:psnodeone`.
  - The `,1` is the **Target Portal Group Tag (`tpg1`)**, NOT LUN 1!
  - `sendtargets` **never** lists LUNs.
- **Viewing LUNs on Client Nodes**:
  - Use `lsblk`, `lsscsi`, or `iscsiadm -m session -P 3`.
  - If a LUN was added after the node booted, trigger a rescan on the client:
    ```bash
    sudo iscsiadm -m session --rescan
    # or:
    for h in /sys/class/scsi_host/host*; do echo "- - -" | sudo tee $h/scan; done
    ```
- **LUN 0 & SCSI `REPORT LUNS`**:
  - Standard SCSI initiators issue `REPORT LUNS` to LUN 0 to discover devices. LIO defaults to starting at LUN 0 if unspecified. In this dual-drive setup, OS is LUN 1 and Games is LUN 2 (to match legacy DHCP `:::1:`).
  - If initiators require LUN 0 for discovery, a 1MB ramdisk controller (`/backstores/ramdisk/controller`) can be mapped to LUN 0 across all targets.

---

## 3. Merging Mechanism (`lanparty merge`)

### Syntax:
```bash
lanparty merge <HOST> [os | games | all]
```
*(Default when volume parameter is omitted: `all`)*

### OS Merge (Smart Block Diff):
- Base image is unmounted locally.
- Python-based diff scanner (`diff-sync-devices`) reads chunks from the client snapshot (`/dev/mapper/cached-$HOST`) and destination base (`/dev/$VGROUP/$BASE_IMAGE`), comparing blocks in memory and only writing modified blocks.
- Cleans up and recreates a fresh `$HOST-cow` overlay.

### Games Merge (Fast Thin-Delta):
- `thin-delta-sync` extracts block mappings from the thin pool metadata using `thin_dump -m` on the metadata LV (`${VG}-${POOL}_tmeta`).
- Merges contiguous modified chunks (up to 32MB) and skips all unwritten blocks.
- Temporarily activates and unlocks destination base storage with `blockdev --setrw /dev/$VGROUP/$GAMES_IMAGE`.
- Directly streams modified delta blocks to the destination in seconds.
- Re-secures `GAMES_IMAGE` (`lvchange -p r && lvchange -an`), removes the merged snapshot, and creates a fresh empty thin snapshot.
- Restarts iSCSI publishing for the host (`start-iscsi "$MERGE_HOST"`).

---

## 4. Configuration Reference (`/etc/lanparty.conf`)

Key variables relevant to the storage stack:
```bash
VGROUP="vg1"                     # Volume Group containing base images and thin pool
BASE_IMAGE="main"                # 250GB OS Base LV (SAS SSD)
GAMES_IMAGE="master"             # 4TB Games Base LV (RAID 0), or empty "" for OS-only
GAMES_LUN=2                      # LUN exposed for Games (default: 2)
THIN_POOL="scratchpool"          # LVM Thin Pool on NVMe scratch SSD
OS_COW_QUOTA="150G"              # Virtual quota for OS snapshot
GAMES_SNAPSHOT_QUOTA="350G"      # Virtual quota for Games snapshot
CACHE_LOOP_DEVICE="/dev/loop7"   # Dedicated loop device for OS read caching
```

---

## 5. Binary Installation & Sync Path
On the server (`gamebootserver`), ensure `/usr/local/bin/lanparty` is a symbolic link to the repository version:
```bash
ln -sf ~/lanparty/lanparty /usr/local/bin/lanparty
chmod +x ~/lanparty/lanparty
```
This ensures running `git pull` in `~/lanparty` immediately applies all code updates to the active `lanparty` command.
