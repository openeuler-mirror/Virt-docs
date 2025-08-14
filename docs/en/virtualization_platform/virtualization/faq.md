# Common Issues and Solutions

## Issue 1: QEMU Hot Patch Created with LibcarePlus Fails to Load

The problem arises when the QEMU version does not match the hot patch version. To resolve this, download the source code for the corresponding QEMU version and ensure the environments for creating the hot patch and building the QEMU package are identical. The buildID can verify consistency. If users lack the QEMU build environment, they can **build and install the package themselves**, then use the buildID from `/usr/libexec/qemu-kvm` in the self-built package.

## Issue 2: Hot Patch Created with LibcarePlus Is Loaded but Not Effective

This occurs because certain functions are not supported, including dead loops, non-exiting functions, recursive functions, initialization functions, inline functions, and functions shorter than 5 bytes. To address this, verify if the patched function falls under these constraints.

## Issue 3: The First Result Displayed by the kvmtop Tool Is Calculated from Two Samples with a 0.05-Second Interval, Resulting in Significant Fluctuations

This issue stems from a defect in the open-source top framework, and there is currently no solution available.
