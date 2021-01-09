title: Platform IO Command line
date: 2019-01-08T17:13:17Z
tags: platform IO,coding, ARM, notes
Summary: >
	Notes about Installing, Platform IO Command line interface on Arch Linux.


As part of my mucking around with ARM based compilers, I have also
given (https://platformio.org)[Platform IO] a crack.  First
impressions are it is much nicer to use than the MBED command line
equivalent.


# Installation

NOTE:  You will need the relevant ARM tool chains. 

## Platform IO setup

Requires Python 2, as always I like to make use of a virtualenv to save
my global python ending up full of cruft.

~~~
dang@dang-laptop% virtualenv -p /usr/bin/python2 env
~~~

Install

~~~
(env)  dang@dang-laptop%  pip install platformio
~~~

Check it works

~~~
(env)  dang@dang-laptop% platformio
Usage: platformio [OPTIONS] COMMAND [ARGS]...

Options:
  --version          Show the version and exit.
  -f, --force        Force to accept any confirmation prompts.
  -c, --caller TEXT  Caller ID (service).
  -h, --help         Show this message and exit.

Commands:
  account   Manage PIO Account
  boards    Embedded Board Explorer
  ci        Continuous Integration
  debug     PIO Unified Debugger
  device    Monitor device or list existing
  home      PIO Home
  init      Initialize PlatformIO project or update existing
  lib       Library Manager
  platform  Platform Manager
  remote    PIO Remote
  run       Process project environments
  settings  Manage PlatformIO settings
  test      Local Unit Testing
  update    Update installed platforms, packages and libraries
  upgrade   Upgrade PlatformIO to the latest version
(env)  dang@dang-laptop  ~/Documents/India/PlatformIO  
~~~

## Getting More of the Toolchain working

Install the Udev Rules file

https://docs.platformio.org/en/latest/faq.html#platformio-udev-rules

> NOTE:  As per usual they suggest directly installing from CURL.  As usual this is a Bad Idea.
> Take a look at the file in Nano to avoid any surprises.

Download and check the file
~~~
dang@dang-laptop /tmp % curl -fsSL https://raw.githubusercontent.com/platformio/platformio-core/develop/scripts/99-platformio-udev.rules > 99-platformio-udev.rules
dang@dang-laptop /tmp % nano 99-platformio-udev.rules 
~~~

Install and update UDEV

~~~
dang@dang-laptop /tmp % sudo cp 99-platformio-udev.rules /etc/udev/rules.d/ 
[sudo] password for dang: 
2 dang@dang-laptop /tmp % sudo udevadm control --reload-rules                                                       dang@dang-laptop /tmp % sudo udevadm trigger
~~~


# Try one of the Demo Projects

Clone the STM toolchain with examples
https://github.com/platformio/platform-ststm32/tree/master

As I am interested in using the mbed OS, the blink example (from the Online version)
looks like a good bet.

```platform-ststm32/examples/mbed-blink```


## Building the Example

Unfortunately, when we try to run the example, 

A Bit afraid of adding this to the Board  Lets try something a bit different
According to the MBED docs the device I have is supported, but not in the build list

~~~
▶ platformio run -e nucleo_f746zg
Error: Unknown environment names 'nucleo_f746zg'. Valid names are 'nucleo_f103rb, disco_l053c8, nucleo_l053r8, nucleo_f767zi, disco_f100rb, mts_dragonfly_f411re, cloud_jam, nucleo_f207zg, cloud jam l4, disco_f303vc, nucleo_f030r8, nucleo_f401re, disco_f407vg, mote_l152rc, elmo_f411re, nucleo_l152re, nucleo_f302r8'
(env) 
~~~

However according to the Docs it is supported, so lets add a new target platform to the project.
Taking the definition from an example that has support for the Nucleo F746 (cube32_blink) 

In ```platformio.ini``` we add

~~~
[env:nucleo_f746zg]
platform = ststm32
framework = mbed
board = nucleo_f746zg
build_flags = -DF7
~~~

This time building appears to work

~~~
 platformio run -e nucleo_f746zg

Processing nucleo_f746zg (platform: ststm32; board: nucleo_f746zg; framework: mbed)
---------------------------------------------------------------------------------------------------------------------
Verbose mode can be enabled via `-v, --verbose` option
CONFIGURATION: https://docs.platformio.org/page/boards/ststm32/nucleo_f746zg.html
PLATFORM: ST STM32 > ST Nucleo F746ZG
HARDWARE: STM32F746ZGT6 216MHz 320KB RAM (1MB Flash)
...
<SNIP>
...
===================================================== [SUMMARY] =====================================================
Environment disco_f407vg                [SKIP]
Environment disco_f303vc                [SKIP]
Environment disco_f100rb                [SKIP]
Environment nucleo_f030r8               [SKIP]
Environment nucleo_f103rb               [SKIP]
Environment nucleo_f207zg               [SKIP]
Environment nucleo_f302r8               [SKIP]
Environment nucleo_f401re               [SKIP]
Environment nucleo_f767zi               [SKIP]
Environment nucleo_l053r8               [SKIP]
Environment nucleo_l152re               [SKIP]
Environment disco_l053c8                [SKIP]
Environment elmo_f411re                 [SKIP]
Environment mote_l152rc                 [SKIP]
Environment mts_dragonfly_f411re        [SKIP]
Environment cloud_jam                   [SKIP]
Environment cloud jam l4                [SKIP]
Environment nucleo_f746zg               [SUCCESS]
============================================ [SUCCESS] Took 7.50 seconds ===
~~~

TO Upload to the board using we need to specify the location to upload

~~~
▶ udiskie-mount /dev/sda 
mounted /org/freedesktop/UDisks2/block_devices/sda on /run/media/dang/NODE_F746ZG
~~~

And It looks like everything works as Required
