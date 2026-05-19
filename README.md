# HyperGuard

AMD-V hypervisor that protects Windows kernel callbacks from being unhooked at Ring 0. Built on top of [SimpleSVM](https://github.com/tandasat/SimpleSVM) by Satoshi Tanda as part of my final year project.

## What it does

When EDR products register kernel callbacks (process creation, thread, image load), attackers running at Ring 0 can null those entries and blind the EDR. This project runs a hypervisor underneath Windows using AMD SVM, and uses a timer-based integrity loop to detect and restore any callbacks that get wiped.

The main things I added on top of SimpleSVM:

- Callback array discovery — scans for LEA instructions at runtime to find `PspCreateProcessNotifyRoutine`, `PspCreateThreadNotifyRoutine` and `PspLoadImageNotifyRoutine` without hardcoded offsets
- Snapshot — saves the valid callback pointers on load
- Integrity loop — a KTIMER fires every 10ms, runs a DPC, checks the live arrays against the snapshot and restores anything that changed

## How it works

```text
Windows kernel (guest, Ring 0)
  └── callback arrays: Process / Thread / Image

AMD-V hypervisor (host, below the OS)
  └── KTIMER/KDPC every 10ms
        └── compare live entries vs saved snapshot
              └── restore if nulled or modified
```

## Requirements

- AMD CPU with SVM + Nested Paging
- Windows 10/11 x64
- WDK / Visual Studio 2022
- Test signing enabled

## Notes

- Snapshot is taken once at load time, so callbacks registered after that are not tracked
- 10ms polling is a trade-off between speed and overhead
- AMD only, no Intel VT-x version

## Credit

The hypervisor base (virtualisation, VMEXIT handling, NPT setup, etc.) comes from [SimpleSVM](https://github.com/tandasat/SimpleSVM) by Satoshi Tanda under the MIT license. The callback protection logic and modifications are my own.