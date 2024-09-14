# ZCU102 PetaLinux Setup

This repository provides scripts and detailed instructions for configuring, building, and flashing PetaLinux using a custom exported hardware design.

Tested on Ubuntu 22.04.

## Setup Instructions

### 1. Download PetaLinux
- Download the PetaLinux installer from the official [Xilinx website (PetaLinux 2022.1)](https://www.xilinx.com/member/forms/download/xef.html?filename=petalinux-v2022.1-04191534-installer.run). This guide uses version 2022.1.

### 2. Prepare the Installation Directory
- Copy the installer to the desired installation folder. In this guide, we use `/opt/Xilinx/Petalinux` as the target directory.

### 3. Install PetaLinux
- First, install the required prerequisite packages:

    ```bash
    sudo apt-get install gcc-multilib

    # Add support for i386 packages
    sudo dpkg --add-architecture i386

    sudo apt-get install zlib1g:i386

    sudo apt-get install texinfo
    ```

> **Note**: Depending on your system, additional packages may be required.

### 4. Install PetaLinux
- **Important**: The PetaLinux installer will extract its files into the **current directory** without creating subdirectories. Make sure you are in the desired target directory before running the installer.

To install PetaLinux, run the following commands:

```bash
chmod +x petalinux-v2022.1-04191534-installer.run
./petalinux-v2022.1-04191534-installer.run
```
### 4. Configure PetaLinux

- To configure, build, and flash PetaLinux, you must first source the PetaLinux environment setup script:

    ```bash
    source /opt/Xilinx/Petalinux/settings.sh
    ```

- Navigate to the directory where you want to create your project and run the following commands:

    ```bash
    cd projectfolder
    source /opt/Xilinx/Petalinux/settings.sh

    # For Ultrascale+ devices, use 'zynqMP', and for Zynq-7000, use 'zynq'
    petalinux-create --type project --template zynqMP --name projectName
    cd projectfolder
    ```

---

### 5. Import Hardware Description and Customize the Build

- Import your exported hardware description file (refer to the other repository for details):

    ```bash
    petalinux-config --get-hw-description zcu_102_sgmii_sfp.xsa
    ```

- Now, you can customize your PetaLinux build. For instance, to select the `axi_eth` network interface, navigate through the configuration menu:

    ```text
    Subsystem AUTO Hardware Settings -> Ethernet Settings -> Primary Ethernet -> axi_eth
    ```

    ![AXI_ETH](documentation/figures/screenshot_axi_eth.png)

- To adjust the root filesystem settings, navigate to:

    ```text
    Image Packaging Configuration -> Root Filesystem -> Select Type (EXT4 (SD/eMMC/SATA/USB))
    ```

    - For the device node, select `/dev/mmcblk0p2`, which represents the second partition on the SD card. (You may need to insert the SD card and manually create the two partitions, as detailed in Step 10.) With this selection all the data created within the petalinux instance is stored on the second SD-card partition.

- If you do **not** wish to use a TFTP server, disable the option to *"Copy final images to tftpboot directory"* by pressing `n`.


![ROOT_FS](documentation/figures/root_fs_selection.png)

Finally press Save and Exit.


### 6. Configure U-Boot for SD Card Support

- To configure U-Boot, run the following command:

    ```bash
    petalinux-config -c u-boot
    ```

- Enable SD Card support by navigating through the menu:

    ```text
    Boot Options -> Media -> SD Support for booting from SD/EMMC
    ```

![ROOT_FS](documentation/figures/sd_card_support.png)

- Press `y` to enable SD Card support.


### 7. Add a Custom Application

You can add a custom application to your PetaLinux project by creating a new one with the following command:

```bash
$ petalinux-create -t apps --template c++ --name myapp --enable
```
This command creates an application named *myapp.* Apps must be in lowercase letters otherwise the warning: 

```
WARNING: QA Issue: PN: embeddedRTPS is upper case, this can result in unexpected behavior. [uppercase-pn]
```

pops up. 

The relevant files will be located in the following directory:

```bash
./project-spec/meta-user/recipes-apps/myapp
```

Within the `myapp.bb` file (the recipe file for the application), you can specify various settings for your app. Below is an example of the contents:
```c++
#
# This file is the myapp recipe.
#

SUMMARY = "Simple emyapp application"
SECTION = "PETALINUX/apps"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "file://myapp.cpp \
           file://Makefile \
		  "

S = "${WORKDIR}"

do_compile() {
	     oe_runmake
}

do_install() {
	     install -d ${D}${bindir}
	     install -m 0755 myapp ${D}${bindir}
}
```

In the files directory, you will find `myapp.cpp` and a `Makefile` that you can modify to configure your application.

### 7 . Configure the kernel

By executing the following command, the linux kernel can be configured:

```bash
$ petalinux-config -c kernel
```

For now we do not change anything here and just keep the default values. Anyway it is worth to scroll through the menues. 


### 6. Configure Root Filesystem (RootFS)

To configure the root filesystem, run the following command:

```bash
$ petalinux-config -c rootfs
```

To add our previously created application go to:

```bash
Under apps -> myapp 
```

Here, you can select the previously created `myapp` application, which will then be automatically compiled and included in the build image.

![rootfs_myapp](documentation/figures/select_myapp_rootfs.png)

Additionally, you can select other packages under PetaLinux Package Groups, such as support for X11, Qt, or Python.

### 7. Build PetaLinux

No as we have configured all settings, you can build Petalinux by executing

```bash
$ petalinux-build

[INFO] Sourcing buildtools
[INFO] Building project
[INFO] Sourcing build environment
[INFO] Generating workspace directory
INFO: bitbake petalinux-image-minimal
NOTE: Started PRServer with DBfile: /home/florianfrank/Documents/asoa_package_filtering_root/zcu_102_petalinux_setup/zynq_netboot/build/cache/prserv.sqlite3, Address: 127.0.0.1:43887, PID: 275754
WARNING: Host distribution "ubuntu-22.04" has not been validated with this version of the build system; you may possibly experience unexpected failures. It is recommended that you use a tested distribution.
Loading cache: 100% |##############################################################################################################################################################| Time: 0:00:00
Loaded 5391 entries from dependency cache.
Parsing recipes: 100% |############################################################################################################################################################| Time: 0:00:00
Parsing of 3594 .bb files complete (3589 cached, 5 parsed). 5396 targets, 509 skipped, 0 masked, 0 errors.
NOTE: Resolving any missing task queue dependencies
#########################################################################################################################################################| Time: 0:00:06
INFO: Successfully copied built images to tftp dir: /var/lib/tftpboot
[INFO] Successfully built project
```

You should see the following output. In this example, the TFTP boot option was enabled, which automatically copies the build artifacts to `/var/lib/tftpboot`.

Additionally, a directory `/images/linux/` should now be present, containing all the necessary build files.

### 9. Create a bootable package

In a next step all files to boot the system must be created. To do this the following command must be executed: 

```bash
$ petalinux-package --boot --u-boot images/linux/u-boot.elf --dtb images/linux/system.dtb --fsbl images/linux zynqmp_fsbl.elf --fpga images/linux/system.bit
```

This command creates a `BOOT.BIN`it will fail if you rebuild the system and a BOOT.BIN is already available. 
Therefore you can simply excute the script: 

```bash
$ ../build_boot.sh
```

In your default project folder. It will remove the previous `BOOT.bin` and create a new one.



### 8. Booting the device

There are several options for booting your device. We will cover three methods: 

1. Using JTAG and the hardware server.
2. Booting from an SD card.
3. Booting via TFTP with NFS as the root filesystem.

#### 8.1 Boot Petalinux via JTAG 

When booting the device via JTAG, ensure you are in the current project folder.

First, verify that Vivado is not running with an active connection to the hardware server. If Vivado is open and connected, navigate to the **Hardware Manager** and disconnect the device.

![alt text](documentation/figures/hw_server_close.png)

Afterwards start your hardware server by executing

```bash
$ /opt/Xilinx/Petalinux/tools/xsct/bin/hw_server
```

Do this in a seperate tab because the application blocks the terminal. 

Afterwards set the ZCU102 in JTAG mode by setting the Pins SW6 to 0000. 

![alt text](documentation/figures/switch_JTAG.png)


Afterwards return to the project folder and execute: 

```bash
$ petalinux-boot --jtag --kernel --hw_server-url TCP:127.0.0.1:3121
```

### Prepare SD Card image


```bash 


```



1. Image packageing in petalinux-config



1. Go to vitis
-> new application -> select hardware export
-> select r5 and next r5_0
-> Select OpenAMP echo test






petalinux-package --boot --fsbl images/linux/zynqmp_fsbl.elf --fpga images/linux/system.bit  --u-boot