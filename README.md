# 🚗 Car Hacking Junsun V1 Pro Head unit 📻

```bash
https://www.amazon.com/dp/B0D5GYJPBB?ref=ppx_yo2ov_dt_b_fed_asin_title&th=1
```

```bash
https://xdaforums.com/t/junsun-v1-pro-mt8163-thread.4692130/
```

Android is a wonderful free open source thing.

This is my annotated journey with the Junsun V1 Pro. It started with an enthusiam for apple car play and a dissapointment that it was handsfree only and no tethered usb carplay option was available.

**⚠️Precaution statement - replicate anything you learn here at your own risk!!!**

```bash
System info
System platform: MT8163
Build Number: AJ_2024.09.28.12_3_2224
MCU_version: CVD1810-WJ_23.12.12_399

```

## Root

We want root!

Maybe if I can root, I can find where tethered carplay was turned off or forgotten about. After using Qute terminal emulator on device and being denied every which way, it was clear that root priveledges was the only way to get answers.

### tools

1. Very sparse tutorial - I really had no idea what I was getting myself into. https://xdaforums.com/t/junsun-v1-pro-mt8163-thread.4692130/page-5#post-89932439
2. mtk client - https://github.com/bkerler/mtkclient
3. sp flash tool - https://spflashtool.com/#google_vignette
4. Magisk - https://topjohnwu.github.io/Magisk/

### Overview

In order to root, we need the boot.img so we can patch the kernel with root with Magisk tool. In order to read the boot image we need the precise scalar offset within the on board eMMC internal flash memory for boot img. We also would like its size. The eMMC memory is a byte[] array, 32gb in size. The next problem is I have no reliable sources on what that offset is. There is an online scatter found but its not even valid, as you can see in online_scatter.txt, the preloader and pgpt are supposedly located at the same offset 0x0. ⚠️ CAUTION — POTENTIAL FOR PERMANENT BRICKING. I can easily brick this Chinese marvel of engineering, cost efficiency, and android version spoofing forever by writing the wrong bytes to wrong offsets. (they claim android 13, it really runs 9. Marketing is a hell of a drug).

### Experiment

Rather than guess and brick 🧱, I decided to use mtkClient to read to actual layout and produce an exact scatter. With an exact scatter I can use SP Flash tools and Magisk with confidence. To do any of this though, I have to be in BROM mode first. From wikipedia "BROM (Boot ROM) is a crucial component in embedded systems, responsible for the initial booting process when the device is powered on or reset".
As for SP Flash Tools, the binary requires 🪟 windows or 🐧 linux **x86 only**. I chose to continue using my Macbook and use UTM to emulate x86 linux ........... RIP 😭. I totally would have used windows if my laptop wasn't entirely dependent on its power cord.

So I went the UTM route, which would have worked if it could handle rapidly appearing and disappearing usb connections, like say from a boot loop awaiting protocol handshake to enter BROM. Unfortunately for UTM 4.6.5, this is not the case. Likely there is some null pointer exception crashing things when the usb is no longer who UTM thought it was. UTM simply cannot handle the truth. I tried dowloading the "[nightly](https://github.com/utmapp/UTM/actions/runs/16120262973)" UTM from github actions at behest of ChatGPT and nope, macOS is not having it, not with unsigned apps. They be scary 👻 🔥💰🔥.

Linux Quirks for SP FlashTools and MTK install

```bash
#sudo -i is your friend. Root for yourself.

#to get sp flash tool to work, download libpng and place in lib folder.  cd /tmp

#dpkg-deb -x libpng12-0_1.2.54-1ubuntu1_amd64.deb tmp cd ~/car_hacks/SP_Flash_Tool-5.1916_Linux/lib


#cp /tmp/tmp/lib/x86_64-linux-gnu/libpng12.so.0.* .
#ln -s libpng12.so.0.* libpng12.so.0


#To get valid scatter, using mtk, which got from github and downloaded to pipx venv.

#apt install libfuse2 to get it to work. Otherwise it said it couldn’t find libfuse
```

I will try again later with a battery functional windows machine.

### Procedure for putting device in boot loop & BROM

This is the place is where you can flash new firmware.

1. 🔋 5V Signal - First things first, the 4pin connector ⬅️ needs to recieve 5V . Your headunit will not budge without it. Notice **I did not** say the 4pin connected ➡️ needs to send 5V (normal charging operation). In order to do this, you will need a **USB A male to USB A female 4pin**. I learned the hard way that USB A female to female **will not** transmit 5V signal and a USB male to USB male will transmit even though sometimes it won't. Yes you read that right. I spent the better part of 2 hours poking things with a multimeter diagnosing why 5V signal was so hard to get working. I found a usb A male to male that should supposedly transmit 5V but even that did not transmit the 5V signal!!! 🤯. Apparently some cables are meant for data only and remove the power on for safety reasons. To fix things for today, I took the 4pin that came with the radio, I cut off the USB A female head, and spliced the line onto a USB A male head. Voila, Houston we have liftoff 🔋.
2. Once 5V is hitting the unit, we are good to proceed to the next step. Cycling 12v power. First use a paper clip to depress **RST** for 2 seconds while keeping ignition in the **OFF** position. This makes sure the unit powers off. Now with the key in the ignition turn from **OFF** to **ACC** for $<=0.5$ seconds and back to **OFF**. Do this 2 to 3 times and the display will no longer light up. You should see that the buttons now flash white every 10 seconds or so. Welcome to the boot loop. BTW OFF is actually LOW voltage. This is why the system stays on for a period of time, even without the keys in the ignition. Embedded systems, am I right? 🤖
3. Look for a SIN. Since I was emulating Linux, I first recieved these signals on my macbook pro.

```bash
>>> system_profiler SPUSBDataType

<<<
 MT65xx Preloader:

              Product ID: 0x2000
              Vendor ID: 0x0e8d  (MediaTek Inc.)
              Version: 1.00
              Speed: Up to 480 Mb/s
              Manufacturer: MediaTek
              Location ID: 0x00120000 / 4
              Current Available (mA): 500
              Current Required (mA): 500
              Extra Operating Current (mA): 0


```

3. Send ACK - Right now the machine is looping periodically showing up as `MT65xx Preloader` and then disapearing. Prep the machine for the protocol handshake which enters it to the golden land of BROM. Preparation code from MTK as follows. This code has mtk looking for the handshake.

On UTM Linux emulator, check "do not show prompt when usb device is connected."
Enable USB 2.0 support in input. Disable host passthrough. We will use QEMU usb passthrough backend.

QEMU usb [documentation](https://www.qemu.org/docs/master/system/devices/usb.html#connecting-usb-devices)

```bash
-device usb-host,vendorid=0x0e8d,productid=0x2000
```

We are in preloader. And without scatter. Lets send the ACK to the device. This is the code that will send the ACK to the device and put it into BROM mode.

```bash
>>> mtk plstage \
  --vid 0x0e8d --pid 0x2000 \
  --skipwdt 1

<<<

MTK Flash/Exploit Client Public V2.0.1 (c) B.Kerler 2018-2024

Preloader - Status: Waiting for PreLoader VCOM, please reconnect mobile to brom mode

Port - Hint:

Power off the phone before connecting.
For brom mode, press and hold vol up, vol dwn, or all hw buttons and connect usb.
For preloader mode, don't press any hw button and connect usb.
If it is already connected and on, hold power for 10 seconds to reset.


Port - Device detected :)
Preloader - 	CPU:			MT8163()
Preloader - 	HW version:		0x0
Preloader - 	WDT:			0x10007000
Preloader - 	Uart:			0x11002000
Preloader - 	Brom payload addr:	0x100a00
Preloader - 	DA payload addr:	0x201000
Preloader - 	CQ_DMA addr:		0x10212c00
Preloader - 	Var1:			0xb1
Preloader - HW code:			0x8163
Preloader - Target config:		0x0
Preloader - 	SBC enabled:		False
Preloader - 	SLA enabled:		False
Preloader - 	DAA enabled:		False
Preloader - 	SWJTAG enabled:		False
Preloader - 	EPP_PARAM at 0x600 after EMMC_BOOT/SDMMC_BOOT:	False
Preloader - 	Root cert required:	False
Preloader - 	Mem read auth:		False
Preloader - 	Mem write auth:		False
Preloader - 	Cmd 0xC8 blocked:	False
Preloader - Get Target info
Preloader - 	HW subcode:		0x8a00
Preloader - 	HW Ver:			0xcb00
Preloader - 	SW Ver:			0x1
Main - Connected to device, loading
Main - Sent stage2 to 0x40001000, length 0x4658
Preloader - Jumping to 0x40001000
Preloader - Jumping to 0x40001000: ok.
Main - Jumped to stage2 at 0x40001000.
Main - Successfully loaded stage2
```

Try to Crash into BROM. Doing this can cause it to crash the preloader mode. Meaning remove 5v and start again.

```bash
>>> mtk crash --vid 0x0e8d --pid 0x2000

<<<
Port - Device detected :)
Preloader - 	CPU:			MT8163()
Preloader - 	HW version:		0x0
Preloader - 	WDT:			0x10007000
Preloader - 	Uart:			0x11002000
Preloader - 	Brom payload addr:	0x100a00
Preloader - 	DA payload addr:	0x201000
Preloader - 	CQ_DMA addr:		0x10212c00
Preloader - 	Var1:			0xb1
Preloader - Disabling Watchdog...
Preloader - HW code:			0x8163
Preloader - Target config:		0x0
Preloader - 	SBC enabled:		False
Preloader - 	SLA enabled:		False
Preloader - 	DAA enabled:		False
Preloader - 	SWJTAG enabled:		False
Preloader - 	EPP_PARAM at 0x600 after EMMC_BOOT/SDMMC_BOOT:	False
Preloader - 	Root cert required:	False
Preloader - 	Mem read auth:		False
Preloader - 	Mem write auth:		False
Preloader - 	Cmd 0xC8 blocked:	False
Preloader - Get Target info
Preloader - 	HW subcode:		0x8a00
Preloader - 	HW Ver:			0xcb00
Preloader - 	SW Ver:			0x1
Mtk - We're not in bootrom, trying to crash da...
Exploitation - Crashing da...
Preloader
Preloader - [LIB]: DA_Send status error:DA_INVALID_LENGTH (0x7004)
Preloader - Status: Waiting for PreLoader VCOM, please reconnect mobile to brom mode

Port - Hint:

Power off the phone before connecting.
For brom mode, press and hold vol up, vol dwn, or all hw buttons and connect usb.
For preloader mode, don't press any hw button and connect usb.
If it is already connected and on, hold power for 10 seconds to reset.


...........

Port - Hint:

Power off the phone before connecting.
For brom mode, press and hold vol up, vol dwn, or all hw buttons and connect usb.
For preloader mode, don't press any hw button and connect usb.
If it is already connected and on, hold power for 10 seconds to reset.


......Port - Device detected :)
Preloader - 	CPU:			MT8163()
Preloader - 	HW version:		0x0
Preloader - 	WDT:			0x10007000
Preloader - 	Uart:			0x11002000
Preloader - 	Brom payload addr:	0x100a00
Preloader - 	DA payload addr:	0x201000
Preloader - 	CQ_DMA addr:		0x10212c00
Preloader - 	Var1:			0xb1
Preloader - Disabling Watchdog...
Preloader - HW code:			0x8163
Preloader - Target config:		0x0
Preloader - 	SBC enabled:		False
Preloader - 	SLA enabled:		False
Preloader - 	DAA enabled:		False
Preloader - 	SWJTAG enabled:		False
Preloader - 	EPP_PARAM at 0x600 after EMMC_BOOT/SDMMC_BOOT:	False
Preloader - 	Root cert required:	False
Preloader - 	Mem read auth:		False
Preloader - 	Mem write auth:		False
Preloader - 	Cmd 0xC8 blocked:	False
Preloader - Get Target info
Preloader - 	HW subcode:		0x8a00
Preloader - 	HW Ver:			0xcb00
Preloader - 	SW Ver:			0x1>>

```

## Weird i'm in preloader but i can grab BROM stuff by talking to stage 2 DA legacy.

Have sent it into stage 2 with plstage. Then removed 5v, then added 5v and voila handshake connected to stage 2. And then I can read the GPT table.

```bash
>>> mtk printgpt --vid 0x0e8d --pid 0x2000

<<<
........Port - Device detected :)
Preloader - 	CPU:			MT8163()
Preloader - 	HW version:		0x0
Preloader - 	WDT:			0x10007000
Preloader - 	Uart:			0x11002000
Preloader - 	Brom payload addr:	0x100a00
Preloader - 	DA payload addr:	0x201000
Preloader - 	CQ_DMA addr:		0x10212c00
Preloader - 	Var1:			0xb1
Preloader - Disabling Watchdog...
Preloader - HW code:			0x8163
Preloader - Target config:		0x0
Preloader - 	SBC enabled:		False
Preloader - 	SLA enabled:		False
Preloader - 	DAA enabled:		False
Preloader - 	SWJTAG enabled:		False
Preloader - 	EPP_PARAM at 0x600 after EMMC_BOOT/SDMMC_BOOT:	False
Preloader - 	Root cert required:	False
Preloader - 	Mem read auth:		False
Preloader - 	Mem write auth:		False
Preloader - 	Cmd 0xC8 blocked:	False
Preloader - Get Target info
Preloader - 	HW subcode:		0x8a00
Preloader - 	HW Ver:			0xcb00
Preloader - 	SW Ver:			0x1
DaHandler - Device is unprotected.
DaHandler - Device is in Preloader-Mode.
DALegacy - Uploading legacy da...
DALegacy - Uploading legacy stage 1 from MTK_DA_V5.bin
LegacyExt - Legacy DA2 is patched.
LegacyExt - Legacy DA2 CMD F0 is patched.
Preloader - Jumping to 0x200000
Preloader - Jumping to 0x200000: ok.
DALegacy - Got loader sync !
DALegacy - Reading nand info
DALegacy - Reading emmc info
DALegacy - ACK: 0402a1
DALegacy - Setting stage 2 config ...
DALegacy - Uploading stage 2...
DALegacy - Successfully uploaded stage 2
DALegacy - Connected to stage2
DALegacy - m_int_sram_ret = 0x0
m_int_sram_size = 0x40000
m_ext_ram_ret = 0x0
m_ext_ram_type = 0x2
m_ext_ram_chip_select = 0x0
m_int_sram_ret = 0x0
m_ext_ram_size = 0x80000000
randomid = 0x92ABD0F219C3BD53292794B2BB557A09

m_emmc_ret = 0x0
m_emmc_boot1_size = 0x400000
m_emmc_boot2_size = 0x400000
m_emmc_rpmb_size = 0x400000
m_emmc_gp_size[0] = 0x0
m_emmc_gp_size[1] = 0x0
m_emmc_gp_size[2] = 0x0
m_emmc_gp_size[3] = 0x0
m_emmc_ua_size = 0x748000000
m_emmc_cid = 43413332df01185316057b3b4710a55d
m_emmc_fwver = 1000000000000000


GPT Table:
-------------
proinfo:             Offset 0x0000000000080000, Length 0x0000000000300000
                     Flags 0x00000000, UUID f57ad330-39c2-4488-b09b-00cb43c9ccd4, Type EFI_LINUX_DATA
nvram:               Offset 0x0000000000380000, Length 0x0000000000500000
                     Flags 0x00000000, UUID fe686d97-3544-4a41-21be-167e25b61b6f, Type EFI_LINUX_DATA
protect1:            Offset 0x0000000000880000, Length 0x0000000000a00000
                     Flags 0x00000000, UUID 1cb143a8-b1a8-4b57-51b2-945c5119e8fe, Type EFI_LINUX_DATA
protect2:            Offset 0x0000000001280000, Length 0x0000000000a00000
                     Flags 0x00000000, UUID 3b9e343b-cdc8-4d7f-a69f-b6812e50ab62, Type EFI_LINUX_DATA
persist:             Offset 0x0000000001c80000, Length 0x0000000003000000
                     Flags 0x00000000, UUID 5f6a2c79-6617-4b85-02ac-c2975a14d2d7, Type EFI_LINUX_DATA
seccfg:              Offset 0x0000000004c80000, Length 0x0000000000040000
                     Flags 0x00000000, UUID 4ae2050b-5db5-4ff7-d3aa-5730534be63d, Type EFI_LINUX_DATA
lk:                  Offset 0x0000000004cc0000, Length 0x0000000000060000
                     Flags 0x00000000, UUID 1f9b0939-e16b-4bc9-bca5-dc2ee969d801, Type EFI_LINUX_DATA
lk2:                 Offset 0x0000000004d20000, Length 0x0000000000060000
                     Flags 0x00000000, UUID d722c721-0dee-4cb8-838a-2c63cd1393c7, Type EFI_LINUX_DATA
boot:                Offset 0x0000000004d80000, Length 0x0000000001000000
                     Flags 0x00000000, UUID e02179a8-ceb5-48a9-3188-4f1c9c5a8695, Type EFI_LINUX_DATA
recovery:            Offset 0x0000000005d80000, Length 0x0000000001000000
                     Flags 0x00000000, UUID 84b09a81-fad2-41ac-0e89-407c24975e74, Type EFI_LINUX_DATA
secro:               Offset 0x0000000006d80000, Length 0x0000000000600000
                     Flags 0x00000000, UUID e8f0a5ef-8d1b-42ea-2a9c-835cd77de363, Type EFI_LINUX_DATA
para:                Offset 0x0000000007380000, Length 0x0000000000080000
                     Flags 0x00000000, UUID d5f0e175-a6e1-4db7-c094-f82ad032950b, Type EFI_LINUX_DATA
logo:                Offset 0x0000000007400000, Length 0x0000000000800000
                     Flags 0x00000000, UUID 1d9056e1-e139-4fca-0b8c-b75fd74d81c6, Type EFI_LINUX_DATA
dtbo:                Offset 0x0000000007c00000, Length 0x0000000000800000
                     Flags 0x00000000, UUID 7792210b-b6a8-45d5-91ad-3361ed14c608, Type EFI_LINUX_DATA
vendor:              Offset 0x0000000008400000, Length 0x0000000012c00000
                     Flags 0x00000000, UUID 138a6db9-1032-451d-e991-0fa38ff94fbb, Type EFI_LINUX_DATA
apd:                 Offset 0x000000001b000000, Length 0x0000000040000000
                     Flags 0x00000000, UUID 756d934c-50e3-4c91-46af-02d824169ca7, Type EFI_LINUX_DATA
nvrom:               Offset 0x000000005b000000, Length 0x0000000001000000
                     Flags 0x00000000, UUID a3f3c267-5521-42dd-24a7-3bdec20c7c6f, Type EFI_LINUX_DATA
expdb:               Offset 0x000000005c000000, Length 0x0000000000a00000
                     Flags 0x00000000, UUID 8c68cd2a-ccc9-4c5d-578b-34ae9b2dd481, Type EFI_LINUX_DATA
frp:                 Offset 0x000000005ca00000, Length 0x0000000000100000
                     Flags 0x00000000, UUID 6a5cebf8-54a7-4b89-1d8d-c5eb140b095b, Type EFI_LINUX_DATA
tee1:                Offset 0x000000005cb00000, Length 0x0000000000500000
                     Flags 0x00000004, UUID a0d65bf8-e8de-4107-3494-1d318c843d37, Type EFI_LINUX_DATA
tee2:                Offset 0x000000005d000000, Length 0x0000000000500000
                     Flags 0x00000000, UUID 46f0c0bb-f227-4eb6-2fb8-66408e13e36d, Type EFI_LINUX_DATA
kb:                  Offset 0x000000005d500000, Length 0x0000000000200000
                     Flags 0x00000000, UUID fbc2c131-6392-4217-1eb5-548a6edb03d0, Type EFI_LINUX_DATA
dkb:                 Offset 0x000000005d700000, Length 0x0000000000200000
                     Flags 0x00000000, UUID e195a981-e285-4734-2580-ec323e9589d9, Type EFI_LINUX_DATA
metadata:            Offset 0x000000005d900000, Length 0x0000000002000000
                     Flags 0x00000000, UUID e29052f8-5d3a-4e97-b5ad-5f312ce6610a, Type EFI_LINUX_DATA
vbmeta:              Offset 0x000000005f900000, Length 0x0000000000f00000
                     Flags 0x00000000, UUID 9c3cabd7-a35d-4b45-578c-b80775426b35, Type EFI_LINUX_DATA
system:              Offset 0x0000000060800000, Length 0x0000000080000000
                     Flags 0x00000000, UUID e7099731-95a6-45a6-e5a1-1b6aba032cf1, Type EFI_LINUX_DATA
cache:               Offset 0x00000000e0800000, Length 0x0000000013000000
                     Flags 0x00000000, UUID 8273e1ab-846f-4468-99b9-ee2ea8e50a16, Type EFI_LINUX_DATA
userdata:            Offset 0x00000000f3800000, Length 0x0000000653780000
                     Flags 0x00000000, UUID d26472f1-9ebc-421d-14ba-311296457c90, Type EFI_LINUX_DATA
flashinfo:           Offset 0x0000000746f80000, Length 0x0000000001000000
                     Flags 0x00000000, UUID b72ccbe9-2055-46f4-67a1-4a069c201738, Type EFI_LINUX_DATA

Total disk size:0x0000000748000000, sectors:0x0000000003a40000



```

So much for my spliced usb 4pin to male cable. It makes it about 6% in. I need to get some real hardware. Either that or this doens't work unless 'i'm in BROM.

```bash
Progress: |█---------| 6.2% Read (0x800/0x8000, 04s left) 3.04 MB/sDeviceClass - USBError(5, 'Input/Output Error')
DeviceClass - USBError(32, 'Pipe error')
Traceback (most recent call last):
  File "/Users/nohren/miniconda3/envs/mtk/bin/mtk", line 8, in <module>
    sys.exit(main())
             ^^^^^^
  File "/Users/nohren/miniconda3/envs/mtk/lib/python3.12/site-packages/mtkclient/mtk.py", line 1017, in main
    mtk = Main(args).run(parser)
          ^^^^^^^^^^^^^^^^^^^^^^
  File "/Users/nohren/miniconda3/envs/mtk/lib/python3.12/site-packages/mtkclient/Library/mtk_main.py", line 684, in run
    da_handler.handle_da_cmds(mtk, cmd, self.args)
  File "/Users/nohren/miniconda3/envs/mtk/lib/python3.12/site-packages/mtkclient/Library/DA/mtk_da_handler.py", line 788, in handle_da_cmds
    if self.da_ro(start=start, length=length, filename=filename, parttype=parttype):
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/Users/nohren/miniconda3/envs/mtk/lib/python3.12/site-packages/mtkclient/Library/DA/mtk_da_handler.py", line 390, in da_ro
    return self.mtk.daloader.readflash(addr=start,
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/Users/nohren/miniconda3/envs/mtk/lib/python3.12/site-packages/mtkclient/Library/DA/mtk_daloader.py", line 312, in readflash
    return self.da.readflash(addr=addr, length=length, filename=filename, parttype=parttype, display=display)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/Users/nohren/miniconda3/envs/mtk/lib/python3.12/site-packages/mtkclient/Library/DA/legacy/dalegacy_lib.py", line 1116, in readflash
    checksum = unpack(">H", self.usbread(2))[0]
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
struct.error: unpack requires a buffer of 2 bytes

```

### Read

Open SP Flash Tools

- load the scatter file.
- Click read and check the boot.img box.
- Read boot.img.
- write boot.img to a flash drive.
- Save this somewhere in case you get into trouble.

### Add root to boot.img

- Download the [magisk](https://topjohnwu.github.io/Magisk/) apk onto the same flash drive.
- Install the apk on your head unit.
- Supply the boot.img to it.
- Let it create a root_boot.img. Place that back onto the flash drive.

### Write

- Open SP Flash Tools
- Load scatter file
- Click on write or download
- Check boot.img
- give it root_boot.img
- Download
- Restart
- Verify if working or not

## Car play tethering

... TBD rooting

## Other interesting things

Unlock developer menu - click on the build number 7 times in settings. Enter the current date as `(YYYY-MM-DD)` as password.

Unlock car settings factory menu - click factory and enter password `000000`.

Add a theme img - copy png's to flash drive. Paste them to head unit disk. Select image in display settings.

Only the Zlink build that came with the device works. Not any upgrades. Perhaps the low level drivers are not compatible with the newer Zlink builds. Some things feel very hardcoded.

```bash
Zlink:

ID 808C028D9F7F6DC934DF67E4A4203C6F
SN DF011853434133324710A55D16057B3B
KEY HCZJWLAY6C4481
PLATFORM
HENGCHEN-AJ-tb8163p3_bsp
WIRELESS
CPW AAW
DISPLAY
1280x720-1280x720
FEATURES
CPL CPW AAL AAW QTL QTW AML AMW
VERSION
34696b9e4ef6323d7b9fcacb91627ec8ecd301dc
```
