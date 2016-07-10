---
layout: default
title: SC Joystick Configuration
tagline: DRAFT
description: >-
  This page is a tutorial I wrote to document the development of my inputs to
  Star Citizen.
projpath: SC-Joystick-Configuration
version: 0.0.1 Beta
author: danricho
permalink: /index.html
published: true
---

# Introduction 

When I started playing **Star Citizen [\[1\]](#reference-list)** (SC) I found the key combinations required during "flight" overwhelming. I saw this problem as something that could be overcome by designing a physical "control panel" which would have the most important functions mapped to a dedicated button.

*Note: I only outline setting up the SC "flight" functionality as I primarily use keyboard and mouse for other parts of the game such as First-Person Shooting (FPS).*

# Development Log

## Joysticks

The first thing I decided was that I wanted my flight control to be a **HOSAS** (Hands On Stick And Stick) rather than the perhaps more common HOTAS (Hand On Throttle And Stick) setup. 
My rationale is that the game is primarily space based and the additional axes that the dual sticks afford, provide fine control of all three "strafing" dimensions as well as all three rotational "Pitch, Roll and Yaw" dimensions.

Having decided on the HOSAS setup, the next question was which joysticks I wanted to use. I decided upon two **Thrustmaster T.16000m** joysticks. The factors that decided this choice were:

  - they are very affordable,
  - they are quite easily sourced, 
  - they can be configured as left or right handed, and
  - they are very accurate.
  
![My Dual T.16000m Layout][IMG-STICK_LAYOUT]

## Control Panel

The first step in designing the control panel was to determine how many buttons I would want to implement. I settled at around 60 after deciding which SC functions would be mapped to the panel itself, the buttons on the sticks, neither or both.

![My Control Panel Layout][IMG-PANEL_LAYOUT]

With this decided, I was able to choose the type of buttons and the enclosure I wanted.

I chose the Arduino as the micro-controller platform within the panel as I've used them previously, they are simple, and they work out of the box. 

The specific Arduino I chose was an "eBay knockoff" of the **5V/16MHz Pro Micro [\[4\]](#reference-list)**. This is based on the **Leonardo** range of Arduinos which allow Human Interface Device (HID) emulation as the micro-controller itself (ATmega32u4) has built-in USB communication, eliminating the need for a secondary processor. 

The only limitation with Arduinos for a project like this is that they provide limited general purpose Input/Output (GPIO or I/O) pins.
The workaround I chose for this was to use external devices to add more, as this reduced the complexity of the Arduino software and the hardware required to support the switches.

The Microchip **MCP23017 IC [\[2\]](#reference-list)** is a 16-bit I/O expander with an I<sup>2</sup>C serial control interface. The reasons I chose these chips are as follows:

  - Multiple chips can be used on the serial bus and each adds 16 additional I/O pins.
  - The 16 I/O pins include pull-up resistors meaning that this doesn't need to be implemented in hardware.
  - Adafruit's MCP23017 Library makes interfacing with these pins as easy as it is with the built-in Arduino pins.

![Control Panel Driver Board][IMG-PANEL_DRV_BOARD]

I have used four MCP23017 ICs giving me 64 additional I/O pins, sacrificing only 2 of the Arduino GPIO pins (I<sup>2</sup>C serial communication pins).

With sufficient I/O pins available, I could assemble amd wire it all up.

![Control Panel Wiring][IMG-PANEL_WIRING]

### Firmware

Here is a bare-basic example of how to use the **MCP23017 Library [\[3\]](#reference-list)**:

~~~c++
#include <Wire.h>
#include "Adafruit_MCP23017.h"

// Create a MCP23017 object
Adafruit_MCP23017 myMCP;

void setup() {
  // Setup the MCP with address 0
  myMCP.begin(0);
  // Sets pin 0 of the MCP 
  // to be an input with pull-up resistor
  myMCP.pinMode(0, INPUT); 
  myMCP.pullUp(0, HIGH);
}

void loop() {
  // Reads the current state of pin 0 on the MCP
  bool currentInputState = myMCP.digitalRead(0)
  // Do something with it...
}
~~~

Initially, the Arduino HID emulation was limited to Keyboard and Mouse functionality but MHeironimus has developed the HID Joystick Library.

At the time I am writing this, the release version of the Joystick Library is v1.0.1 which allows multiple or single joysticks with a maximum or 32 buttons to be emulated by the Arduino. I wanted to emulate only one joystick but I wanted at least 60 buttons. Luckily, the library has a (currently Beta) version 2.0 which allows the number of buttons to be specified. 

*Note: Arduino IDE 1.6.6 (or above) is required for this library.*

Here is a bare-basic example of how to use the **HID Joystick Library [\[5\]](#reference-list)**:

~~~c++
#include <Joystick.h>

// Create a Joystick object
Joystick_ myJoystick;

void setup() {
  // Setup the Joystick
  myJoystick.begin();
}

void loop() {
  // State of the button (read something)...
  bool buttonState = true;
  // Report the first (0) button state to buttonState
  myJoystick.setButton(0, buttonState);
}
~~~

My Arduino firmware is available [here][LINK-REPO-ARDFW]. When loaded, the Arduino will be listed as a standard USB Game Controller in Windows (run command: `joy.cpl`). The standard utility will show only the first 32 buttons. To see more, I had success with **Pointy's Joystick Test [\[6\]](#reference-list)**. 

## Controller Fusion

**This section is not complete.**

At this point three controllers were visible to the computer; two T.16000m joysticks and the control panel. The total number of inputs are as follows:

| CONTROLLER | INPUTS |
| :---: | :---: |
|Left T.16000m|Axis 1-3|
|Left T.16000m|Slider|
|Left T.16000m|Hat|
|Left T.16000m|Buttons 1-4 (on stick)|
|Left T.16000m|Buttons 5-16 (on base)|
|Right T.16000m|Axis 1-3|
|Right T.16000m|Slider|
|Right T.16000m|Hat|
|Right T.16000m|Buttons 1-4 (on stick)|
|Right T.16000m|Buttons 5-16 (on base)|
|Arduino Panel|Buttons 1-64|

There were multiple limitations to simply mapping these three controllers directly:

  1. A maximum of 50 buttons are allowed per joystick in SC which means that not all buttons on the control panel could be used.
  2. Only one button can be mapped to each function.
  3. Any custom button functions must be implemented on the Arduino, requiring all actions be mapped to an Arduino "joystick button".

To remove these limitations, multiple options exist. The solution I chose is one I discovered on the SC forums. WhiteMagic's **Joystick Gremlin [\[7\]](#reference-list)** was developed to utilize "virtual joysticks" which can be linked to the physical joysticks in many ways.

### vJoy

Once configured, vJoy is a set and forget tool. It creates the virtual joysticks and unless troubleshooting or changing the vJoy parameters, there is little evidence that it is even installed. I set it up as follows:

  1. Install **vJoy [\[8\]](#reference-list)**.
  2. Launch to configuration interface and create 2 virtual joysticks (1 tab each).
  3. For each joystick, allocate:
     - 3 axis (X, Y, Z)
     - 3 rotations (Rx, Ry, Rz)
     - Slider
     - Hat
     - 50 buttons (maximum allowed in SC)  
  4. Activate vJoy.
  
The virtual joysticks are now be shown in the list of USB game controllers.

### Joystick Gremlin

*Note: The configurations and macro files in the repo are created for Joystick Gremlin v5 which is pre-release at the time of writing. Thanks for all your help WM!*

**TO BE WRITTEN - mention the direct remaps, response curve, cutom modules, etc.**

  1. Install **Joystick Gremlin**.
  2. Assign the buttons of each physical controller (tabs) to virtual actions (vjoy buttons, macros, custom python modules).
    - **examples etc will go here**
  3. Activate Joystick Gremlin.

*My Joystick Gremlin configuration (.xml) and and custom modules (.py) are available [here][LINK-REPO-JGCONF].*

These files should be placed in the  
`%userprofile%\Joystick Gremlin\` directory.  
*Note: Custom modules (.py) should not be in a sub-directory.*

### SC Joystick Mapper

To map the virtual joysticks (vJoy 1 & 2) to SC keybindings, **SC Joystick Mapper [\[9\]](#reference-list)** can be helpful. It creates an XML file which can be imported into the game.

*My SC keybinding (.xml) is available [here][LINK-REPO-SCXML].* 

This file should be placed in the 
`StarCitizen\CitizenClient\USER\Controls\Mappings\` directory.  
The keymapping is loaded by navigating in the menu to: Options, Keybinding, Advanced Controls Customization.
Under Control Profiles, select the keymapping and chose the controllers to load.

# Reference List

[1]  [Star Citizen][LINK-EXT-1] by *Cloud Imperium Games*  
[2]  [MCP23017 datasheet][LINK-EXT-2] by *Microchip*  
[3]  [MCP23017 Arduino Library][LINK-EXT-3] by *Adafruit*  
[4]  [5V/16MHz Pro Micro product page][LINK-EXT-4] by *Sparkfun*  
[5]  [HID Joystick Arduino Library][LINK-EXT-5] by *MHeironimus*  
[6]  [Pointy's Joystick Test][LINK-EXT-6] by *Pointy*  
[7]  [Joystick Gremlin][LINK-EXT-7] by *WhiteMagic*  
[8]  [vJoy][LINK-EXT-8] by *Shaul Eizikovich*  
[9]  [SC Joystick Mapper][LINK-EXT-9] by *SCToolsfactory*  




[comment]: # (==========================================================)
[comment]: # (REFERENCED LINKS AND IMAGES)
[comment]: # (==========================================================)


[IMG-STICK_LAYOUT]: images/sticks_layout.jpg
[IMG-PANEL_LAYOUT]: images/panel_layout.jpg
[IMG-PANEL_DRV_BOARD]: images/panel_driver_board.jpg
[IMG-PANEL_WIRING]: images/panel_wiring.jpg


[LINK-REPO-ARDFW]: https://github.com/danricho/SC-Joystick-Configuration/tree/master/ArduinoFirmware
[LINK-REPO-JGCONF]: https://github.com/danricho/SC-Joystick-Configuration/tree/master/Joystick%20Gremlin
[LINK-REPO-SCXML]: https://github.com/danricho/SC-Joystick-Configuration/blob/master/dual_t16000m_leonardo_SCmap.xml


[LINK-EXT-1]: https://robertsspaceindustries.com/
[LINK-EXT-2]: http://ww1.microchip.com/downloads/en/DeviceDoc/21952b.pdf
[LINK-EXT-3]: https://github.com/adafruit/Adafruit-MCP23017-Arduino-Library
[LINK-EXT-4]: https://www.sparkfun.com/products/12640
[LINK-EXT-5]: https://github.com/MHeironimus/ArduinoJoystickLibrary
[LINK-EXT-6]: http://www.planetpointy.co.uk/joystick-test-application/
[LINK-EXT-7]: http://whitemagic.github.io/JoystickGremlin/
[LINK-EXT-8]: http://vjoystick.sourceforge.net/
[LINK-EXT-9]: https://github.com/SCToolsfactory/SCJMapper-V2

