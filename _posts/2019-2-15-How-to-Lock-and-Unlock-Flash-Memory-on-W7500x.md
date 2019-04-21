---
layout: post
title: How to Lock and Unlock Flash Memory on W7500x
date:   2019-02-15 
author: James Kim
categories: W7500x
tags:	jekyll welcome
cover:  "/assets/images/ispcmd_init.PNG"
---


The Cortex M0 base W7500(and W7500P) from WIZnet has 128Kbytes Code Flash and 512 Bytes Data Flash inside.
Locking read/write operation to Flash memory is an inevitable capability of a MCU, in order to protect his own application code from illegal copy or to avoid mistakes overwriting the original code by accident.
With this post, I will introduce the way how to lock or unlock the flash memories on W7500(P).

## W7500isp python module ##
W7500(P) is limited to lock and/or unlock flash memories only by using ISP mode. In order words, locking and/or unlocking flash memory using JTAG is not allowed.
There is a python module to support various commands to access W7500(P) in ISP mode and it is a CLI(Command Line Interface) type.

Below is the github link for the python module.

[https://github.com/javakys/W7500isp](https://github.com/javakys/W7500isp)

All you do is to download it and just call and use W7500isp.ispcmd object.

## How to use W7500isp module ##
W7500isp module communicates with W7500(P) via COM port on your PC. So you need to install pySerial to use COM port on python interpreter. If it is installed, then you should import serial module in python script or on python interpreter

![_config.yml](/assets/images/capture1.PNG)


Run python interpreter after installation of pySerial

![_config.yml](/assets/images/python_interpreter.PNG)

Execute the preceding processes like below sequence.
* import W7500isp.ispcmd module
* import pySerial.serial module
* Instantiate an object of COM port connected to W7500(P)
* Instantiate an ispcmd object with just instanced COM port object as a parameter 
* run checkisp() member function of ispcmd to check whether W7500(P) is already running ISP modue.

Below is the example of above processes

![_config.yml](/assets/images/ispcmd_init.PNG)


If you get a reply message 'Boot Mode Entered' when you call checkisp() function, then it means that W7500(P) is on ISP mode. Since then, you can call any function of ispcmd module.

## Locking Read Protection ##


The function which shows the current state of read/write lock of Flash memory is readLockFlag(). it returns like below when it called.
![_config.yml](/assets/images/readLockFlag.PNG)

The first response shows the current value of FLOCKR0 and FLOCKR1 in hexadecimal format. Each register is 4 Bytes width so that each digit is 4bits HEX value. All 0's means that read/write lock on all flash pages is unlock. For more detail, please refer to READ.md on above github repository.
When you try to read the value of data in the specific flash area on unlocked state, you can read the value clearly.
![_config.yml](/assets/images/dumpDataFlash.PNG)


Now, let's check what the locked flash will return.
I'll lock the read operation to Data Flash area. Read Lock bits for Page 0 and Page 1 of Data Flash are FLOCKR0 bit2 and bit3. If these two bits are set to '1' then read operation to Data Flash is locked.
![_config.yml](/assets/images/dataFlashReadlock.PNG)

To lock or unlock read and/or write operation to a specific flash area, progLockFlag() function is used.
First parameter is for FLOCKR0 and 'C' set bit2 and bit3 on FLOCKR0 to '1'.
Now read Lock to Data Flash is set and I'll try to read to see What W7500(P) will reply with.
![_config.yml](/assets/images/dumpLockedDataFlash.PNG)


As you can see, reading a flash area with read-locked can just get all 0's reply because that flash area is locked for read operation.
So you can see that your own code will not be disclosed if you set its read-lock.

## Unlocking the locked flash and its affection ##

What happens if you unlock the already locked flash area? Unlocking the locked flash area makes W7500(P) erase the whole content on that area promptly. So even though someone unlocks the locked flash area to take your code illegally, he can't get anything.
![_config.yml](/assets/images/dumpUnlockedDataFlash.PNG)

As the above example, if you call progLockFlag() with all 0's parameters to unlock the locked Data Flash, then all data on Data Flash will be erased automatically.
If you call dumpDataFlash() to read data, you can see all FF's reply.
This proves that unlocking the flash memory erases the content.

## Conclusion ##

By now, I introduced that how to lock and unlock the Flash memory from read/write operations, how to protect your own code with these capabilities. I hope W7500isp module is helpful for users using W7500(W7500P) 