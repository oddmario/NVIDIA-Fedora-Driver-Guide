# NVIDIA Fedora Driver Guide
A little guide to help you install & manage the NVIDIA GPU driver on your Fedora system(s)

I am personally an **Fedora 40** user at the moment, so this is mostly what this guide applies to (though I believe it should work alright on newer releases, and also on older releases which are not old very old `[something like Fedora 38+]`)

## Index of content
- [Driver installation](#driver-installation)
- [Driver uninstallation](#driver-uninstallation)
- [Issues faced after installing the NVIDIA drivers, and how to solve them](#issues-faced-after-installing-the-nvidia-drivers-and-how-to-solve-them)
  * [Fix VLC black screen issue](#getting-a-black-screen-on-video-players-vlc-etc)
  * [Wayland is no longer enabled/not visible on the login screen](#wayland-is-not-shown-as-an-option-on-the-login-screen-or-the-cog-icon-of-the-login-screen-doesnt-show-at-all)
  * [Fix Wayland issues (flickering, etc.)](#the-experience-on-wayland-is-not-the-smoothest-fix-wayland-issues)
- [References](#references)

-----

## Driver installation

1. Follow the following sections in the same order:
   - https://rpmfusion.org/Howto/NVIDIA#Latest.2FBeta_driver for the beta driver, or https://rpmfusion.org/Howto/NVIDIA#Current_GeForce.2FQuadro.2FTesla for the stable driver.
   - https://rpmfusion.org/Howto/NVIDIA#VDPAU.2FVAAPI
   - https://rpmfusion.org/Howto/NVIDIA#Vulkan
   - https://rpmfusion.org/Howto/NVIDIA#NVENC.2FNVDEC
     
2. Run `sudo dracut --force` to update the initramfs
3. Reboot the system to finalise the driver installation and configuration

-----

## Driver uninstallation

1. Follow https://rpmfusion.org/Howto/NVIDIA#Uninstall_the_NVIDIA_driver
2. Reboot the system

-----

## Issues faced after installing the NVIDIA drivers, and how to solve them

### Getting a black screen on video players (VLC, etc)

This may happen because most of the video/media players require openh264 which is not shipped by default with Fedora.

Install `ffmpeg` and `libvacodec` using `sudo dnf install ffmpeg libavcodec-freeworld --allowerasing` as long as you have the rpmfusion-free repo enabled.

This will install openh264 as well and shall fix the problem.

### Wayland is not shown as an option on the login screen (or the cog icon of the login screen doesn't show at all)

1. Edit the `/etc/gdm/custom.conf` file using `sudo nano /etc/gdm/custom.conf`
2. Ensure that `WaylandEnable=true` is set in that file and make sure that it's uncommented (does not start with a `#`)
3. Run `sudo ln -s /dev/null /etc/udev/rules.d/61-gdm.rules`
4. Reboot the system

### The experience on Wayland is not the smoothest (fix Wayland issues)

This may happen for a lot of reasons. For a while now, NVIDIA has been known to have issues with the Wayland windowing system. However, NVIDIA has been working on making this better.
And this has actually already gotten much better starting from the NVIDIA driver 555.42.02 which added [explicit sync](https://9to5linux.com/developer-explains-why-explicit-sync-will-finally-solve-the-nvidia-wayland-issues) support.

So first of all, make sure to have:
- Version 555.42.02 or a higher version of the Nvidia driver
- GNOME 46.1 or a higher version on your Fedora installation

#### Pay attention please
The packages shipped by the RPMFusion repository should already handle configuring everything for you to ensure the smoothest experience possible (for both Xorg and Wayland). However, if you are still experiencing issues, continue reading below:

* Your system may be using the Mesa driver instead of the NVIDIA one on Wayland sessions. You can confirm this by typing `glxinfo|egrep "OpenGL vendor|OpenGL renderer*"`

  In order to solve this:
     
  1. Edit `/etc/default/grub` using `sudo nano /etc/default/grub`
  2. Add `nvidia-drm.modeset=1` and `nvidia-drm.fbdev=1` inside your `GRUB_CMDLINE_LINUX` (i.e. `GRUB_CMDLINE_LINUX="nvidia-drm.modeset=1 nvidia-drm.fbdev=1"`)
  3. Run `sudo grub2-mkconfig -o /etc/grub2.cfg`
  4. Reboot the system
 
* You may have the GSP firmware of Nvidia enabled, and this is known to cause some performance issues on the beta 555.42.02 version of the driver. Maybe this will be fixed in the future, but for now, we can disable the GSP firmware if needed.

  You can check whether the GSP firmware is enabled or no by typing `nvidia-smi -q | grep "GSP Firmware"` — if it says `N/A` then the firmware is not enabled. If otherwise (it shows a version for GSP firmware) then the firmware is enabled.

  To disable the GSP firmware, please follow the below steps:

  1. Edit `/etc/default/grub` using `sudo nano /etc/default/grub`
  2. Add `nvidia.NVreg_EnableGpuFirmware=0` inside your `GRUB_CMDLINE_LINUX`
  3. Run `sudo grub2-mkconfig -o /etc/grub2.cfg`
  4. Reboot the system

  See https://forums.developer.nvidia.com/t/major-kde-plasma-desktop-frameskip-lag-issues-on-driver-555/293606 for more information on this issue.
   
* for Google Chrome (and Chromium-based browsers in general), you may need to switch the "Preferred Ozone platform" flag to "Wayland" or "auto" (this is not needed if you have both GNOME 46.1+ and Nvidia driver 555.42.02+). Follow the steps below in order to apply this:
  1. Go to chrome://flags
  2. Search "Preferred Ozone platform"
  3. Set the flag to "Wayland" or "auto"
  4. Restart the browser
* for some Electron apps, you may need to pass the same Ozone platform flag as we did above. For example `code --enable-features=UseOzonePlatform,WaylandWindowDecorations --ozone-platform-hint=auto` for Visual Studio Code

* You may not have the preserve video memory allocations module parameter enabled, and this can cause issues particularly when suspending and resuming the system, usually in the form of graphical artifacts or a broken desktop environment.

  You can check whether the module parameter is enabled or not by typing `sudo cat /proc/driver/nvidia/params | grep "PreserveVideoMemoryAllocations"`. If the value is `0` or missing, then the parameter is not enabled.

  To enable the preserve video memory allocations module paramter, please follow the below steps:
  
  1. Create or edit `/etc/modprobe.d/nvidia.conf` using `sudo nano /etc/modprobe.d/nvidia.conf`
  2. Add `options nvidia NVreg_PreserveVideoMemoryAllocations=1` to a new line
  3. Run `sudo dracut --regenerate-all --force`
  4. Reboot the system
  5. Run `sudo cat /proc/driver/nvidia/params | grep "PreserveVideoMemoryAllocations"` to verify the parameter is now set

  If you are still experiencing issues with suspend/resume after enabling this module parameter, you may want to take a look at Nvidia's [power management documentation](https://download.nvidia.com/XFree86/Linux-x86_64/435.17/README/powermanagement.html) to double check that the relevant `systemd` services are installed and enabled.
      
-----

## References
- https://askubuntu.com/questions/271613/am-i-using-the-nouveau-driver-or-the-proprietary-nvidia-driver
- https://github.com/lutris/docs/blob/master/InstallingDrivers.md
- https://askubuntu.com/questions/66328/how-do-i-install-the-latest-nvidia-drivers-from-the-run-file/66335#66335
- https://askubuntu.com/questions/219942/how-to-uninstall-manually-installed-nvidia-drivers/220729#220729
- https://askubuntu.com/questions/1403854/cant-use-wayland-with-nvidia-510-drivers-on-ubuntu-22-04-lts
- https://bbs.archlinux.org/viewtopic.php?pid=2133404#p2133404
- https://wiki.archlinux.org/title/NVIDIA#DRM_kernel_mode_setting
- https://askubuntu.com/questions/68028/how-do-i-check-if-ubuntu-is-using-my-nvidia-graphics-card/624793#624793
- https://www.reddit.com/r/linux_gaming/comments/17fn30q/comment/k6bug3m/
- https://www.reddit.com/r/linux_gaming/comments/17ubgrl/nvidia_libnvidiaeglwayland1_do_i_need_to_install/
- https://www.reddit.com/r/Fedora/comments/rkzp78/make_chrome_run_on_wayland_permanently/
- https://www.reddit.com/r/Fedora/comments/1afkoge/how_to_make_vscode_run_in_wayland_mode/
- https://us.download.nvidia.com/XFree86/Linux-x86_64/555.42.02/README/installationandconfiguration.html
- https://www.reddit.com/r/archlinux/comments/1cxc36m/comment/l528uff/
- https://forums.developer.nvidia.com/t/major-kde-plasma-desktop-frameskip-lag-issues-on-driver-555/293606
- https://download.nvidia.com/XFree86/Linux-x86_64/510.39.01/README/gsp.html
- https://download.nvidia.com/XFree86/Linux-x86_64/435.17/README/powermanagement.html
- https://www.reddit.com/r/linux_gaming/comments/1bjhx8w/explicit_sync_protocol_just_merged_on_wayland/
- https://www.reddit.com/r/linux_gaming/comments/1c9izpc/gnome_461_released_with_explicit_sync/
- https://www.reddit.com/r/linux_gaming/comments/1cx8739/nvidia_555_driver_now_out_explicit_sync_support/
- https://www.tecmint.com/install-nvidia-drivers-in-linux/
- https://rpmfusion.org/Howto/NVIDIA
- https://discussion.fedoraproject.org/t/video-playback-broken-after-upgrading-to-f39-libopenh264-so-7-is-missing-openh264-support-will-be-disabled/100019/2
