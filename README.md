2 Ansible playbooks for getting a complete installation of Arch Linux directly from the live ISO, without rebooting, and with hooks to run personal user scripts.

## Why not archinstall?

archinstall is a way more comprehensive script compared to these playbooks, with more out-of-the-box configuration options. At the same time it is less configurable when it comes to user's own scripts to get the exact system you want. If you are not settled on the exact system you want, or the [limitations](#limitations) of this project are affecting you - use archinstall, if you already know what you need and are looking for a fast, reproducible, and scriptable install that you can quickly fix and re-run if something fails - try these playbooks.

## How it works

1. `install.yml` runs against the live ISO. It only does what is needed to reach phase 2 or is otherwise possible to do only on the live ISO: partitions, filesystems, mounts, a minimal pacstrap (only base, python, and openssh), fstab, and starting a new instance of sshd on port 2222 running inside the new system (via arch-chroot), with a temporary root password.

2. `postinstall.yml` connects to that sshd, so Ansible works inside the new system as if it was already booted. This is where the actual installation happens.

### Order of steps in postinstall.yml

0. Prompt for the user password to be set
1. Install core_packages (kernel, firmware, microcode, sudo, GPU drivers, ...)
2. Timezone, locale, console keymap, hostname, /etc/hosts
3. Install systemd-boot, create the boot entry, enable systemd-boot-update.service
4. Run user-defined `Pre-install script` as root
5. Full system upgrade before installing all packages
6. Install everything from `packages.txt`
7. Create the user, and allow wheel to use sudo (if the user is in wheel)
8. Run user-defined `Post-install script` as your user, with passwordless sudo only while it runs
9. Enable / disable / mask services
10. Rebuild the initramfs (mkinitcpio -P)
11. Set the root password (last, see "Re-running")

### Hook scripts

Everything specific to you goes into two optional scripts in `playbooks/files/`:

- `pre-install.sh` runs as root, before all the packages are installed. This is a place where you can add repositories (e.g. CachyOS or Chaotic AUR) or apply some changes to /etc/pacman.conf, or add some pacman hooks, etc.

- `post-install.sh` runs as your user after all packages are installed. It has passwordless sudo for its duration only (removed afterwards, even if the script fails). Use it for cloning your dotfiles, installing AUR packages, applying themes, anything that needs installed packages or belongs in $HOME.

The rules for both scripts:
- They are copied from your machine and run on the target. They need a shebang, e.g. `#!/usr/bin/env bash`.
- A non-zero exit code stops the playbook, and the script's `stderr` is displayed.
- `stdout` is not visible during the run, `stdin` is not your TTY, so don't use interactive elements.
- They run in full on every re-run, so write them to be re-run-safe (e.g. skip `git clone` if the directory exists).
- They don't run in a login shell. The post-install script gets Arch's Perl directories added to PATH (some AUR builds need them).

### Re-running

Re-running after a failure is safe, most steps check the current state first and finished steps report ok, the hook scripts run every time.

The root password is set last, so after a failed run the temporary password still works and you can simply run `postinstall.yml` again. After a successful complete run, root has your real password and Ansible can't connect anymore.

### Requirements

- Your machine (the controller): Ansible with the `community.general` and `ansible.posix` collections, `sshpass` and `python-passlib`. On Arch:
  ```sh
  sudo pacman -S ansible sshpass python-passlib
  ```
- The target: a UEFI x86_64 machine, booted from the Arch ISO.

### Prepare the live ISO

1. Connect to the internet (e.g. iwctl for Wi-Fi).
2. Allow root to log in over SSH without a password (the ISO's root has an empty password):
   ```sh
   echo -e 'PermitRootLogin yes\nPermitEmptyPasswords yes' >> /etc/ssh/sshd_config
   systemctl restart sshd
   ```
3. Note the IP address: `ip a`.
4. Prepare the disk:
    - Empty disk: nothing to do. The playbook creates the partitions from the `filesystems` variable.
    - Existing partitions: the playbook only formats the partitions listed in `filesystems`.
      - The ESP must have the type EFI System.
      - A partition with an old filesystem signature is not reformatted. Clear it with `wipefs -a /dev/<partition>`.

**Warning:** every partition listed in `filesystems` is formatted.

### Run the playbooks

Set the ISO's IP in `inventory/hosts.yml`, then set the variables in both playbooks (see Configuration). After that run both playbooks in order:
```sh
ansible-playbook playbooks/install.yml
ansible-playbook playbooks/postinstall.yml
```

After the install stop the chroot sshd, unmount, and reboot:
```sh
pkill -f 'sshd -D -p 2222'
umount -R /mnt
reboot
```

## Configuration

```
ansible.cfg
inventory/hosts.yml          # ISO IP address, temporary root password
playbooks/
├── install.yml              # phase 1: disk partitions and filesystems
├── postinstall.yml          # phase 2: the rest of installation configuration
└── files/
    ├── packages.txt         # optional packages to install
    ├── pre-install.sh       # optional hook, runs as root
    └── post-install.sh      # optional hook, runs as your user
```

**Never remove a variable.** If you want to skip some optional steps - leave its variable's value empty. These are the required variables that cannot be empty, everything else optional: `ansible_host`, `tmp_root_password`, `disk`, `filesystems`, `timezone`, `locale`, `lang`, `console_keymap`, `hostname`, `username`

### inventory/hosts.yml

| variable | meaning |
|---|---|
| ansible_host (both the live ISO and the chrooted system) | the ISO's IP address |
| tmp_root_password | temporary root password of the new system during the install |

### install.yml

| variable | meaning |
|---|---|
| disk | the target disk, e.g. `/dev/nvme0n1` |
| filesystems | one entry per partition, in disk order |

Each filesystems entry:

| key | meaning |
|---|---|
| dev | partition device, e.g. `/dev/nvme0n1p1` |
| fstype | e.g. `vfat`, `ext4`, `xfs` |
| opts | optional mkfs options, e.g. `-F32` |
| mount | mount point including `/mnt`, e.g. `/mnt`, `/mnt/boot`, `/mnt/home` |
| size | size in GiB (used only when partitioning an empty disk) |
| flags | optional, `[esp]` for the EFI system partition |

The ESP must be the vfat entry with the `esp` flag, mounted at /mnt/boot.

### postinstall.yml

| variable | meaning |
|---|---|
| timezone | e.g. `Europe/Berlin` |
| locale | locale to generate, e.g. `en_US.UTF-8` |
| lang | system language (LANG) |
| console_keymap | console keyboard layout, e.g. `us`, `de-latin1` |
| hostname | the machine's name |
| username | your user |
| user_groups | extra groups; include `wheel` to get sudo |
| user_shell | login shell, e.g. `/usr/bin/zsh` (must be installed by `core_packages` or `packages.txt`); empty = bash |
| packages | package list file in `playbooks/files/`; empty = skip |
| pacman_upgrade_flags | extra pacman options for the system upgrade |
| pacman_install_flags | extra pacman options for installing `packages.txt` |
| pre_install_script / post_install_script | hook scripts, e.g. `files/pre-install.sh`; empty = skip |
| kernel_params | kernel command line after `root=...`, e.g. `rw quiet` |
| core_packages | packages installed first; may contain `gpu_*` names |
| gpu_packages | the packages behind each `gpu_*` name |
| services_enabled / services_disabled / services_masked | systemd units to enable / disable / mask |

`packages.txt` must have one package per line. Empty lines and lines starting with # are ignored. If some package names do not exist, the play fails (fix the list and run again).

## Limitations

- UEFI only, systemd-boot only.
- The linux kernel only (no linux-lts or linux-zen).
- The ESP must be mounted at /boot.
- Plain filesystems only: no btrfs subvolumes, LUKS, LVM or swap.
- Automatic partitioning only on a disk with no partitions at all. Otherwise partition manually.
- Root and the user share the same password.
- Phase 2 runs on the live ISO's kernel, so runtime changes (rfkill, sysctl, modprobe) have no effect.

### Security notes

- `tmp_root_password` is stored in plain text in the inventory. It only matters during the install. Don't reuse it for the main password.
- During the install, the live ISO sshd accepts passwordless root logins (stopped after the `install.yml` play) and the chroot sshd accepts root logins on port 2222 with a password stored in plain sight. Don't run the install on an untrusted network.