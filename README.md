# Car Hacking

Android is a wonderful free open source thing

## Root

**SP Flash tool, Windows computer and scatter**

The head units persistent storage is flash memory in the eMMC.  This is a byte[] array, 32gb in size.
The scatter file tells us where the boot partition is... this number is a scalar offset from the start or 0 byte.  
In physical terms, each byte in the byte array represents a flash memory container.
We are going to compare what we see with what the scatter file tells us.  
When we extract the boot.img we give to magisk tool to add the backdoor.

Then we write to the new magisk boot.img **ONLY!** to the boot partition.  Writing anywhere else means soft/hard brick.  BEWARE.

Read phase
1. Download SP Flash tool
2. 