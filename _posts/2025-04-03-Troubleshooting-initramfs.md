---
layout: post
title: "Troubleshooting and modifing initramfs"
category: posts
---

The initramfs (initial RAM filesystem) plays a critical role in the boot process of Rocky Linux. In this article, we’ll walk through what initramfs is, how it’s built with dracut, and the steps you can take to troubleshoot or customize it. Rocky 9.5 distro was used in my case.

## Understanding initramfs

   **What is initramfs?**  
   The initramfs is an initial root filesystem that is loaded into RAM by the bootloader (usually GRUB) and then by the Linux kernel before the real root filesystem is mounted. This temporary filesystem provides the tools needed to mount the final filesystem and start the real system.  
   *Correction:* Although GRUB loads the initramfs image into memory, it is the kernel that mounts and uses it during the early boot process.

## Building the initramfs with dracut

1. **How is the initramfs built?**  
   In Rocky Linux 9, dracut is used to create the initramfs image. The resulting file is typically saved in the `/boot` directory along with your kernel images.

2. **The dracut command:**  
   You can force dracut to rebuild the initramfs and include specific modules with the following command:  
   ```bash
   dracut --force --add 'crypt lvm multipath'
   ```  
   This command rebuilds the initramfs image, ensuring that modules for disk encryption (`crypt`), logical volume management (`lvm`), and multipath support (`multipath`) are included.  
   *Note:* Some users prefer to add modules individually (e.g., `--add crypt --add lvm --add multipath`), but the combined form works as long as the module names are separated by spaces.

## Exploring the initramfs Contents

1.   **Listing the contents:**  
   You can inspect the contents of an initramfs image using the `lsinitrd` command. For example:  
   ```bash
   lsinitrd /boot/initramfs-<kernel-version>.img
   ```  
   **Additional Tip:** You can also run `lsinitrd` without specifying the path to an image file. In such cases, it will display the contents of the initramfs for the currently running kernel.

2.   **Unpacking the initramfs:**  
   If you need to modify the contents, you can unpack the initramfs into your current working directory with:  
   ```bash
   lsinitrd --unpack /boot/initramfs-<kernel-version>.img
   ```  
   This extracts the filesystem so you can browse or modify its files.

## Dracut Modules and Customization

1.   **Location of dracut modules:**  
   Dracut modules can be found in the directory:  
   ```bash
   /usr/lib/dracut/modules.d
   ```  
   Here you will see various modules that dracut uses to build the initramfs image.

2.   **Modifying dracut behavior with systemd vs. non-systemd:**  
   When customizing or modifying dracut modules on a system that uses systemd, you should modify the corresponding `.service` files rather than `.sh` scripts. The shell scripts might not be copied into the final initramfs if systemd units are available.

## Entering the initramfs Environment

   **Breaking into the initramfs shell:**  
   During boot, you may need to troubleshoot by entering an emergency shell within the initramfs. This can be done by adding a kernel parameter. For example:  
   - Use `rd.break` to immediately break into the initramfs shell.  
   - More granular control is available using parameters such as `rd.break=pre-mount` to stop the boot process at a specific phase.

## A Word of Caution on Console Parameters

   **Console visibility warning:**  
   When passing kernel parameters, be cautious with console settings. For instance, having a parameter like `console=ttyS0` may result in an invisible emergency shell (where a blinking cursor appears but no prompt is visible).  

Remember, always keep backups of your original initramfs and configuration files before making any changes. Happy troubleshooting!

