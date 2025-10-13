## Physical mem template for Windows Kernel Exploits by Juan Sacco
<<jsacco@exploitpack.com>> https://exploitpack.com

In this exploit, the core technique here was hijacking a legitimate syscall (NtShutdownSystem) to act as a gate into arbitrary kernel exports.

First, resolved the virtual address of a target kernel routine (PsGetCurrentProcess, DbgPrint, ExAllocatePoolWithTag, etc) by parsing the ntoskrnl export table.

Then, temporarily patched the syscall’s entry point with a tiny jump stub (FF 25 ... absolute JMP) that redirected execution into that routine. When user mode invoked the syscall, control safely crossed into ring 0 and landed directly at the chosen export  executed with kernel privileges. The stub was restored immediately after.

Step by step of the exploit:
Obtain the kernel base address,
Use get_kmodule_export (thanks back.engineering!) to parse PE export directory in kernel memory,
Read the export address table,
Find the Relative virtual address of my export,
Add Relative virtual address to kernel base to obtain the routine offset address,
Patch the syscall "NtShutdownSystem" with small shellcode to call it from usermode and jump to my routine,
Call to NtShutdownSystem() from user-mode,
The kernel executes my export instead of NtShutdownSystem() with full kernel context,
Restore the original syscall bytes to avoid a BSOD,
Repeat for all the exports and profit.
For this exploit I'm going to use the tool I made based on vdm, this tool is able to use the physical memory primitives and use "NTShutdownSystem" as gate for our syscalls and exports, to obtain arbitrary code execution in Kernel Space, but for this to happen we need to provide the device name, and the vulnerable IOCTL control codes.

Let's start by finding the device name. IDA pro was smart enough to discover the Driver Entry, and there we found the device name as shown in this screenshot:

<img src="https://cdn.shopify.com/s/files/1/0918/4162/6445/files/Screenshot_from_2025-09-24_10-26-47.png?v=1758702729">

Then we needed to find IOCTL codes that used in a vulnerable way ZwMapViewOfSection and ZwUnMapViewOfSection. For this I have used Driver Buddy Reloaded, as a comment this plugin has been edited by me, to make it work with recent IDA Pro versions.

Here I discovered the locations of ZwMapViewOfSection and ZwUnMapViewOfSection.
<img src="https://cdn.shopify.com/s/files/1/0918/4162/6445/files/Screenshot_from_2025-09-24_10-34-26.png?v=1758702890">

Then I have followed the location of that call to ZwMapView of section:

<img src="https://cdn.shopify.com/s/files/1/0918/4162/6445/files/Screenshot_from_2025-09-24_10-35-42.png?v=1758703002">

Then I have decompiled the code using Hexrays decompiler and found the vulnerable part that maps physical memory into the calling process with ZwMapViewOfSection.

The code does not validate the physical addresses, sizes, or intended use. That means a malicious caller could request a mapping over kernel memory or hardware MMIO regions.

<img src="https://cdn.shopify.com/s/files/1/0918/4162/6445/files/Screenshot_from_2025-09-24_10-40-20.png?v=1758703236">

Then I followed the xrefs of this function, the objective of doing that is to find the driver dispatcher routine, to locate the IOCTL code that will point to this function.

<img src="https://cdn.shopify.com/s/files/1/0918/4162/6445/files/Screenshot_from_2025-09-24_10-37-10.png?v=1758703203">

And here is the first IOCTL code for ZwMapViewOfSection.

<img src="https://cdn.shopify.com/s/files/1/0918/4162/6445/files/Screenshot_from_2025-09-24_10-44-11.png?v=1758703477">

As we located the dispatcher we can easily find the next IOCTL code we need: ZwUnmapViewOfSection, here below is the IOCTL code for that.

<img src="https://cdn.shopify.com/s/files/1/0918/4162/6445/files/Screenshot_from_2025-09-24_10-45-32.png?v=1758703546">

Now we are ready to launch our exploit with the device name and the ioctl codes for ZwMapViewOfSection and ZwUnMapViewOfSection. As a result we have obtained Arbitrary Kernel Code Execution, process tokens have been swapped from System 4 to the current process, effectively escalating the current user to System.

<img src="https://cdn.shopify.com/s/files/1/0918/4162/6445/files/Screenshot_from_2025-09-18_19-35-29.png?v=1758702199">

Credits: https://blog.back.engineering/
