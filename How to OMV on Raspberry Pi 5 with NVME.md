# Raspberry Pi 5 NVMe + OMV Setup Manual

## Goal

* OMV *System* and *Storage* on a single NVME
* OMV System fully isolated from the Storage partition (and SD card)
* SD card: Raspberry Pi OS Full (maintenance / installer)
* NVMe p1+p2: Raspberry Pi OS Lite (OMV system)
  * p1 → boot
  * p2 → root
* NVMe p3: Storage partition (for OMV services)

---

## 1. Prerequisites

* Raspberry Pi 5
* SD card (Raspberry Pi OS Full)
* NVMe drive + NVMe hat
* Network connection
* `rpi-imager` installed on Raspberry Pi OS Full

---

## 2. Flash Raspberry Pi OS Lite to NVMe

1. Boot Raspberry Pi OS Full from SD card.
2. Insert NVMe drive.
3. Run **Raspberry Pi Imager**:

   * OS: Raspberry Pi OS **Lite** (64-bit)
   * Storage: entire NVMe device (`/dev/nvme0n1`)
   * Allow Imager to repartition (automatic)

After flashing, NVMe layout:
```
nvme0n1p1  512M  boot
nvme0n1p2  2.3G  root
```

---

## 3. Partition NVMe for OMV System + Data

**Steps to resize system and create data partition:**

* Stay booted from SD.
* Ensure NVMe root is not mounted.
* Check filesystem for errors.
* Resize root filesystem to 64 GB.
* Resize root partition to match filesystem.
* Create NVMe data partition and format with label `OMV_Storage`.

Expected output of `lsblk -f`:
```bash
nvme0n1
├─nvme0n1p1 vfat   FAT32 bootfs 1515-D8F3
└─nvme0n1p2 ext4   1.0   rootfs 0b68493e-4b4c-43aa-b88d-df23601fa0e0
```

Execute:
```bash
sudo umount /dev/nvme0n1p2
sudo e2fsck -f /dev/nvme0n1p2
sudo resize2fs /dev/nvme0n1p2 64G
sudo parted /dev/nvme0n1 --script \
  resizepart 2 64GiB \
  mkpart primary ext4 64GiB 100%
sudo mkfs.ext4 -L OMV_Storage /dev/nvme0n1p3
lsblk -f
```

Expected output of `lsblk -f`:
```
nvme0n1
├─nvme0n1p1 vfat   FAT32 bootfs      1515-D8F3
├─nvme0n1p2 ext4   1.0   rootfs      0b68493e-4b4c-43aa-b88d-df23601fa0e0
└─nvme0n1p3 ext4   1.0   OMV_Storage d59d74e2-f497-4dc8-8223-af4e7c30b982
```

* p1 + p2 → OMV system
* p3 → OMV data (~1.8 TB in example)

---

## 4. Boot NVMe and configure Raspberry Pi OS Lite

1. Set NVMe as first boot device via `raspi-config` or EEPROM.
2. Set Boot Order to NVMe
3. Reboot into NVMe Raspberry Pi OS Lite
4. Verify boot from NVMe `findmnt /`:
```bash
/   /dev/nvme0n1p2 ext4   rw,noatime
```

---

## 5. Install OpenMediaVault on NVMe

```bash
sudo apt update
sudo apt upgrade
wget -O - https://raw.githubusercontent.com/OpenMediaVault-Plugin-Developers/installScript/master/preinstall | sudo bash
sudo reboot
wget -O - https://raw.githubusercontent.com/OpenMediaVault-Plugin-Developers/installScript/master/install | sudo bash
sudo reboot
```

## 6. Configure OMV Storage

Mount the storage partition in OMV here:
`http://RASPBERRY_PI_IP_ADDRESS/#/storage/filesystems/mount`