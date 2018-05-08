# CHiP Hacking
## Overview
This repository is where I will record any progress I make as I hack around with my [WowWee CHiP Robot Dog](https://wowwee.com/chip).

<img src="https://raw.githubusercontent.com/adamgreen/CHiP/master/images/20180507-02.jpg" alt="WowWee CHiP w/ Real Dog" width="320" height="240" /> <img src="https://raw.githubusercontent.com/adamgreen/CHiP/master/images/20180507-01.jpg" alt="WowWee CHiP" width="320" height="240" />

## Interesting Links
[WowWee's Official CHiP Page](https://wowwee.com/chip)<br>
[WowWee CHiP iOS SDK](https://github.com/WowWeeLabs/CHIP-iOS-SDK#wowwee-chip-ios-sdk)<br>
[WowWee CHiP Android SDK](https://github.com/WowWeeLabs/CHIP-Android-SDK#wowwee-chip-android-sdk)<br>
[WowWee CHiP Robot Toy Dog Teardown - fictiv Blog](https://www.fictiv.com/blog/posts/wowwee-chip-robot-toy-dog-teardown)<br>
[Photos of CHiP Internals from FCC ID Database](https://fccid.io/OKP0805A/Internal-Photos/Internal-Photos-3123283)<br>
[My SmartBand Repair](#chip-smartband-repair)<br>

## CHiP SmartBand Repair
When I first played around with my new CHiP robot, I noticed that the SmartBand wouldn't turn on. I plugged in the USB charging cable and the SmartBand's LED blinked with an inconsistent random blinking pattern, almost like there was a loose connection. I left it on the charger for a few hours, it still continued to blink in that same random pattern, and it would never activate.

If this had been a toy purchase for a child, I would have been pretty disappointed and tried to contact WowWee about getting a warranty replacement or returned it to the store where I bought it. Since I had every intention to take it apart at some point, voiding the warranty, I decided to take the SmartBand apart to see what I could see. Getting into the SmartBand was easy as it was only held together with 4 small Phillips head screws, accessible from the back of the band.

<img src="https://raw.githubusercontent.com/adamgreen/CHiP/master/images/20180507-03.jpg" alt="Back of SmartBand" width="320" height="240" />

There are a few pieces of plastic that make up the SmartBand and they might be hard to get back together in the correct orientation except that WowWee added assymetric keying to make it a cinch.

<img src="https://raw.githubusercontent.com/adamgreen/CHiP/master/images/20180507-04.jpg" alt="SmartBand Apart" width="320" height="240" /> <img src="https://raw.githubusercontent.com/adamgreen/CHiP/master/images/20180507-05.jpg" alt="Top of SmartBand PCB" width="320" height="240" />

I was pretty suspicious that the Lithium Ion battery was defective and wouldn't take a charge. Once I had the SmartBand apart, I could easily access the pads to which the battery attached to the PCB. Putting the voltmeter on those pads showed less than 0.5V from the battery. That is definitely too low for a Lithium Ion battery and it was probably destroyed by being allowed to discharge too much. 

<img src="https://raw.githubusercontent.com/adamgreen/CHiP/master/images/20180507-06.jpg" alt="Bottom of SmartBand PCB" width="320" height="240" />

I found that Adafruit sold a [3.7v 150mAh Lithium Ion Polymer Battery](https://www.adafruit.com/product/1317) that was comparable in size and capacity (the original was 130mAh). I added one of these batteries to my next Adafruit order and used it to replace the bad one which originally shipped with the CHiP. **NOTE:** If you are going to replace the battery yourself, you need to take caution and not short the batteries (original or replacement) as you are undertaking the repair.
* I cut each of the original battery's wires (black and red) close to the pads. I cut the wires one at a time and not both together so as not to short the battery across the surface of the cutters.
* If you look closely at the pads on the PCB, you should be able to see **+** and **-** symbols designating the polarity of the pads. If you can't see them, you should add markings to the PCB yourself to indicate which pad is to receive the red wire and which the black.
* I used the soldering iron to remove what was left of the battery wires from the PCB pads.
* The wires on the new battery from Adafruit include a connector that I didn't need and the wires are a bit longer than required for the SmartBand. I cut each of the new battery's wires to a length comparable to that of the wires on the original battery. Again, I cut each wire separately so as not to short the new battery across the cutting surface.
* I then stripped a few mm of insulation from the red, positive, wire of the new battery and tinned it in preparation for soldering to the **+** pad of the PCB. I added some flux to the **+** PCB pad and then soldered the red wire to it.
* I then did the same with the black, ground, wire and the **-** PCB pad. Be very careful to not short the ground wire to the positive wire or pad while you are soldering it down.
* I placed a small piece of Kapton insulating tape on top of the soldered pads to make sure that it wasn't easy to short. You could use black electrical tape instead.
* I removed the small foam pad from the old battery and attached it in roughly the same location on the new battery. It makes sure that the battery fits snugly in the SmartBand and the battery itself doesn't rest right up against the PCB.

<img src="https://raw.githubusercontent.com/adamgreen/CHiP/master/images/20180507-07.jpg" alt="New battery soldered onto PCB" width="320" height="240" />

Once I replaced the battery, the SmartBand started working as expected. Problem fixed!
