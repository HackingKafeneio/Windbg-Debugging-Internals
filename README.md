# Windbg-Debugging-Internals
* Source taken from ReactOS

The `main` of windows kernel set the IDT handlers for debugging,, 
```cpp
KIDT_INIT KiInterruptInitTable[] =
{
    ...
    {0x01, 0x00, 0x00, KiDebugTrapOrFault},
    ....
    {0x03, 0x03, 0x00, KiBreakpointTrap},
    ...
}

VOID
NTAPI
KiSystemStartup(IN PLOADER_PARAMETER_BLOCK LoaderBlock)
{
    ...
    if (Cpu == 0)
    {
        ...
        /* Initialize debugging system */
        KdInitSystem(0, KeLoaderBlock);

        /* Check for break-in */
        if (KdPollBreakIn()) DbgBreakPointWithStatus(DBG_STATUS_CONTROL_C);
    }
    ...
}
```
