---
layout: post
title:  "Installing refind with Windows 10"
listed: false
---

In this blog post I'll be covering the process of installing and troubleshooting [**refind**](https://www.rodsbooks.com/refind/) bootloader from a machine running Windows 10 1903 (although I hope this will survive any updates from Microsoft).

![Screenshot of our desired result](images/refind_screenshot.jpg)
_Our desired result_

### What our goals are
 - Install and theme refind from a Windows 10 machine
 - Make refind the default bootloader
 - Prevent Windows from setting itself as the bootloader after every boot

### What our goals aren't
 - Conver refind's installation from any other Operating System (if you're not on Windows 10 and wants to install refind, take a look [here](https://www.rodsbooks.com/refind/installing.html))
 - Make Secure Boot happy (I may or may not make a blog post about this in the future. If you want to use Secure Boot - which I recommend -, you can start by taking a look at [the official documentation](https://www.rodsbooks.com/refind/secureboot.html) on this topic)

### What you'll need
 - A Windows 10 bootable USB stick (or DVD if you live in a cave)
 - [refind](https://www.rodsbooks.com/refind/getting.html) (duh)
 - [Explorer++](https://github.com/derceg/explorerplusplus) or some other file explorer that can be launched as Administrator (I'll assume that you have one, but it is not required at all. If you feel confortable, you can do all the file moving/editing from your command prompt)
 <br>

### Getting started
First, we'll need access to the Windows' FAT32 EFI partition. In order to do that, we'll have to mount it.

From an Administrator command prompt, issue the following command (where B: is the letter you want to assign to the partition):

{% highlight sh %}
mountvol B: /s
{% endhighlight %}

After this, we should have our partition mounted at `B:`. In order to access it, you'll have to launch Explorer++ as Administrator (see the note¹ below). If you open it up you should see the following structure:

{% highlight markdown %}
.
├── EFI
|   ├── Boot
|   |   └─── bootx64.efi
|   ├── Microsoft
|   |   ├─── Boot
|   |   └─── Recovery
{% endhighlight %}

**NOTE¹**: The old trick of killing `explorer.exe` and launching it from an elevated command prompt doesn't seem to be working anymore. The B: drive shows up but is not accessible. I prefer not to mess around with this drive's permissions, so I'm using this alternative method.