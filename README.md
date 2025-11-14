# Framework Laptop 16 Expansion Bay Modules
![ExpansionBay](https://user-images.githubusercontent.com/28994301/226250626-f826d576-5e1f-4112-a077-6ac71b7851d8.png)

The Framework Laptop 16 has an Expansion Bay system that contains both the cooling fans
and an interface for high-power, high-speed PCIe peripherals like GPUs.  Expansion Bay
modules can extend in both thickness and depth to accomodate higher power requirements.

There are currently two Expansion Bay modules available:

 1. Expansion Bay Shell - This module has two cooling fans for the system as well as an open area 
 in the middle that can be used for PCIe peripherals like SSDs or even additional I/O that can
 be exposed on the back of the system.  This GitHub repository documents the mechanical 
 and electrical specifications for the cards that can go in the Shell.
 2. Graphics Module - This module is thicker and deeper, and contains a GPU card along with
 the heatsink for it.

## Custom mechanical parts

Note that one custom part is needed for development: the threaded insert that the interposer 
screws into.  Two pieces are needed on each Expansion Bay Module.  A drawing is in the PCB Reference Design
folder.  You can submit a request for free samples of these for development purposes at:
https://forms.gle/t4oS8qWWoaaXqVot9

## A developer's guide to Expansion Bay Integration - JJ Shapiro

Although Framework have done an incredible job at publishing as much original documentation as possible, it can still be difficult for new developers to grasp how to properly electrically integrate new modules - especially in the case of EEPROM and power delivery. This should hopefully clear some things up! 

Contents:

1. Power delivery
2. EEPROM
3. Writing to and from the EEPROM
4. Some useful PCIE info
5. Fan electrical
6. Prototyping tips and tricks

## 1. Power delivery

Power delivery is not nearly as complex as it might initially seem. Two pins on the interposer are pulled to ground by 330k ohm resistors, which tell the EC that you've installed the interposer correctly and it's safe to deliver power. It's easy to think of these like a continuity check. If the EC doesn't read 330k at all then it knows there's been a connection issue and it's not safe to deliver power. If it reads significantly higher there could be something between the pins of the connector and the pads, and hence it also won't provide power. 
The resistors should be connected to ID0 and ID1, pulled to ground, (see below)


<img width="400" height="638" alt="image" src="https://github.com/user-attachments/assets/7dd60b7b-1425-482d-9ef3-a6a5f176252d" /> <br />



<img width="400" height="294" alt="image" src="https://github.com/user-attachments/assets/059781be-da4a-4816-af34-51df3e76b992" />

The reference schematic provided suggests ±1% is a good range for the resistors, so this is what I'd recommend. And that's it for power delivery! The expansion bay should now be providing 3.3v, 5v and 20v respectively.

## 2. EEPROM 

The EEPROM is a bit more complex. It uses the BR24G32FVT-3GE2 - an xx24 EEPROM chip which is fairly standard. The two critical components to understand here are the EEPROM addressing and general power delivery. The BR24 has _adjustable_ addresses, which in Framework's case is set to the lowest possible value. What this means in practicality is that A0, A1 and A2 should be pulled to ground to set the correct address, (again see below):


<img width="400" height="146" alt="image" src="https://github.com/user-attachments/assets/7f80ad25-7a3d-43fb-bef9-09c48bc808c4" />

<br />

<img width="400" height="516" alt="image" src="https://github.com/user-attachments/assets/ae565d0b-cb88-40c8-a46b-431f7743dba9" />


This can be a confusing part of the reference schematic, as the Framework design has options to change the address, which is what I've shown above. I can't actually see any instance in which you'd want to change this though, so you should be able to get away with just tying all of the address pins to ground. 

Power to the EEPROM is as simple as hooking up VCC to 3.3v and GND to ground. I'd recommend a 0.1uF cap on the VCC line though, as there's potential for the 3.3v bus to be shared amongst other noisy devices. 


<img width="400" height="388" alt="image" src="https://github.com/user-attachments/assets/558a4022-7b54-4c01-95ea-037927265427" />


Be careful with the WP, (write protection), pin! Depending on whether it is pulled up or down, you won't be able to write anything to the EEPROM, which can be an issue for prototyping. Pulled _down_ switches protection _off_, while pulling it _up_, (to 3.3v), switches it on for production-ready board. 

<img width="400" height="802" alt="image" src="https://github.com/user-attachments/assets/a8a9d44f-5da5-45a9-9762-1c87bddec3ed" />

SDA and SCL are what you really need to actually use the EEPROM, and they work exactly how you'd expect any I2C device to. They should be connected SDA -> I2C DAT AlW on the interposer connection, and SCL - > I2C CLK ALW. It should be standard, but pull up both lines with 4.7k resistors.

<img width="400" height="150" alt="image" src="https://github.com/user-attachments/assets/6e46c6ba-48e0-4ae5-90cf-5949aa23f3fe" />

<br />

<img width="400" height="732" alt="image" src="https://github.com/user-attachments/assets/950c9e35-7bd0-42a6-9f84-54fb82774420" />

That covers it for the electrical integration of the EEPROM.

## 3. Writing to and from the EEPROM

Writing to and from the EEPROM is done solely with framework_tool, an application which should come pre-installed on every Framework Laptop. All the critical ones for expansion bay development are listed [here](https://github.com/FrameworkComputer/framework-system/blob/main/EXAMPLES_ADVANCED.md "@embed"), in the framework_tool advanced examples library. 
The critical ones for quick reference are here though:

### Dump current descriptor (e.g. for backup)
> framework_tool --dump-gpu-descriptor-file foo.bin -vvvv
Dumping to foo.bin
Wrote 153 bytes to foo.bin

### Update just the serial number
> framework_tool --flash-gpu-descriptor GPU FRAKMQCP41500ASSY1 -vvvv
> framework_tool --flash-gpu-descriptor 13 FRAKMQCP41500ASSY1 -vvvv
> framework_tool --flash-gpu-descriptor 0x0D FRAKMQCP41500ASSY1 -vvvv

### Update everything from a file
> framework_tool --flash-gpu-descriptor-file pcie_4x2.bin -vvvv

### Check the state of the expansion bay
> framework_tool --expansion-bay -vvvv

## 4. Some useful PCIE info

TODO

## 5. Fan electrical

The electrical design for the fans is very simple thanks to the fact that most of it is handled by the mainboard over the interposer. The fans are both 4 pin, 12v with a 0.5mm pin connector pitch that faces downwards when installed regularly. _The connector shown on the M.2 reference board is incorrect and should not be used!_ Any 0.5mm pitch 4 pin downwards facing FPC connector should work fine though. I've added 0.1uF caps on the 12v lines where they aren't normally found on the reference schematic, but these might not be necessary. 

<img width="400" height="1306" alt="image" src="https://github.com/user-attachments/assets/b69d28fe-e3da-4228-bf53-a02f7babeb5d" />

If you do choose to use an extra capacitor, make sure it's rated for the full 12v, or even higher to be safe. Connection to the interposer is simple, but you'll need an 100M ferrite bead filter on the 12v line:

<img width="400" height="118" alt="image" src="https://github.com/user-attachments/assets/30f981cb-aa06-4337-ac45-fb358c488f56" />

## 6. Prototyping tips and tricks

I've found that when the expansion bay door switch, (a small trigger which is compressed when the interposer door is closed), is left open the Framework refuses to let the interposer connect. To be safe, putting white-tack or capton tape to hold the swtich down may solve your issue. 

If you're trying to prototype a GPU style design and nothing is being displayed, plugging an HDMI cable into the mainboard itself and not the GPU will force output through the IGPU. 

Make sure to check for shorts between VCC and GND before booting on your board. It's much easier to fix bad solder joints before your board has exploded than afterwards. 

And that's it! Good luck to anyone trying this - make a forum post if you make anything interesting, and make sure to add your project to this [community wiki](https://community.frame.work/t/list-of-company-or-individually-driven-projects/18559 "@embed") for ongoing or successful ideas











## License
Expansion Bay © 2023 by Framework Computer Inc is licensed under CC BY 4.0. To view a copy of this license, visit http://creativecommons.org/licenses/by/4.0/
