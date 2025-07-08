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

**⚠️Nothing tested after this point!**

```bash
mtk plstage --vid 0e8d --pid 2000 --skipwdt 1

OR

mtk plstage init --vid 0e8d --pid 2000 --debugmode


```

Once you are in BROM read the scatter.

```bash
mtk readgpt --json gpt.json

mtk gpt2scatter gpt.json MyScatter.txt

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

Only the Zlink build that came with the device works. Not any upgrades. Perhaps the low level drivers are not compatible with the new Zlink builds.

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
