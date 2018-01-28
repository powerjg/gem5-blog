---
layout: post
tags: howto
authors: Swapnil Haria
excerpt: >-
    This post describes how to emulate new persistent memory technolgies in gem5 by Swapnil Haria.
    Swapnil used this support in his ASPLOS 2017 publication: An Analysis of Persistent Memory Use with WHISPER. Sanketh Nalli, Swapnil Haria, Mark D. Hill, Michael M. Swift, Haris Volos, Kimberly Keeton.
title: Persistent Memory emulation in gem5
---

:Authors: Swapnil Haria

This post is from `Swapnil Haria`_, a 4th year grad student at University of Wisconsin working with `Mark Hill`_ and `Mike Swift`_.
It describes how to emulate new persistent memory technolgies in gem5.
Swapnil used this support in his ASPLOS 2017 publication: *An Analysis of Persistent Memory Use with WHISPER*. Sanketh Nalli, Swapnil Haria, Mark D. Hill, Michael M. Swift, Haris Volos, Kimberly Keeton.

.. _Swapnil Haria: http://pages.cs.wisc.edu/~swapnilh/

.. _Mark Hill:  http://pages.cs.wisc.edu/~markhill/

.. _Mike Swift: http://pages.cs.wisc.edu/~swift/



Why is this necessary?
~~~~~~~~~~~~~~~~~~~~~~~~~
Persistent Memory (PM) is a catch-all term simply meaning any non-volatile memory (NVM)
technology attached on the memory bus. Currently, there is no support for PM in gem5, which
mean that researchers use existing DRAM models to emulate PM. Moreover, there is no good way of distinguishing PM accesses from DRAM accesses. However, this is not ideal for the following reasons:
- PM latencies will be vastly different from DRAM latencies and also asymmetrical.
- Different memory controller architectures will be needed for NVM.
In this post, we develop a system configuration which will separate PM from DRAM making it easier to solve the issues above.

How does this work?
~~~~~~~~~~~~~~~~~~~~
Future hybrid-memory systems are expected to have a physical address space with separate ranges for volatile and persistent memory. We configure such a system in gem5 using existing NUMA support. Specifically, we mimic PM using a NUMA node with only memory and no CPU. This ensures that PM is not local memory for any CPU and requires special allocation.


.. figure:: /assets/pm-system.png
   :width: 300pt
   :align: center

Note that these steps only works in Full-System mode.

#. Build the Linux kernel with following NUMA config options enabled (CONFIG_NUMA=y, CONFIG_NUMA_EMU=y). I used linux version 4.14-rc6.
#. We will configure 24GB physical memory split across three memory controllers (with a gap for devices between 3 and 4GB):

.. code-block:: python

       system = X86System()
       system.mem_ranges = [AddrRange('3GB'),
            AddrRange(Addr('4GB'), size = convert.toMemorySize('5GB')),
            AddrRange(Addr('9GB'), size = convert.toMemorySize('8GB')),
            AddrRange(Addr('17GB'), size = convert.toMemorySize('8GB'))]


#. We configure the memory layout registers read by BIOS:

.. code-block:: python

       entries = []
       entries.append(X86E820Entry(addr = 0x100000000,
            size = '%dB' % (self.mem_ranges[1].size()), range_type = 1))
        entries.append(X86E820Entry(addr = 0x240000000,
            size = '%dB' % (self.mem_ranges[2].size()), range_type = 1))
        entries.append(X86E820Entry(addr = 0x440000000,
            size = '%dB' % (self.mem_ranges[2].size()), range_type = 1))
        system.e820_table.entries = entries


#. A separate memory controller is created for each address range and thus PM-specific controllers can be used for the last address range.
#. Add 'numa=fake=3' to linux boot command line. If using default config files, this can be found in gem5/configs/common/FSConfig.py. This sets up three NUMA nodes with memory of equal sizes.
#. Configure the system to only simulate 2 CPUS (i.e., one less than the number of NUMA nodes).
#. We use libnuma/numactl to ensure all volatile memory allocations are serviced from memory on nodes 0,1:

.. code-block:: sh

     numactl --membind=0,1 ./queue_nvm


#. All PM allocations can be done on the PM node via numa_alloc_onnode() API provided by libnuma.
#. Alternatively, use a backing file to emulate PM. Allocate file on a tmpfs volume (shows up as /dev/shm typically). tmpfs is a temporary, memory-resident file system. We ensure that tmpfs is configured to only allocate memory on node 2:

.. code-block:: sh

	mount -o remount,mpol=bind=static:2 /dev/shm
