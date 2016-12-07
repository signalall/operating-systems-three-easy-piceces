# 18 分页：介绍


有时说操作系统采用两种方法之一来解决大多数空间管理问题。 第一种方法是将事物分割成可变大小的部分，就像我们在虚拟内存中看到的那样。 不幸的是，这种解决方案具有固有的困难。 特别地，当将空间划分为不同大小的块时，
空间本身可能变得**分散**，因此
随着时间的推移分配变得更具有挑战性。

因此，第二种方法可能值得考虑：把
空间划分成固定大小的部分。 在虚拟内存中，我们称之为**分页**，它来自一个早期重要的系统，Atlas [KE + 62，L78]。
第二种方法不是将进程的地址空间拆分成若干个
可变大小的逻辑段（例如，代码，堆，堆栈），而是将它分成固定大小的单元，我们每个单元称为页面。 相应地，我们把物理存储看做是由一个许多固定大小的槽的数组，槽也被称为页框。每个页框可以包含一个虚拟内存页。我们的挑战：

> 如何用页面来虚拟化内存
> 我们如何用页面虚拟化内存，以避免分段的问题？ 什么是基本技术？ 我们如何做
这些技术工作得很好，并保证空间和时间开销最小？


## 18.1 一个简单的示例及其概述
```
// Extract the VPN from the virtual address
VPN = (VirtualAddress &VPN_MASK) >> SHIFT;

// Form the address of the page-table entry (PTE)
PTEAddr = PTBR + (VPN *sizeof(PTE));

// Fetch the PTE
PTE = AccessMemory(PTEAddr);

// Check if process can access the page
if (PTE.Valid == False) RaiseException(SEGMENTATION_FAULT);
else if (CanAccess(PTE.ProtectBits) == False)
    RaiseException(PROTECTION_FAULT);
else // Access is OK: form physical address and fetch it
    offset = VirtualAddress & OFFSET_MASK;
    PhysAddr = (PTE.PFN << PFN_SHIFT) | offset;
    Register = AccessMemory(PhysAddr);

```