+++++++++++++++++++++++++++++++++++++
Debug Hybrid Graphics issues on Linux
+++++++++++++++++++++++++++++++++++++

:date: 2019-09-11 15:50
:tags: linux
:category: linux
:slug: debug-hybrid-graphics-issues-linux
:authors: Victor Stinner

`Hybrid Graphics <https://wiki.archlinux.org/index.php/Hybrid_graphics>`_ is a
complex hardware and software solution to achieve longer laptop battery life:
an **integrated** graphics device is used by default, and a **discrete**
graphics device with higher graphics performances is enabled on demand.

.. image:: {static}/images/hybrid_graphics.jpg
   :alt: Hybrid Graphics
   :target: https://www.theregister.co.uk/2010/02/09/inside_nvidia_optimus/

If it is designed and implemented carefully, users should not notice that a
laptop has two graphical devices.

Sadly, the Linux implementation is not perfect yet. I had to debug different
graphics issues on GNOME last months, so I decided to write down an article
about this technology.

This article is about the **GNOME** desktop environment with **Wayland**
running on **Fedora** 30, with Linux kernel **vgaswitcheroo** in muxless mode
(more about that above).


Hybrid Graphics
===============

Hybrid Graphics are known under different names:

* Linux kernel `vgaswitcheroo
  <https://www.kernel.org/doc/html/latest/gpu/vga-switcheroo.html>`_
* `PRIME <https://wiki.archlinux.org/index.php/PRIME>`_ in Linux open source
  GPU drivers (nouveau, ati, amdgpu and intel), the "muxless" flavor of hybrid graphics
* `Bumblebee <https://wiki.archlinux.org/index.php/bumblebee>`_:
  `NVIDIA Optimus <https://wiki.archlinux.org/index.php/NVIDIA_Optimus>`_
  for Linux
* "AMD Dynamic Switchable Graphics" for Radeon
* "Dual GPUs"
* etc.

Nowadays, most manufacturers utilizes the **muxless** model:

    Dual GPUs but **only one of them is connected to outputs**. The other one
    is merely used to **offload rendering**, its results are copied over PCIe
    into the framebuffer. On Linux this is supported with DRI PRIME.

In 2010, the first generation hybrid model used the **muxed** model:

    Dual GPUs with a hardware multiplexer chip to switch outputs between GPUs.
    This model makes the user choose (at boot time or at login time) between
    the two power/graphics profiles and is almost fixed throughout the user
    session.

Note: The development to support hybrid graphics in Linux started in 2010.

Does my Linux have Hybrid Graphics?
===================================

On Linux, Hybrid Graphics is used if the ``/sys/kernel/debug/vgaswitcheroo/``
directory exists.

No Hybrid Graphics, single graphics device::

    $ sudo cat /sys/kernel/debug/vgaswitcheroo/switch
    cat: /sys/kernel/debug/vgaswitcheroo/switch: No such file or directory

Hybrid Graphics with two graphics devices::

    $ sudo cat /sys/kernel/debug/vgaswitcheroo/switch
    0:IGD:+:Pwr:0000:00:02.0
    1:DIS: :DynOff:0000:01:00.0

Command to list graphics devices::

    $ lspci|grep VGA
    00:02.0 VGA compatible controller: Intel Corporation HD Graphics 530 (rev 06)
    01:00.0 VGA compatible controller: NVIDIA Corporation GM107GLM [Quadro M1000M] (rev a2)


Hardware
========

My employer gave me a Lenovo P50 laptop to work in December 2017. It is my only
computer at home, so I needed a powerful laptop (even if it's heavy for
traveling to conferences). The CPU, RAM and battery are great, but the hybrid
graphics caused me some headaches.

My Lenovo P50 has two GPUs::

    $ lspci|grep VGA
    00:02.0 VGA compatible controller: Intel Corporation HD Graphics 530 (rev 06)
    01:00.0 VGA compatible controller: NVIDIA Corporation GM107GLM [Quadro M1000M] (rev a2)

* The **Integrated Graphics Device** is a **Intel** IGP (Intel HD Graphics 530)
* The **Discrete Graphics Device** is a **NVIDIA** GPU (NVIDIA Quadro M1000M)

I didn't know that that the laptop had two graphics device when I chose the
laptop model. I discovered hybrid graphics when I started to debug graphics
issues.


BIOS
====

Hybrid graphics can be configured in the BIOS:

* **Discrete Graphics mode** will achieve higher graphics performances.
* **Hybrid Graphics mode** (default) runs as Integrated Graphics mode to
  achieve longer battery life, and Discrete Graphics is enabled on demand.

On my Lenovo P50, using the **Discrete Graphics mode** removes "00:02.0 VGA
compatible controller: Intel Corporation HD Graphics 530" from ``lspci``
command output: the **Intel IGP is fully disabled**. The Linux kernel only
sees the NVIDIA GPU.


Linux kernel
============

On Linux, hybrid graphics is handled by **vgaswitcheroo**::

    $ sudo cat /sys/kernel/debug/vgaswitcheroo/switch
    0:IGD:+:Pwr:0000:00:02.0
    1:DIS: :DynPwr:0000:01:00.0

* ``IGD`` stands for **Integrated** Graphics Device
* ``DIS`` stands for **DIScrete** Graphics Device
* "+" marks the **active** card
* ``Pwr``: the graphics device is **always active**
* ``DynPwr``: the graphics device is actived **on demand**

The last field (ex: ``0000:00:02.0``) is based on the PCI identifier::

    $ lspci|grep VGA
    00:02.0 VGA compatible controller: Intel Corporation HD Graphics 530 (rev 06)
    01:00.0 VGA compatible controller: NVIDIA Corporation GM107GLM [Quadro M1000M] (rev a2)

On my laptop, hybrid graphics is detected by an `ACPI
<https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface>`_
"Device-Specific Method" (DSM)::

    $ journalctl -b -k|grep 'VGA switcheroo'
    Sep 11 02:29:54 apu kernel: VGA switcheroo: detected Optimus DSM method \_SB_.PCI0.PEG0.PEGP handle

See: `VGA Switcheroo (Linux kernel documentation)
<https://www.kernel.org/doc/html/latest/gpu/vga-switcheroo.html>`_.


OpenGL
======

`Mesa <https://en.wikipedia.org/wiki/Mesa_(computer_graphics)>`_ provides
``glxinfo`` utility to get information about the OpenGL driver currently used::

    $ glxinfo|grep -E 'Device|direct rendering'
    direct rendering: Yes
        Device: Mesa DRI Intel(R) HD Graphics 530 (Skylake GT2)  (0x191b)

On this example, the discrete Intel IGP is used.

In Firefox, go to **about:support** page and search for the ``Graphics``
section to get information about compositing, WebGL, GPU, etc.


DRI_PRIME environment variable
==============================

Set DRI_PRIME=1 environment variable to run an application with the
**discrete** GPU.

Example::

    $ DRI_PRIME=1 glxinfo|grep -E 'Device|rendering'
    direct rendering: Yes
        Device: NV117 (0x13b1)

switcheroo-control
==================

`switcheroo-control <https://github.com/hadess/switcheroo-control>`_ is a
deamon controlling ``/sys/kernel/debug/vgaswitcheroo/switch`` (Linux kernel).
It can be accessed by DBus.

When the daemon starts, it looks for ``xdg.force_integrated=VALUE`` parameter
in the Linux command line. If *VALUE* is ``1``, ``true`` or ``on``, or if
``xdg.force_integrated=VALUE`` is not found in the command line, the daemon
writes ``DIGD`` into ``/sys/kernel/debug/vgaswitcheroo/switch`` (delayed
**switch to the integrated graphics device**: my Intel IGP)

If ``xdg.force_integrated=0`` is found in the command line, the daemon leaves
``/sys/kernel/debug/vgaswitcheroo/switch`` unchanged.

systemd:

* Check if the service is running: ``sudo systemctl status switcheroo-control.service``
* Disable the service: ``sudo systemctl disable switcheroo-control.service``
  and ``sudo systemctl stop switcheroo-control.service``

On Fedora, switcheroo-control is installed by default.

It is unclear to me if this daemon is still useful for my setup. It seems like
the the Linux kernel switcheroo uses the integrated Intel IGP by default
anyway.


Disable the discrete GPU by blacklisting its driver
===================================================

To debug graphical bugs, I wanted to ensure that the discrete NVIDIA GPU is
never used.

I found the solution of fully disabling the nouveau driver in the Linux kernel:
add ``modprobe.blacklist=nouveau`` to the Linux kernel command line. On Fedora,
you can use::

    sudo grubby --update-kernel=ALL --args="modprobe.blacklist=nouveau"

To reenable nouveau, remove the parameter. On Fedora::

    sudo grubby --update-kernel=ALL --remove-args="modprobe.blacklist=nouveau"


Demo!
=====

For this test, my laptop is not connected to anything (no power cable, no
external monitor, no dock).

When my laptop is idle (no 3D application is running), the NVIDIA GPU is
**suspended**::

    $ cat /sys/bus/pci/drivers/nouveau/0000\:01\:00.0/enable
    0
    $ cat /sys/bus/pci/drivers/nouveau/0000\:01\:00.0/power/runtime_status
    suspended

I explicitly run a 3D application on it::

    DRI_PRIME=1 glxgears

The NVIDIA GPU becomes **active**::

    $ cat /sys/bus/pci/drivers/nouveau/0000\:01\:00.0/enable
    2
    $ cat /sys/bus/pci/drivers/nouveau/0000\:01\:00.0/power/runtime_status
    active

I stop the 3D application. A few seconds later, the NVIDIA GPU is **suspended**
again::

    $ cat /sys/bus/pci/drivers/nouveau/0000\:01\:00.0/enable
    0
    $ cat /sys/bus/pci/drivers/nouveau/0000\:01\:00.0/power/runtime_status
    suspended


Graphics devices and monitors
=============================

When I disabled the nouveau driver using ``modprobe.blacklist=nouveau`` kernel
command line parameter, I was no longer able to use external monitors. I
understood that:

* The **Intel** IGP is connected to the **internal** laptop screen
* The **NVIDIA** GPU is connected to the **external** monitors (DisplayPort
  and HDMI ports)

When my laptop has **no external monitor** connected, the **discrete** NVIDIA
GPU is **actived on demand** (suspended when idle)

When I connect my laptop to **two external monitors** (using my dock), the
**discrete** NVIDIA GPU is **always active**::

    $ cat /sys/bus/pci/drivers/nouveau/0000\:01\:00.0/power/runtime_status
    active


Links
=====

* https://wiki.archlinux.org/index.php/Hybrid_graphics
* https://www.kernel.org/doc/html/latest/gpu/vga-switcheroo.html
* https://wiki.archlinux.org/index.php/PRIME
* https://help.ubuntu.com/community/HybridGraphics
* https://en.wikipedia.org/wiki/Nvidia_Optimus
* https://en.wikipedia.org/wiki/AMD_Hybrid_Graphics
* https://nouveau.freedesktop.org/wiki/Optimus
