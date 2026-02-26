# PetaLinux Guide

Please be aware that this guide is for 2022.1. Each version has its own different files.

## Before Installation

1.  sudo dpkg-reconfigure dash
    Say no

2.  Verify it:
    ls -l /bin/sh

    Should show:
    /bin/sh -> bash

3.  Go to 2022.1_PetaLinux Package_List.xlxs
    Find the appropraite package list for your OS
    For Ubuntu 20.04, run below command

    sudo apt-get install iproute2 gawk python3 python build-essential gcc git make net-tools libncurses5-dev tftpd zlib1g-dev libssl-dev flex bison libselinux1 gnupg wget git-core diffstat chrpath socat xterm autoconf libtool tar unzip texinfo zlib1g-dev gcc-multilib automake zlib1g:i386 screen pax gzip cpio python3-pip python3-pexpect xz-utils debianutils iputils-ping python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev pylint3

4.  Install extra packages
    For Ubuntu:
    sudo apt-get install libtinfo5

    For RHEL/CentOS and so on
    sudo apt-get install ncurses-libs

5.  petalinux error: No space left on device or exceeds fs.inotify.max_user_watches?

### Create a TFTP Server
1.  sudo apt update; sudo apt install tftpd-hpa

2.  sudo nano /etc/default/tftpd-hpa

    TFTP_USERNAME="tftp"
    TFTP_DIRECTORY="/srv/tftp"
    TFTP_ADDRESS="0.0.0.0:69"
    TFTP_OPTIONS="--secure --create"

3.  sudo mkdir -p /srv/tftp
    sudo chown tftp:tftp /srv/tftp
    sudo chmod 755 /srv/tftp

4.  Start and enable it
    sudo systemctl start tftpd-hpa
    sudo systemctl enable tftpd-hpa

5.  sudo systemctl status tftpd-hpa
    Should output -> "active (running)

## Installing the PetaLinux Tool

1.  mkdir petalinux

Move petalinux-v2022.1-04191534-installer.run to this folder (Can be downloaded from Xilinx Archive)

2.  cd petalinux
3.  mkdir install

4.  chmod 755 ./petalinux-v2022.1-04191534-installer.run 
5.  ./petalinux-v2022.1-04191534-installer.run --dir $PWD/install --platform aarch64

## PetaLinux Working Enviroment Setup

This setup should be done everytime you want to access the petalinux commands like:

- petalinux-config
- petalinux-build and etc...

1.  cd petalinux/install

2.  source settings.sh

## Generating the .xsa File

### **VERY IMPORTANT:**

While generating the XSA for our case when configuring the PCIe on Zynq UltraScale MPSoC IP on Vivado, first:
    
1.  Enable advanced options

2.  Go to Advanced PCIe Settings

3.  Select Root Port and NOT endpoint mode

4.  This resets the pcie setting on the "High Speed" tab so re-assign them.

5.  Generate the .xsa with these settings.

## Create a Project & Import Hardware Configuration
1.  Move your .xsa file which you generated from Vivado to the petalinux folder

2.  cd petalinux/install

3.  petalinux-create --type project --template zynqMP --name ps_nvme_alinx

4.  cd ps_nvme_alinx

5.  petalinux-config --get-hw-description ../../ps_nvme_wrapper.xsa

6.  Subsystem AUTO Hardware Settings -> Serial Settings
    Check if this corresponds to your ps_uart port with baud rate 115200 (ex. psu_uart_0)

7.  Subsystem AUTO Hardware Settings -> SD/SDIO Settings
    Check if this corresponds to your SD Card connection (ex. psu_sd_1)

8.  Save and exit 

### Build Optimization

For UltraScale MPSoC you need following, but you can disable the optionals (this is not tested)

-   First Stage Bootloader (FSBL)   -> optional
-   PMU Firmware (PMUF)             -> optional
-   Trusted Firmware-A (TF-A)       -> mandatory

1.  petalinux-config
    Go to sub-pages where these options are listed and configure it

2.  Save and exit

## Kernel Customization

To enable certain peripherals run below command, for this setup we will enable NVMe drivers.

To configure other areas check the commands listed on page 16 of PetaLinux Tools Documentation Reference Guide

0.  cd petalinux/install/ps_nvme_alinx

1.  petalinux-config -c kernel
    This will take several minutes.

2.  Device drivers -> NVME Support
    Enable all blocks especially, NVM Express block device

3.  Enable: Bus options->PCI support->Message Signaled Interrupts (MSI and MSI-X)

4.  Enable: Bus options->PCI support->Enable PCI resource re-allocation detection

Check the step 10 of the following URL to enable PCI related drivers:
https://www.fpgadeveloper.com/2016/04/connecting-an-ssd-to-an-fpga-running-petalinux.html/

5.  Save and exit.

## Root Filesystem Configuration

0.  cd petalinux/install/ps_nvme_alinx

1.  petalinux-config -c rootfs

2.  Enable PCI utils (for lspci): Filesystem Packages->console/utils->pciutils->pciutils

3.  Filesystem Packages->base->util-linux   ->util-linux
                                            ->util-linux-blkid
                                            ->util-linux-fdisk
                                            ->util-linux-mkfs
                                            ->util-linux-mount

4.  Filesystem Packages->base->e2fsprogs    ->e2fsprogs
                                            ->e2fsprogs-mke2fs

5.  Save and exit.

## Device Tree Configuration


For our case in addition to enabling the PCIe on Vivado and from the kernel customization steps we alsa have to enable it from the device tree configuration files:

-   All of the user configurations are made on system_user.dtsi:

    petalinux/install/ps_nvme_alinx/project-spec/meta-user/recipes-bsp/device-tree/files/system_user.dtsi

-   This .dtsi file is included on system-top.dts:

    petalinux/install/ps_nvme_alinx/components/plnx_workspace/device-tree/device-tree

-   On system-top.dts all the other .dtsi files are included which creates the whole device tree structure.

-   To edit our system_user.dtsi we have to know what each peripheral correspond the the right syntax for this check the documentation on:

    petalinux/install/ps_nvme_alinx/build/tmp/work-shared/zynqmp-generic/kernel-source/Documentation/devicetree/bindings

-   Here for example if we go to the "pcie" folder we can see all the different pci configurations for different pci devices (chips)

-   For our case we are only concerned with the UltraScale+ Devices Integrated Block for PCIExpress (short: PS PCIe). This is probably shown on:

    - xilinx-pcie.txt

When checking it this corresponds to the AXI PCIe Bridge IP but actually we need the:

    - xilinx-nwl-pcie.txt

I do not know what NWL corresponds to but this driver file is explained on the Xilinx/Confluence as **Linux ZynqMP PS-PCIe Root Port Driver**

Also on our **zynqmp.dtsi** where we only instantiated the PS PCIe Port the following code snipped is generated:

		pcie: pcie@fd0e0000 {
			compatible = "xlnx,nwl-pcie-2.11";
			status = "disabled";
			#address-cells = <3>;
			#size-cells = <2>;
			#interrupt-cells = <1>;
			msi-controller;
			device_type = "pci";
			interrupt-parent = <&gic>;
			interrupts = <0 118 4>,
				     <0 117 4>,
				     <0 116 4>,
				     <0 115 4>,	/* MSI_1 [63...32] */
				     <0 114 4>;	/* MSI_0 [31...0] */
			interrupt-names = "misc", "dummy", "intx",
					  "msi1", "msi0";
			msi-parent = <&pcie>;
			reg = <0x0 0xfd0e0000 0x0 0x1000>,
			      <0x0 0xfd480000 0x0 0x1000>,
			      <0x80 0x00000000 0x0 0x1000000>;
			reg-names = "breg", "pcireg", "cfg";
			ranges = <0x02000000 0x00000000 0xe0000000 0x00000000 0xe0000000 0x00000000 0x10000000	/* non-prefetchable memory */
				  0x43000000 0x00000006 0x00000000 0x00000006 0x00000000 0x00000002 0x00000000>;/* prefetchable memory */
			interrupt-map-mask = <0x0 0x0 0x0 0x7>;
			bus-range = <0x00 0xff>;
			interrupt-map = <0x0 0x0 0x0 0x1 &pcie_intc 0x1>,
					<0x0 0x0 0x0 0x2 &pcie_intc 0x2>,
					<0x0 0x0 0x0 0x3 &pcie_intc 0x3>,
					<0x0 0x0 0x0 0x4 &pcie_intc 0x4>;
			#stream-id-cells = <1>;
			iommus = <&smmu 0x4d0>;
			power-domains = <&zynqmp_firmware PD_PCIE>;
			pcie_intc: legacy-interrupt-controller {
				interrupt-controller;
				#address-cells = <0>;
				#interrupt-cells = <1>;
			};
        }

Here we can see it is "disabled" by default. We have enable this on our system-user.dtsi file.

Plus the xlnx-nwl-pcie-2.11 also looks for reset-gpios (ie. perst signal) to work properly. On our alinx board this is connected to the MIO37 pin.

Since the gpio and pcie blocks on zynqmp.dtsi are "disabled" by default. We also have to enable them beforehand.

Therefore the system-user.dtsi should be updated like below:

/include/ "system-conf.dtsi"
/ {
};

&gpio {
	status = "okay";
};

&pcie {
	status = "okay";
	/* This property tells the driver to use MIO 37 as the reset signal. */
	reset-gpios = <&gpio 37 GPIO_ACTIVE_LOW>;
};

## Build and Package Boot Image

0.  cd petalinux/install/ps_nvme_alinx

1.  petalinux-build
    This will take around 15 minutes.

    If you want to first clean the older build:
    petalinux-build -x mrproper

2.  petalinux-package --boot --u-boot --force
    Use --force if you want to overwrite existing package

## Packaging Prebuild Images & BSP (optional)

PetaLinux board support packages (BSPs) are useful for distribution between teams and customers. Customized PetaLinux project can be shipped to next level teams or external customers through BSPs.

0.  cd petalinux/install/ps_nvme_alinx

1.  petalinux-package --prebuilt --fpga $PWD/project-spec/hw-description/ps_nvme_wrapper.bit

2.  petalinux-package --bsp -p $PWD --output MY_BSP

## Copying Files to an SD Card

0.  cd petalinux/install/ps_nvme_alinx

1.  Create two partitions for the SD Card. You can use "gparted" or other ways to manage this.

    - for BOOT files create a fat32 partition
    - for ROOT files create a ext4 partition

2.  Label each partition:

    - sudo mlabel -i /dev/sda1 ::boot
    - sudo e2label /dev/sda2 root

3.  Copy and paste below files on /images/linux to the boot partition (you can use file manager for this operation)

    - BOOT.BIN
    - image.ub
    - boot.scr

4.  You can not access ext4 partition via file system manager and using sudo creates more problem (ie. deleting the entire operation system)

5.  First mount this partition on /mnt

    sudo mount /dev/sda2 /mnt

6.  Then copy the rootfs.tar.gz to this location

    sudo cp images/linux/rootfs.tar.gz /mnt/

7.  Go to that folder
    nautilus /mnt/

8.  Open up a new terminal run below commands:
    sudo tar -xvzf rootfs.tar.gz
    sudo rm rootfs.tar.gz; sudo umount --lazy /mnt

9.  Eject the SD Card from file explorer

10. Plug in the SD Card to the carrier board and power it.

## Petalinux Operations

login: petalinux
password : asks you to enter

---

If everything is done properly:

- sudo lspci -v
- sudo lsblk -f

Should return proper output regarding the nvme device.

When the nvme is formatted on windows, it creates four partition where the last two partitions are not recognized by petalinux.

So I have done below test to validate simple read/write operations.

1.  Unmount any mounted partitions
    sudo umount /run/media/nvme0n1p1

2.  sudo fdisk /dev/nvme0n1
    Delete every other partition
    d
    1
    d
    2
    d
    3
    d

3.  Create a basic single partition
    n
    Partition 1
    First sector (default)
    Last sector 1000000 -> 510 MB (do not create a single huge partition)
    Erase older format signature
    w   (DO NOT FORGET THIS, it saves it, at this step)

4.  Format new partition
    sudo mkfs.ext4 /dev/nvme0n1p1

5.  Mount it and create a file inside the nvme
    sudo mount /dev/nvme0n1p1 /mnt

6.  Create a test file
    cd /mnt
    sudo mkdir test; cd test
    sudo touch test.txt
    sudo vi test.txt
    esc and :q! -> do not save and exit
    esc and :wq -> save and exit

7.  sudo umount /dev/nvme0n1p1

8.  sudo shutdown now

9.  Connect the nvme device to your host PC. Check if the text.txt exists and... DONE!!!

**NOTE**: Creating a 1TB single partition and formatting it to ext4, crashed the petalinux.