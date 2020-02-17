+++
Tags = ["bios","elitebook","840 g1", "password","HP"]
Description = ""
date = "2020-02-17T22:35:15+01:00"
title = "Resetting your HP BIOS password by modifying the main BIOS chip"
draft = false
+++

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;My inner security expert turned on the BIOS password on a HP laptop (EliteBook 840 G1), straight away after I bought it, almost over 6 years ago. Of course I forgot what it was after a while, since I did not have a reason to change any of the settings, but this changed recently. I really wanted to play with [WSL2](https://docs.microsoft.com/en-us/windows/wsl/wsl2-index) (Windows subtype for Linux) and Docker for Windows, and one of the prerequisites for both is to [enable Hyper-V](https://docs.microsoft.com/en-us/windows/wsl/wsl2-faq) which is a setting in BIOS.\
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Is there a way I can remove the BIOS password? This blog post describes my attempts which finally resulted in success recently. 

<!--more-->

## Solutions I decided not to go with

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;These are the solutions I have considered, but ended up not going with, because they weren't reasonable or simply not possible anymore (HP support). Note that item 4 and the solution I ended up going with, both need the BIOS chip to be desoldered. This might sound scary, but it's actually quite simple with a little bit of hardware skills and the necessary equipment (or having a friend with both).

1. Contact official HP support
2. Buy a new laptop
3. HP BIOS Unlock 
4. Buy an unlocked BIOS chip on Ebay

##### Contact official HP support
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;This was the way to go until [HP decided to stop providing the solution](https://support.hp.com/us-en/document/c06368824) sometime in the middle of 2019. The process was to send HP the serial number of your laptop for which they would generate a SMC.bin file and provide the instructions how to override the existing BIOS. Note that any SMC.bin files you will find on the internet, will not work for you, because the serial numbers will not match. I assume the serial number in the binary file is encrypted, which makes it impossible for you to replace.

##### Buy a new laptop
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;To much of an overkill, even though the [laptop](https://support.hp.com/gb-en/document/c03961746) is 6 years old I7 with 16 GB of RAM it is still in mint condition and does everything I need.

##### HP BIOS Unlock tool
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;There is a [tool](http://mazzifsoftware.blogspot.com/p/hp-bios-unlock.html) that works with some HP models, but not all of them. I think the newer HP models have extra security that does not allow you to flash the BIOS just like this tool does.\
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;This is not the same tool as mentioned and used later in the solution.

##### Buy unlocked BIOS chip on Ebay
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;This involves replacing the main BIOS chip on your motherboard with an unlocked one you buy on Ebay. In fact the seller recommended replacing two chips. Main BIOS chip + EC (Embed Controller) chip, but he doesn't explain why.\
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;You also need to let them know the SKU, serial number and the model of your laptop. BIOS image dumps are not transferable, you can't use the same dump on two machines - even if they are exactly the same model. They will take this information to [initialize the ME region (Data section)](https://www.win-raid.com/t1658f39-Guide-Clean-Dumped-Intel-Engine-CS-ME-CS-TXE-Regions-with-Data-Initialization.html).\
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;I have not decided to go with this solution because I found a more elegant one. Solution presented next will specifically target only the BIOS password and will leave the ME Data region unchanged.

## Solution
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;In the end I decided to go with the least invasive solution that doesn't involve changing any hardware parts or making massive modifications to the BIOS dump file.\
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;None of the work in this post is my own. This post is just an aggregate of everything I learned during my research, that gave me the understanding and confidence I needed to get the job done. The amount of information you can find about the hardware components is incredible. Next time one of my gadgets breaks, I will definitely put more effort into trying to get it fixed rather than throwing it away and buying a new one :). You can find all the links to hardware forums used in this project later on.

The final solution includes:

1. Identifying the correct chip
2. Desoldering main bios chip
3. Modifying the content
4. Putting the main BIOS chip back

##### Identifying the correct chip
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;There are two BIOS flash memory chips and first we need to identify which one we need. Why are there motherboards with multiple BIOS chips I don't know, I guess separation of concerns. Anyone interested in this can read it [here](http://www.bios-chip24.com/epages/63730052.sf/en_GB/?ObjectPath=/Shops/63730052/Categories/Informationen_zum_Bios/Bios_Chip_lokalisierung). As I already mentioned some sellers on Ebay are offering 2 unlocked bios chips, but after reading online, I learned this is not necessary.\
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;In my case the correct chip was marked with a red dot, and was located right above the CMOS battery.

{{< figure link="/img/chip_location.png" src="/img/chip_location_small.png" caption="main BIOS memory chip above the CMOS battery">}}

{{< figure link="/img/winbond_25q128fvsq.jpg" src="/img/winbond_25q128fvsq_small.jpg" caption="Winbond 25Q128FVSQ-1318">}}

##### Desoldering main BIOS chip
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;This step is easy if you have the right tools and experience (it took a hobby enthusiast 10 minutes). Place the chip into the adapter and plug it into an EEPROM programmer. In my case the main BIOS chip was a "Winbond 25Q128FVSQ-1318" and when copied its contents I got a dump of size 16MB. You can find all of the equipment used later in the post. Make sure you keep a copy of your original BIOS dump...just in case!!

{{< figure link="/img/Programmer_Adapter_Socket.png" src="/img/Programmer_Adapter_Socket_small.png" caption="SOIC8 SOP8 to DIP8 adapter for the BIOS chip">}}

##### Modifying the content
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;It is easy if you know what you need to do. In my case I had to clear 87 bytes starting at address 0DB3470. I learned this on [HP EliteBook 840 G1 BIOS Admin Password](https://forums.mydigitallife.net/threads/solved-hp-elitebook-840-g1-bios-admin-password.61719/), there is also an elegant Python application called [HP BIOS unlock tool](https://www.badcaps.net/forum/showthread.php?t=79956). I rather used the Python script. The application takes a single argument which should be the path to your 16MB BIOS image. After running it, the script should create an unlocked version of your original image. You can view the result in the hex editor below (unlocked vs. locked BIOS dump content).

{{< figure link="/img/bios_dump.png" src="/img/bios_dump.png">}}

##### Putting the main BIOS chip back
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Nothing much to add here. Clean the pads a bit and solder the chip back. Make sure you pay attention to markings on the motherboard in order to connect pin 1 correctly (or just take a picture of the chip's rotation before you desolder it).\

That's it, you have successfully unlocked your BIOS.

## Bitlocker
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;First thing I did after removing the BIOS password was to upgrade the BIOS version (you can't upgrade your BIOS if your BIOS is locked). I was swapping between two primary hard-drives and after I re-connected the bootable drive that was encrypted using Bitlocker I could not boot anymore without providing the Bitlocker backup key.\
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Either upgrading the BIOS or modifying the BIOS dump changed the fingerprint of my laptop so make sure you have the Bitlocker backup key ready in case your hard drive is encrypted using this technology.

## Equipment Used

* __Adapter__ - SOIC8 SOP8 to DIP8 Wide-body Seat Wide 150mil 200mil Programmer Adapter
* __Programmer__ - TL866A Tall Speed in-circuit Programmer USB
* __Soldering station__ - soldering iron + hot air gun
* [HP Unlocking tool](https://www.badcaps.net/forum/showthread.php?t=79956)


## Conclusion
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;It took me a while to realize the main BIOS chip will have to be taken off the motherboard. My main concern was this "surgical" procedure would render the laptop useless. But it turned out the 8 pin memory chip is quite big, easy to work with and located away from any critical components. And if there would be a problem with the modified BIOS dump, I could just copy back the original content I backed up before the modification.

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Two online resources I got a lot information from:\

* [Badcaps Forums - Salvation For Your Electronics!](https://www.badcaps.net/forum/)
* [My Digital Life Forums](https://forums.mydigitallife.net/)