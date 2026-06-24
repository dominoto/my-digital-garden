---
{"dg-publish":true,"permalink":"/resources/ghost-of-tsushima-crashing-with-dxgi-error/","created":"2024-07-30","updated":"2025-10-12","dg-note-properties":{"type":["Article"],"topics":["[[Gaming]]"],"tags":null,"created":"2024-07-30","modified":"2025-10-12","author":["Domagoj Zubac"]}}
---


**Topics:** ==[[Resources/Gaming\|Gaming]]==

[Ghost of Tsushima](https://en.wikipedia.org/wiki/Ghost_of_Tsushima) is an action-adventure video game developed by Sucker Punch Productions and released in 2020 on consoles. It also released on PC in 2024, but with an annoying issue:

> DXGI_ERROR_DEVICE_HUNG

![9dqlopdqp51d1.png](/img/user/_Meta/Attachments/9dqlopdqp51d1.png)

If this error sounds familiar maybe the rest of this post will help you, so read on.

Since the problem is probably with the renderer (other fixes like rolling back drivers or changing in-game graphic settings didn't help), we will change it from DX12 to Vulkan using [vkd3d](https://github.com/HansKristian-Work/vkd3d-proton) and [dxvk](https://github.com/doitsujin/dxvk).

1. Download latest **vkd3d-proton-x.xx.tar.zst** [from here](https://github.com/HansKristian-Work/vkd3d-proton/releases) and extract (you can use [7-zip](https://www.7-zip.org/)) the x64 version of **d3d12.dll** and **d3d12core.dll** in the same folder as **GhostOfTsushima.exe**
2. Download latest **dxvk-x.x.tar.gz** [from here](https://github.com/doitsujin/dxvk/releases) and extract (you can use [7-zip](https://www.7-zip.org/)) the x64 version of **dxgi.dll** in the same folder as **GhostOfTsushima.exe**
3. Open new file in Notepad, paste the following and save it as **dxvk.conf** in the same folder with **GhostOfTsushima.exe**:
```
# These settings override the reported PCI vendor and device IDs to the application. In this case, 10de is a common ID for NVIDIA GPUs, and 24c9 may be a custom or fictional ID. This can cause the application to behave differently depending on the selected values. You need to spoof the gpu because game will not start without it. I have a Radeon and it's working properly even though it is spoofed as Nvidia.
dxgi.customDeviceId = 24c9
dxgi.customVendorId = 10de

# By default, dxvk hides NVIDIA GPUs from the application, which can help work around crashes or low performance in games, especially those using Unreal Engine. Since this setting is set to False, the application will see the NVIDIA GPU as-is.
dxgi.hideNvidiaGpu = False

# UMA ( Unified Memory Architecture) emulation is disabled. UMA is a technique used to emulate a multi-GPU configuration on a single GPU. Disabling emulation means the application will not expect to see multiple GPUs.
dxgi.emulateUMA = False

# Enable HDR if you want to use it.
dxgi.enableHDR = True
```

> [!important]
> Be careful with **dxvk.conf** file, extension must be *.conf*, not .conf.txt!
> If you don't use HDR, feel free to change `dxgi.enableHDR` to `False`.
> If you are planning to use ReShade or something similar, ==install it for Vulcan== instead of DX12.

Happy gaming! :)

---
References:
https://raw.githubusercontent.com/doitsujin/dxvk/master/dxvk.conf
https://www.reddit.com/r/ghostoftsushima/comments/1cuts8x/comment/l5vhh9w/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button