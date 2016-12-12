# 20 分页：更小的表

```
VPN = (VirtualAddress &VPN_MASK) >> SHIFT;
(Success, TlbEntry) = TLB_Lookup(VPN);
if (Success == True) {// TLB Hit
    if (CanAccess(TlbEntry.ProtectBits) == True) {
        Offset = VirtualAddress & OFFSET_MASK;
        PhysAddr = (TlbEntry.PFN << SHIFT) | Offset;
        Register = AccessMemory(PhysAddr);
    } else {
        RaiseException(PROTECTION_FAULT)
    }
} else { // TLB Miss
    // first, get page directory entry
    PDIndex = (VPN & PD_MASK) >> PD_SHIFT;
    PDEAddr = PDBR + (PDIndex * sizeof(PDE));
    PDE = AccessMemory(PDEAddr);
}
if (PDE.Valid == False) {
    RaiseException(SEGMENTATION_FAULT);
} else { // PDE is valid: now fetch PTE from page table
    PTIndex = (VPN & PT_MASK) >> PT_SHIFT;
    PTEAddr = (PDE.PFN << SHIFT) + (PTIndex * sizeof(PTE));
    PTE = AccessMemory(PTEAddr);
}
if (PTE.Valid == False) {
    RaiseException(SEGMENTATION_FAULT);
} else if (CanAccess(PTE.ProtectBits) == False) {
    RaiseException(PROTECTION_FAULT);
} else {
    TLB_Insert(VPN, PTE.PFN, PTE.ProtectBits);
    RetryInstruction();
}
```