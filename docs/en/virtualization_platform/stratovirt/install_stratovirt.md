# Installing StratoVirt

## Software and Hardware Requirements

### Minimum Hardware Requirements

- Processor architecture: Only the AArch64 and x86_64 processor architectures are supported. AArch64 requires ARMv8 or a later version that supports virtualization extension. x86_64 requires VT-x support.

- 2-core CPU
- 4 GiB memory
- 16 GiB available drive space

### Software Requirements

Operating system: openEuler 21.03

## Component Installation

To use StratoVirt virtualization, it is necessary to install StratoVirt. Before the installation, ensure that the openEuler Yum source has been configured.

1. Run the following command as user **root** to install the StratoVirt component:

   ```shell
   # yum install stratovirt
   ```

2. Check whether the installation is successful.

   ```shell
   $ stratovirt -version
   StratoVirt 2.1.0
   ```
