title: Getting Started with Arm MBED Online compiler
date: 2019-01-07T11:51:41Z
tags: mbed, coding, ARM, notes
Summary: >
	Building and uploading a demo created the ARM mbed online compiler


The ARM Mbed online compiler seems to be a nice balance in getting
code written for embedded devices.  It avoids having to install a
tool chain (which can be tricksy) and gives a short lead time on
"Getting Something Working(TM)" for student engagement.

# First Steps

You will need to create an account at https://www.mbed.com/en/

Our first stage is to "Register" the target board with the online
compiler (and like many of these things it could not be that much
easier).

I am using the *STM32 Nucelo-F746ZG*, which is MBED compatible, 
to register the board with the online compiler simply 

 1. Plug it in over USB
 1. Navigate to the new Drive that is created
 1. Open the MBED.htm in whatever web browser takes your fancy.
 
# Getting Some Demo Code Running.

Lets get some demo code up and running.

 1. Click Import, and search for the *Mbed OS example Blinky* program

We should get some code that looks like this

~~~ c
/* mbed Microcontroller Library
 * Copyright (c) 2018 ARM Limited
 * SPDX-License-Identifier: Apache-2.0
 */

#include "mbed.h"
#include "stats_report.h"

DigitalOut led1(LED1);

// main() runs in its own thread in the OS
int main()
{
    SystemReport sys_state(500 /* Loop delay time in ms */);

    while (true) {
        // Blink LED and wait 0.5 seconds
        led1 = !led1;
        wait_ms(500);

        // Following the main thread wait, report on the current system status
        sys_state.report_state();
    }
}

~~~

 1. Cick the *Compile* button
 1. Copy the code to the USB drive on the device
 
~~~
 dang@dang-laptop /run/media/dang/NODE_F746ZG % mv ~/Downloads/mbed-os-example-blinky.NUCLEO_F746ZG .
~~~
 
The Device should update and blink one of the user LEDS.


# Testing with a quick modification.

Lets makes sure things work properly by modifying the code to blink all the debugging LEDS

~~~
/* mbed Microcontroller Library
 * Copyright (c) 2018 ARM Limited
 * SPDX-License-Identifier: Apache-2.0
 */

#include "mbed.h"
#include "stats_report.h"

DigitalOut led1(LED1);
DigitalOut led2(LED2);
DigitalOut led3(LED3);
int count=0;

// main() runs in its own thread in the OS
int main()
{
    SystemReport sys_state(500 /* Loop delay time in ms */);
    printf("Hello World");
    while (true) {
        // Blink LED and wait 0.5 seconds
        led1 = !led1;
        wait(0.5f);  
        led2 = !led2;
        wait(0.5f);
        led3 = !led3;
        wait(0.5f);

        // Following the main thread wait, report on the current system status
        sys_state.report_state();
    }
}
~~~

After uploading we get all 3 leds flashing in a sequence.

