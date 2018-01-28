---
title: Using 1GB pages in gem5
authors: Swapnil Haria
excerpt:
---



Why is this necessary?
~~~~~~~~~~~~~~~~~~~~~~~~~
The overheads of virtual memory (mostly from address translation) have been steadily increasing with increasing main memory sizes. They can be as high as 50% of overall execution for unvirtualized systems and higher still for virtualized systems. Using large pages (2MB or 1GB) effectively increases the range of memory covered by TLB entries and reduces the number of costly TLB misses. Huge pages is also commonly used as a baseline when evaluating new virtual memory proposals.
We first understand 1GB pages can be supported in gem5 and then understand how to leverage them for our workloads.

Enabling 1GB pages
~~~~~~~~~~~~~~~~~~~~
x86-64 processors convey supported processor features to the OS via the CPUID interface. The CPUID information determines the flags shown via cat /proc/cpuinfo.

#. First, we configure these registers correctly to indicate support for 1GB pages. When the CPUID opcode is executed along with a register number, CPU registers EAX through EDX are populated as per the format specified in the Intel manual: http://pdinda.org/icsclass/doc/AMD_ARCH_MANUALS/CPUID_Specification.pdf
#. We need to set bit 26 in EDX register for CPUID 0x8000_0001 to indicate CPU support for 1GB pages.  This will set the corresponding flag 'pdpe1gb' in /proc/cpuinfo.
#. Further, we need to ensure that gem5 can actually handle 1GB pages. For this, we need to update the pagetable walker logic. Page table walks for 1GB pages end at the PDP level (2 levels above the 4KB PTE). The pagetable walker code is present in function Fault Walker::WalkerState::stepWalk(PacketPtr &write) in src/arch/x86/pagetable_walker.cc
#. A page table walk ends whenever the pagetable entry has its PS bit set (bit 7, starting from 0). We update the pagetable walker logic to also expect this at the PDP stage in addition to the PD and the PTE stages:

.. code-block:: c++

      case LongPDP:
        DPRINTF(PageTableWalker,
                "Got long mode PDP entry %#016x.\n", (uint64_t)pte);
        nextRead = ((uint64_t)pte & (mask(40) << 12)) + vaddr.longl2 * dataSize;
        doWrite = !pte.a;
        pte.a = 1;
        entry.writable = entry.writable && pte.w;
        entry.user = entry.user && pte.u;
        if (!pte.ps) {
            // 2 MB or 4 KB page
            if (badNX || !pte.p) {
                doEndWalk = true;
                fault = pageFault(pte.p);
                break;
            }
            nextState = LongPD;
            walker->PDPages.insert(nextRead >> 12);
            walker->PDEntries.insert(nextRead);
            break;
        } else {
            // 1 GB page
            warn_once("Encountered long mode 1GB page:0x%x\n", entry.vaddr);
            entry.logBytes = 30; // 1GB page so 30 bits for page offset
            entry.paddr = (uint64_t)pte & (mask(22) << 30);
            entry.uncacheable = uncacheable;
            entry.global = pte.g;
            entry.patBit = bits(pte, 12);
            entry.vaddr = entry.vaddr & ~((2 * (1 << 29)) - 1);
            doTLBInsert = true;
            doEndWalk = true;  // End page walk here
            break;
        }

With this, 1GB pages are supported in gem5!

Using huge pages
~~~~~~~~~~~~~~~~
The following steps are required for using huge pages:

#. Kernel boot command line must be update to indicate hugepage sizes desired as well as the default hugepage size (1GB vs 2MB in x86-64). This is done via :  `hugepagesz=1GB default_hugepagesz=1GB`

#. We also need libhugetlbfs (https://github.com/libhugetlbfs/libhugetlbfs/) installed on the disk image for ensuring allocations are made using hugepages instead of regular 4K pages.

#. Using libhugetlbfs, we first create a pool of hugepages:

.. code-block:: sh

   ./libhugetlbfs/obj/hugeadm --pool-pages-min 1GB:10
   ./libhugetlbfs/obj/hugeadm --pool-pages-max 1GB:30

#. We check the available pools to ensure that the above configuration worked correctly:

.. code-block:: sh

   ./libhugetlbfs/obj/hugeadm --pool-list

#. Finally, we run the application using the hugectl utility (https://linux.die.net/man/8/hugectl) to guide the allocation policy. It is recommended to turn on verbosity to ensure that allocations via hugepages actually occur. Thus to only use hugepages for memory on the heap, we launch our application thus:

.. code-block:: sh

    hugectl --heap=1GB --verbose=3 ./application
