# iPadOS 18 on iPad 6

IMPORTANT: the iPad 6 needs to be on 17.7.x already

Download an iPad 7 18.x IPSW
And iPad 6 17.7.2 IPSW (other 17.7.x can work too but I used this one)

For some dependencies check the `projects` folder in this repo for submodules which is easiest way to get the correct version/branch.

Obtain openra1n special version `ipad6` branch: https://github.com/asdfugil/openra1n (submodule: `projects/openra1n`)

`img4` is from img4lib (submodule: `projects/img4lib`)
palera1n: https://github.com/palera1n/palera1n/releases
pongoterm (pongoOS terminal): https://github.com/palera1n/PongoOS/blob/iOS15/scripts/pongoterm.c

## Current Limitations

- Tethered (will always be the case)
- Lightning AV and VGA Adapters do not work
- Wastes storage because I am too lazy to patch SSV checks in iBoot
- System Camera app cannot take videos
- "Erase All Contents and Settings" will cause a bootloop
- Baseband, and by extension Cellular model activation does not work (so setup the cellular model before starting the process)

## Build palera1n KPF

palera1n KPF: https://github.com/palera1n/PongoOS `ios18.4` branch (with cryptex patches)
submodule: `projects/PongoOS-KPF`

```
cd projects/PongoOS-KPF
gmake -j10 DEV_BUILD=1
```

## Build Turdus PongoOS

Turdus PongoOS: https://github.com/turdus-m3rula/PongoOS
submodule: `projects/PongoOS-Turdus`

```
cd projects/PongoOS-Turdus
gmake -j10 DEV_BUILD=1
```

## get turdus files

download `https://sep.lol/files/releases/v1.0.4/046aaa68a518c270f3050363c4ce6e0eab0129797a8a0b711f1e9f526266970c47eb4b482298d620e9b8d6546f764fd3/sep_racer`

This is the same file as the module embedded within turdus 1.0.4, save it as it will be used later.

## Decrypt Root FS and System Cryptex

Decompress the IPSW file (it is a zip file with another extension)
There will be two `.dmg.aea` files, the bigger one is the encrypted root fs and the smaller one is the system cryptex.
There is also a small `.dmg` file that is less than 20 MB in size, that is the app cryptex.

### Get keys

aeota: https://github.com/dhinakg/aeota
submodule: `projects/aeota`

```
# Decrypt the bigger rootfs .dmg.aea file
aea decrypt -i <XXX-XXXXXXX-XXX.dmg.aea> -o root.dmg -key-value "base64:$(python3 /path/to/aeota/get_key.py <XXX-XXXXXXX-XXX.dmg.aea>)"
# Decrypt the smaller system cryptex .dmg.aea file
aea decrypt -i <XXX-XXXXXXX-XXX.dmg.aea> -o os.dmg -key-value "base64:$(python3 /path/to/aeota/get_key.py <XXX-XXXXXXX-XXX.dmg.aea>)"
```

## SSH Ramdisk

https://github.com/verygenericname/sshrd_script
submodule: `projects/SSHRD_Script`


```
cd projects/SSHRD_Script
./sshrd.sh 17.7
./sshrd.sh boot
```

### SSH commands

`inetcat` is part of libusbmuxd.

Download a file from the device
```
scp -o 'ProxyCommand=inetcat 22' -o 'UserKnownHostsFile /dev/null' -o 'StrictHostKeyChecking no' root@ramdisk:/path/on/source/file/on/device /path/to/destination/file/on/host
```
Upload a file to the device
```
scp -o 'ProxyCommand=inetcat 22' -o 'UserKnownHostsFile /dev/null' -o 'StrictHostKeyChecking no' /path/to/source/file/on/host root@ramdisk:/path/to/destination/file/on/device
```
SSH
```
ssh -o 'ProxyCommand=inetcat 22' -o 'UserKnownHostsFile /dev/null' -o 'StrictHostKeyChecking no' root@ramdisk
```

Password: alpine

The following commands shall be run on the device.

### Mount tmpfs
```
mount_tmpfs /mnt5
```

### Create bootloader

iBootPatch2: https://github.com/asdfugil/ibootpatch2 (ipad6 branch)
submodule: `projects/ibootpatch2`

compile commands on host
```
cd projects/ibootpatch2
gmake -j10
```

Dump onboard blobs on the iPad:

```
dd if=/dev/disk2 of=/mnt5/disk2.bin
```

Download /mnt/disk2.bin onto the host, run the following commands on the PC to create a IM4M file which will also be used later.

Obtain iPad 6 firmware decryption key for "LLB": https://theapplewiki.com/wiki/Firmware_Keys/17.x#iPad 
```
# Bootloader from iPad 6 IPSW
# Example key is for 17.7.2
img4 -i /path/to/iPad_64bit_TouchID_Restore/Firmware/all_flash/LLB.ipad7b.RELEASE.im4p -k b9117f477dd1bf95e70c3c8de9103f787e86f64f57f1d6e48cfe3ac9199bae3851dd3925fc29715cb9b882e1b6d59ccd -o LLB.bin
# patch signature checks using the iBoot64Patcher in SSHRD Script
# or compile: https://github.com/Cryptiiiic/iBoot64Patcher
./SSHRD_Script/<OS; probably Darwin>/iBoot64Patcher LLB.bin LLB2.bin
iBootPatch2 LLB2.bin LLB3.bin
img4 -i disk2.bin -m IM4M
img4 -i LLB3.bin -A -T ibss -M IM4M -o LLB.img4
```

Now continue running commands on the device.

### Mount preboot
```
mount_apfs /dev/disk1s5 /mnt6
```

Now mark down the boot manifest hash, it is the longest folder name in `/mnt6`, that name is also stoerd in `/mnt6/active`:
```
cat /mnt6/active
mkdir -p /mnt6/cryptex1/currend
cp -a /mnt6/cryptex1/current/apticket.*.im4m /mnt6/cryptex1/currend
cp -a /mnt6/cryptex1/current/*.{root_hash,trustcache} /mnt6/cryptex1/currend
```

## Make iOS 18 Root FS
### Create file system
```
ls /dev/disk1s*
/sbin/newfs_apfs -A -D -o role=r -v Xystem /dev/disk0s1
ls /dev/disk1s*
```
Mount the new volume that appeared, on wifi models it should be `/dev/disk1s8`, and on cellular it should be `/dev/disk1s9`.

```
mount_apfs /dev/disk1s8 /mnt8 # DOUBLE CHECK THE VOLUME PATH!
```

Upload root fs to `/mnt8/root.dmg`

Unmount filesystems
```
umount /mnt8
umount /mnt6
```

APFS invert
```
/System/Library/Filesystems/apfs.fs/apfs_invert -d /dev/disk0s1 -s <LAST NUMBER IN VOLUME DEVICE FILE NAME (8 OR 9)> -n root.dmg
```

Mount preboot and iPadOS 18
```
mount_apfs /dev/disk1s5 /mnt6
mount_apfs /dev/disk1s8 /mnt8
```


Upload system cryptex to `/mnt6/cryptex1/currend/os.dmg`
Upload app cryptex to `/mnt6/cryptex1/currend/app.dmg`

### Mount iOS 17
```
mount_apfs -o ro /dev/disk1s1 /mnt1
```

### Add iPad 6 specific files

```
find /mnt1 -iregex '.*j7[[1-2]b.*' -type f -exec /bin/sh -c 'dirname="$(echo "{}" | sed -E '\''s|^/mnt1(/.+)/.+$|\1|'\'')"; filename="$(echo "{}" | sed -E '\''s|/mnt1/.+/(.+)$|\1|'\'')"; mkdir -p "/mnt8/${dirname}"; cp -an "{}" "/mnt8/${dirname}/${filename}";' \;

ln -s J171.Default.plist /mnt8/System/Library/EventTimingProfiles/J71b.Default.plist
ln -s J171.Touch.plist /mnt8/System/Library/EventTimingProfiles/J71b.Touch.plist
ln -s J171.Pencil.plist /mnt8/System/Library/EventTimingProfiles/J71b.Pencil.plist

ln -s J172.Default.plist /mnt8/System/Library/EventTimingProfiles/J72b.Default.plist
ln -s J172.Touch.plist /mnt8/System/Library/EventTimingProfiles/J72b.Touch.plist
ln -s J172.Pencil.plist /mnt8/System/Library/EventTimingProfiles/J72b.Pencil.plist
```

### Downgrade components
```
mv /mnt8/Library/Wallpaper{,.bak}
cp -a /mnt1/Library/Wallpaper /mnt8/Library

mv /mnt8/Library/Audio/Plug-Ins{,.bak}
cp -a /mnt1/Library/Audio/Plug-Ins /mnt8/Library/Audio
```

### Patch RootFS and wrap up
```
sed -i -e 's|cryptex1/current|cryptex1/currend|' /mnt8/usr/lib/dyld
ldid -Icom.apple.dyld -S /mnt8/usr/lib/dyld
```

## Create and upload preboot files
Run commands on the host

### Patch device tree

devicetree-parse: https://github.com/khanhduytran0/devicetree-parse
submodule: `projects/devicetree-parse`

Wi-Fi model: DeviceTree.j71bap.im4p, diff file `dt-j71bap.diff`
Cellular model: DeviceTree.j72bap.im4p, diff file `dt-j72bap.diff`
Unpack and patch device tree with the diff file

```
img4 -i /path/to/iPad_64bit_TouchID_Restore/Firmware/all_flash/DeviceTree.j71bap.im4p -o DeviceTree
devicetree-parse DeviceTree > DeviceTree_j71bap.jsonc
patch DeviceTree_j71bap.jsonc dt-j71bap.diff
```

### Wrapping up files
Wrap kernel in img4
```
img4 -i /path/to/iPad_10.2_Restore/kernelcache.release.ipad7c -M /path/to/onboard/IM4M -o kernelcachd
```

Wrap patched device tree
```
devicetree-repack DeviceTree_j71bap.jsonc devicetred
img4 -i devicetred -M /path/to/onboard/IM4M -A -T dtre -o devicetred.img4
```

Upload the `kernelcachd` to `/mnt6/<boot-manifest-hash>/System/Library/Caches/com.apple.kernelcaches/kernelcachd`
Upload the `devicetred.img4` to `/mnt6/<boot-manifest-hash>/usr/standalone/firmware/devicetred.img4`

Find the correct SEP firmware for your device, wifi variant uses j171 and cellular variant uses j172,
save it somewhere convenient.

```
cp /path/to/iPad_10.2_Restore/Firmware/all_flash/sep-firmware.j171.RELEASE.im4p sep-firmware.im4p
```

AVE firmware
```
img4 -i /path/to/iPad_10.2_Restore/Firmware/ave/AppleAVE2FW_H9.im4p -M /path/to/onboard/IM4M -o EVA.img4
```
Upload `EVA.img4` to `/mnt6/<boot-manifest-hash>/usr/standalone/firmware/FUD/EVA.img4`

Reboot
```
reboot
```

## Now boot into iOS 17 rootless with palera1n
Run this command on the host:
```
palera1n -l
```

Run commands on the device using palera1n's internal ssh:
```
ssh -o 'ProxyCommand=inetcat 44' -o 'UserKnownHostsFile /dev/null' -o 'StrictHostKeyChecking no' root@palera1n
```

### Fixing up /var

```
mount_apfs /dev/disk1s8 /cores/fs/fake
rm -rf /private/var/staged_system_apps
mv /cores/fs/fake/private/var/staged_system_apps /private/var
nvram p1-fakefs-rootdev=disk1s8
snaputil -c orig-fs /cores/fs/fake
```

## Kill iOS 17 while keeping it recoverable

```
cd /private/preboot/<boot-manifest-hash>/System/Library/Caches/com.apple.kernelcaches
mv kernelcache kernelcache.bak
```

## Build the boot ramdisks

https://github.com/palera1n/jbinit `ios18.4` branch
submodule: `projects/jbinit`

https://static.palera.in/binpack.tar
https://static.palera.in/artifacts/loader/universal_lite/palera1nLoader.ipa
https://static.palera.in/artifacts/loader/universal_lite/palera1nLoaderTV.ipa

```
# get binpack.tar palera1nLoader.ipa palera1nLoaderTV.ipa into the src directory of jbinit
cd projects/jbinit
gmake -j10 DEV_BUILD=1
```

## Now boot iOS 18!

Pongo.bin should be from Turdus' pongoOS repo and checkra1n-kpf-pongo should be from palera1n's PongoOS repo.
The following commands assume one used the `projects` folder in this repo.

```
PROJECTS="/path/to/ipad6-ipados18/projects"

palera1n -D
${PROJECTS}/openra1n/openra1n ${PROJECTS}/PongoOS-Turdus/build/Pongo.bin
sleep 2;
irecovery -f LLB.img4
sleep 1;
palera1n -fp -k ${PROJECTS}/PongoOS-Turdus/build/Pongo.bin
sleep 1;
printf 'fuse lock\n/send %s\nmodload\n/send %s\nsep payload\nsep sep_flag 0x12\nsep pwn\n/send %s\nmodload\npalera1n_flags 0x1\n/send %s\nramdisk\n/send %s\noverlay\nxargs %s\nbootx\n' \
	"/path/to/sep_racer" \
	"/path/to/sep-firmware.im4p" \
	"${PROJECTS}/PongoOS-KPF/build/checkra1n-kpf-pongo" \
	"${PROJECTS}/jbinit/src/ramdisk.dmg" \
	"${PROJECTS}/jbinit/src/binpack.dmg" \
	'serial=3' | pongoterm
```

## First Boot

For Wi-Fi models, the iPad can be setup from scratch if iPadOS 17's setup is never completed,
cellular models will need to be already setup due to baseband issues. Due to the wallpaper
downgrade the iPad will use iPadOS 17 wallpaper by default. iPadOS 18 wallpaper is available
online in Settings after the device has been set up.

## Credits

- [Turdus M3rula](https://github.com/turdus-m3rula) - SEP firmware loader
- [checkra1n](https://checkra.in) - the original checkm8 bootkit
- [mineek](https://github.com/mineek) - openra1n sigcheck
