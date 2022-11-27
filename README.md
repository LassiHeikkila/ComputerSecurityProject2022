# Computer Security Project - Hacking TP-Link C200
Course: `521253S`

## Group 
Lassi Heikkilä `Y68725369`

Mikko Isotalo `######`


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
The SPI flash on this device is 8MB, which is quite small and fits into RAM, so we can read all of it in one go by running:
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

## References
[hacefresko](https://github.com/hacefresko)'s great post about the same device was very useful for getting initial access:
[tp-link-tapo-c200-unauthenticated-rce](https://www.hacefresko.com/posts/tp-link-tapo-c200-unauthenticated-rce).

[nervous-inhuman](https://github.com/nervous-inhuman) has done great work reverse engineering the device:
[tplink-tapo-c200-re](https://github.com/nervous-inhuman/tplink-tapo-c200-re)

FCC database contains some good pictures of the PCBs, which were used to check if there is a UART on the device prior to purchase:
[TE7C200](https://fccid.io/TE7C200)

Hanna Gustafsson and Hanna Kvist have written a Master's thesis paper containing information about the device:
[thesis paper](https://www.diva-portal.org/smash/get/diva2:1679623/FULLTEXT01.pdf)
