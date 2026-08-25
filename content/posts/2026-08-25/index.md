---
title: "Cutting, Bending, Soldering"
description: "How to organize disk space without losing your mind"
date: 2026-08-25T23:00:00+02:00
math: false
license: "CC BY-NC-SA 4.0"
hidden: false
comments: true
draft: false
tags:
    - PC
    - Linux
categories:
    - Software
---

# Introduction

Three years ago, I bought a computer with a preinstalled operating system commonly known as Windows.

You may ask: why?

Surely it would have been possible to find a machine with an empty drive and save myself the trouble.

Well, I happened to find a computer with what was then the latest Core i5, 64 GB of RAM, a 1 TB NVMe drive, and an NVIDIA RTX 3050 at a price low enough that the correct order of operations was:

1. buy the computer,
2. ask questions later.

And Windows?

It came with the machine.

Might as well keep it. You never know when it might become useful.

After several months, however, *working* with this otherwise magnificent operating system started to resemble an experiment designed to determine the limits of human patience.

I wanted it gone.

Or, at the very least, I wanted its influence over my life significantly reduced.

Using Windows itself, I managed to shrink its partition and install **Ubuntu**, which had long been my default Linux distribution both on PCs and on many ARM64 machines.

Ubuntu received 244 GiB.

In theory, that should be plenty. With a little maintenance, that amount of space can last for a very long time.

The problem was that the rest of the drive was being wasted spectacularly.

Windows later allowed me to reclaim another 115 GiB or so, but no more.

Where was the administrator's control over his own computer?

Where was the sacred right to partition a disk according to one's own unreasonable plans?

This led to a bold decision:

delete the main NTFS partition, reinstall the *hostile operating system* into the smallest sensible amount of space, and use everything else to live dangerously.

For Fedora, which I had never had time for.

For FreeBSD, which I had never had room for.

For Arch Linux, so that I could finally approach random strangers in the street and say:

> I use Arch, btw.

For Slackware, so that I could watch its installer produce a `segmentation fault` and reassure myself that reports of the distribution's death had indeed been greatly exaggerated.

And so on.

That is exactly what I did.

# Backup

First things first: backups.

Important files from NTFS first, then the `ext4` partition as well.

Because you never know.

A question for the brave:

how often do you **not** make backups?

And what happens if the power disappears halfway through repartitioning the drive?

I'm seriously asking.

For a friend.

Who also does not make backups.

The obvious tool is `rsync`:

```bash
rsync -av --info=progress2 /source-directory/ /destination-directory/
```

The destination will probably be an external NTFS drive already containing other important backups.

Converting it to `ext4` right now would merely create another project that nobody asked for.

Run `rsync` as many times as necessary.

Including the Windows user directory.

You never know when something from there might turn out to be useful.

At worst, six months later you discover several dozen gigabytes of Go, Firefox, `pip`, and other caches proving that civilization once existed there.

# A Quick Trip Through Windows

The world has moved on since Windows XP.

You no longer need to keep an installation DVD hidden in a drawer somewhere.

Windows installation media can be downloaded from the Internet, and on machines originally sold with Windows, activation will normally happen automatically using the OEM key stored in UEFI firmware.

For peace of mind, however, you can still retrieve the key before doing anything destructive:

```powershell
(Get-CimInstance -ClassName SoftwareLicensingService).OA3xOriginalProductKey
```

On Linux, the key stored in the ACPI MSDM table can often be displayed with:

```bash
sudo strings /sys/firmware/acpi/tables/MSDM
```

On my machine, that produced the correct activation key as well.

To make reinstalling Windows slightly less painful, I prepared the USB stick using **Rufus**.

Rufus detects Windows installation images and offers several additional installation options.

After pressing `Start`, I was presented with the **Windows User Experience** dialog.

I selected, among other things:

* no requirement for a Microsoft account,
* creation of a local user with a name of my choosing,
* fewer telemetry and privacy questions,
* disabling automatic BitLocker device encryption.

With the installer prepared, the time finally came to delete some partitions.

Using a proper operating system, naturally.

# What's on the Plate?

`gparted` revealed the following landscape:

| Partition        |           Size | Purpose                                |
| ---------------- | -------------: | -------------------------------------- |
| `/dev/nvme0n1p1` |        300 MiB | **EFI System Partition**               |
| `/dev/nvme0n1p2` |        128 MiB | **Microsoft Reserved Partition (MSR)** |
| `/dev/nvme0n1p3` |     546.37 GiB | **Windows C:**                         |
| unallocated      | **115.86 GiB** | free space                             |
| `/dev/nvme0n1p4` |       1.12 GiB | **Windows Recovery / WinRE**           |
| `/dev/nvme0n1p5` |     244.14 GiB | existing Ubuntu                        |
| unallocated      |       1.05 GiB | a small gap                            |
| `/dev/nvme0n1p6` |      21.07 GiB | **Dell factory recovery image**        |
| `/dev/nvme0n1p7` |       1.49 GiB | **Dell support / diagnostics**         |
| unallocated      |       1.71 MiB | alignment                              |

The plan:

> Delete `nvme0n1p3` and `nvme0n1p4`. Windows Setup can recreate what it needs. Leave `p1`, `p2`, `p5`, `p6`, and `p7` alone.

Snip.

Gone.

# Installing Windows

On Dell laptops, pressing **F12** during boot opens the temporary boot menu, from which the Windows installer USB stick can be selected.

The installer started correctly.

There was just one tiny problem:

> The only visible drive was the USB stick itself.

The solution was simple.

Back into the firmware setup using **F2**, then change the storage-controller mode from **RAID** to **AHCI**.

After rebooting, the NVMe drive appeared in Windows Setup.

This is where a little attention becomes important:

> Do not simply keep clicking *Next*.

Windows is perfectly happy to consume all available space for itself.

I manually created a partition of roughly 150 GiB and installed Windows there.

Partition created.

Windows installed.

# Initial Windows Configuration

The objective here was simple: prevent Windows from quietly reserving unreasonable amounts of disk space for things I did not particularly care about.

First, disable hibernation:

```powershell
powercfg /h off
```

This also removes `hiberfil.sys`.

Next:

```text
sysdm.cpl
```

Then:

**Advanced → Performance → Settings → Advanced → Virtual Memory → Change**

Automatic pagefile management can be disabled and a fixed limit selected.

In my case:

* Initial size: `1024 MB`
* Maximum size: `4096 MB`

This is not universal advice.

My machine has 64 GB of RAM. On systems with significantly less memory, leaving pagefile management to Windows may be the better option.

Next:

**Advanced → Startup and Recovery → Settings → Write debugging information → None**

If full crash dumps are not useful to you, there is little reason to reserve many gigabytes for them.

Then:

```text
SystemPropertiesProtection
```

For `C:`, System Protection can either be disabled or given a small storage limit.

I kept it enabled but limited.

After major updates, the component store can also be cleaned:

```powershell
DISM /Online /Cleanup-Image /AnalyzeComponentStore
DISM /Online /Cleanup-Image /StartComponentCleanup
```

And finally:

**Settings → System → Storage → Temporary Files**

Phew.

Windows had now been successfully confined to its 150-gigabyte room.

It was time to return to the actual experiment.

# The Second Operating System

For the second operating system — from the normal family — I chose **Fedora KDE**.

With every intention of eventually making it my main system.

The `fvwm3` configuration can wait for long autumn evenings, when ordinary life stops providing enough problems.

## Installing Fedora

Fedora boots into a Live environment, which has several advantages.

First, a complete system and disk tools are available before installation.

Second, you can open a browser, play a movie, and enjoy a genuine *sit back and relax* experience.

Another company advertised that concept back in 1995.

It did not go particularly well.

Fedora's installer was also quite enthusiastic about using all available disk space, so I politely declined and prepared the partitions myself.

I allocated 2 GiB of `ext4` for `/boot`.

The root filesystem received roughly 215 GiB of `btrfs`.

It is worth keeping Fedora's usual Btrfs subvolume layout:

* `root` → `/`
* `home` → `/home`

There is no need for a separate `/home` partition.

Both subvolumes share the same Btrfs storage pool, so free space is not artificially divided between the operating system and user files.

The installer then needed three mount points:

* `/boot`,
* `/`,
* `/boot/efi`.

The last one comes with an important rule:

> do **not** format the existing EFI System Partition.

Unless crying and gnashing of teeth happens to be part of the plan.

At this point, tea can be prepared while the installer does its work.

## `UnsupportedPartitioningError`

There was, naturally, one surprise.

The installer refused to use the Btrfs filesystem I had prepared for `/` and stopped with `UnsupportedPartitioningError`.

In my particular case, manually creating the expected subvolumes solved the problem:

```bash
sudo mount /dev/nvme0n1p9 /mnt/btrfs

sudo btrfs subvolume create /mnt/btrfs/root
sudo btrfs subvolume create /mnt/btrfs/home

sudo btrfs subvolume list /mnt/btrfs
```

In other words:

* mount the filesystem,
* create `root`,
* create `home`,
* verify that they actually exist.

The next installation attempt completed successfully.

Fedora installed its own GRUB configuration afterward.

Fortunately, it also discovered the other operating systems and added them to the boot menu.

## First Boot

After the first successful boot, I created the normal administrative user.

Its UID and GID were the usual:

```text
1000:1000
```

Exactly the same as my user account under Ubuntu.

A small detail, but a very convenient one when files are shared between installations.

The root password remained unset.

*Use sudo, Luke.*

# Mounting the NTFS Partition

Once the Windows partition was known, it could be added to `/etc/fstab`.

My first version used the kernel `ntfs3` driver:

```fstab
UUID=NTFS-PART-UUID  /mnt/c  ntfs3  rw,nosuid,nodev,nofail,uid=1000,gid=1000,umask=022,windows_names  0  0
```

Later, however, I found that `ntfs-3g` behaved much better in my particular setup, so that is what I ultimately kept.

# Fonts

With the Windows partition mounted, its fonts can also be reused.

After all, the files are already sitting on a machine whose Windows license came with the hardware.

For manually installed system-wide fonts, a suitable location is:

```bash
sudo mkdir -p /usr/local/share/fonts/microsoft
```

Then:

```bash
sudo cp /mnt/c/Windows/Fonts/*.ttf /usr/local/share/fonts/microsoft/
sudo fc-cache -fv
```

And this is where another adventure began.

`fc-cache` reported no usable fonts.

`fc-scan` failed:

```bash
fc-scan /usr/local/share/fonts/microsoft/arial.ttf
```

while:

```bash
file /usr/local/share/fonts/microsoft/arial.ttf
```

identified the file merely as:

```text
data
```

Something was clearly wrong.

After several experiments, the issue disappeared when I switched the Windows filesystem from the kernel `ntfs3` driver to `ntfs-3g`.

I remounted it, copied the font files again, rebuilt the Fontconfig cache, and this time everything was recognized correctly.

I am **not** claiming that `ntfs3` inherently breaks font files.

I am saying that this was the exact symptom on my machine, and changing the driver fixed it.

That distinction becomes rather important once something is published on the Internet.

I could, of course, have installed one of the packages that retrieves Microsoft's traditional core fonts.

But why add another mechanism for downloading data when the files I actually wanted were already sitting on the neighboring partition?

# Standard Software

After this unexpected NTFS lesson, normal life could finally begin.

First:

```bash
sudo dnf upgrade --refresh
```

Then the basic toolkit:

```bash
sudo dnf install \
    fish rsync curl wget zip unzip ansible git \
    emacs vim firefox gimp inkscape jq yq mc htop tmux \
    gcc gcc-g++ make cmake python3-pip python3-devel \
    podman buildah skopeo
```

At this point the computer finally begins to resemble a workstation.

# Additional Software

## KeePassXC

**KeePassXC** is available directly from the Fedora repositories:

```bash
sudo dnf install keepassxc
```

Years ago, applications like this required considerably more effort.

Civilization occasionally moves forward.

## HeidiSQL

**HeidiSQL** now offers a Linux build and an RPM package.

Installing it is therefore considerably less exotic than it once would have been.

The Linux version uses Qt rather than .NET.

## Opera

Opera provides its own RPM repository.

Import the signing key:

```bash
sudo rpm --import https://rpm.opera.com/rpmrepo.key
```

Add the repository:

```bash
sudo tee /etc/yum.repos.d/opera.repo <<'RPMREPO'
[opera]
name=Opera packages
type=rpm-md
baseurl=https://rpm.opera.com/rpm
gpgcheck=1
gpgkey=https://rpm.opera.com/rpmrepo.key
enabled=1
RPMREPO
```

Then install:

```bash
sudo dnf install opera-stable
```

Not `opera-developer`.

I have no idea why I briefly considered installing that one.

## Visual Studio Code

Following the recommendations of the company whose extremely enthusiastic CEO once ran around a stage shouting:

> Developers! Developers! Developers!

first import _their_ signing key:

```bash
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
```

Then add the repository:

```bash
printf '%s\n' \
'[code]' \
'name=Visual Studio Code' \
'baseurl=https://packages.microsoft.com/yumrepos/vscode' \
'enabled=1' \
'autorefresh=1' \
'type=rpm-md' \
'gpgcheck=1' \
'gpgkey=https://packages.microsoft.com/keys/microsoft.asc' \
| sudo tee /etc/yum.repos.d/vscode.repo >/dev/null
```

And install:

```bash
sudo dnf install code
```

From this point on, a normal:

```bash
sudo dnf upgrade
```

will update VS Code along with everything else.

## VSCodium

The process is similar, except that the packages no longer come from the repository belonging to the company whose enthusiastic CEO...

All right.

Enough.

```bash
printf '%s\n' \
'[gitlab.com_paulcarroty_vscodium_repo]' \
'name=VSCodium' \
'baseurl=https://paulcarroty.gitlab.io/vscodium-deb-rpm-repo/rpms/' \
'enabled=1' \
'gpgcheck=1' \
'repo_gpgcheck=1' \
'gpgkey=https://gitlab.com/paulcarroty/vscodium-deb-rpm-repo/raw/master/pub.gpg' \
'metadata_expire=1h' \
| sudo tee /etc/yum.repos.d/vscodium.repo >/dev/null

sudo dnf install codium
```

## Brave

Right.

Definitely enough about enthusiastic CEOs.

```bash
sudo dnf install dnf-plugins-core

sudo dnf config-manager addrepo \
    --from-repofile=https://brave-browser-rpm-release.s3.brave.com/brave-browser.repo

sudo dnf install brave-browser
```

## Spotify

Spotify comes from Flatpak.

Possibly because the company is Swedish and its CEO, in turn...

No.

Never mind.

```bash
flatpak remote-add --if-not-exists flathub \
    https://dl.flathub.org/repo/flathub.flatpakrepo

flatpak install flathub com.spotify.Client
```

## VirtualBox

First enable RPM Fusion:

```bash
sudo dnf install \
    https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-44.noarch.rpm \
    https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-44.noarch.rpm
```

Then:

```bash
sudo dnf install VirtualBox akmod-VirtualBox kernel-devel
```

To explicitly build the module for the currently running kernel:

```bash
sudo akmods --kernels "$(uname -r)"
```

Future kernel updates should normally be handled automatically by `akmods`.

## A Better-Looking GRUB Menu

Once a computer contains several operating systems, it seems only reasonable to make the screen used to choose between them somewhat pleasant to look at.

Fortunately, there is no need to spend an evening designing one.

There is **Tela**:

```bash
git clone https://github.com/vinceliuice/grub2-themes.git
cd grub2-themes
sudo ./install.sh -t tela -s 1080p
```

And suddenly the boot menu looks as though somebody actually expected a human being to see it.

# Enough for One Day

In a single day, I managed to:

* confine Windows to an amount of disk space I can tolerate,
* preserve Ubuntu,
* add Fedora,
* learn something about Btrfs,
* discover an interesting difference between `ntfs3` and `ntfs-3g`,
* recover the fonts,
* install the basic workstation software,
* and make GRUB look respectable.

Doctors do not recommend more excitement than this in a twenty-four-hour period.

And there is still plenty of unallocated disk space left.

FreeBSD is waiting.

So is Arch.

Slackware, one hopes, is still waiting too.

So there will be more.

Probably sooner rather than later.
