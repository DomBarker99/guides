# Proxmox — Snapshot and VM Management Guide

A practical guide to managing virtual machines and snapshots in the Proxmox VE web UI. Covers creating, configuring, and controlling VMs, with a focus on using snapshots to safely test changes and roll back when things go wrong.

---

## Table of Contents

1. [VM Lifecycle](#vm-lifecycle)
2. [Creating a VM](#creating-a-vm)
3. [VM Configuration](#vm-configuration)
4. [Starting, Stopping, and Controlling VMs](#starting-stopping-and-controlling-vms)
5. [Snapshots](#snapshots)
6. [Cloning VMs](#cloning-vms)
7. [Backups and Restore](#backups-and-restore)
8. [Templates](#templates)
9. [Quick Reference](#quick-reference)

---

## VM Lifecycle

A VM in Proxmox goes through a simple lifecycle:

```
Create  -->  Configure  -->  Start  -->  Snapshot / Clone / Backup
                                              |
                                         Rollback (if needed)
                                              |
                                         Destroy (when done)
```

Everything in this guide is done through the Proxmox web UI, accessible at `https://<proxmox-host-ip>:8006` in your browser.

---

## Creating a VM

### Step 1: Upload an ISO

Before creating a VM, you need an installation image:

1. In the sidebar, expand your node and click on your storage (e.g., **local**)
2. Go to the **ISO Images** tab
3. Click **Upload** and select your ISO file (e.g., Ubuntu Server 24.04)

### Step 2: Create the VM

1. Click **Create VM** in the top-right corner
2. Walk through the configuration tabs:

| Tab | Key Settings |
|-----|-------------|
| **General** | Node, VM ID (auto-assigned or pick your own), Name |
| **OS** | Select your uploaded ISO image |
| **System** | BIOS type (SeaBIOS or OVMF/UEFI), SCSI controller (VirtIO SCSI recommended) |
| **Disks** | Disk size, storage location, format (raw or qcow2) |
| **CPU** | Number of cores and sockets |
| **Memory** | RAM in MB |
| **Network** | Bridge (usually `vmbr0`), model (VirtIO recommended) |

3. Review the summary and click **Finish**

The VM will appear in the sidebar under your node. It is created in a stopped state — you still need to start it and run through the OS installer.

---

## VM Configuration

All VM settings are managed through the tabs that appear when you select a VM in the sidebar.

### Hardware Tab

The **Hardware** tab shows every virtual device attached to the VM: CPU, memory, disks, network adapters, and the CD/DVD drive.

To change a setting:

1. Select the VM in the sidebar
2. Go to the **Hardware** tab
3. Select the item you want to change (e.g., **Memory**)
4. Click **Edit** in the toolbar above the list
5. Modify the value and click **OK**

Common changes:

| Setting | Where to Find It |
|---------|-----------------|
| CPU cores | **Hardware** > **Processors** > **Edit** |
| RAM | **Hardware** > **Memory** > **Edit** |
| Add a disk | **Hardware** > **Add** > **Hard Disk** |
| Add a network adapter | **Hardware** > **Add** > **Network Device** |
| Remove the install ISO | **Hardware** > select **CD/DVD Drive** > **Edit** > set to **Do not use any media** |

### Resizing a Disk

1. Go to the **Hardware** tab
2. Select the disk (e.g., **scsi0**)
3. Click **Resize disk** in the toolbar (or under **Disk Action**)
4. Enter the amount of space to add (e.g., `10` to add 10 GB)
5. Click **Resize disk**

**Note:** You can grow a disk but you cannot shrink it. After growing the disk in Proxmox, you still need to expand the filesystem inside the VM (e.g., using `growpart` and `resize2fs` on Linux).

### Options Tab

The **Options** tab controls VM behavior settings:

| Setting | What It Does |
|---------|-------------|
| **Start at boot** | Automatically start this VM when the Proxmox host boots |
| **Boot Order** | Which devices to boot from and in what order |
| **QEMU Guest Agent** | Enable communication between Proxmox and the VM (recommended — improves shutdown reliability and snapshot consistency) |
| **Start/Shutdown order** | Set priority when multiple VMs auto-start |

To change any of these, select the option and click **Edit**.

---

## Starting, Stopping, and Controlling VMs

The power control buttons appear in the top-right corner when you select a VM in the sidebar.

| Button | What It Does |
|--------|-------------|
| **Start** | Powers on the VM |
| **Shutdown** | Sends a graceful ACPI shutdown signal to the guest OS |
| **Stop** | Immediately kills the VM (like pulling the power cable) |
| **Reboot** | Graceful restart |
| **Pause** | Freezes the VM in place (CPU halted, memory preserved) |
| **Resume** | Unfreezes a paused VM |

**Shutdown vs Stop:** Always prefer **Shutdown** for a clean power-off. **Stop** is equivalent to cutting power and can cause filesystem corruption or data loss inside the VM. Only use it if the VM is unresponsive to shutdown.

### Console Access

To interact with the VM directly:

1. Select the VM in the sidebar
2. Click the **Console** tab (or click **>_ Console** in the top-right dropdown)
3. A noVNC window opens showing the VM's display

This is how you run through the OS installer on a new VM and how you access VMs that you cannot reach over SSH.

---

## Snapshots

Snapshots capture the exact state of a VM at a point in time — disk contents, configuration, and optionally RAM state. They let you make risky changes (OS upgrades, config edits, package installs) with a guaranteed rollback point.

### How Snapshots Work

- Snapshots are stored as a diff layer on top of the current disk state — they do not create a full copy
- Taking a snapshot is fast because it only starts tracking changes from that point forward
- Rolling back replays the original state and discards changes made after the snapshot
- Snapshots are **not backups** — they live on the same storage as the VM. If the storage fails, both the VM and its snapshots are lost

### When to Snapshot

- Before an OS or kernel upgrade
- Before installing or removing packages that affect system services
- Before editing critical config files (networking, Docker, etc.)
- Before running a destructive script for the first time

### Taking a Snapshot

1. Select the VM in the sidebar
2. Go to the **Snapshots** tab
3. Click **Take Snapshot**
4. Fill in the fields:

| Field | What to Enter |
|-------|--------------|
| **Name** | A short descriptive name (e.g., `before-docker-install`) |
| **Description** | Optional free-text note explaining what this snapshot is for |
| **Include RAM** | Check this to capture the live memory state (see below) |

5. Click **Take Snapshot**

The snapshot appears in the tree view. `NOW` always represents the current live state of the VM.

**With or without RAM?**
- **Without RAM** (default): Faster, smaller. When you roll back, the VM will be in a stopped state and you start it fresh. Good for most cases.
- **With RAM**: Slower, larger. When you roll back, the VM resumes exactly where it was — same processes, same open connections. Useful when reproducing a specific runtime state matters.

### Rolling Back a Snapshot

**This is a destructive operation.** Rolling back discards all changes made since the snapshot was taken — disk writes, config changes, everything.

1. Select the VM > **Snapshots** tab
2. Select the snapshot you want to roll back to in the tree
3. Click **Rollback**
4. Confirm the action

**The VM must be stopped before rolling back.** If it is running, use the **Shutdown** button first (or **Stop** if the VM is unresponsive).

**What happens during rollback:**
- The disk is reverted to the exact state at snapshot time
- Any data written after the snapshot is permanently lost
- If the snapshot included RAM, the VM resumes in its captured running state
- If the snapshot did not include RAM, the VM will be stopped after rollback — start it manually

### Deleting a Snapshot

Deleting a snapshot merges its diff layer into the base image. This frees up the space the snapshot was using without affecting the current state of the VM.

1. Select the VM > **Snapshots** tab
2. Select the snapshot in the tree
3. Click **Remove**
4. Confirm

**This does not roll back.** Deleting a snapshot just removes the rollback point. The current state of the VM is unchanged.

### Snapshot Best Practices

- **Name snapshots descriptively** — `before-docker-install` is better than `snap1`
- **Don't accumulate snapshots** — each snapshot adds I/O overhead because Proxmox must track diffs. Delete snapshots you no longer need.
- **Snapshot before, not after** — the point is to have a clean state to return to
- **Don't rely on snapshots as backups** — they are tied to the same storage. Use proper backups (see [Backups and Restore](#backups-and-restore)) for disaster recovery.

---

## Cloning VMs

Cloning creates a copy of a VM. Useful for spinning up a second instance from a known-good configuration.

### Full Clone vs Linked Clone

| Type | How It Works | Pros | Cons |
|------|-------------|------|------|
| **Full clone** | Complete independent copy of all disks | Fully standalone, can be moved to different storage | Uses the same amount of disk space as the original |
| **Linked clone** | Copy-on-write reference to the original's disk | Fast to create, uses minimal initial space | Depends on the original VM — cannot delete the source without breaking the clone |

### Cloning a VM

1. Right-click the VM in the sidebar (or select it and click **More > Clone**)
2. Fill in the fields:

| Field | What to Enter |
|-------|--------------|
| **Target node** | The Proxmox node to create the clone on (default: same node) |
| **VM ID** | Auto-assigned or pick your own |
| **Name** | A name for the new VM |
| **Mode** | **Full Clone** or **Linked Clone** |
| **Target Storage** | Where to store the cloned disk (full clone only) |

3. Click **Clone**

**After cloning**, update anything that should be unique inside the new VM: hostname, static IP address, SSH host keys, and any application-level identifiers.

---

## Backups and Restore

Backups are full exports of a VM that can be stored on separate storage and restored even if the original is destroyed. Unlike snapshots, backups protect against storage failure.

### Creating a Backup

1. Select the VM in the sidebar
2. Go to the **Backup** tab
3. Click **Backup now**
4. Configure the backup:

| Setting | Options |
|---------|---------|
| **Storage** | Where to save the backup (e.g., `local`, a NFS share, etc.) |
| **Mode** | **Snapshot** (no downtime), **Suspend** (brief pause), or **Stop** (full stop — most consistent) |
| **Compression** | **ZSTD** (recommended — fast and good compression), **LZO**, or **GZIP** |

5. Click **Backup**

| Backup Mode | Behavior |
|-------------|----------|
| **Snapshot** | Backs up the VM while it is running (no downtime). Uses a temporary snapshot internally. |
| **Suspend** | Briefly pauses the VM during backup to ensure consistency, then resumes. |
| **Stop** | Stops the VM, backs it up, then restarts it. Most consistent but causes downtime. |

### Scheduling Automated Backups

1. Go to **Datacenter** in the sidebar (the top-level item)
2. Click the **Backup** tab
3. Click **Add**
4. Configure the schedule (e.g., daily at 2:00 AM), target storage, which VMs to include, compression, and retention policy
5. Click **Create**

### Restoring from a Backup

1. In the sidebar, click on your backup storage (e.g., **local**)
2. Go to the **Content** tab
3. Select the backup file (they are named with the VM ID and timestamp)
4. Click **Restore**
5. Choose the target VM ID and storage location
6. Click **Restore**

You can restore to the original VM ID (overwriting it) or to a new VM ID to create a separate copy.

---

## Templates

Templates are read-only VM images you can clone from repeatedly. They are ideal for creating a base image (e.g., Ubuntu Server with Docker pre-installed) and spinning up new VMs from it quickly.

### Creating a Template

1. Set up a VM exactly how you want the base image to be (install the OS, configure packages, etc.)
2. Shut down the VM
3. Right-click the VM in the sidebar > **Convert to Template**
4. Confirm

**This is irreversible.** Once a VM is converted to a template, it cannot be started or modified. You can only clone from it. The VM icon in the sidebar changes to indicate it is now a template.

### Cloning from a Template

The workflow is the same as regular cloning:

1. Right-click the template in the sidebar > **Clone**
2. Choose **Full Clone** or **Linked Clone**, set a name and VM ID
3. Click **Clone**

The new VM is a fully working copy you can start and modify.

---

## Quick Reference

| Task | How To |
|------|--------|
| Create a VM | **Create VM** button (top-right) |
| Start / Shutdown / Stop | Power buttons (top-right when VM is selected) |
| Edit hardware (CPU, RAM, disk) | VM > **Hardware** tab > select item > **Edit** |
| Resize a disk | VM > **Hardware** > select disk > **Disk Action** > **Resize** |
| Change boot order | VM > **Options** tab > **Boot Order** > **Edit** |
| Enable auto-start | VM > **Options** tab > **Start at boot** > **Edit** |
| Open console | VM > **Console** tab |
| Take a snapshot | VM > **Snapshots** > **Take Snapshot** |
| Roll back a snapshot | VM > **Snapshots** > select snapshot > **Rollback** |
| Delete a snapshot | VM > **Snapshots** > select snapshot > **Remove** |
| Clone a VM | Right-click VM > **Clone** |
| Convert to template | Right-click VM > **Convert to Template** |
| Backup a VM | VM > **Backup** > **Backup now** |
| Schedule backups | **Datacenter** > **Backup** > **Add** |
| Restore a backup | Storage > **Content** > select backup > **Restore** |