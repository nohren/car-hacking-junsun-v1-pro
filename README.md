# 🚗 Car Hacking Junsun V1 Pro Head unit 📻

```bash
https://www.amazon.com/dp/B0D5GYJPBB?ref=ppx_yo2ov_dt_b_fed_asin_title&th=1
```
```bash
https://xdaforums.com/t/junsun-v1-pro-mt8163-thread.4692130/
```


Android is a wonderful free open source thing.

This is my annotated journey with the Junsun V1 Pro.  It started with an enthusiam for apple car play and a dissapointment that it was handsfree only and no tethered usb carplay option was available.

**⚠️Precaution statement - replicate anything you learn here at your own risk!!!**

## Root 
We want root! 

Maybe if I can root, I can find where tethered carplay was turned off or forgotten about. After using Qute terminal emulator on device and being denied every which way, it was clear that root priveledges was the only way to get answers.

### tools
1. Very sparse tutorial - I really had no idea what I was getting myself into. https://xdaforums.com/t/junsun-v1-pro-mt8163-thread.4692130/page-5#post-89932439
2. mtk client - https://github.com/bkerler/mtkclient
3. sp flash tool - https://spflashtool.com/#google_vignette
4. Magisk - https://topjohnwu.github.io/Magisk/

### Overview
In order to root, we need the boot.img so we can patch the kernel with root with Magisk tool. In order to read the boot image we need the precise scalar offset within the on board eMMC internal flash memory for boot img.  We also would like its size.  The eMMC memory is a byte[] array, 32gb in size. The next problem is I have no reliable sources on what that offset is.  There is an online scatter found but its not even valid as  If I trust blindly whatever i find online, such as preloader and pgpt are supposedly located at the same position 0x0, I may overwrite something critical and brick the poor machine forever.

### Experiment

Rather than guess, I decided to use mtkClient to read the memory and produce an exact scatter.  With an exact scatter i can use SP Flash tools to read out the boot.img bytes and Magisk to create a new boot.img with root.  Then use SP flash tool to write that new boot.img to ROM.  And to do all this btw you have to be BROM mode. From wikipedia "BROM (Boot ROM) is a crucial component in embedded systems, responsible for the initial booting process when the device is powered on or reset". SP Flash Tools binary requires windows or linux. I chose to continue using my Macbook and use UTM to emulate x86 linux ... RIP.  I totally would have used windows if my laptop wasn't entirely dependent on its power cord.

In the end UTM would have worked if it could handle rapidly appearing and disapearing usb connections, like say from a boot loop awaiting protocol handshake to enter BROM. Unfortunely for UTM 4.6.5, this is not the case. Likely there is some null pointer exception crashing things when the usb is no longer who UTM thought it was.  UTM simply cannot handle it.  I tried dowloading the "[nightly](https://github.com/utmapp/UTM/actions/runs/16120262973)" UTM from github actions at behest of ChatGPT and nope, macOS is not having it, not with unsigned apps.

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

1. 🔋 5V Signal - First we need to send 5V to the 4pin connector. Your headunit will not budge without it. You will need a **USB A male to USB A female 4pin**.  I learned the hard way that USB A female to female will not transmit 5V signal and a USB male to USB male even though it will transmit the 5v won't fit onto the male harness receptor... you need a female for that. I spent the better part of 2 hours poking things with a multimeter diagnosing why 5V signal was so hard to get working. I found a usb A male to male that should supposedly help transmit 5V but even that did not transmit the 5V signal!!! 🤯. Apparently some cables are meant for data only and remove the power on purpose.  So i took the 4pin that came with the radio, I cut off the USB A female head, and spliced those wires to the USB A male. Voila.
2. Once 5V is hitting the unit, you are good to proceed to the next step.  Cycling 12v power. First use a paper clip to depress **RST** for 2 seconds while keeping ignition in **LOW**. Now key the ignition to **ACC** for $<=0.5$ seconds and back to **LOW**. Do this 2 to 3 times and the display will no longer light up. You should see that the buttons now flash white every 10 seconds or so. Welcome to the boot loop.
3. Right now the machine is looping periodically showing up as `MT65xx Preloader` and then disapearing. Prep the machine for the protocol handshake which enters it to the golden land of BROM. Preparation code from MTK as follows. This code has mtk looking for the handshake.

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
-  Install the apk on your head unit. 
-  Supply the boot.img to it.  
-   Let it create a root_boot.img.  Place that back onto the flash drive.

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
