# Vendor U-Boot on the EX530v

Notes from disassembling the stock bootloader (EcoNet U-Boot 2014, `tcboot`, the LZMA
blob inside `boot` at offset `0x24400`, link address `0x81e00000`) and from testing on
the device.

## Environment

```
bootcmd=flash tpimg;bootm 81800200
bootdelay=1
ipaddr=192.168.1.1
serverip=192.168.1.100
loadaddr=0x81800000
```

Any key within one second stops autoboot at `ECNT>`. `tftpboot` and `bootm` work; the
loader understands gzip and LZMA FIT images.

## `flash tpimg`: which slot is booted

1. Reads the boot flag (below). `1` selects image1 (slot B), anything else image0 (slot A).
2. Reads the 0x200 byte header from **raw** flash `0x2cc0000` (slot B) or `0x4c0000`
   (slot A) to `0x81800000`.
3. Reads the payload: length = little-endian u32 at header offset `0x78`.
4. Checks one thing only: little-endian u32 at offset `0` must be `0x03000003`, otherwise
   `tp image format error`. **There is no MD5 or signature check.**
5. Sets the `root=` variable (`/dev/mtdblock5` for image0, `/dev/mtdblock8` for image1),
   the MAC address from `misc` + `0x8f100`, and rebuilds `bootargs` from the environment.
6. `bootm 81800200` boots the FIT that follows the header (FIT hashes are verified).

So a slot B image is: a 0x200 byte header (magic `03 00 00 03`, FIT size at `0x78`, the
rest zero) followed by a normal FIT with kernel + DTB. The `ex530v-tp-header` build step in
`en7523.mk` produces it.

Because U-Boot always adds `root=/dev/mtdblock8 ro`, the OpenWrt kernel is built with
`CONFIG_CMDLINE_FORCE`. The root file system is then found automatically: the `ubi`
partition is attached and the `rootfs` volume becomes `ubiblock0_0`.

## Boot flag used by autoboot

8 bytes at raw flash **`0x5940000`** (start of the stock `reserve` partition):

- byte 7 = (sum of bytes 0..6) & 0xff, otherwise `Boot flag csum incorrect, return image0`
- image1 is selected when byte 4 and byte 5 are both non-zero

On a new unit the area is erased (`0xff`), so the stock firmware (image0) boots.

It is written by the **hidden** sub-command `flash bflag <0|1>` (not listed in
`help flash`). On erased flash, `flash bflag 1` leaves:

```
00 00 ff 01 01 01 00 02
```

OpenWrt's sysupgrade handler (`ex530v_set_bootflag` in
`target/linux/airoha/en7523/base-files/lib/upgrade/platform.sh`) writes
`00 00 00 01 01 01 00 03` to the `reserve` partition after a successful flash, unless a
valid image1 flag is already there. That is why a single `sysupgrade -n` from the RAM
image is enough to install.

You can check it with:

```
flash read 5940000 10 85000000
md.b 85000000 10
```

## The `bootflag` command is a trap

`bootflag read` / `bootflag swap` use a **different** flag: one ASCII character (`'0'` /
`'1'`) at raw flash `0x1bfc0000`. `flash tpimg` never reads it, so `bootflag swap` does not
change which slot boots. It does read-modify-write the whole erase block at `0x1bfc0000`.
The OpenWrt layout ends the UBI area just before that block, maps it as `bootflag` and
leaves it alone.

## BMT pool

The loader reports `bmt pool size: 163`. The last 163 blocks (`0x1d740000` .. end) are the
vendor bad-block management pool: a `RAWB` table in its first block and a `BMT` table in
the last one. OpenWrt does not map this area. Nothing may write it.
