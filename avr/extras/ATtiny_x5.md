# ATtiny 25/45/85
![x5 pin mapping](Pinout_x5.jpg "Arduino Pin Mapping for ATtiny x5-family")

Specification         |    ATtiny85    |      ATtiny85  |    ATtiny85    |     ATtiny45   |       ATtiny45 |      ATtiny25  |
----------------------|----------------|----------------|----------------|----------------|----------------|----------------|
Bootloader (if any)   |                |       Optiboot |  Micronucleus  |                |       Optiboot |                |
Uploading uses        |   ISP/SPI pins | Serial Adapter | USB (directly) |   ISP/SPI pins | Serial Adapter |   ISP/SPI pins |
Flash available user  |     8192 bytes |     7552 bytes |     6586 bytes |     4096 bytes |     3456 bytes |           2048 |
RAM                   |      512 bytes |      512 bytes |      512 bytes |      256 bytes |      256 bytes |            128 |
EEPROM                |      512 bytes |      512 bytes |      512 bytes |      256 bytes |      256 bytes |            128 |
GPIO Pins             |      5 + RESET |      5 + RESET |      5 + RESET |      5 + RESET |      5 + RESET |      5 + RESET |
ADC Channels          |   4 (incl RST) |   4 (incl RST) |   4 (incl RST) |   4 (incl RST) |   4 (incl RST) |   4 (incl RST) |
Differential ADC      |     1/20x gain |     1/20x gain |     1/20x gain |     1/20x gain |     1/20x gain |     1/20x gain |
PWM Channels          |  4: PA5-7, PB2 |  4: PA5-7, PB2 |  4: PA5-7, PB2 |  4: PA5-7, PB2 |  4: PA5-7, PB2 |  4: PA5-7, PB2 |
Interfaces            |            USI |            USI |      vUSB, USI |            USI |            USI |            USI |
Clocking Options:     |         in MHz |         in MHz |         in MHz |         in MHz |         in MHz |         in MHz |
Int. Oscillator or PLL| 16, 8, 4, 2, 1 | 16, 8, 4, 2, 1 |    16.5. 16, 8 | 16, 8, 4, 2, 1 | 16, 8, 4, 2, 1 | 16, 8, 4, 2, 1 |
Internal, with tuning |    16.5, 12, 8 |    16.5, 12, 8 |    16.5, 12, 8 |    16.5, 12, 8 |    16.5, 12, 8 |    16.5, 12, 8 |
External Crystal      |   All Standard |   All Standard |  Not supported |   All Standard |   All Standard |   All Standard |
External Clock        |   All Standard |   All Standard |  Not supported |   All Standard |   All Standard |   All Standard |
Int. WDT Oscillator   |        128 kHz |        128 kHz |        128 kHz |        128 kHz |        128 kHz |        128 kHz |
LED_BUILTIN           |        PIN_PB1 |        PIN_PB1 |        PIN_PB1 |        PIN_PB1 |        PIN_PB1 |        PIN_PB1 |

## Programming
Any of these parts can be programmed by use of any ISP programmer. 4k and 8k parts can be programmed over the software serial port using Optiboot, and 8k parts can be programmed via Micronucleus. Be sure to read the section of the main readme on the ISP programmers and IDE versions. 1.8.13 is recommended for best results.

### Optiboot Bootloader
This core includes an Optiboot bootloader for the ATtiny85/45, operating using software serial at at the standard ATTinyCore baud rates (which have changed in 2.0.0 for improved reliability see [the Optboot reference](./Ref_Optiboot.md)) - the software serial uses the AIN0 and AIN1 pins (see UART section below). The bootloader uses 640b of space, leaving 3456 or 7552b available for user code. In order to work on these parts, which do not have hardware bootloader support (hence no BOOTRST functionality), "Virtual Boot" is used. This works around this limitation by rewriting the vector table of the sketch as it's uploaded - the reset vector gets pointed at the start of the bootloader, while the EE_RDY vector gets pointed to the start of the application.

Due to a defect in Optiboot, it is possible for the bootloader to trash itself and the installed application; In this case ISP reprogramming is required to fix it. This means that **optiboot is not suitable for production systems on this part** - eventually, the bug will get triggered, and they will need to be rebootloaded; in a production setting this is simply not acceptable. I know an what must be done to fix this but getting from that to code which does that has proven extrordinarily difficult, I have attempted several times, each time reaching a point where I had no idea how to proceed to finish the fix. I know what value it needs to write to what address, but not now to get from where I was to there.

Optiboot is available for the 85 and 45. It would fill more than 1/4th of the flash on the 25 and make the device very difficult to do anything useful with, and so we do not provide false hope by offering support.

### Micronucleus VUSB Bootloader
This core includes a Micronucleus bootloader that supports the ATtiny85, allowing sketches to be uploaded directly over USB. The board definition runs at 16.5 MHz via the internal PLL, adjusting the clock speed up slightly to get 16.5 MHz, and leaves it that way when the sketch is launched unless a slower clock speed is selected. These lower clock speeds are not compatible with USB libraries. See the document on [Micronucleus usage](Ref_Micronucleus.md) for more information. D- is on pin 3, D+ is on pin 4. Note that for the most part, libraries that make an ATtiny85 work as a USB device don't work correctly.

In 2.0.0, all of the usual micronucleus entry methods are available. It is shockingly robust considering the hackjob it is built upon.

Note that VUSB is only supported for loading code. After much very disappointing discussion with relevant experts and background research I am forced to say that VUSB is not supported for emulating other USB peripherals, as the hardware does not provide a means to meet the timing constraints in the context of an arduino sketch. Some people have gotten limited functionality to work. This is the exception not the rule.

### LED_BUILTIN is on PB1
Both optiboot and micronucleus will try to blink it to indicate bootloader status.

## Features

### PLL Clock
The ATtiny x5-family parts have an on-chip PLL. This is clocked off the internal oscillator and nominally runs at 64 MHz when enabled. It is possible to clock the chip off 1/4th of the PLL clock speed, providing a 16MHz clock option without a crystal (this has the same accuracy problems as the internal oscillator driving it). Alternately, or in addition to using it to derive the system clock, Timer1 can be clocked off the PLL. See below. For use with USB libraries, a 16.5 MHz clock option is available; with the Micronucleus bootloader, a tuned value calculated from the USB clock is used, and this is the default clock option, otherwise, a heuristic is used to determine the tuned speed.

### Timer1 is a high speed timer
This means it can be clocked at 64 MHz from the on-chip PLL. In the past a menu option was provided to configure this. It never worked, and in any event is insufficient to do much of practical use with. It was eliminated for 2.0.0. Instead, see the [ATTinyCore library](../libraries/ATTinyCore/README.md)

#### By default, PB1 uses timer1
Since it is the more capable timer (it can be clocked at 64 MHz from an internal PLL and has every power of two as a prescling option. This can be overridden with the tools -> PB1 Timer menu.

### I2C Support
There is no hardware I2C peripheral. I2C functionality can be achieved with the hardware USI. This is handled transparently via the special version of the Wire library included with this core. **You must have external pullup resistors installed** in order for I2C functionality to work at all. We only support use of the builtin universal Wire.h library. If you try to use other libraries and encounter issues, please contact the author or maintainer of that library - there are too many of these poorly written libraries for us to provide technical support for.

### SPI Support
There is no hardware SPI peripheral. SPI functionality can be achieved with the hardware USI. This should be handled transparently via the SPI library. Take care to note that the USI does not have MISO/MOSI, it has DI/DO; when operating in master mode, DI is MISO, and DO is MOSI. When operating in slave mode, DI is MOSI and DO is MISO. The #defines for MISO and MOSI assume master mode (as this is much more common, and the only mode that the SPI library has ever supported). As with I2C, we only support SPI through the included universal SPI library, not through any other libraries that may exist, and can provide no support for third party SPI libraries.

### UART (Serial) Support
There is no hardware UART. If running off the internal oscillator, you may need to calibrate it to get the speed close enough to the correct speed for UART communication to work. The core incorporates a built-in software serial named Serial - this uses the analog comparator pins, in order to use the Analog Comparator's interrupt, so that it doesn't conflict with libraries and applications that require PCINTs.  TX is AIN0, RX is AIN1. Although it is named Serial, it is still a software implementation, so you cannot send or receive at the same time. The SoftwareSerial library may be used; if it is used at the same time as the built-in software Serial, only one of them can send *or* receive at a time (if you need to be able to use both at the same time, or send and receive at the same time, you must use a device with a hardware UART). While one should not attempt to particularly high baud rates out of the software serial port, [there is also a minimum baud rate as well](Ref_TinySoftSerial.md)

Though TX defaults to AIN0, it can be moved to any pin using Serial.setTxBit(n) where n is the number in the pin name using Pxn notation (in this case, also the arduino pin number) (2.0.0+ only - was broken in earlier versions)..

To disable the RX channel (to use only TX), select "TX only" from the Builtin SoftSerial tools menu. To disable the TX channel, simply don't print anything to it, and set it to the desired pinMode after Serial.begin()

### Tone Support
Tone() uses Timer1. If the high speed functionality of Timer1 has been enabled (see link above), tone() will not produce the expected frequencies, but rather ones 2 or 4 times higher. For best results, use pin 1 or 4 for tone - this will use Timer1's output compare unit to generate the tone, rather than generating an interrupt to toggle the pin. In this way, "tones" can be generated up into the MHz range.  If using SoftwareSerial or the builtin software serial "Serial", tone() will work on pin 1 or 4 while the software serial is active but not on any other pins. Tone will disable PWM on pins 1 and 4.

### Servo Support
Although the timers are quite different, and historically there have been issues with the Servo library, we include a builtin Servo library that supports the Tiny x5 series. As always, while a software serial port is receiving or transmitting, the servo signal will glitch (this includes the builtin software serial "Serial).

### Servo and Tone break PB4 (and possibly PB1 for PWM)
The servo library and the tone function require full control of timer1.  This has two unfortunate consequences:
* There is no PWM available on the Timer1 pins (PB4 and - by default - PB1) if either tone() is outputting a tone, or the servo library is used.
* The Servo library cannot be used at the same time as tone

I realize that this is sometimes painful, but you're using an 8-pin tinyAVR that's over a decade old! Really, what do you expect? (this is not an issue on modern tinyAVRs (the ones supported by megaTinyCore))

## ADC Features
The ATtiny85 has a differential ADC, unlike even some ATmega parts, but like many other ATtiny devices. Gain of 1x or 20x is available, and two differential pairs are available. The ADC supports both bipolar mode (-512 to 511) and unipolar mode (0-1023) when taking differential measurements; you can set this using `setADCBipolarMode(true or false)`. On many AVR devices with a differential ADC, only bipolar mode is available. All of the channels can have the positive and negative inputs swapped; they advise taking a measurement in bipolar mode, and then swapping the direction if needed and switching to unipolar mode to double the effective resoluition.

### ADC Reference options
The ATtiny85 has both the 2.56 and 1.1V reference options, a rarity among the classic tinyAVR parts. It even supports an external reference, not that you can usually spare the pins to use one.

| Reference Option   | Reference Voltage           | Uses AREF Pin        | Aliases/synonyms                         |
|--------------------|-----------------------------|----------------------|------------------------------------------|
| `DEFAULT`          | Vcc                         | No, pin available    |                                          |
| `EXTERNAL`         | Voltage applied to AREF pin | Yes, ext. voltage    |                                          |
| `INTERNAL1V1`      | Internal 1.1V reference     | No, pin available    | `INTERNAL`                               |
| `INTERNAL2V56`     | Internal 2.56V reference    | No, pin available    | `INTERNAL2V56_NO_CAP` `INTERNAL2V56NOBP` |
| `INTERNAL2V56_CAP` | Internal 2.56V reference    | Yes, w/cap. on AREF  |                                          |


### Internal Sources
| Voltage Source  | Description                            |
|-----------------|----------------------------------------|
| ADC_INTERNAL1V1 | Reads the INTERNAL1V1 reference        |
| ADC_GROUND      | Measures ground (offset cal?)          |
| ADC_TEMPERATURE | Internal temperature sensor            |

### Differential Channels
The ADC on the x5-series can act as a differential ADC with selectable 1x or 20x gain. It can operate in either unipolar or bipolar mode, and the polarity of the two differential pairs can be inverted in order to maximize the resolution available in unipolar mode (they envision users making a measurement in bipolar mode, reversing the input polarity if needed before taking a more accurate measurement in unipolar mode.) ATTinyCore wraps the IPR bit into the name of the differential ADC channel. Note the organization of the channels - there are two differential pairs A0/A1, and A2/A3 - but there is no way to take a differential measurement betwween A0 and A2 or A3 for example.

| Positive   | Negative   |Gain | Name            | Notes            |
|------------|------------|-----|-----------------|------------------|
| ADC0 (PB5) | ADC0 (PB5) | 1x  | DIFF_A0_A0_1X   | For offset cal.  |
| ADC0 (PB5) | ADC0 (PB5) | 20x | DIFF_A0_A0_20X  | For offset cal.  |
| ADC0 (PB5) | ADC1 (PB2) | 1x  | DIFF_A0_A1_1X   |                  |
| ADC0 (PB5) | ADC1 (PB2) | 20x | DIFF_A0_A1_20X  |                  |
| ADC1 (PB2) | ADC0 (PB5) | 1x  | DIFF_A1_A0_1X   | Input Reversed   |
| ADC1 (PB2) | ADC0 (PB5) | 20x | DIFF_A1_A0_20X  | Input Reversed   |
| ADC2 (PB4) | ADC2 (PB4) | 1x  | DIFF_A2_A2_1X   | For offset cal.  |
| ADC2 (PB4) | ADC2 (PB4) | 20x | DIFF_A2_A2_20X  | For offset cal.  |
| ADC2 (PB4) | ADC3 (PB3) | 1x  | DIFF_A2_A3_1X   |                  |
| ADC2 (PB4) | ADC3 (PB3) | 20x | DIFF_A2_A3_20X  |                  |
| ADC3 (PB3) | ADC2 (PB4) | 1x  | DIFF_A3_A2_1X   | Input Reversed   |
| ADC3 (PB3) | ADC2 (PB4) | 20x | DIFF_A3_A2_20X  | Input Reversed   |

### Temperature Measurement
To measure the temperature, select the 1.1v internal voltage reference, and analogRead(ADC_TEMPERATURE); This value changes by approximately 1 LSB per degree C. This requires calibration on a per-chip basis to translate to an actual temperature, as the offset is not tightly controlled - take the measurement at a known temperature (we recommend 25C - though it should be close to the nominal operating temperature, since the closer to the single point calibration temperature the measured temperature is, the more accurate that calibration will be without doing a more complicated two-point calibration (which would also give an approximate value for the slope)) and store it in EEPROM (make sure that `EESAVE` fuse is set first, otherwise it will be lost when new code is uploaded via ISP) if programming via ISP, or at the end of the flash if programming via a bootloader (same area where oscillator tuning values are stored). See the section below for the recommended locations for these.

### Tuning Constant Locations
These are the recommended locations to store tuning constants. In the case of OSCCAL, they are what are checked during startup when a tuned configuration is selected. They are not otherwiseused by the core.

ISP programming: Make sure to have EESAVE fuse set, stored in EEPROM

Optiboot used: Saved between end of bootloader and end of flash.

| Tuning Constant        | Location EEPROM | Location Flash |
|------------------------|-----------------|----------------|
| Temperature Offset     | E2END - 4       | FLASHEND - 7   |
| Temperature Slope      | E2END - 3       | FLASHEND - 6   |
| Unspecified            | N/A             | FLASHEND - 5   |
| Tuned OSCCAL 12 MHz    | E2END   2       | FLASHEND - 4   |
| Tuned OSCCAL 8.25 MHz  | E2END - 1       | FLASHEND - 3   |
| Tuned OSCCAL 8 MHz     | E2END           | FLASHEND - 2   |
| Bootloader Signature 1 | Not Used        | FLASHEND - 1   |
| Bootloader Signature 2 | Not Used        | FLASHEND       |

Mironucleus used: Micronucleus boards store a tuning value to the application section, but a separate sketch could also use a different means of calibration and store a value in the flash. The recommended locationsare shown below.

| Tuning Constant        | Location Flash         |
|------------------------|------------------------|
| Tuned OSCCAL 8.25 MHz  | BOOTLOADER_ADDRESS - 4 |
| Temperature Offset     | FLASHEND - 3           |
| Temperature Slope      | FLASHEND - 2           |
| Tuned OSCCAL 8.25 MHz  | FLASHEND - 1           |
| Tuned OSCCAL 8 MHz     | FLASHEND               |

## Purchasing ATtiny85 Boards
As the ATtiny85 is available in an easy-to-solder through-hole DIP package, a board can be easily made by just soldering the part into prototyping board.
I (Spence Konde) sell a specialized prototyping board that combines an ISP header with prototyping space and outlines to fit common SMD parts.
* [ATtiny85 prototyping board](https://www.tindie.com/products/drazzy/attiny85-project-board/)
* Micronucleus boards, and socketed boards for chips bootloaded with Micronucleus are readily available all over the internet, very cheaply, in several models. Search for things like "Digispark ATtiny85", "ATtiny85 USB" and so on.

## Interrupt Vectors
This table lists all of the interrupt vectors available on the ATtiny x5-family, as well as the name you refer to them as when using the `ISR()` macro. Be aware that a non-existent vector is just a "warning" not an "error" (for example, if you misspell a vector name) - however, when that interrupt is triggered, the device will (at best) immediately reset (and not clearly - I refer to this as a "dirty reset") The catastrophic nature of the failure often makes debugging challenging. Vector addresses are "word addressed". vect_num is the number you are shown in the event of a duplicate vector error, as well as the interrupt priority (lower number = higher priority), if, for example, several interrupt flags are set while interrupts are disabled, the lowest numbered one would run first.

|vect_num | Address| Vector Name        | Interrupt Definition                |
|---------|--------|--------------------|-------------------------------------|
|       0 | 0x0000 | RESET_vect         | Not an interrupt - this is a jump to the start of your code.  |
|       1 | 0x0001 | INT0_vect          | External Interrupt Request 0        |
|       2 | 0x0002 | PCINT0_vect        | Pin Change Interrupt 0              |
|       3 | 0x0003 | TIMER1_COMPA_vect  | Timer/Counter1 Compare Match A      |
|       4 | 0x0004 | TIMER1_OVF_vect    | Timer/Counter1 Overflow             |
|       5 | 0x0005 | TIMER0_OVF_vect    | Timer/Counter0 Overflow             |
|       6 | 0x0006 | EE_RDY_vect        | EEPROM Ready                        |
|       7 | 0x0007 | ANA_COMP_vect      | Analog Comparator                   |
|       8 | 0x0008 | ADC_vect           | ADC Conversion Complete             |
|       9 | 0x0009 | TIMER1_COMPB_vect  | Timer/Counter1 Compare Match B      |
|      10 | 0x000A | TIMER0_COMPA_vect  | Timer/Counter0 Compare Match A      |
|      11 | 0x000B | TIMER0_COMPB_vect  | Timer/Counter0 Compare Match B      |
|      12 | 0x000C | WDT_vect           | Watchdog Time-out (Interrupt Mode)  |
|      13 | 0x000D | USI_START_vect     | USI START                           |
|      14 | 0x000E | USI_OVF_vect       | USI Overflow                        |
