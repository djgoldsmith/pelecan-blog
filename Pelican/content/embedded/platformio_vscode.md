title: "Platformio_vscode"
date: 2019-01-11T16:39:55Z
tags: ["platform IO","coding", "ARM", "notes"]
Summary: >
	Installing platformIO in visual studio code

# Install VS Code

~~~
dang@dang-laptop ~$ sudo pacman -S code
~~~

And Start the **code** program

In the Editor:

 - Select **Extensions**  (Box looking thing on the RHS)
 - Platformio-ide
 - Install
 
Click Reload wait till installed and click reload again


> It appears that platformIO installs in a virtualenv somewhere, Very nice


# Arduino Project

Should be quick and easy to do.

New Project -> Select UNO Board -> Rest is default

Navigate to the Code folder **src/main.cpp**

## Teh Codez

Edit the code to be something like

~~~
#include <Arduino.h>

int LED=13;


void setup() {
  // put your setup code here, to run once:
  Serial.begin(9600);
  pinMode(LED, OUTPUT);
  }

void loop() {
  // put your main code here, to run repeatedly:
  
  Serial.print("Tick....");
  digitalWrite(PIN, HIGH);
  delay(500);

  Serial.println("Tock");
  digitalWrite(PIN, LOW);
  delay(500);
  }
~~~

## Compiling and Uploading

 - Use the Build Button on the toolbar at the bottom of the page
 - Use the Upload Button on the toolbar

# MBED Project

Create a new project as before but select the relevant Target Board

Code is 

~~~
#include <mbed.h>

DigitalOut myled(LED1);

int main() {

  // put your setup code here, to run once:

  while(1) {
    // put your main code here, to run repeatedly:
    myled = 1;
    wait(1);
    myled = 0;
    wait(1);
  }
}
~~~

To Upload we need to mount the device again but then it works without a flaw

## Multiple Targets
 
I also have an old skool NXP LPC

https://docs.platformio.org/en/latest/boards/nxplpc/lpc1768.html

~~~
[env:lpc1768]
platform = nxplpc
board = lpc1768
~~~

Upload and it works (after a reset)

One Issue is it builds and tries to upload both tasks
We can fix this by selecting the task required, using the *run task* (tickbox) option on the menu.

# Trying moving things from the Sim

In the MBED simulator they have a nice LCD demo

http://ec2-52-211-146-247.eu-west-1.compute.amazonaws.com:7829/#lcd

Copy and paste the code into vscode

## Install external library

To get the LCD working we need an external Library

Click Platform IO Home, and the Install Libraries

Search for *C12832*

Install it

## Fix the Lookup for the SPI 

~~~
C12832 lcd(p5, p7, p6, p8, p11);
//C12832 lcd(SPI_MOSI, SPI_SCK, SPI_MISO, p8, p11);
~~~


# Notes

There is a little bit irritating in the auto complete (ie brackets, and "smart" newlines.
Don't like the toolbar at the bottom of the page, Suppose I will get used to it.

