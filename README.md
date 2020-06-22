Reverse engineer Swann camera

# Overview
Another month and another device that has robust hardware but the software simply sucks.
I'm talking about the "Swann floodlight security camera", Swann has something like 10 apps to deal with the different hardware they sell, they are all extremely outdated or simply do not work.

I bought a SWWHD-FLOCAM and though the Swann apps were clearly outdated, it worked more or less.

# The problem
Around a month ago I started receiving notifications saying that the existing app "SAFE by Swann" would stop working since "The software company we used to build the SAFE by Swann app has recently had financial troubles and are ceased operating, leaving us with a problem."

The recommendation is to download yet another app called "SwannSecurity".
Me being an understanding customer, I proceeded to do so.
Here is where the problems started, the camera just wouldn't sync/setup with the new app.
I tried everything I could think of to get it working but no luck (everything with the exception of contacting Swann of course, I'm to shy). My suspicion is that the device creates a Wifi access point with a different name than what the app is expecting.

# Last resort AKA reverse engineer

## Discovery
Secretly excited that nothing worked it gave me an excuse to poke around. This is after all a security camera, I was eager to get it working.
Opening the device a Hi3518 IC is visible, googling a bit it seems that some poke work has been done already for similar cameras. The chip itself is pretty much open source with an SDK and toolchain available.

After some quick browsing of the PCB the usual glorious "oooo" is seen (oo in this case), the UART port pins that is (TX, RX, VCC, GND).

The pins are small but I managed to solder some wires to it and fire up the serial monitor and voila, Uboot with a root shell.

If the keyboard is smashed while the device is rebooting the booting procedure is halted and a Uboot shell is provided, a new filesystem can be flashed this way using `sf` and some `loadb` trickery. Even though tftp is available the camera doesn't have any network ports, therefore loading binary files over the serial port is required, there also seems to be a way of loading files over USB and I can recognize the micro-usb pins on the sdcard pcb but nothing is soldered there, thanks Swann.
Also sdcard? What sdcard? The camera has no sdcard visible/accessible but after opening the camera there is a pcb that has the reset button, sdcard with a 8GB card and usb.

## Digging through the OS

### Getting Access
The outputs that everyone wants:
```
$ which telnetd
/usr/sbin/telnetd
```

```
$ which ftpd
/usr/sbin/ftpd
```

After finding that the device had telnet I proceeded to fire it up by issuing `telnetd &`.
This sucessfully spawned telnet but after trying to connect it required the root login which the password was nowhere to be seen (UART autologins as root).

Seeing all the crappy passwords that this types of devices have I thought it couldn't be to hard to get it:
```
$ cat /etc/shadow
root:AxF/qjbydMnd.:0:0:99999:7:::
bin:*:12963:0:99999:7:::
daemon:*:12963:0:99999:7:::
adm:*:12963:0:99999:7:::
lp:*:12963:0:99999:7:::
sync:*:12963:0:99999:7:::
shutdown:*:12963:0:99999:7:::
halt:*:12963:0:99999:7:::
uucp:*:12963:0:99999:7:::
operator:*:12963:0:99999:7:::
nobody:*:12963:0:99999:7:::
```

Hum, is that DESCrypt? Isn't descrypt vulnerable to collisions? Nice choice.
Anyway its salted, john the ripper with the rockyou wordlist or brute force yielded nothing but hashcat took care of it in 1 minute or so `AxF/qjbydMnd.:a0n1ipc`

After looking around the filesystem is mounted as read-only but the files that actually manage the camera are mounted at `/mnt/mtd` as read-write, so the poking continues.

### The filesystem and camera working principles
Some juicy info:
```
$ ls -1
AACVOICE
DevImportId.ini
LOGFILE
Ozvision
PIR.ko
SwannMicroAgent.sh
SystemConfig.ini
SystemConfig_backup.ini
_format_sdcard.sh
aoni_ipc
aoni_ipc.sh
aoni_ipc.sh_bak
boot.sh
boot.sh.bak
curl
daemon
fastboot.sh
fdisk.param
hisi_check_format.sh
hostapd
lib
load3518e_979
logs
ntpclient
oz_start_wait_netok.sh
product_type
rtl8192eu.ko
sensorIQ
smart_ap
socket_system_server
timstamp.conf
tmp
udhcpc
upgrade
version
wdt.ko
wpa_supplicant.conf
```

```
$ ps xa
 ...
 1105 root      17:09 /mnt/mtd/aoni_ipc
 1270 root       0:11 /mnt/mtd/Ozvision/bin/cloud_agents_watchdog -k /mnt/mtd/Ozvision/key/<TOO_MUCH_INFO>.key -l /mnt/mtd/Ozvision/conf/config.dat -i /mnt/mtd/Ozvision/conf/device.ini -g gateway.safebyswann.com 
 1272 root       0:38 /mnt/mtd/Ozvision/bin/cloud_gw_manager -k /mnt/mtd/Ozvision/key/<TOO_MUCH_INFO>.key -d <TOO_MUCH_INFO> -g gateway.safebyswann.com -c start -r55
 1274 root       0:03 /mnt/mtd/Ozvision/bin/cloud_p2pv2_tunnel -s /mnt/mtd/Ozvision/key/<TOO_MUCH_INFO>.key -d <TOO_MUCH_INFO> -g swann.signaling.p2pbox.net -r rtsp://127.0.0.1:8282/av0_0 -o start
 1282 root       0:01 /mnt/mtd/Ozvision/bin/cloud_adapter -d <TOO_MUCH_INFO> -l /mnt/mtd/Ozvision/conf/config.dat -c start -r55
 1284 root       0:12 /mnt/mtd/Ozvision/bin/cloud_uploader -k /mnt/mtd/Ozvision/key/<TOO_MUCH_INFO>.key -d <TOO_MUCH_INFO> -g gateway.safebyswann.com -l /mnt/mtd/Ozvision/conf/config.dat -c start -r55
 ...
```

The running apps got me interested, so I did what any sane person does and tried to break things, killing each of the processes until the camera became unresponsive in the iOS app.
This resulted in me realising that `cloud_*` apps are responsible for spying on me and the `aoni_ipc` is the actual camera management app.
Also note that a rtsp address is cleary present at `rtsp://<camera_ip>:8282/av0_0` and it can be accessed sucessfully with VLC, yet, Swann states that the camera does not have any rtsp interface.

### API
The camera more or less works with all the cloud apps closed but not when `aoni_ipc` is closed.
So I proceeded to kill the app and spawn it in a telnet shell in order to get some more info, this resulted in a very verbose log and it seems that the app has some kind of API, sweet.
After popping the binary in ida/ghidra and digging a bit the main method that handles all API calls looks as follows:
```
 char *pcVar1;
    int iVar2;
    undefined4 uVar3;
    
    pcVar1 = strstr(param_1,"req");
    if ((pcVar1 != (char *)0x0) && (pcVar1 = strstr(param_1,"is alive"), pcVar1 != (char *)0x0)){
        return 1000;
    }
    pcVar1 = strstr(param_1,"req");
    if (pcVar1 != (char *)0x0) {
        pcVar1 = strstr(param_1,"telnet enable");
        if (pcVar1 != (char *)0x0) {
            return 0x3ec;
        }
        pcVar1 = strstr(param_1,"get camera name");
        if (pcVar1 != (char *)0x0) {
            return 0x3f1;
        }
        pcVar1 = strstr(param_1,"set camera name");
        if (pcVar1 != (char *)0x0) {
            return 0x3f2;
        }
        pcVar1 = strstr(param_1,"set video");
        if (pcVar1 != (char *)0x0) {
            return 0x3f3;
        }
        pcVar1 = strstr(param_1,"get alarm");
        if ((pcVar1 != (char *)0x0) &&
           (pcVar1 = strstr(param_1,"get alarm time interval"), pcVar1 == (char *)0x0)) {
            return 0x3f4;
        }
        pcVar1 = strstr(param_1,"set alarm");
        if ((pcVar1 != (char *)0x0) &&
           (pcVar1 = strstr(param_1,"set alarm time interval"), pcVar1 == (char *)0x0)) {
            return 0x3f5;
        }
        pcVar1 = strstr(param_1,"set alarm time interval");
        if (pcVar1 != (char *)0x0) {
            return 0x3f6;
        }
        pcVar1 = strstr(param_1,"get alarm time interval");
        if (pcVar1 != (char *)0x0) {
            return 0x3f7;
        }
        pcVar1 = strstr(param_1,"set rec state");
        if (pcVar1 != (char *)0x0) {
            return 0x3f9;
        }
        pcVar1 = strstr(param_1,"get device info");
        if (pcVar1 != (char *)0x0) {
            return 0x3f8;
        }
        pcVar1 = strstr(param_1,"get privacy mode");
        if (pcVar1 != (char *)0x0) {
            return 0x3fa;
        }
        pcVar1 = strstr(param_1,"set privacy mode");
        if (pcVar1 != (char *)0x0) {
            return 0x3fb;
        }
        pcVar1 = strstr(param_1,"start recording");
        if (pcVar1 != (char *)0x0) {
            return 0x3fc;
        }
        pcVar1 = strstr(param_1,"stop recording");
        if (pcVar1 != (char *)0x0) {
            return 0x3fd;
        }
        pcVar1 = strstr(param_1,"take snapshot");
        if (pcVar1 != (char *)0x0) {
            return 0x3fe;
        }
        pcVar1 = strstr(param_1,"fw upgrade control");
        if (pcVar1 != (char *)0x0) {
            return 0x3ff;
        }
        pcVar1 = strstr(param_1,"fw upgrade status");
        if (pcVar1 != (char *)0x0) {
            return 0x400;
        }
        pcVar1 = strstr(param_1,"flip image");
        if (pcVar1 != (char *)0x0) {
            return 0x401;
        }
        pcVar1 = strstr(param_1,"set two way audio enabled");
        if (pcVar1 != (char *)0x0) {
            return 0x402;
        }
        pcVar1 = strstr(param_1,"set time zone");
        if (pcVar1 != (char *)0x0) {
            return 0x403;
        }
        pcVar1 = strstr(param_1,"localstorage device get info");
        if (pcVar1 != (char *)0x0) {
            return 0x404;
        }
        pcVar1 = strstr(param_1,"localstorage get directory name");
        if (pcVar1 != (char *)0x0) {
            return 0x405;
        }
        pcVar1 = strstr(param_1,"localstorage device format");
        if (pcVar1 != (char *)0x0) {
            return 0x406;
        }
        pcVar1 = strstr(param_1,"localstorage playback");
        if (pcVar1 != (char *)0x0) {
            return 0x3ee;
        }
        pcVar1 = strstr(param_1,"list wifi");
        if (pcVar1 != (char *)0x0) {
            return 0x407;
        }
        pcVar1 = strstr(param_1,"set wifi");
        if (pcVar1 != (char *)0x0) {
            return 0x408;
        }
        pcVar1 = strstr(param_1,"set environment");
        if (pcVar1 != (char *)0x0) {
            return 0x409;
        }
        pcVar1 = strstr(param_1,"get time zone");
        if (pcVar1 != (char *)0x0) {
            return 0x40a;
        }
        pcVar1 = strstr(param_1,"set icr mode");
        if (pcVar1 != (char *)0x0) {
            return 0x40b;
        }
        pcVar1 = strstr(param_1,"get icr mode");
        if (pcVar1 != (char *)0x0) {
            return 0x40c;
        }
        pcVar1 = strstr(param_1,"get siren state");
        if (pcVar1 != (char *)0x0) {
            return 0x40e;
        }
        pcVar1 = strstr(param_1,"set siren state");
        if (pcVar1 != (char *)0x0) {
            return 0x40d;
        }
        pcVar1 = strstr(param_1,"get siren autooff");
        if (pcVar1 != (char *)0x0) {
            return 0x410;
        }
        pcVar1 = strstr(param_1,"set siren autooff");
        if (pcVar1 != (char *)0x0) {
            return 0x40f;
        }
        pcVar1 = strstr(param_1,"get motion detect zone");
        if (pcVar1 != (char *)0x0) {
            return 0x412;
        }
        pcVar1 = strstr(param_1,"set motion detect zone");
        if (pcVar1 != (char *)0x0) {
            return 0x411;
        }
        pcVar1 = strstr(param_1,"get light state");
        if (pcVar1 != (char *)0x0) {
            return 0x414;
        }
        pcVar1 = strstr(param_1,"set light state");
        if (pcVar1 != (char *)0x0) {
            return 0x413;
        }
        pcVar1 = strstr(param_1,"get pir sensor");
        if (pcVar1 != (char *)0x0) {
            return 0x415;
        }
        pcVar1 = strstr(param_1,"set pir sensor");
        if (pcVar1 != (char *)0x0) {
            return 0x416;
        }
        pcVar1 = strstr(param_1,"get all pir sensors");
        if (pcVar1 != (char *)0x0) {
            return 0x417;
        }
        pcVar1 = strstr(param_1,"set all pir sensors");
        if (pcVar1 != (char *)0x0) {
            return 0x418;
        }
        pcVar1 = strstr(param_1,"get light pir autooff");
        if (pcVar1 != (char *)0x0) {
            return 0x419;
        }
        pcVar1 = strstr(param_1,"set light pir autooff");
        if (pcVar1 != (char *)0x0) {
            return 0x41a;
        }
        pcVar1 = strstr(param_1,"reboot device");
        if (pcVar1 != (char *)0x0) {
            return 0x41b;
        }
        pcVar1 = strstr(param_1,"reset device");
        if (pcVar1 != (char *)0x0) {
            return 0x41c;
        }
        iVar2 = API_get_command(param_1);
        if (iVar2 != 0) {
            return 0x3ed;
        }
```

I got to admit, the first API call "telnet enable" made me feel a bit better about Swann, since by being able to enable telnet using an API new users don't need to physically open the camera in order to fiddle with it.

The API works by sending raw TCP commands to port 9030, so for example to turn on the flood lights the following can be sent:
```
$ nc <camera_ip> 9030
$ {"req":"set light state","state":"on","type":1}
```

To enable telnet:
```
$ nc <camera_ip> 9030
$ {"req":"telnet enable","state":"enable","type":1}
```

"telnet enable" state needs to be "enable" not "on", enable, on, 1, who cares.

# Notes
The camera also has httpd which spawns a web server, so some web interface can be made available, the camera has no web interface enabled by default.
The firmware upgrade procedure is performed every now and then by querying "https://firmware.safebyswann.com" with the camera version and some other fields, all the upgrades seem to be OTA, not full fledged filesystems and they are just a tar file with the contents of `/mnt/mtd`. The upgrades are stored in `/mnt/mtd/upgrades`
How `aoni_ipc` works behind the scenes, at least to activate the floodlights, is by calling the `himm` command, whcih I believe it allows to set/read the GPIO, for example to set the lights on issue `hmmi 0x2013000c 7`, to set them off issue `hmmi 0x2013000c 0`.
There is a driver to interact with the PIR sensor but I didn't dive on how it works exactly.
The firmware on my camera is `v1.5.3`

Enjoy!