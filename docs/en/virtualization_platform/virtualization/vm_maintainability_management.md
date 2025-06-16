# VM Maintainability Management 

## VM NMI Watchdog

### Overview

The NMI Watchdog is a mechanism used to detect hardlockup in Linux. Even if normal interrupts are disabled, non-maskable interrupt (NMI) can interrupt the code execution and further detect hardlockup. The current Arm architecture does not support native NMI, so it enables Pseudo-NMI based on the interrupt priority and configures Performance Monitoring Interrupt (PMI) as NMI to implement NMI Watchdog (PMU Watchdog).

### Precautions

- The VM OS needs to support Pseudo-NMI and corresponding kernel parameters needs to be configured.
- The configuration of the VM PMU Watchdog is the same as that of the host PMU Watchdog. The XML file configuration is not required.
- Both SDEI Watchdog and PMU Watchdog are NMI Watchdogs, whereas the SDEI Watchdog has a higher priority than PMU Watchdog. Therefore, the PMU Watchdog can be enabled only when the SDEI Watchdog is disabled. The VM does not support SDEI Watchdog, so you need to disable it by configuring kernel parameters.

### Procedure

To configure the NMI Watchdog for a VM in ARM architecture,perform the following steps:

1. Add the following parameters to the VM boot configuration file grub.cfg:

    ```conf
    nmi_watchdog=1 pmu_nmi_enable hardlockup_cpu_freq=auto irqchip.gicv3_pseudo_nmi=1 disable_sdei_nmi_watchdog hardlockup_enable=1
    ```

2. Check whether the PMU Watchdog is successfully loaded on the VM.
    If the loading is successful, information similar to the following is displayed in the dmesg log of the kernel: 

    ```text
    [2.1173222] NMI watchdog: CPU0 freq probed as 2399999942 HZ.
    ```
