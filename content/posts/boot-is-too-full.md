+++
title = 'Fix "Partition /boot is too full" Error on Linux'
date = 2024-05-22T22:53:35-04:00
draft = false
+++

## Saving Space on Your /boot Partition

I recently ran into trouble because I followed the old advice of allocating only ~200MB for the `/boot` partition. While this might have been a reasonable choice a decade ago, when files were smaller and disk space was more expensive, it’s no longer sufficient for modern systems. On top of that, I’m dual-booting with Windows, which further complicates disk space management.

Anyway, that's the situation I faced. The next question was: how do I fix this? After searching online, I found several recommendations that suggested deleting `initramfs-linux-fallback.img` to free up space. While it’s a large file and might seem unnecessary, it’s not advisable to remove it without understanding its purpose.

## Why You Shouldn't Delete the Fallback Image

The `initramfs-linux.img` file, which is the default boot image, is critical for booting your computer. It contains the necessary kernel modules that tell the OS how to interact with your hardware based on your current setup. In most cases, this file is enough to boot your system.

But what happens when you make hardware changes (e.g., adding new RAM)? Since the `initramfs-linux.img` only contains the modules needed for the hardware present when it was generated, it might not have the correct modules for your new hardware. This is where the `initramfs-linux-fallback.img` comes in. The fallback image contains all the kernel modules that Linux supports, allowing the system to handle new or unrecognized hardware. If the fallback image is deleted and your system requires a module not found in the standard `initramfs`, it may fail to recognize new hardware.

Thus, deleting the fallback image could leave you in a difficult situation if your system relies on modules that are only available in that file.

## Compress Instead of Delete

Rather than deleting the fallback image, a better solution is to compress it to save space. The `mkinitcpio` tool makes this easy to do. Here’s how:

### 1. Edit `/etc/mkinitcpio.conf`

By default, the `mkinitcpio.conf` file already includes a compression setting. To reduce the size of your initramfs images, you can increase the compression level in the `COMPRESSION_OPTIONS` section. Most compression tools (like `gzip` and `xz`) support compression levels from 0 (least compressed) to 9 (most compressed). Check the manual page of the tool you select for the best options. For example, `gzip` is commonly used, but if you're really pressed for space, `xz` offers a higher compression ratio.

### 2. Use High Compression if Necessary

If your `/boot` partition is very limited, you can try `xz` with the `-9e` option for maximum compression. Keep in mind that this is a very slow process — on my laptop, it took over five minutes. If you don't need such extreme compression, you can start with a lower level like `-6` and adjust upward as needed. Be aware that higher compression levels may increase the time and CPU required to generate initramfs images.

### 3. Consult the Arch Linux Wiki

For more detailed documentation on different configuration options, I recommend checking out the [Arch Linux Wiki](https://wiki.archlinux.org/title/Mkinitcpio). It’s a valuable resource even if you’re using a different Linux distribution.

### 4. Test Your Config

After making your changes, run `sudo mkinitcpio -P` to rebuild your initramfs images. This command will regenerate the images for all installed kernels. Make sure the command runs without errors and use `du` to check the compressed size of the files to ensure they are small enough for your `/boot` partition.

### 5. Reboot Carefully

Once you're satisfied with the size of your images, you can reboot your system. I recommend performing a reboot to ensure everything works as expected. Be cautious when making these changes because a corrupted `initramfs-linux.img` can prevent your system from booting. It’s a good idea to have a backup or recovery method in place (such as a live USB) just in case something goes wrong.

Good luck!
