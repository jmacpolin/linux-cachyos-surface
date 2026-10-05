# linux-cachyos-surface-7.1: the tested 7.1 kernel as a fallback

The `7.1` and `7.2` branches both build a package named `linux-cachyos-surface`, so installing the 7.2 build removes the 7.1 kernel. This PKGBUILD takes the 7.1.3 packages that were tested on a Surface Pro 9 and installs the same kernel under the name `linux-cachyos-surface-7.1`, next to the newer build. It gets its own `/boot/vmlinuz-linux-cachyos-surface-7.1`, its own initramfs, and its own boot entry. Nothing is recompiled.

## Use

1. Find the original packages from the 7.1 build. `makepkg` leaves them in the package directory:

   ```bash
   ls -l ~/linux-cachyos-surface/linux-cachyos-surface/*7.1.3*.pkg.tar.zst
   ```

   If they've moved, `sudo find / -xdev -name '*cachyos-surface*7.1.3*.pkg.tar.zst'` will find them.

2. Copy them into this directory and build and install the renamed packages:

   ```bash
   cp ~/linux-cachyos-surface/linux-cachyos-surface/linux-cachyos-surface-7.1.3-1-x86_64.pkg.tar.zst \
      ~/linux-cachyos-surface/linux-cachyos-surface/linux-cachyos-surface-headers-7.1.3-1-x86_64.pkg.tar.zst \
      ~/linux-cachyos-surface/fallback/linux-cachyos-surface-7.1/
   cd ~/linux-cachyos-surface/fallback/linux-cachyos-surface-7.1
   makepkg -si
   ```

   `makepkg` checks both files against the sha256 sums of the tested build, so it refuses any other 7.1 build.

3. Check that both kernels are installed and have boot images:

   ```bash
   pacman -Q | grep cachyos-surface      # linux-cachyos-surface 7.2.x and linux-cachyos-surface-7.1 7.1.3-1
   ls /boot                              # vmlinuz-linux-cachyos-surface-7.1 and its initramfs
   ```

   On CachyOS the bootloader hook (Limine, systemd-boot or GRUB) adds the boot entry automatically. If it doesn't appear, regenerate your bootloader's entries.

Booted into it, `uname -r` shows `7.1.3-1-cachyos-surface`: the kernel release is unchanged, only the package name differs.

To remove it later, run `sudo pacman -R linux-cachyos-surface-7.1 linux-cachyos-surface-7.1-headers`.

If the original packages are gone, rebuild from the `7.1` branch instead. That branch now builds a package with the same `linux-cachyos-surface-7.1` name, so it also installs alongside the main package.
