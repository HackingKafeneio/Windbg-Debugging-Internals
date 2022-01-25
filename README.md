# Windbg-Debugging-Internals
* Source taken from ReactOS `https://github.com/reactos/reactos/blob/1e01afab990b9fb9255d0c0d253ca141d5731a65/ntoskrnl/ke/amd64/kiinit.c#L431`

The windows kernel main function `KiSystemStartup`  sets the IDT handlers for debugging.

`KiBreakpointTrap` is responsible for handling breakpoints `int3` and `KiDebugTrapOrFault` is handling the debugging `single-step` functionality.

In case we need a startup-break(CTRL-K), windbg will send a break packet right after the initialization, that's going to execute `DbgBreakPointWithStatus`-`int3` function `(1)`.

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
        /* Setup the IDT */
        KeInitExceptions();

        /* Initialize debugging system */
        KdInitSystem(0, KeLoaderBlock);

        /* Check for break-in */
        if (KdPollBreakIn()) 
            DbgBreakPointWithStatus(DBG_STATUS_CONTROL_C); //(1)
    }
    ...
}

BOOLEAN
NTAPI
KdPollBreakIn(VOID)
{
    ...
    /* Now get a packet */
    if (KdReceivePacket(PACKET_TYPE_KD_POLL_BREAKIN,
                                    NULL,
                                    NULL,
                                    NULL,
                                    NULL) == KdPacketReceived) {
        /* Successful breakin */
        return TRUE;
    }
    ...
}

.PROC DbgBreakPointWithStatus
    mov eax, ecx
    int 3
    ret
.ENDP

```

* `KdInitSystem` spawns a `DPC` handler(`KdpTimeSlipDpcRoutine`) that is executed every couple of nanoseconds, checking if a `break` packet has arrived from debuggers input device(`serial`, `net`, `usb` etc). Windows10 seem to directly spawn the dpc-timer handler, ReactOS checks for the break `windbg` packet in `HalpClockInterrupt`

```cpp
BOOLEAN
NTAPI
KdInitSystem(IN ULONG BootPhase,
             IN PLOADER_PARAMETER_BLOCK LoaderBlock)
{
    ...
    KeInitializeDpc(&KdpTimeSlipDpc, KdpTimeSlipDpcRoutine, NULL);
    KeInitializeTimer(&KdpTimeSlipTimer);
    ExInitializeWorkItem(&KdpTimeSlipWorkItem, KdpTimeSlipWork, NULL);
    ...
}
```
```cpp
VOID
HalpClockInterrupt(VOID)
{
    /* Call the kernel */
    KeUpdateSystemTime(KeGetCurrentThread()->TrapFrame,
                       HalpCurrentTimeIncrement,
                       CLOCK2_LEVEL);
}

VOID
FASTCALL
KeUpdateSystemTime(IN PKTRAP_FRAME TrapFrame,
                   IN ULONG Increment,
                   IN KIRQL Irql)
{
    ...
     /* If the debugger is enabled, check for break-in request */
    if (KdDebuggerEnabled && KdPollBreakIn())
    {
        /* Break-in requested! */
        DbgBreakPointWithStatus(DBG_STATUS_CONTROL_C);
    }
    ...
```

Once an `int3` instruction gets executed, whether is executed directly from an overwritten instruction(by setting a breakpoint to a function) or from the `dpc` handler by sending a break packet, is going to execute `DispatchException` that will end up calling  `KdpTrap`.
`KdpTrap` is the main parser of `windbg` protocol. Is going to end up in a loop that will handle every packet until we continue the execution with the `g` command.

In case of `single-step`, the `single-step` bit is set in `eflags` and windbg will send a continue-packet. Once the next instruction(after breakpoint) is going to be executed, cpu will execute the `int1`(`KiDebugTrapOrFault`) handler and remove the `single-step` bit from `eflags`. `KiDebugTrapOrFault` will end up calling `KdpTrap` doing pretty much the same thing.

```asm
PUBLIC KiBreakpointTrap
FUNC KiBreakpointTrap
    /* Push pseudo error code */
    EnterTrap TF_SAVE_ALL

    /* Check if the frame was from kernelmode */
    test word ptr [rbp + KTRAP_FRAME_SegCs], 3
    jz KiBreakpointTrapKMode
    ...
KiBreakpointTrapKMode:
    /* Dispatch the exception */
    DispatchException STATUS_BREAKPOINT, 3, BREAKPOINT_BREAK, 0, 0

    /* Return */
    ExitTrap TF_SAVE_ALL
ENDFUNC

```

```cpp
BOOLEAN
NTAPI
KdpTrap(IN PKTRAP_FRAME TrapFrame,
        IN PKEXCEPTION_FRAME ExceptionFrame,
        IN PEXCEPTION_RECORD ExceptionRecord,
        IN PCONTEXT ContextRecord,
        IN KPROCESSOR_MODE PreviousMode,
        IN BOOLEAN SecondChanceException)
{
    
}
```
