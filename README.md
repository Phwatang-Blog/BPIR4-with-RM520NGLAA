# BPIR4-with-RM520NGLAA
 A guide to setup the Banana Pi BPI-R4 with a Quectel RM520N-GLAA modem to establish a connection to the internet

The guide assumes a fresh install of OpenWRT (v24.10.0 at time of writing) on the BPI-R4 and that the RM520N-GLAA has been installed correctly.

## End Goal
By the end, we want to have a working cellular modem "box" whose main goal is to provide an internet connection to the main router of a home network.

Main steps:
1. Configure OpenWRT to communicate with the modem
2. Establish a cellular connection and map it to be the WAN interface
3. Install basic diagnostics and utilities to monitor the Bananapi and the modem
4. Create a "management" port to adjust settings of the Bananapi from another network

## 1. Communicate With The Modem
Substeps:
1. If the modem is in **QMI mode**, install the following packages:
    - kmod-usb-net-qmi-wwan 
    - kmod-usb-serial-option
    - uqmi

   If the modem is in **MBIM mode**, install the following packages:
    - kmod-usb-net-cdc-mbim 
    - kmod-usb-serial-option
    - umbim
   
   Both sets of packages can be installed without interfering with each other. Feel free to install both sets
   if unsure which mode your modem is currently in.
2. Install the following packages:
    - modemmanager
    - luci-proto-modemmanager
3. Perform reboot
4. At this point, `ls /dev` should show various `ttyUSBx` and a `cdc-wdm0`.
    - Example succesful output:
      ```
      root@OpenWrt:~# ls /dev
      bus           i2c-2         loop4         mmcblk0p4     port          ttyS1         ttyS6         ubi0_2
      cdc-wdm0      i2c-3         loop5         mmcblk0p5     ppp           ttyS10        ttyS7         ubi0_3
      console       i2c-4         loop6         mmcblk0rpmb   ptmx          ttyS11        ttyS8         ubi0_4
      fd            i2c-5         loop7         mtd0          pts           ttyS12        ttyS9         ubi0_5
      fit0          kmsg          mmcblk0       mtd0ro        random        ttyS13        ttyUSB0       ubi0_6
      fitrw         log           mmcblk0boot0  mtd1          shm           ttyS14        ttyUSB1       ubi_ctrl
      full          loop-control  mmcblk0boot1  mtd1ro        stderr        ttyS15        ttyUSB2       ubiblock0_4
      gpiochip0     loop0         mmcblk0p1     mtdblock0     stdin         ttyS2         ttyUSB3       urandom
      hwrng         loop1         mmcblk0p128   mtdblock1     stdout        ttyS3         ubi0          watchdog
      i2c-0         loop2         mmcblk0p2     null          tty           ttyS4         ubi0_0        watchdog0
      i2c-1         loop3         mmcblk0p3     pmsg0         ttyS0         ttyS5         ubi0_1        zero
      ```

## 2. Establish Cellular Connection as WAN
Substeps:
1. Delete the default `wan6` interface. IPv6 on the WAN will be handled by modemmanager.
2. Modify the `wan` interface to use `ModemManager` as the protcol:
   ![](Step2_2.png)
3. Adjust appropriate settings. Save and apply.
   ![](Step2_3.png)
4. At this point, OpenWRT should be using the modem as its internet source.

   If not, possibly perform a reboot. Also check system logs to see if modemmanager ran into an issue. The utilities in the next step will help to diagnose and resolve issues.

## 3. Install Diagnostics and Utilities
Substeps:
1. Add [4IceG's repository](https://github.com/4IceG/Modem-extras) to opkg
2. Update opkg and install the following packages:
    - luci-app-modemband
       - Allows easy adjustment for what cellular bands the modem should search for
    - luci-app-3ginfo-lite
       - Give basic information about the current cellular connection
    - luci-app-atcommands
       - Allows for easy sending of AT commands to the modem
   
   A new `Modem` tab will appear at the top of the Luci interface:
   ![](Step3_2.png)
3. We will now configure the packages we've just installed to communicate with the modem. 

   For each section under ```Modem```, goto ```Configuration``` and set (if present):
    - Interface to `wan`
    - Communication port to `/dev/ttyUSB2`

   Example configuration for 3g-info-lite:
   ![](Step3_3.png)

## 4. Creating a Management Port
The management port will be used such that the Bananapi can be configured (ssh/http/... etc) from another network. Feel free to skip this step. If you don't know why it would be useful, then you probably don't need it.

There are many ways to do this. We will do this the simple way by setting up one of the Bananapi's ports to use DHCP and adjust the lan subnet range if necessary.

Substeps:
1. Select a (port) Device to use as the management interface (and remove it from `br-lan` if necessary). 

   We will use `Switch port: "wan"` going forward. (This was the port used for the wan interface before being replaced with ModemManager).

2. Add a new firewall zone for the management port. Set both `input` and `output` to `accept`.
   ![](Step4_2.png)

3. Add a new interface for this device and set the protcol to DHCP
   ![](Step4_3.png)

   Under `Advanced Settings` of the new interface, uncheck the following:
    - `Use default gateway`
    - `Use DNS server advertised by peer`
4. Check the subnet range of the management network the management interface will be plugged into.

   Adjust the `lan` interface to work in a different subnet. 
    - E.g if the management network operates in 192.168.1.1/24, adjust the `lan` interface to operate in 192.168.30.1/24
    - Don't forget to access the Bananapi through its new ip address after the change

   Example adjustment of `lan` interface:
   ![](Step4_4.png)

5. Add a new traffic rule to reject any traffic coming from `lan` zone going to `Device` zone on ports:
    - 22 (ssh)
    - 80 (http)
    - 443 (https)
   ![](Step4_5.png)

## Other Things To Look Out For
Some other things to check for if the guide isn't working:
 - Ensure the modem is using sim slot 1
    - `AT+QUIMSLOT?` should output a 1
    - slot 1 is for physical sims, slot 2 is for esim?
 - Ensure sim pin detection is disabled
    - `AT+QSIMDET?` will output a 2 number tuple. First number should be 0.

## Basic Checks
 - Check usbspeed
    - AT+QCFG="usbspeed"
 - Check data protocol
    - AT+QCFG="usbnet"
 - Check sleep mode
    - AT+QSCLK?
 - Check band information
    - AT+QENG="servingcell"
    - AT+QNWCFG=?
    - AT+QNWPREFCFG="lte_band"
    - AT+QNWPREFCFG="nsa_nr5g_band"

## AT Commands Not In Manual
There are many AT commands not specified in the manual.

Some useful hidden commands are given by the AT command: `AT+QNWCFG=?`.
 - The commands `AT+QNWCFG: "nr5g_pref_freq_list",(0-32),<EARFCN_list>
` and `AT+QNWCFG: "lte_pref_freq_list",(0-32),<EARFCN_list>` allow for specifying a band priority hierachy
    - Upon further testing, these commands appear to do nothing when applied. I.e querying the frequency list before and after modifications gives no change

## Resources
Online guides/forums/manuals used:
 - https://openwrt.org/docs/guide-user/network/wan/wwan/ltedongle
    - General cellular setup
 - https://ten64doc.traverse.com.au/applications/cellular/
    - Setting WAN interface to be the modem
 - https://forum.openwrt.org/t/looking-for-a-guide-on-configuring-lte-band-aggregation/186063
    - Setting cellular band preferences
 - https://forum.openwrt.org/t/luci-app-luci-app-3ginfo-3ginfo-gui-info-about-3g-lte-connection/67893
    - Displaying cellular signal metrics
 - https://forums.quectel.com/uploads/short-url/x7O7XpZA1jJboG9xct7VxaeWLXx.pdf
    - RM520N-GL AT Command Manual

Repos/packages:
 - https://github.com/4IceG/Modem-extras
    - 4IceG's repo
 - https://forums.quectel.com/t/rm502q-ae-band-preference-command/15630/8
    - Hidden band prioritisation commands
 - https://github.com/gSpotx2f/luci-app-cpu-status-mini
    - luci-app-cpu-status-mini
 - https://github.com/4IceG/luci-app-modemband
    - luci-app-modemband
 - https://github.com/4IceG/luci-app-3ginfo-lite
    - luci-app-3ginfo
 - https://github.com/4IceG/luci-app-atcommands
    - luci-app-atcommands

## Updating Modem Firmware
Firmware can be updated through Quectel's QFirehose utility.

If you have a USB to m.2 NGFF adapter, then you have the easy option of connecting the modem into the adapter and plugging the adapter into a proper computer.

### Updating Directly From BPIR4
I used the BPiR4 to perform the firmware update.

At time of writing, the [opkg qfirehose](https://openwrt.pkgs.org/23.05/openwrt-packages-aarch64_cortex-a72/qfirehose_1.4.9-1_aarch64_cortex-a72.ipk.html) has not been updated to OpenWRT 24.x so most of this procedure has to be done through the terminal.

The general procedure is to:
1. Download a qfirehose zip onto the bpi
2. Unzip and grant executible permissions with `chmod 775`
3. Download latest firmware for RM520N
4. Execute QFirehose and tell it to use the firmware downloaded in step 3

See method 2 of this [guide](https://docs.gl-inet.com/router/en/4/tutorials/upgrade_quectel_module_software/) for more details.

See [4IceG's repo](https://github.com/4IceG/RM520N-GL) for the latest firmware.

## A Sour Note on RM520N-GLAP
If step 1 of the guide seems to not work, you may have ended up purchasing a **RM520N-GLAP** instead of the **RM520N-GLAA**.
 - The AP version does not have USB communication support whereas the AA version does
 - Product listing for the modem is frequently abreviated to **RM520N-GL** so double-check the listing details for the variant

I burnt a few days trying to get the AP version to work with no success. I eventually folded and purchased the AA version.

Incase anyone finds this helpful, here were my findings:
 - Install the `mhi-pci-generic` package. This gets the drivers for communication to the modem. Output of `lspci -v` should look like:
   ```
   root@OpenWrt:~# lspci -v
   ...
   ...

   0003:01:00.0 Unassigned class [ff00]: Qualcomm Technologies, Inc Device 0308
         Subsystem: Qualcomm Technologies, Inc Device 0308
         Flags: bus master, fast devsel, latency 0, IRQ 123
         Memory at 20200000 (64-bit, non-prefetchable) [size=4K]
         Memory at 20201000 (64-bit, non-prefetchable) [size=4K]
         Capabilities: [40] Power Management version 3
         Capabilities: [50] MSI: Enable+ Count=8/32 Maskable+ 64bit+
         Capabilities: [70] Express Endpoint, IntMsgNum 0
         Capabilities: [100] Advanced Error Reporting
         Capabilities: [148] Secondary PCI Express
         Capabilities: [168] Physical Layer 16.0 GT/s <?>
         Capabilities: [18c] Lane Margining at the Receiver
         Capabilities: [19c] Transaction Processing Hints
         Capabilities: [228] Latency Tolerance Reporting
         Capabilities: [230] L1 PM Substates
         Capabilities: [240] Data Link Feature <?>
         Kernel driver in use: mhi-pci-generic
   ```
 - Use the [GFriend Custom OpenWRT Image](https://github.com/mdsdtech/GFriendWRT/releases/tag/V060225-R4) as a tool for sending AT commands
   
   Preview:
   ![](GFriend%20AT%20command.PNG)

 - Ensure to disable sim pin detection on the modem
    - AT+QSIMDET=0,0

From these I was able to get a `wwan0mbim0`, `wwan0qcdm0` and `wwan0qmi0` to appear in the output of `ls /dev/`.

I think I was also able to get the modem to successfully establish a cellular connection. Some of my system logs were:
```
Fri Apr 25 03:56:16 2025 daemon.info [2736]: <inf> [modem0] processing user request to register modem...
Fri Apr 25 03:56:16 2025 daemon.info [2736]: <inf> [modem0] already registered automatically in network '....', automatic registration not launched...
Fri Apr 25 03:56:16 2025 daemon.info [2736]: <inf> [modem0] modem registered
Fri Apr 25 03:56:16 2025 daemon.notice netifd: wan (5402): successfully registered the modem
Fri Apr 25 03:56:16 2025 daemon.notice netifd: wan (5402): starting connection with apn '....'...
Fri Apr 25 03:56:16 2025 daemon.info [2736]: <inf> [modem0] processing user request to connect modem...
Fri Apr 25 03:56:16 2025 daemon.info [2736]: <inf> [modem0]   apn: ....
Fri Apr 25 03:56:16 2025 daemon.info [2736]: <inf> [modem0]   ip type: ipv4v6
Fri Apr 25 03:56:16 2025 daemon.info [2736]: <inf> [modem0]   allowed auth: unknown
Fri Apr 25 03:56:16 2025 daemon.info [2736]: <inf> [modem0]   allow roaming: yes
Fri Apr 25 03:56:16 2025 daemon.notice [2736]: <msg> [modem0] simple connect started...
Fri Apr 25 03:56:16 2025 daemon.notice [2736]: <msg> [modem0] simple connect state (6/10): register
Fri Apr 25 03:56:16 2025 daemon.info [2736]: <inf> [modem0] already registered automatically in network '....', automatic registration not launched...
Fri Apr 25 03:56:16 2025 daemon.notice [2736]: <msg> [modem0] simple connect state (7/10): wait to get packet service state attached
Fri Apr 25 03:56:16 2025 daemon.notice [2736]: <msg> [modem0] simple connect state (8/10): bearer
Fri Apr 25 03:56:16 2025 daemon.notice [2736]: <msg> [modem0] simple connect state (9/10): connect
Fri Apr 25 03:56:16 2025 daemon.notice [2736]: <msg> [modem0] state changed (registered -> connecting)
Fri Apr 25 03:56:16 2025 daemon.notice [2736]: <msg> [modem0] state changed (connecting -> connected)
Fri Apr 25 03:56:16 2025 daemon.notice [2736]: <msg> [modem0] simple connect state (10/10): all done
Fri Apr 25 03:56:16 2025 daemon.notice netifd: wan (5402): successfully connected the modem
Fri Apr 25 03:56:16 2025 daemon.notice netifd: wan (5402): signal refresh rate is not set
Fri Apr 25 03:56:16 2025 daemon.notice netifd: wan (5402): network operator name: ....
Fri Apr 25 03:56:16 2025 daemon.notice netifd: wan (5402): network operator MCCMNC: ....
Fri Apr 25 03:56:16 2025 daemon.notice netifd: wan (5402): registration type: home
Fri Apr 25 03:56:16 2025 daemon.notice netifd: wan (5402): access technology: lte, 5gnr
Fri Apr 25 03:56:16 2025 daemon.notice netifd: wan (5402): signal quality: 30%
Fri Apr 25 03:56:16 2025 daemon.notice netifd: wan (5402): IPv4 connection setup required in interface wan: static
Fri Apr 25 03:56:16 2025 daemon.notice netifd: wan (5402): adding IPv4 address ...., netmask 255.255.255.252
Fri Apr 25 03:56:16 2025 daemon.notice netifd: wan (5402): adding default IPv4 route via ....
Fri Apr 25 03:56:16 2025 daemon.notice netifd: wan (5402): adding primary DNS at ....
Fri Apr 25 03:56:16 2025 daemon.notice netifd: wan (5402): adding secondary DNS at ....
Fri Apr 25 03:56:16 2025 daemon.notice netifd: Interface 'wan' is now up
Fri Apr 25 03:56:16 2025 daemon.notice netifd: Network device 'mhi_hwip0' link is up
```

However, ModemManager never seemed to receive any packets:
![](RM520N-GLAP_no_receive.jpg)

Potentially useful resources:
 - https://openwrt.org/inbox/toh/sinovoip/bananapi_bpi-r4
 - https://github.com/4IceG/RM520N-GL?tab=readme-ov-file
 - https://forum.banana-pi.org/t/bpi-r4-how-to-activate-key-b-pcie2-11280000-solved/17295?page=2
 - https://forum.banana-pi.org/t/banana-pi-bpi-r4-4g-5g-module-sim-card-missing-in-modem-manager/17944/16
    - https://github.com/mdsdtech/GFriendWRT/releases/tag/V060225-R4
 - https://forum.openwrt.org/t/send-at-command-while-using-modemmanager/196440
 - https://openwrt.org/docs/guide-user/network/wan/wwan/ltedongle
 - https://forum.gl-inet.com/t/how-to-installing-vanilla-openwrt-on-gl-x3000/45404/73?page=4
    - https://forum.openwrt.org/t/need-help-rm520ngl-ap-openwrt-and-cellular-connectivity/207460
