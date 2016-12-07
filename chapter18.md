# 18 分页：介绍
#
18 分页：介绍

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