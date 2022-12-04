# Computer Security Project - Hacking TP-Link C200
Project for course `521253S` at the University of Oulu.

Aim: 
- explore hardware hacking
- try to gain root access to TP-Link C200 and [Inteno FG500]()
- dump root filesystem
- try to find vulnerabilities in the product(s)

## Group 
Lassi Heikkilä `Y68725369`

Mikko Isotalo `2264612`

## Initial exploration

### Physical access
The device was taken apart by prying open the case,
and undoing all the screws that were holding in the PCBs and motors.
Luckily nothing was really glued in, so it was nice and tidy inside the case.

The PCB has a clearly labeled UART port:
![](./photos/c200-pcb-top-uart-contacts.jpg)

However, it does not have pins soldered onto it so that needed to be done:
![](./photos/c200-pcb-top-soldered-uart-pins.jpg)

and connect wires to it:
![](./photos/c200-pcb-top-connected-uart-wires.jpg)


### Shell access from UART
Thanks to hacefresko & nervous-inhuman, we know that the expected baudrate is 57600.
We used a Raspberry Pi to connect to the UART, using minicom:
```console
sudo minicom -b 57600 -D /dev/serial0 -C somefile.log
```

Then when power is plugged in, the U-boot boot sequence and some Linux kernel logs can be observed:

[boot-log](./logs/c200-fw-v1.1.14-boot.log)

Again, thanks to hacefresko & nervous-inhuman, we know that the shell login credentials are:  
username: `root`  
password: `slprealtek`  

When the boot has progressed far enough, shell is available by simply pressing enter and entering the credentials.

The system is running OpenWrt Linux version 12.09-rc1 with kernel version 3.10.27.

This is before installing latest available firmware update.

Full exploration log (potentially sensitive info redacted):
[shell log](./logs/c200-fw-v1.1.14-shell-exploration.log)

Some highlights:

#### Viewing `/etc/passwd`:
```console
root@SLP:~# cat /etc/passwd
root:$1$QhxxhIXI$w5srjXWcEn1.D0geuDaUa.:0:0:root:/root:/bin/ash
nobody:*:65534:65534:nobody:/var:/bin/false
admin:*:500:500:admin:/var:/bin/false
guest:*:500:500:guest:/var:/bin/false
ftp:*:55:55:ftp:/home/ftp:/bin/false
```

#### Dump config values with `uci`:
```console
root@SLP:~# uci export uhttpd
package uhttpd

config uhttpd 'main'
        option listen_https '443'
        option home '/www'
        option rfc1918_filter '1'
        option max_requests '8'
        option cert '/tmp/uhttpd.crt'
        option key '/tmp/uhttpd.key'
        option cgi_prefix '/cgi-bin'
        option lua_prefix '/luci'
        option lua_handler '/usr/lib/lua/luci/sgi/uhttpd.lua'
        option script_timeout '180'
        option network_timeout '180'
        option tcp_keepalive '0'

config cert 'px5g'
        option days '3600'
        option bits '1024'
        option country 'CN'
        option state 'China'
        option location 'China'
        option commonname 'TP-Link'
```

#### View running processes with `ps`:
```console
root@SLP:~# ps 
  PID USER       VSZ STAT COMMAND
    1 root      2324 S    init
    2 root         0 SW   [kthreadd]
    3 root         0 SW   [ksoftirqd/0]
    4 root         0 SW   [kworker/0:0]
    5 root         0 SW<  [kworker/0:0H]
    6 root         0 SW   [kworker/u2:0]
    7 root         0 SW   [rcu_preempt]
    8 root         0 SW   [rcu_bh]
    9 root         0 SW   [rcu_sched]
   10 root         0 SW<  [khelper]
   11 root         0 SW<  [writeback]
   12 root         0 SW<  [bioset]
   13 root         0 SW<  [kblockd]
   14 root         0 SW   [khubd]
   15 root         0 SW   [kworker/0:1]
   16 root         0 SW   [kswapd0]
   17 root         0 SW   [fsnotify_mark]
   18 root         0 SW<  [crypto]
   27 root         0 SW   [kworker/u2:1]
   46 root         0 SW<  [deferwq]
   47 root         0 SW<  [kworker/0:1H]
  261 root      2332 S    -ash
  276 root         0 DW   [reset_thread]
  287 root         0 SW<  [cryptodev_queue]
  300 root       864 S    /sbin/hotplug2 --override --persistent --set-rules-f
  316 root       888 S    /sbin/ubusd
  341 root      8036 S    tp_manage
  375 root      3420 S    /usr/bin/ledd
  378 root      3220 S    /usr/sbin/netlinkd
  382 root      5464 S <  /usr/bin/system_state_audio
  452 root      1636 S    /sbin/netifd
  453 root      1516 S    /usr/sbin/connModed
  461 root      1528 S    /usr/sbin/connModed
  464 root     10128 S    /usr/sbin/wlan-manager
  581 root         0 SW   [RTW_CMD_THREAD]
  610 root      1196 S    wpa_supplicant -B -Dwext -iwlan0 -P/tmp/supplicant_p
  645 root     11440 S    /usr/bin/dsd
  678 root      4412 S    /bin/cloud-brd -c /var/etc/cloud_brd_conf
  779 root     14972 S    /bin/cloud-client
  783 root     11700 S    /bin/cloud-service
  945 root      2316 S    /usr/sbin/telnetd -b 127.0.0.1
  985 root      3792 S    /usr/sbin/uhttpd -f -h /www -T 180 -A 0 -n 8 -R -r C
  995 root      5880 S    /usr/bin/rtspd
  998 root      5988 S    /usr/bin/relayd
 1007 root      4612 S    /usr/bin/p2pd
 1010 root     11152 S    /bin/dn_switch
 1012 root      4200 S    /bin/storage_manager
 1054 root     40632 S    /bin/cet
 1116 root     30228 S    /bin/vda
 1123 root      3804 S    /bin/wtd
 1132 root     11336 S    /bin/nvid
 1185 root      2324 S    udhcpc -p /var/run/static-dhcpc.pid -s /lib/netifd/s
 1260 root      2328 S    /usr/sbin/ntpd -n -p time.nist.gov -p 128.138.140.44
 1280 root      3836 S    /usr/bin/motord
14266 root      2320 R    ps
```

#### View open ports with netstat
```console
root@SLP:~# netstat -lntup
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:8800            0.0.0.0:*               LISTEN      1054/cet
tcp        0      0 127.0.0.1:929           0.0.0.0:*               LISTEN      1007/p2pd
tcp        0      0 0.0.0.0:2020            0.0.0.0:*               LISTEN      1132/nvid
tcp        0      0 0.0.0.0:554             0.0.0.0:*               LISTEN      1054/cet
tcp        0      0 127.0.0.1:23            0.0.0.0:*               LISTEN      945/telnetd
tcp        0      0 127.0.0.1:921           0.0.0.0:*               LISTEN      998/relayd
tcp        0      0 127.0.0.1:922           0.0.0.0:*               LISTEN      995/rtspd
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      985/uhttpd
udp        0      0 0.0.0.0:20002           0.0.0.0:*                           341/tp_manage
udp        0      0 0.0.0.0:3702            0.0.0.0:*                           1132/nvid
udp        0      0 0.0.0.0:38643           0.0.0.0:*                           1260/ntpd
```

We can see that telnet is running but only on the local interface, so it is not accessible remotely.

### U-Boot exploration
U-boot boot process can be stopped by quickly entering `slp` when "Autobooting in x seconds" is displayed.

Full u-boot shell exploration log can be found here: [uboot log](./logs/c200-fw-v1.1.14-uboot-shell.log)

We can see that there are several useful commands available, such as `printenv`, `tftpboot` and `tfptput`.

There are also commands for reading and writing the SPI flash memory, which is probably the most interesting part for us. It would be nice to download the entire firmware image from the device.

#### Dumping SPI flash
The SPI flash on this device is 8MB, which is quite small and fits into RAM, so we can read all of it in one go by running some [sf commands](https://u-boot.readthedocs.io/en/latest/usage/cmd/sf.html):
```console
sf probe
sf read 80000000 0 800000
```

Then the memory can be displayed by running:
```console
md.b 80000000 800000
```

Printing the memory contents through the serial port took a long time.

The hexdump form of the output can be found [here](./data/spi-flash.hex), and in binary form [here](./data/spi-flash.bin).

### Extracting rootfs from SPI flash image
We can find the partition sizes and offsets from the [boot log](./logs/c200-fw-v1.1.14-boot.log):
```text
[    0.403000] 0x000000000000-0x00000001d800 : "factory_boot"
[    0.414000] mtd: partition "factory_boot" doesn't end on an erase block -- force read-only
[    0.432000] 0x00000001d800-0x000000020000 : "factory_info"
[    0.443000] mtd: partition "factory_info" doesn't start on an erase block boundary -- force read-only
[    0.462000] 0x000000020000-0x000000040000 : "art"
[    0.472000] 0x000000040000-0x000000050000 : "config"
[    0.483000] 0x000000050000-0x000000070000 : "boot"
[    0.493000] 0x000000070000-0x0000001c1200 : "kernel"
[    0.503000] mtd: partition "kernel" doesn't end on an erase block -- force read-only
[    0.519000] 0x0000001c1200-0x000000430000 : "rootfs"
[    0.529000] mtd: partition "rootfs" doesn't start on an erase block boundary -- force read-only
[    0.547000] 0x000000430000-0x0000007ffe00 : "rootfs_data"
[    0.558000] mtd: partition "rootfs_data" doesn't end on an erase block -- force read-only
[    0.575000] 0x0000007ffe00-0x000000800000 : "verify"
[    0.585000] mtd: partition "verify" doesn't start on an erase block boundary -- force read-only
[    0.604000] 0x000000070000-0x000000800000 : "firmware"
```

So we can expect the following partitions:

| begin            | end              | size (hex) | size (decimal) | name           | note                                     |
| ---------------- | ---------------- | ---------- | -------------- | -------------- | ---------------------------------------- |
| `0x000000000000` | `0x00000001d800` | `1d800`    | `120832`       | `factory_boot` |                                          |
| `0x00000001d800` | `0x000000020000` | `2800`     | `10240`        | `factory_info` |                                          |
| `0x000000020000` | `0x000000040000` | `20000`    | `131072`       | `art`          |                                          |
| `0x000000040000` | `0x000000050000` | `10000`    | `65536`        | `config`       |                                          |
| `0x000000050000` | `0x000000070000` | `20000`    | `131072`       | `boot`         |                                          |
| `0x000000070000` | `0x0000001c1200` | `151200`   | `1380864`      | `kernel`       |                                          |
| `0x0000001c1200` | `0x000000430000` | `26ee00`   | `2551296`      | `rootfs`       | squashfs based on boot logs              |
| `0x000000430000` | `0x0000007ffe00` | `3cfe00`   | `3997184`      | `rootfs_data`  | for overlayfs?                           |
| `0x0000007ffe00` | `0x000000800000` | `200`      | `512`          | `verify`       | why does this overlap with firmware?     |
| `0x000000070000` | `0x000000800000` | `790000`   | `7929856`      | `firmware`     | maybe this is just a "virtual" partition |

#### Taking apart the `spi-flash.bin` based on partition table
We can use `dd` to separate the partitions.

Using block size 1 for input and output and specifying correct offset and size for the partition should be enough to extract a partition from the image. 

It is possible to also use different block sizes for faster `dd` operation, but that is left as an exercise for the reader. It takes a few minutes to extract one of the bigger partitions with blocksize 1, but as a one time operation that is fine.

So, e.g. to extract `config` partition, we can run the following command. Keep in mind that skip and count are given with base-10 integers here.

```console
dd if=spi-flash.bin of=spi-flash-config.bin bs=1 skip=262144 count=65536
```

We can also try to figure out what is contained on all of the partitions:
```console
$ for img in spi-flash-*.bin; do file $img; done
spi-flash-art.bin: data
spi-flash-boot.bin: data
spi-flash-config.bin: data
spi-flash-factory_boot.bin: data
spi-flash-factory_info.bin: data
spi-flash-kernel.bin: data
spi-flash-rootfs.bin: Squashfs filesystem, little endian, version 4.0, xz compressed, 2499196 bytes, 789 inodes, blocksize: 131072 bytes, created: Wed Aug 25 04:58:05 2021
spi-flash-rootfs_data.bin: Squashfs filesystem, little endian, version 4.0, xz compressed, 3568406 bytes, 298 inodes, blocksize: 131072 bytes, created: Wed Aug 25 04:58:08 2021
spi-flash-verify.bin: data
```

So we know that `rootfs` and `rootfs_data` contain a `squashfs` filesystem, but nothing about the others.

`binwalk` tells us that that some of the other partitions are compressed with LZMA or gzip:
```console
$ for img in spi-flash-*.bin; do echo "$img:" && binwalk -B $img; done
spi-flash-art.bin:

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
256           0x100           gzip compressed data, from Unix, last modified: 2021-08-25 04:58:02

spi-flash-boot.bin:

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
24576         0x6000          LZMA compressed data, properties: 0x5D, dictionary size: 8388608 bytes, uncompressed size: 120124 bytes

spi-flash-config.bin:

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------

spi-flash-factory_boot.bin:

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
24576         0x6000          LZMA compressed data, properties: 0x5D, dictionary size: 8388608 bytes, uncompressed size: 219032 bytes

spi-flash-factory_info.bin:

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------

spi-flash-kernel.bin:

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
512           0x200           LZMA compressed data, properties: 0x6D, dictionary size: 8388608 bytes, uncompressed size: 4112988 bytes

spi-flash-rootfs.bin:

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             Squashfs filesystem, little endian, version 4.0, compression:xz, size: 2499196 bytes, 789 inodes, blocksize: 131072 bytes, created: 2021-08-25 04:58:05

spi-flash-rootfs_data.bin:

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             Squashfs filesystem, little endian, version 4.0, compression:xz, size: 3568406 bytes, 298 inodes, blocksize: 131072 bytes, created: 2021-08-25 04:58:08
3932672       0x3C0200        LZMA compressed data, properties: 0x6D, dictionary size: 8388608 bytes, uncompressed size: 4112988 bytes

spi-flash-verify.bin:

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
```

So `boot`, `factory_boot`, and `kernel` have some LZMA compressed data, which we might be able to uncompress with `7z`.

`config` partition doesn't contain any parts that `file` or `binwalk` are able to recognize, while `binwalk` entropy plot shows that the beginning of the partition containing data has very high entropy, so it may be encrypted.
![](screenshots/spi-flash-config-entropy.png)

Extracted partitions:
| partition name | link                                                            | notes                                      |
| -------------- | --------------------------------------------------------------- | ------------------------------------------ |
| `factory_boot` | [spi-flash-factory_boot.bin](./data/spi-flash-factory_boot.bin) | could this contain some more fancy u-boot? |
| `factory_info` | [spi-flash-factory_info.bin](./data/spi-flash-factory_info.bin) | contains some plain strings                |
| `art`          | [spi-flash-art.bin](./data/spi-flash-art.bin)                   | related to WiFi hardware                   |
| `config`       | [spi-flash-config.bin](./data/spi-flash-config.bin)             | possibly encrypted data                    |
| `boot`         | [spi-flash-boot.bin](./data/spi-flash-boot.bin)                 | probably u-boot binary                     |
| `kernel`       | [spi-flash-kernel.bin](./data/spi-flash-kernel.bin)             | kernel binary                              |
| `rootfs`       | [spi-flash-rootfs.bin](./data/spi-flash-rootfs.bin)             | squashfs                                   |
| `rootfs_data`  | [spi-flash-rootfs_data.bin](./data/spi-flash-rootfs_data.bin)   | squashfs                                   |
| `verify`       | [spi-flash-verify.bin](./data/spi-flash-verify.bin)             |                                            |

#### Extracting the root filesystem
We know that `rootfs` partition contains a [squashfs filesystem image](https://tldp.org/HOWTO/SquashFS-HOWTO/whatis.html) based on boot logs.

Thanks to [this answer on StackExchange](https://unix.stackexchange.com/a/80312), we can extract the filesystem from the partition image by running the following command in a directory where we want to extract the filesystem to:
```console
unsquashfs spi-flash-rootfs.bin
```

This produces a lot of errors due to failing symbolic links, but it also produces a directory called `squashfs-root`, containing our filesystem.
```
$ tree .
.
├── bin
│   ├── busybox
│   ├── cloud-brd
│   ├── cloud-client
│   ├── cloud-service
│   ├── cmd_dsd.sh
│   ├── ipcalc.sh
│   ├── logControl.sh
│   ├── login.sh
│   ├── mmc_test
│   ├── rts_set_gpio
│   ├── tapo_kill.sh
│   ├── timezone.sh
│   ├── tp_manage
│   ├── ubus
│   └── uc_convert
├── dev
├── etc
│   ├── banner
│   ├── cloud-client
│   │   └── 2048_newroot.cer
│   ├── cloud-sdk
│   │   ├── 2048_newroot.cer
│   │   ├── cloud_config.cfg
│   │   └── cloud_service.cfg
│   ├── cloud_service
│   │   └── ipc_service.cer
│   ├── crontabs
│   ├── default
│   │   ├── hostapd.accept
│   │   ├── hostapd_default.conf
│   │   └── hostapd.deny
│   ├── diag.sh
│   ├── dsd_convert.json
│   ├── group
│   ├── hosts
│   ├── hotplug2-common.rules
│   ├── hotplug2-init.rules
│   ├── hotplug2.rules
│   ├── hotplug.d
│   │   ├── block
│   │   │   └── 10-mmc
│   │   ├── button
│   │   │   ├── 00-button
│   │   │   └── 10-reset
│   │   ├── ieee1394
│   │   │   └── 10-ieee1394
│   │   ├── iface
│   │   │   ├── 00-netstate
│   │   │   ├── 10-sysctl
│   │   │   ├── 30-multicast-routing
│   │   │   ├── 70-cloud
│   │   │   └── 80-upnpc
│   │   └── usb
│   │       └── 10-usb
│   ├── init.d
│   │   ├── boot
│   │   ├── cfgdev_info
│   │   ├── cfgmac
│   │   ├── check_dup_list_from_update
│   │   ├── check_upgrade
│   │   ├── cloud_client
│   │   ├── cloud_sdk
│   │   ├── cloud_service
│   │   ├── done
│   │   ├── dsd
│   │   ├── firewall
│   │   ├── is_cal.script
│   │   ├── ledd
│   │   ├── luci_fixtime
│   │   ├── msglog
│   │   ├── netlinkd
│   │   ├── network
│   │   ├── rcS
│   │   ├── recover-manager
│   │   ├── sysctl
│   │   ├── sysfixtime
│   │   ├── sysntpd
│   │   ├── system_state_audio
│   │   ├── tp_manage
│   │   ├── ubus
│   │   ├── udhcpd
│   │   ├── uhttpd
│   │   ├── umount
│   │   ├── wlan
│   │   └── wlan-manager
│   ├── inittab
│   ├── modules.d
│   │   ├── 19-k_netlink_ipc
│   │   ├── 1-k_hi_reset
│   │   ├── 20-lib-textsearch
│   │   ├── 40-ipt-core
│   │   ├── 41-ipt-conntrack
│   │   ├── 45-ipt-filter
│   │   ├── 50-cryptodev
│   │   └── 63-port_status_notify
│   ├── netlink.d
│   │   ├── host_data_collect
│   │   │   └── notify_cloud
│   │   ├── lan_status_notify
│   │   │   └── cleanTMP.sh
│   │   └── wan_status_notify
│   │       ├── 10-wifi-ethernet-switch
│   │       └── 20-auto_down_wan
│   ├── openwrt_release
│   ├── openwrt_version
│   ├── passwd
│   ├── preinit
│   ├── profile
│   ├── protocols
│   ├── rc.common
│   ├── rc.d
│   ├── rc.local
│   ├── root.cer
│   ├── rtl
│   │   ├── 8188FTV_mask.txt
│   │   ├── PHY_REG_PG.txt
│   │   └── wifi_efuse_8188fu.map
│   ├── services
│   ├── shadow
│   ├── shells
│   ├── sys_conf_data
│   ├── sysctl.conf
│   ├── sysupgrade.conf
│   ├── tp_manage
│   │   ├── priv-key.pem
│   │   └── server-cert.pem
│   └── usr_conf_data
├── lib
│   ├── config
│   │   └── uci.sh
│   ├── configs
│   ├── firstboot
│   │   ├── 05_firstboot_skip
│   │   ├── 10_determine_parts
│   │   ├── 10_no_fo_clear_overlay
│   │   ├── 10_reset_has_mini_fo
│   │   ├── 20_has_mini_fo
│   │   ├── 20_no_fo_mount_jffs
│   │   ├── 20_reset_clear_jffs
│   │   ├── 30_is_rootfs_mounted
│   │   ├── 30_no_fo_pivot
│   │   ├── 30_reset_copy_rom
│   │   ├── 40_copy_ramoverlay
│   │   ├── 40_no_fo_copy_ramoverlay
│   │   ├── 50_pivot
│   │   ├── 99_10_no_fo_cleanup
│   │   └── 99_10_with_fo_cleanup
│   ├── functions
│   │   ├── boot.sh
│   │   ├── get_region_code.sh
│   │   ├── network.sh
│   │   ├── parse_uci_config.sh
│   │   ├── service.sh
│   │   └── uci-defaults.sh
│   ├── functions.sh
│   ├── ld-uClibc-0.9.33.so
│   ├── libblobmsg_json.so
│   ├── libcrypt-0.9.33.so
│   ├── libdl-0.9.33.so
│   ├── libgcc_s.so.1
│   ├── libiniparser.so
│   ├── libm-0.9.33.so
│   ├── libpthread-0.9.33.so
│   ├── libpthread.so
│   ├── librt-0.9.33.so
│   ├── libsecurity.so
│   ├── libubox.so
│   ├── libubus.so
│   ├── libuci.so
│   ├── libuClibc-0.9.33.so
│   ├── modules
│   │   └── 3.10.27
│   │       ├── 8188fu.ko
│   │       ├── cryptodev.ko
│   │       ├── gpio_leds.ko
│   │       ├── gpio-motor.ko
│   │       ├── k_hi_reset.ko
│   │       ├── k_netlink_ipc.ko
│   │       ├── port_status_notify.ko
│   │       ├── ts_bm.ko
│   │       ├── ts_fsm.ko
│   │       ├── ts_kmp.ko
│   │       ├── xt_conntrack.ko
│   │       ├── xt_mac.ko
│   │       ├── xt_multiport.ko
│   │       ├── xt_state.ko
│   │       └── xt_string.ko
│   ├── netifd
│   │   ├── common_api.sh
│   │   ├── dhcp.script
│   │   ├── mode_switch.sh
│   │   ├── netifd-proto.sh
│   │   ├── proto
│   │   │   └── dhcp.sh
│   │   ├── smart_lan_ip.sh
│   │   ├── static-dhcp.script
│   │   ├── static_dhcp_start.sh
│   │   └── static_dhcp_stop.sh
│   ├── network
│   │   └── config.sh
│   ├── preinit
│   │   ├── 05_mount_skip
│   │   ├── 10_check_for_mtd
│   │   ├── 10_essential_fs
│   │   ├── 10_indicate_preinit
│   │   ├── 20_check_jffs2_ready
│   │   ├── 20_device_fs_mount
│   │   ├── 30_device_fs_daemons
│   │   ├── 40_dont_mount_jffs2
│   │   ├── 40_init_shm
│   │   ├── 40_mount_devpts
│   │   ├── 50_choose_console
│   │   ├── 60_init_hotplug
│   │   ├── 70_pivot_jffs2_root
│   │   ├── 80_mount_root
│   │   ├── 90_init_console
│   │   ├── 90_mount_no_jffs2
│   │   ├── 99_10_mount_no_mtd
│   │   ├── 99_10_read_uc_data
│   │   └── 99_10_run_init
│   └── upgrade
│       ├── common.sh
│       ├── keep.d
│       │   ├── base-files
│       │   ├── base-files-essential
│       │   └── uhttpd
│       └── luci-add-conffiles.sh
├── mnt
├── overlay
├── proc
├── rom
│   └── note
├── root
├── sbin
│   ├── debug_ir_cut_switch_test.sh
│   ├── debug_ptz_test.sh
│   ├── debug_speaker_test.sh
│   ├── devstatus
│   ├── factorywrite
│   ├── firstboot
│   ├── fw
│   ├── hotplug2
│   ├── hotplug-call
│   ├── ifstatus
│   ├── ifup
│   ├── led.sh
│   ├── luci-reload
│   ├── mount_root
│   ├── msglogd
│   ├── mtd
│   ├── netifd
│   ├── slprestore
│   ├── slpupgrade
│   ├── sound
│   ├── sysupgrade
│   ├── track-reload
│   ├── ubusd
│   ├── uci
│   ├── udevtrigger
│   ├── wget2nand
│   └── wifi
├── sp_rom
├── sys
├── tmp
│   └── log
├── usr
│   ├── bin
│   │   ├── cloudMode.sh
│   │   ├── cloud_upgrade
│   │   ├── consoleredict
│   │   ├── curl
│   │   ├── dsd
│   │   ├── dsd_exec
│   │   ├── get_new_fwinfo
│   │   ├── get_sys_status
│   │   ├── jshn
│   │   ├── ledd
│   │   ├── onboarding
│   │   ├── recover-manager
│   │   ├── resolveip
│   │   └── tp_event.sh
│   ├── lib
│   │   ├── libaes.so
│   │   ├── libcommon.so
│   │   ├── libcrypto.so.1.0.0
│   │   ├── libcurl.so.4.3.0
│   │   ├── libdecrypter.so
│   │   ├── libdsd_client.so
│   │   ├── libiconv.so.2.4.0
│   │   ├── libip4tc.so.0.0.0
│   │   ├── libiwinfo.so
│   │   ├── libjson.so.0.0.1
│   │   ├── libmediautil.so
│   │   ├── libmsglog.sh
│   │   ├── libmsglog.so
│   │   ├── libnl-tiny.so
│   │   ├── libpwr.so
│   │   ├── libssl.so.1.0.0
│   │   ├── libuciassist.so
│   │   ├── libwlanmanager.so
│   │   ├── libwlanubus.so
│   │   ├── libwpactrl.so
│   │   ├── libz.so.1.2.7
│   │   ├── lua
│   │   │   ├── iwinfo.so
│   │   │   ├── libmsglog.lua
│   │   │   ├── luci
│   │   │   │   └── torchlight
│   │   │   │       └── tp_event.lua
│   │   │   ├── ubus.so
│   │   │   └── uci.so
│   │   ├── opkg
│   │   │   ├── info
│   │   │   │   ├── alsa.control
│   │   │   │   ├── alsa-lib.control
│   │   │   │   ├── alsa-lib.list
│   │   │   │   ├── alsa.list
│   │   │   │   ├── base-files.conffiles
│   │   │   │   ├── base-files.control
│   │   │   │   ├── base-files.list
│   │   │   │   ├── busybox.control
│   │   │   │   ├── busybox.list
│   │   │   │   ├── cet.control
│   │   │   │   ├── cet.list
│   │   │   │   ├── chn_info.control
│   │   │   │   ├── chn_info.list
│   │   │   │   ├── cloud-brd.control
│   │   │   │   ├── cloud-brd.list
│   │   │   │   ├── cloud-sdk.control
│   │   │   │   ├── cloud-sdk.list
│   │   │   │   ├── cloud-service.control
│   │   │   │   ├── cloud-service.list
│   │   │   │   ├── config_dev_info.control
│   │   │   │   ├── config_dev_info.list
│   │   │   │   ├── config_mac.control
│   │   │   │   ├── config_mac.list
│   │   │   │   ├── connMode.control
│   │   │   │   ├── connMode.list
│   │   │   │   ├── curl.control
│   │   │   │   ├── curl.list
│   │   │   │   ├── diagnose.control
│   │   │   │   ├── diagnose.list
│   │   │   │   ├── dn_switch.control
│   │   │   │   ├── dn_switch.list
│   │   │   │   ├── dosfstools.control
│   │   │   │   ├── dosfstools.list
│   │   │   │   ├── dsd.control
│   │   │   │   ├── dsd.list
│   │   │   │   ├── factory_restore.control
│   │   │   │   ├── factory_restore.list
│   │   │   │   ├── firewall_ipc.conffiles
│   │   │   │   ├── firewall_ipc.control
│   │   │   │   ├── firewall_ipc.list
│   │   │   │   ├── firmware_upgrade.control
│   │   │   │   ├── firmware_upgrade.list
│   │   │   │   ├── hotplug2.conffiles
│   │   │   │   ├── hotplug2.control
│   │   │   │   ├── hotplug2.list
│   │   │   │   ├── iperf-mt.control
│   │   │   │   ├── iperf-mt.list
│   │   │   │   ├── iptables.control
│   │   │   │   ├── iptables.list
│   │   │   │   ├── iptables-mod-filter.control
│   │   │   │   ├── iptables-mod-filter.list
│   │   │   │   ├── iptables-mod-iprange.control
│   │   │   │   ├── iptables-mod-iprange.list
│   │   │   │   ├── jshn.control
│   │   │   │   ├── jshn.list
│   │   │   │   ├── kernel.control
│   │   │   │   ├── kernel.list
│   │   │   │   ├── kmod-cryptodev.control
│   │   │   │   ├── kmod-cryptodev.list
│   │   │   │   ├── kmod-ipt-conntrack.control
│   │   │   │   ├── kmod-ipt-conntrack.list
│   │   │   │   ├── kmod-ipt-core.control
│   │   │   │   ├── kmod-ipt-core.list
│   │   │   │   ├── kmod-ipt-filter.control
│   │   │   │   ├── kmod-ipt-filter.list
│   │   │   │   ├── kmod-k_hi_reset.control
│   │   │   │   ├── kmod-k_hi_reset.list
│   │   │   │   ├── kmod-k_netlink_ipc.control
│   │   │   │   ├── kmod-k_netlink_ipc.list
│   │   │   │   ├── kmod-lib-crc-ccitt.control
│   │   │   │   ├── kmod-lib-crc-ccitt.list
│   │   │   │   ├── kmod-lib-textsearch.control
│   │   │   │   ├── kmod-lib-textsearch.list
│   │   │   │   ├── kmod-port_status_notify.control
│   │   │   │   ├── kmod-port_status_notify.list
│   │   │   │   ├── kmod-rtl8188fu.control
│   │   │   │   ├── kmod-rtl8188fu.list
│   │   │   │   ├── ledd.control
│   │   │   │   ├── ledd.list
│   │   │   │   ├── libblobmsg-json.control
│   │   │   │   ├── libblobmsg-json.list
│   │   │   │   ├── libc.control
│   │   │   │   ├── libc.list
│   │   │   │   ├── libcurl.control
│   │   │   │   ├── libcurl.list
│   │   │   │   ├── libdecrypter.control
│   │   │   │   ├── libdecrypter.list
│   │   │   │   ├── libevent.control
│   │   │   │   ├── libevent.list
│   │   │   │   ├── libfreetype.control
│   │   │   │   ├── libfreetype.list
│   │   │   │   ├── libgcc.control
│   │   │   │   ├── libgcc.list
│   │   │   │   ├── libiconv-full.control
│   │   │   │   ├── libiconv-full.list
│   │   │   │   ├── libiniparser.control
│   │   │   │   ├── libiniparser.list
│   │   │   │   ├── libip4tc.control
│   │   │   │   ├── libip4tc.list
│   │   │   │   ├── libiwinfo.control
│   │   │   │   ├── libiwinfo.list
│   │   │   │   ├── libiwinfo-lua.control
│   │   │   │   ├── libiwinfo-lua.list
│   │   │   │   ├── libjson.control
│   │   │   │   ├── libjson.list
│   │   │   │   ├── liblua.control
│   │   │   │   ├── liblua.list
│   │   │   │   ├── libmediautil.control
│   │   │   │   ├── libmediautil.list
│   │   │   │   ├── libminiupnpc.control
│   │   │   │   ├── libminiupnpc.list
│   │   │   │   ├── libncurses.control
│   │   │   │   ├── libncurses.list
│   │   │   │   ├── libnl-tiny.control
│   │   │   │   ├── libnl-tiny.list
│   │   │   │   ├── libopenssl.control
│   │   │   │   ├── libopenssl.list
│   │   │   │   ├── libpcap.control
│   │   │   │   ├── libpcap.list
│   │   │   │   ├── libpng.control
│   │   │   │   ├── libpng.list
│   │   │   │   ├── libpthread.control
│   │   │   │   ├── libpthread.list
│   │   │   │   ├── libpwr.control
│   │   │   │   ├── libpwr.list
│   │   │   │   ├── librt.control
│   │   │   │   ├── librt.list
│   │   │   │   ├── libsecurity.control
│   │   │   │   ├── libsecurity.list
│   │   │   │   ├── libsyslog.control
│   │   │   │   ├── libsyslog.list
│   │   │   │   ├── libubox.control
│   │   │   │   ├── libubox.list
│   │   │   │   ├── libubus.control
│   │   │   │   ├── libubus.list
│   │   │   │   ├── libubus-lua.control
│   │   │   │   ├── libubus-lua.list
│   │   │   │   ├── libuci.control
│   │   │   │   ├── libuci.list
│   │   │   │   ├── libuci-lua.control
│   │   │   │   ├── libuci-lua.list
│   │   │   │   ├── libxtables.control
│   │   │   │   ├── libxtables.list
│   │   │   │   ├── lua.control
│   │   │   │   ├── lua.list
│   │   │   │   ├── miniupnpc.control
│   │   │   │   ├── miniupnpc.list
│   │   │   │   ├── motord.control
│   │   │   │   ├── motord.list
│   │   │   │   ├── mtd.control
│   │   │   │   ├── mtd.list
│   │   │   │   ├── netifd.control
│   │   │   │   ├── netifd.list
│   │   │   │   ├── nvid.control
│   │   │   │   ├── nvid.list
│   │   │   │   ├── onboarding.control
│   │   │   │   ├── onboarding.list
│   │   │   │   ├── onvif.control
│   │   │   │   ├── onvif.list
│   │   │   │   ├── openssl-util.conffiles
│   │   │   │   ├── openssl-util.control
│   │   │   │   ├── openssl-util.list
│   │   │   │   ├── p2pd.control
│   │   │   │   ├── p2pd.list
│   │   │   │   ├── public.control
│   │   │   │   ├── public.list
│   │   │   │   ├── realtek_isp_tuning.control
│   │   │   │   ├── realtek_isp_tuning.list
│   │   │   │   ├── realtek_mpp.control
│   │   │   │   ├── realtek_mpp.list
│   │   │   │   ├── realtek_tools.control
│   │   │   │   ├── realtek_tools.list
│   │   │   │   ├── recover-manager.control
│   │   │   │   ├── recover-manager.list
│   │   │   │   ├── resolveip.control
│   │   │   │   ├── resolveip.list
│   │   │   │   ├── rtl-hostapd.control
│   │   │   │   ├── rtl-hostapd.list
│   │   │   │   ├── rtl-wpa-supplicant.control
│   │   │   │   ├── rtl-wpa-supplicant.list
│   │   │   │   ├── smart_lan_ip.control
│   │   │   │   ├── smart_lan_ip.list
│   │   │   │   ├── storage_manager.control
│   │   │   │   ├── storage_manager.list
│   │   │   │   ├── syslog.control
│   │   │   │   ├── syslog.list
│   │   │   │   ├── system_state_audio.control
│   │   │   │   ├── system_state_audio.list
│   │   │   │   ├── tp_manage.control
│   │   │   │   ├── tp_manage.list
│   │   │   │   ├── t_relayd.control
│   │   │   │   ├── t_relayd.list
│   │   │   │   ├── t_rtspd.control
│   │   │   │   ├── t_rtspd.list
│   │   │   │   ├── ubus.control
│   │   │   │   ├── ubusd.control
│   │   │   │   ├── ubusd.list
│   │   │   │   ├── ubus.list
│   │   │   │   ├── uc_convert.control
│   │   │   │   ├── uc_convert.list
│   │   │   │   ├── uci.control
│   │   │   │   ├── uci.list
│   │   │   │   ├── uclibcxx.control
│   │   │   │   ├── uclibcxx.list
│   │   │   │   ├── udt.control
│   │   │   │   ├── udt.list
│   │   │   │   ├── uhttpd.conffiles
│   │   │   │   ├── uhttpd.control
│   │   │   │   ├── uhttpd.list
│   │   │   │   ├── uhttpd-mod-tls.control
│   │   │   │   ├── uhttpd-mod-tls.list
│   │   │   │   ├── u_netlink_ipc.control
│   │   │   │   ├── u_netlink_ipc.list
│   │   │   │   ├── vda.control
│   │   │   │   ├── vda.list
│   │   │   │   ├── wireless-tools-rtl.control
│   │   │   │   ├── wireless-tools-rtl.list
│   │   │   │   ├── wlan-manager.control
│   │   │   │   ├── wlan-manager.list
│   │   │   │   ├── wtd.control
│   │   │   │   ├── wtd.list
│   │   │   │   ├── zlib.control
│   │   │   │   └── zlib.list
│   │   │   ├── lists
│   │   │   └── status
│   │   └── uhttpd_tls.so
│   ├── sbin
│   │   ├── connModed
│   │   ├── fdisk_sd_card
│   │   ├── iwconfig
│   │   ├── makeRoomForUpgrade.sh
│   │   ├── mount_disk
│   │   ├── mount_nas
│   │   ├── nfsroot
│   │   ├── test_nas
│   │   ├── wlan-manager
│   │   └── wpa_supplicant
│   └── share
│       ├── libubox
│       │   └── jshn.sh
│       └── udhcpc
│           └── default.script
└── www

60 directories, 497 files
```

#### `libdecrypt.so`
There is an interesting looking dynamic library found in `/usr/lib/libdecrypter.so`.
`binwalk` tells us that it contains an RSA private key.
```console
$ binwalk -B libdecrypter.so

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             ELF, 32-bit LSB MIPS-I shared object, MIPS, version 1 (SYSV)
5952          0x1740          Base64 standard index table
6304          0x18A0          PEM RSA private key
```
However, that is a false detection. What `binwalk` actually detected was a formatting string containing the RSA key header:
```
-----BEGIN RSA PRIVATE KEY-----
```

Still, it's nice to see that maybe we can find something interesting from this shared object.

Dumping the `strings` from the file yields the following:
```console
$ strings libdecrypter.so
_init
_gp_disp
_fini
_ITM_deregisterTMCloneTable
_ITM_registerTMCloneTable
__cxa_finalize
_Jv_RegisterClasses
libcrypto.so.1.0.0
libmsglog.so
rsa_base64_encode
rsa_base64_encode_alloc
malloc
rsa_isbase64
rsa_base64_decode
rsa_base64_decode_alloc
sprintf
strlen
strcpy
rsa_encrypt
msglog
BIO_new_mem_buf
PEM_read_bio_RSA_PUBKEY
RSA_public_encrypt
calloc
memcpy
RSA_free
BIO_free_all
PEM_read_bio_RSAPrivateKey
RSA_private_decrypt
rsa_decrypt
libc.so.0
_ftext
_fdata
_edata
__bss_start
_fbss
_end
 	9'l
1!@H
0!(E
0!(E
$!(@
$!0`
'!(`
'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/
?456789:;<=
 !"#$%&'()*+,-./0123
-----BEGIN PUBLIC KEY-----
-----BEGIN RSA PRIVATE KEY-----
-----END PUBLIC KEY-----
-----END RSA PRIVATE KEY-----
DECRYPTER
Wrong input. plaintext_len %d.
Base64 decode public key error.
Create BIO buffer for public key error.
call RSA_public_encrypt failed
Base64 encode ciphertext error.
Malloc for ciphertext_out failed.
wrong input. ciphertext_len %d.
Base64 decode ciphtertext error.
Calloc mem error.
MIICXAIBAAKBgQC4D6i0o...part of the key redacted...QmIseOs9f0=
MIGfMA0GCSqGSIb3DQEBA...part of the key redacted...rtZQQIDAQAB
MIGfMA0GCSqGSIb3DQEBA...part of the key redacted...yYVbQIDAQAB
GCC: (GNU) 3.3.2
GCC: (Realtek RSDK-4.8.5p1 Build 2521) 4.8.5 20150209 (prerelease)
.shstrtab
.note.gnu.build-id
.dynamic
.hash
.dynsym
.dynstr
.rel.dyn
.init
.text
.MIPS.stubs
.fini
.rodata
.eh_frame
.ctors
.dtors
.jcr
.data
.got
.sdata
.bss
.comment
.gnu.attributes
.mdebug.abi32

```
There are a few strings that look an awful lot like public or private RSA keys.

After some experimenting with `openssl`, we determined that the private key and the second public key are a pair, but so far no idea what they are used to encrypt and decrypt.

#### Reverse engineering `cloud-client` binary

The `cloud-client` binary probably deals with communicating with the TP-Link cloud service, so it would be interesting to know how it works and how the connection is secured.

We could take a look at it using `Ghidra`.

The binary loads a few shared libraries, so those may be intersting as well:
```console
$ readelf -d cloud-client | grep NEEDED
 0x00000001 (NEEDED)                     Shared library: [libssl.so.1.0.0]
 0x00000001 (NEEDED)                     Shared library: [libcrypto.so.1.0.0]
 0x00000001 (NEEDED)                     Shared library: [libpthread.so.0]
 0x00000001 (NEEDED)                     Shared library: [libuci.so]
 0x00000001 (NEEDED)                     Shared library: [libubus.so]
 0x00000001 (NEEDED)                     Shared library: [libubox.so]
 0x00000001 (NEEDED)                     Shared library: [libjson.so.0]
 0x00000001 (NEEDED)                     Shared library: [libblobmsg_json.so]
 0x00000001 (NEEDED)                     Shared library: [libcurl.so.4]
 0x00000001 (NEEDED)                     Shared library: [librt.so.0]
 0x00000001 (NEEDED)                     Shared library: [libmsglog.so]
 0x00000001 (NEEDED)                     Shared library: [libz.so.1]
 0x00000001 (NEEDED)                     Shared library: [libaes.so]
 0x00000001 (NEEDED)                     Shared library: [libdecrypter.so]
 0x00000001 (NEEDED)                     Shared library: [libsecurity.so]
 0x00000001 (NEEDED)                     Shared library: [libdsd_client.so]
 0x00000001 (NEEDED)                     Shared library: [libm.so.0]
 0x00000001 (NEEDED)                     Shared library: [libdl.so.0]
 0x00000001 (NEEDED)                     Shared library: [libc.so.0]
```

## References
### Literature
[hacefresko](https://github.com/hacefresko)'s great post about the same device was very useful for getting initial access:
[tp-link-tapo-c200-unauthenticated-rce](https://www.hacefresko.com/posts/tp-link-tapo-c200-unauthenticated-rce).

[nervous-inhuman](https://github.com/nervous-inhuman) has done great work reverse engineering the device:
[tplink-tapo-c200-re](https://github.com/nervous-inhuman/tplink-tapo-c200-re)

FCC database contains some good pictures of the PCBs, which were used to check if there is a UART on the device prior to purchase:
[TE7C200](https://fccid.io/TE7C200)

Hanna Gustafsson and Hanna Kvist have written a Master's thesis paper containing information about the device:
[thesis paper](https://www.diva-portal.org/smash/get/diva2:1679623/FULLTEXT01.pdf)

### Used tools
- [`minicom`](https://linux.die.net/man/1/minicom)
- [`binwalk`](https://www.kali.org/tools/binwalk/)
- [`unsquashfs`](https://tldp.org/HOWTO/SquashFS-HOWTO/mksqoverview.html)
- [`dd`](https://man7.org/linux/man-pages/man1/dd.1.html)
- [`readelf`](https://man7.org/linux/man-pages/man1/readelf.1.html)
- [`Ghidra`](https://github.com/NationalSecurityAgency/ghidra)
- 