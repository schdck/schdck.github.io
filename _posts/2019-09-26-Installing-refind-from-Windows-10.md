---
layout: post
title:  "Installing refind from Windows 10"
listed: true
---

In this blog post I'll be covering the process of installing and troubleshooting [**refind**](https://www.rodsbooks.com/refind/) bootloader from a machine running Windows 10 1903 (although I hope this will survive any updates from Microsoft).

At first, I tried following the [official documentation](https://www.rodsbooks.com/refind/installing.html#windows) and replacing the `{bootmgr}`'s path with refind's, but it didn't work. Then, I used `BCDEDIT.exe` to modify every single path to `bootmgfw.efi` I cound find but to no effect. After every reboot, Windows kept showing up no matter what.

So, I'll be describing here the steps I had to take in order to make everything work as expected, and some guide to troubleshoot any error that you stumble on. Here's a screenshot of the final result:

![Screenshot of the end result](images/refind_screenshot.jpg)

### What our goals are
 - Install and theme refind from a Windows 10 machine
 - Make refind the default bootloader
 - Prevent Windows from setting itself as the bootloader after every boot

### What our goals aren't
 - Cover the details of installing refind from any other Operating System (if you're not on Windows 10 and wants to install refind, take a look [here](https://www.rodsbooks.com/refind/installing.html))
 - Make Secure Boot happy (I may or may not make a blog post about this in the future. If you want to use Secure Boot - which I recommend -, you can start by taking a look at [the official documentation](https://www.rodsbooks.com/refind/secureboot.html) on this topic)

### What you'll need
 - [refind](https://www.rodsbooks.com/refind/getting.html) (duh)
 - [Explorer++](https://github.com/derceg/explorerplusplus) or some other file explorer that can be launched as Administrator (I'll assume that you have one, but it is not required at all. If you feel confortable, you can do all the file moving/editing from your command prompt)
 - A Windows 10 bootable USB stick (or DVD if you live in a cave)¹

¹**NOTE**: The Windows 10 bootable is not required, but is recommended in case you mess things up.

### Before we get started

**UPDATE:** Before you proceed, I recommend trying the steps on the official documentation and using this *only as last resort*. If you do have to follow this path, I recommend you **backup your refind installation folder** after the installation is complete. When you are installing some OSes (e.g. Linux), they will overwrite refind's files and prevent you from booting into refind (but booting into the fresh installed OS will work). If this happens, all you have to do is login into the OS you just installed and restore your refind backup.

First of all, go ahead and disable UEFI's Secure Boot. I won't go into details on how to do it, since it changes from system to system. If you don't know how to do it just search around. Pay attention not to disable UEFI, just Secure Boot.

### Getting started
In ordet to get started, we'll need access to the Windows' FAT32 EFI partition. To do that, we'll have to mount it. From an elevated command prompt, issue the following command (where `B:` is the letter you want to assign to the partition):

{% highlight PowerShell %}
mountvol B: /s
{% endhighlight %}

After this, you should have your partition mounted at `B:` (or whatever letter you specified). In order to access it, you'll have to launch Explorer++ as Administrator (see the note below). If you open it up you should see the following structure:

{% highlight markdown %}
.
├── EFI
|   ├── Boot
|   |   └─── bootx64.efi
|   ├── Microsoft
|   |   ├─── Boot
|   |   └─── Recovery
{% endhighlight %}

**NOTE**: The old trick of killing `explorer.exe` and launching it from an elevated command prompt doesn't seem to be working anymore. The `B:` drive shows up but is not accessible. I prefer not to mess around with this drive's permissions, so I'm using this alternative method.

#### Microsoft Windows' Boot Environment

Microsoft's folder intuitively contains files used to boot Windows. I'll summarize what I think is important about the boot process, but if you want to know more, I suggest you start [here](https://uefi.org/sites/default/files/resources/UEFI-Plugfest-WindowsBootEnvironment.pdf) (this link is where I'm getting all the info below).

**NOTE**: I'm not an expert, so I may say something that is not 100% accurate. If you find something that you think is incorrect, please drop me a line.

With just Windows installed, this is what the boot flow looks like:

![Windows' boot flow](images/refind_windows_boot_flow.png)

 - **UEFI Firmware**: Performs CPU and Chipset initialization, load drivers, etc.

 - **UEFI Boot Manager**: Loads UEFI device drivers and loads boot application

 - **Windows Boot Manager** (`EFI\Microsoft\Boot\bootmgfw.efi`):  Is responsible for loading the Windows Loader (`C:\Windows\System32\winload.efi`) chosen by the user (in case there's more than one Windows installed).

#### Replacing Windows Boot Manager

Our goal here is to replace Windows Boot Manager with refind (but, of course, allowing refind to call the Windows Boot Manager later if that's what the user wants).

In order to do that, we'll use the EFI fallback file. Here's a quick definition, but if you want to know more, take a look [here](https://www.happyassassin.net/2014/01/25/uefi-boot-how-does-that-actually-work-then/) and [here](https://www.rodsbooks.com/refind/installing.html#naming).

> The firmware will look through each EFI system partition on the disk in the order they exist on the disk. Within the ESP, it will look for a file with a specific name and location. On an x86-64 PC, it will look for the file \EFI\BOOT\BOOTx64.EFI.

So... let's recap: we know that the firmware looks for Windows Boot Manager on `EFI\Microsoft\Boot\bootmgfw.efi` and we know that, if it doesn't find it, it'll look for whatever file is at `EFI\Boot\bootx64.efi`. Hmm...

You probably realized where we are going here, but in case you dind't: our plan is to set refind as the EFI fallback file and move Windows Boot Loader to some place where the firmware can't find it. Then, we can create an entry in refind pointing to the place where we moved Windows' files.

This is actually quite simple, but will require a few hacks. Lets begin.

#### Installing refind
First of all, rename the existing `EFI\Boot` folder to something like `EFI\Boot.old`, so that you have a backup in case you need it. With this out of the way, create a new `EFI\Boot` folder and copy refind's files there. Rename `refind_x64.efi` to `bootx64.efi`.

One more thing, add the following entry to your `refind.conf` file (this is important, trust me), replacing `{folder}` to anything **BUT** `Microsoft`, I'll use `_Microsoft`:

```
menuentry Windows {
    loader \EFI\{folder}\Boot\bootmgfw.efi
    icon \EFI\Boot\themes\minimal\icons\os_win8.png
}
```

After this step, refind is properly set up as the EFI fallback. All we have to do now is make Windows Boot Loader unfindable by the UEFI firmware.

#### Renaming the Microsoft folder
This should'n need a dedicated section, but is not as straightforward as it seems. The `EFI\Windows` folder cannot be renamed from Windows (since it is kept in use).

So, you'll need to get a command prompt at boot. In order to do that I usually use the Windows bootable (just press `Shift + F10` on menu), but [there are other ways to do it](https://winaero.com/blog/open-command-prompt-boot-windows-10/) in case you didn't set up your bootable media.

At the boot command prompt, mount the EFI partition and navigate to the EFI folder:

{% highlight PowerShell %}
mountvol B: /s
cd /d B:\EFI
{% endhighlight %}

At this point, if you explore this folder, you should have your `Boot` folder with refind's files and `Microsoft` folder with Windows' ones. All you have to do is rename the Microsoft folder to the same name you used on your refind's `menuentry`.

⚠️Remember to replace `{folder}` by the **exact** same thing you used in your `refind.conf` file.

{% highlight PowerShell %}
ren Microsoft {folder}
{% endhighlight %}

And that's it. Close the command prompt and restart your computer. Next time you boot your firmware should not find Windows Boot Manager and fallback to refind. Since we manually added the Windows entry to refind, you should be able to boot into Windows from there.

<br><br>
### Troubleshooting
#### I messed things up and now I can't boot my computer
Keep calm. This probably happened to me a few dozen times in the process of trying to install refind.

All you have to do is fix your EFI partition. I usually just format the partition and ask Windows to rebuild it, this way you should have a fresh start. Note that this will only work if Windows is the only operating system installed in this partition.

To format and rebuild the partition, boot into your Windows 10 media (I hope you have one), launch a command prompt and execute the following:

{% highlight PowerShell %}
mountvol B: /s
format B: /FS:FAT32
bcdboot C:\Windows /s B: /f UEFI
{% endhighlight %}

This should rebuild your EFI partition and you _should_ be able to boot into Windows now.

#### Refind shows up without any option (hangs)
This happened to me. In my case it was an issue with the NTFS driver, so all I had to do was remove the `ntfs_x64.efi` file from the `EFI\Boot\drivers_x64` folder.

**NOTE**: You don't need the NTFS driver to boot into Windows, since the EFI partition is using FAT. For more info, take a look [here](https://www.rodsbooks.com/refind/drivers.html)

If it isn't a drive issue for you, see if any of the below helps. This is a quote from refind's author Roderick W. Smith:

> This sort of problem normally indicates a filesystem issue -- rEFInd is getting stuck in an infinite loop attempting to read one of the filesystems on the disk. I recommend you try the following:<br><br>
>  **1.** If you have any external disks attached to the computer, try
   unplugging them. Such disks sometimes cause problems. If this fixes the problem but you need to use the external disk on a regular basis, you may need to further debug the problem as below; but if it's just a USB flash drive that you don't need to leave permanently attached, then this should be the end of it.<br>
>  **2.** Remove (or move) all the EFI filesystem drivers from the "drivers", "drivers_x64", and "drivers_ia32" subdirectories of the rEFInd installation directory. (Normally there'll be only one drivers subdirectory; just remove all those driver files.) Reboot.
>  **3.** If rEFInd hangs even with no EFI filesystem drivers installed, then the problem is likely with the ESP or some other FAT filesystem. You can try doing a filesystem check on such partition(s) with dosfsck in Linux, CHKDSK in Windows, or similar utilities. In an extreme case, you could try backing up the ESP (and/or other FAT partitions), creating a fresh filesystem, and restoring the data.<br>
>   **4.** If rEFInd comes up and shows a menu after removing the filesystem drivers, even if the menu options are incomplete, then this is good; it indicates that there's a problem with one of the filesystems or their drivers. You can then begin restoring the driver(s) that you need. Normally this will be just one, for whatever filesystem holds your Linux kernels. (You do NOT need the NTFS driver to boot Windows.)<br>
>   **5.** If rEFInd works at this point, then you're done -- the problem was in some superfluous driver(s) that you weren't using.
>   **6.** If rEFInd hangs, then you can try booting in some other way and using a filesystem check utility like fsck on the filesystem(s) that can be read with the driver(s) you restored. With any luck this will fix the problem.<br>
>   **7.** If the problem persists at this point, you have a number of options:<br>
>      **a)** Remove the offending filesystem driver, install GRUB, and use it to boot the Linux kernel. GRUB uses a different filesystem driver and so may not be affected.<br>
>      **b)** If you don't already use one, create a small (~1GiB) partition to use as /boot, using a different filesystem than your main Linux partition. You can then install the EFI driver for the /boot partition, copy the contents of /boot to it, and adjust /etc/fstab to mount this partition at /boot. The ext4fs, Btrfs, and ReiserFS EFI drivers are the fastest ones.<br>
>      **c)** Do as in option b, but use FAT, which requires no special driver. This works better with some distributions than others, though. Debian-based distributions often use symbolic links, which aren't supported on FAT, in /boot, for instance. Some Arch Linux users like to use FAT on /boot (or mount the ESP at /boot), by contrast.<br>
>      **d)** Completely wipe and re-install the offending partition(s). It's possible that the EFI driver is flaking out because of leftover data that wasn't completely erased when you created the partition. If you opt to do this, I recommend completely zeroing the partition (with "dd if=/dev/zero of=/dev/{wherever}" or something similar) before restoring it.<br>
>      **e)** Try a different EFI filesystem driver. The efifs project (http://efi.akeo.ie/) offers a large number of EFI filesystem drivers, and its variant of the rEFInd driver might work better on your system.<br><br>
> There may be some other options and potential causes I'm forgetting.
