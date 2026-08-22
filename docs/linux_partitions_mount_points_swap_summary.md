# Linux Disk Partitions, Mount Points, and Swap — Summary

## 1. `lsblk`

`lsblk` means **list block devices**.

It displays storage devices and their structure, including:

- Hard drives
- SSDs
- NVMe drives
- USB drives
- Partitions
- LVM volumes
- Their mount points

Example:

```text
NAME   SIZE TYPE MOUNTPOINTS
sda    100G disk
├─sda1  30G part /
├─sda2  50G part /home
└─sda3  20G part /backup
```

Useful command:

```bash
lsblk -f
```

This can also show filesystem information such as `ext4`, UUIDs, and mount points.

---

## 2. Disk storage is not memory

When discussing partitions, the correct term is usually **disk storage**, not memory.

- **RAM / memory**: temporary working memory used by running programs.
- **Disk storage**: persistent storage where the operating system, files, and partitions exist.

For example:

```text
RAM  → temporary memory
Disk → persistent storage
```

---

## 3. What partitions provide

Separating a disk into different partitions or filesystems provides:

1. **Organization**
2. **Storage isolation**
3. **Storage limits**
4. **Protection**
5. **Separation of concerns**

Example:

```text
/       → operating system
/home   → user files
/backup → backup files
```

A disk could be divided like this:

```text
Disk: 100 GB

/       → 30 GB
/home   → 50 GB
/backup → 20 GB
```

Each filesystem has its own allocated capacity.

---

## 4. Why separating `/home` and `/backup` protects the system

The root filesystem `/` contains important operating system directories, including:

```text
/etc
/usr
/var
/bin
```

If `/` becomes completely full, the operating system can have problems:

- Programs may fail to create files.
- Logs may fail to write.
- Updates may fail.
- The system can become unstable.

If `/home` is a separate filesystem and becomes full:

```text
/       → still has its own available space
/home   → full
```

The files in `/home` cannot automatically consume the space allocated to `/`.

The same applies to `/backup`. Backups can become large, so putting them on a separate filesystem prevents backup files from consuming all the space reserved for the operating system.

### Important point

The protection comes from `/`, `/home`, and `/backup` being on **separate filesystems with separate allocated storage**.

Simply creating directories such as:

```text
/home
/backup
```

does **not** provide this protection if they are all stored on the same filesystem.

---

## 5. Separation of concerns

Separate filesystems allow different types of data to be isolated:

```text
/        → system and operating system files
/home    → user files and personal data
/backup  → backup data
```

This is a form of **separation of concerns** because each area has a different responsibility.

A good summary is:

> Separating a disk into different partitions or filesystems provides organization, storage isolation, resource limits, protection against one area filling the entire disk, and separation of concerns between system files, user data, and backups.

---

## 6. Swap

**Swap** is disk space that Linux can use as an extension of RAM.

When physical RAM becomes heavily used, the Linux kernel can move less-active memory pages from RAM to swap space.

Example:

```text
RAM is almost full
        ↓
Kernel identifies less-used memory
        ↓
Moves some memory pages to swap
        ↓
Frees RAM for more active processes
```

Swap is much slower than RAM because it uses disk or SSD storage.

Swap can also be used for **hibernation**, where the contents of RAM are saved so the computer can later restore its previous state.

---

## 7. Filesystems and mount points

Linux presents storage as a single directory tree:

```text
/
├── etc
├── usr
├── var
├── home
│   ├── user1
│   └── user2
└── backup
```

Different filesystems can be attached to different locations in this tree.

For example:

```text
Partition 1 → ext4 → mounted at /
Partition 2 → ext4 → mounted at /home
Partition 3 → ext4 → mounted at /backup
```

The location where a filesystem becomes accessible is called its **mount point**.

---

## 8. Choosing mount points during installation

During Linux installation, when manually configuring storage, the installer may let you choose a mount point such as:

```text
/
```

```text
/home
```

```text
/boot
```

or **Other**, where you can specify another mount point, such as:

```text
/backup
```

By choosing a mount point, you are telling the operating system:

> Mount this filesystem at this location in the Linux directory tree.

For example:

```text
Partition 1 → filesystem → /
Partition 2 → filesystem → /home
Partition 3 → filesystem → /backup
```

---

## 9. How Linux knows where files go

Once the filesystems are mounted correctly, the file path determines which filesystem stores the data.

For example:

```text
/etc/...              → filesystem mounted at /
/usr/...              → filesystem mounted at /
/var/...              → filesystem mounted at /
/home/user/file.txt   → filesystem mounted at /home
/backup/file.tar      → filesystem mounted at /backup
```

So if `/home` is a separate filesystem, everything stored under `/home` goes to that filesystem.

If `/backup` is separate, files under `/backup` go to that filesystem.

The key concept is **mounting**.

Linux does not simply treat a partition as "`/home`" by its name. Instead, the system is configured to:

> Take this filesystem and mount it at `/home`.

---

## 10. Complete example

Suppose you have a 100 GB disk:

```text
100 GB Disk
│
├── 30 GB → ext4 → mounted at /
│
├── 50 GB → ext4 → mounted at /home
│
├── 15 GB → ext4 → mounted at /backup
│
└── 5 GB  → swap
```

Linux then presents everything as one directory tree:

```text
/
├── etc       ← stored on the / filesystem
├── usr       ← stored on the / filesystem
├── var       ← stored on the / filesystem
├── home      ← separate filesystem mounted here
│   └── user
└── backup    ← separate filesystem mounted here
```

The physical storage may be separated, but Linux combines it into one logical directory structure.

---

## Core takeaway

The relationship between the main concepts is:

```text
Physical disk
      ↓
Partitions
      ↓
Filesystems (for example, ext4)
      ↓
Mount points
      ↓
Linux directory tree
```

For example:

```text
Physical Disk
    ↓
Partition
    ↓
ext4 filesystem
    ↓
Mounted at /home
    ↓
Files written to /home are stored there
```

Therefore, when you configure your disk during installation, you decide how storage is divided and where each filesystem is mounted. This gives you organization, storage limits, isolation, system protection, and separation of concerns.
