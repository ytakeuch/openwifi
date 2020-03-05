# openwifi
<img src="./openwifi-arch.jpg" width="900">

**openwifi:** Linux mac80211 compatible full-stack IEEE802.11/Wi-Fi design based on SDR (Software Defined Radio).

This repository includes Linux driver and software. [openwifi-hw](https://github.com/open-sdr/openwifi-hw) repository has the FPGA design. [[Project document](https://github.com/open-sdr/openwifi/tree/master/doc)]

[Demo [video](https://youtu.be/NpjEaszd5u4) and video [download](https://users.ugent.be/~xjiao/openwifi-low-aac.mp4)]   [openwifi [maillist](https://lists.ugent.be/wws/subscribe/openwifi)] [[Cite openwifi project](#Cite-openwifi-project)]

Openwifi code has dual licenses. AGPLv3 is the opensource license. For non-opensource license, please contact Filip.Louagie@UGent.be. Openwifi project also leverages some 3rd party modules. It is user's duty to check and follow licenses of those modules according to the purpose/usage. You can find [an example explanation from Analog Devices](https://github.com/analogdevicesinc/hdl/blob/master/LICENSE) for this compound license conditions. [[How to contribute]](https://github.com/open-sdr/openwifi/blob/master/CONTRIBUTING.md). 

**Features:**

- 802.11a/g
- 802.11n MCS 0~7 (Only PHY rx for now. Full system support of 802.11n will come soon)
- 20MHz bandwidth; 70 MHz to 6 GHz frequency range
- Mode tested: Ad-hoc; Station; AP, Monitor
- DCF (CSMA/CA) low MAC layer in FPGA (10us SIFS is achieved)
- Configurable channel access priority parameters:
  - duration of RTS/CTS, CTS-to-self
  - SIFS/DIFS/xIFS/slot-time/CW/etc
- Time slicing based on MAC address
- Easy to change bandwidth and frequency: 
  - 2MHz for 802.11ah in sub-GHz
  - 10MHz for 802.11p/vehicle in 5.9GHz
- On roadmap: **802.11ax**

**Performance (AP: openwifi at channel 44, client: TL-WDN4200 N900 USB Dongle):**
- AP --> client: 30.6Mbps(TCP), 38.8Mbps(UDP)
- client --> AP: 17.0Mbps(TCP), 21.5Mbps(UDP)

**Supported SDR platforms:** (Check [Porting guide](#Porting-guide) for your new board if it isn't in the list)

board_name|board combination|status
-------|-------|----
zc706_fmcs2|Xilinx ZC706 dev board + FMCOMMS2/3/4|Done
zed_fmcs2|Xilinx zed board + FMCOMMS2/3/4|Done
adrv9364z7020|ADRV9364Z7020 SOM + ADRV1CRR-BOB carrier board|Done
adrv9361z7035|ADRV9361Z7035 SOM + ADRV1CRR-BOB carrier board|Done
adrv9361z7035_fmc|ADRV9361Z7035 SOM + ADRV1CRR-FMC carrier board|Done
zc702_fmcs2|Xilinx ZC702 dev board + FMCOMMS2/3/4|Done
zcu102_fmcs2|Xilinx ZCU102 dev board + FMCOMMS2/3/4|Future
zcu102_9371|Xilinx ZCU102 dev board + ADRV9371|Future

- board_name is used to identify FPGA design in openwifi-hw/boards/
- Don't have any boards? Or you like JTAG boot instead of SD card? Check our test bed [w-iLab.t](https://doc.ilabt.imec.be/ilabt/wilab/tutorials/openwifi.html) tutorial.

[[Quick start](#Quick-start)]
[[Basic operations](#Basic-operations)]
[[Update FPGA](#Update-FPGA)]
[[Update Driver](#Update-Driver)]
[[Update sdrctl](#Update-sdrctl)]
[[Easy Access and etc](#Easy-Access-and-etc)]

[[Build openwifi Linux img from scratch](#Build-openwifi-Linux-img-from-scratch)]
[[Special note for 11b](#Special-note-for-11b)]
[[Porting guide](#Porting-guide)]
[[Cite openwifi project](#Cite-openwifi-project)]

## Quick start
- Burn [openwifi image](https://users.ugent.be/~xjiao/openwifi-1.1.0-taiyuan.img.xz) into a SD card (Double clikc in Ubuntu or "Open With Disk Image Writer"). You can see two partitions (BOOT and rootfs) when you insert the SD card to your PC. You need to use **correct files in the BOOT partition** according to the **platform you have**. Just **overwrite** the files in the base directory with the files in **board_name** directory of BOOT partiton.
- Connect two antennas to RXA/TXA ports. Config the board to SD card boot mode (check your board manual). Insert the SD card to the board. 
- Power on. login to the board from your PC (PC Ethernet should have IP 192.168.10.1) with one time password **analog**.
  ```
  ssh root@192.168.10.122
  ```
- Setup routing/NAT **on the PC** for the board -- this is **important** for post installation/config.
  ```
  sudo sysctl -w net.ipv4.ip_forward=1
  sudo iptables -t nat -A POSTROUTING -o ethY -j MASQUERADE
  sudo ip route add 192.168.13.0/24 via 192.168.10.122 dev ethX
  ```
  **ethX** is the PC NIC name connecting the board. **ethY** is the PC NIC name connecting internet.
  
  If you want, uncommenting "net.ipv4.ip_forward=1" in /etc/sysctl.conf to make IP forwarding persistent on PC.
- Run **one time** script on board to complete post installation/config (After this, password becomes **openwifi**)
  ```
  cd ~/openwifi && ./post_config.sh
  ```
- Run openwifi AP together with the on board webserver
  ```
  ~/openwifi/fosdem.sh
  ```
- After you see the "openwifi" SSID on your device (Phone/Laptop/etc), connect it. Browser to 192.168.13.1 on your deivce, you should see the webpage hosted by the webserver on board.
  - Note 1: If your device doesn't support 5GHz (ch44), please change the **hostapd-openwifi.conf** on board and re-run fosdem.sh.
  - Note 2: After ~2 hours, the Viterbi decoder will halt (Xilinx Evaluation License). Just power cycle the board if it happens.

## Basic operations
The board actually is an Linux/Ubuntu computer which is running **hostapd** to offer Wi-Fi AP functionality over the Wi-Fi Network Interface (NIC). The NIC is implemented by openwifi-hw FPGA design. We use the term **"On board"** to indicate that the commands should be executed after ssh login to the board. **"On PC"** means the commands should run on PC.
- Bring up the openwifi NIC sdr0:
  ```
  cd ~/openwifi && ./wgd.sh
  ```
- Use openwifi as client to connect other AP (Change wpa-connect.conf on board firstly):
  ```
  route del default gw 192.168.10.1
  wpa_supplicant -i sdr0 -c wpa-connect.conf &
  dhclient sdr0
  ```
- Use openwifi in ad-hoc mode: Please check **sdr-ad-hoc-up.sh** and **sdr-ad-hoc-join.sh**.
- Use openwifi in monitor mode: Please check **monitor_ch.sh**.
- The Linux native Wi-Fi tools/Apps (iwconfig/ifconfig/iwlist/iw/hostapd/wpa_supplicant/etc) can run over openwifi NIC in the same way as commercial Wi-Fi chip. 
- **sdrctl** is a dedicated tool to access openwifi driver/FPGA, please check doc directory for more information. 

## Update FPGA

Since the pre-built SD card image might not have the latest bug-fixes/updates, it is recommended to udpate the fpga bitstream on board.

- Install Vivado/SDK 2017.4.1 (If you don't need to generate new FPGA bitstream, WebPack version without license is enough)
- Setup environment variables (use absolute path):
  ```
  export XILINX_DIR=your_Xilinx_directory
  export OPENWIFI_DIR=your_openwifi_directory
  export BOARD_NAME=your_board_name
  ```
- Get the latest FPGA bitstream from openwifi-hw, generate BOOT.BIN and transfer it on board via ssh channel:
  ```
  $OPENWIFI_DIR/user_space/get_fpga.sh $OPENWIFI_DIR
  $OPENWIFI_DIR/user_space/boot_bin_gen.sh $OPENWIFI_DIR $XILINX_DIR $BOARD_NAME
  scp $OPENWIFI_DIR/kernel_boot/boards/$BOARD_NAME/output_boot_bin/BOOT.BIN root@192.168.10.122:
  ```
- On board: Put the BOOT.BIN into the BOOT partition.
  ```
  mount /dev/mmcblk0p1 /mnt
  cp ~/BOOT.BIN /mnt
  umount /mnt
  ```
  **Power cycle** the board to load new FPGA bitstream.

## Update Driver

Since the pre-built SD card image might not have the latest bug-fixes/updates, it is recommended to udpate the driver on board.
- Prepare Analog Devices Linux kernel source code (only need to run once):
  ```
  $OPENWIFI_DIR/user_space/prepare_kernel_src.sh $OPENWIFI_DIR $XILINX_DIR
  ```
- Compile the latest openwifi driver
  ```
  $OPENWIFI_DIR/driver/make_all.sh $OPENWIFI_DIR $XILINX_DIR
  ```
- Copy the driver files to the board via ssh channel
  ```
  scp `find $OPENWIFI_DIR/driver/ -name \*.ko` root@192.168.10.122:openwifi/
  ```
  Now you can use **wgd.sh** on board to load the new openwifi driver.

## Update sdrctl
- Copy the sdrctl source files to the board via ssh channel
  ```
  scp `find $OPENWIFI_DIR/user_space/sdrctl_src/ -name \*.*` root@192.168.10.122:openwifi/sdrctl_src/
  ```
- Compile the sdrctl **on board**:
  ```
  cd ~/openwifi/sdrctl_src/ && make && cp sdrctl ../ && cd ..
  ```
## Easy Access and etc

- FPGA and driver on board update scripts
  - Setup [ftp server](https://help.ubuntu.com/lts/serverguide/ftp-server.html) on PC, allow anonymous and change ftp root directory to $OPENWIFI_DIR.
  - On board:
  ```
  ./sdcard_boot_update.sh $BOARD_NAME
  (Above command downloads uImage, BOOT.BIN and devicetree.dtb, then copy them into boot partition. Remember to power cycle)
  ./wgd.sh remote
  (Above command downloads driver files, and brings up sdr0)
  ```
- Access the board disk/rootfs like a disk: 
   - On PC: "File manager --> Connect to Server...", input: sftp://root@192.168.10.122/root
   - Input password "openwifi"

## Build openwifi Linux img from scratch
- Download [2017_R1-2018_01_29.img.xz](http://swdownloads.analog.com/cse/2017_R1-2018_01_29.img.xz) from [Analog Devices Wiki](https://wiki.analog.com/resources/tools-software/linux-software/zynq_images). Burn it to a SD card.
- Insert the SD card to your Linux PC. Find out the mount point (that has two sub directories BOOT and rootfs), and setup environment variables (use absolute path):
  ```
  export SDCARD_DIR=sdcard_mount_point
  export XILINX_DIR=your_Xilinx_directory
  export OPENWIFI_DIR=your_openwifi_directory
  export BOARD_NAME=your_board_name
  ```
- Run script to update SD card:
  ```
  $OPENWIFI_DIR/user_space/update_sdcard.sh $OPENWIFI_DIR $XILINX_DIR $BOARD_NAME $SDCARD_DIR
  ```
- Now you can start from [Quick start](#Quick-start)

## Special note for 11b

Openwifi only applies OFDM as its modulation scheme and as a result, it is not backward compatible with 802.11b clients or modes of operation. This is usually the case during beacon transmission, connection establishment, and robust communication.

As a solution to this problem, openwifi can be fully controlled only if communicating with APs/clients instantiated using hostapd/wpa_supplicant userspace programs respectively.

For hostapd program, 802.11b rates can be suppressed using configuration commands (i.e. supported_rates, basic_rates) and an example configuration file is provided (i.e. hostapd-openwifi.conf). One small caveat to this one comes from fullMAC Wi-Fi cards as they must implement the *NL80211_TXRATE_LEGACY* NetLink handler at the device driver level.

On the other hand, the wpa_supplicant program on the client side (commercial Wi-Fi dongle/board) cannot suppress 802.11b rates out of the box in 2.4GHz band, so there will be an issue when connecting openwifi (OFDM only). A patched wpa_supplicant should be used at the client side.
```
$OPENWIFI_DIR/user_space/build_wpa_supplicant_wo11b.sh $OPENWIFI_DIR
```
## Porting guide

This section explains the porting work by showing the differences between openwifi and Analog Devices reference design. We use **2018_r1** of [HDL Reference Designs](https://github.com/analogdevicesinc/hdl).
- Open the fmcomms2 + zc706 reference design at hdl/projects/fmcomms2/zc706 (Please read Analog Devices help)
- Open the openwifi design zc706_fmcs2 at openwifi-hw/boards/zc706_fmcs2 (Please read openwifi-hw repository)
- "Open Block Design", you will see the differences between openwifi and the reference design. Both in "diagram" and in "Address Editor".
- The address/interrupts of FPGA blocks hooked to the ARM bus should be put/aligned to the devicetree file openwifi/kernel_boot/boards/zc706_fmcs2/devicetree.dts. Linux will parse the devicetree.dtb when booting to know information of attached deivce (FPGA blocks in our case).
- We use dtc command to get devicetree.dts converted from devicetree.dtb in [Analog Devices Linux image](https://wiki.analog.com/resources/tools-software/linux-software/zynq_images), then do modification according to what we have added/modified to the reference design.
- Please learn the script in [[Build openwifi Linux img from scratch](#Build-openwifi-Linux-img-from-scratch)] to understand how we generate devicetree.dtb, BOOT.BIN and Linux kernel uImage and put them together to build the full SD card image.

## Cite openwifi project

Any use of openwifi project which results in a publication should include a citation via (bibtex example):
```
@electronic{openwifigithub,
            author = {Xianjun, Jiao and Wei, Liu and Michael, Mehari},
            title = {open-source IEEE802.11/Wi-Fi baseband chip/FPGA design},
            url = {https://github.com/open-sdr/openwifi},
            year = {2019},
}
```
Openwifi was born in [ORCA project](https://www.orca-project.eu/) (EU's Horizon2020 programme under agreement number 732174).
