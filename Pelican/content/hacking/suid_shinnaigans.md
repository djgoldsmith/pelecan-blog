title: SUID Shenanigans
date: 2019-03-19T19:28:56Z
tags: hacking, linux
Summary: >
	Spotted some interesting new behaviour with the SUID bit in Linux


While prepping for the overflow lecture noticed some new behaviour with
SUID, Haven't been able to find any changes in the Kernel log, so
thought I would play to work out the behaviour

# Path Poisoning Weirdness

## Old Behaviour

In the Linux Trainer, the following code was used in a path poisoning example.

~~~c
#include <stdio.h>
#include <stdlib.h>

void main(void){
  printf("Attempting to access file:\n");
  system("cat /home/level18/File.txt");
}
~~~

Once installed, SUID permissions were set

~~~bash
level18@f0432d98b859:~$ ls -l
total 32
-r-------- 1 elev18  elev18     36 Feb 11 10:00 File.txt
-r-------- 1 elev18  elev18     64 Feb 11 10:00 Pass.txt
---s--x--x 1 elev18  level18 16624 Feb 11 10:00 runme
-rwxr--r-- 1 level18 level18   141 Feb 11 10:00 source.c
level18@f0432d98b859:~$ 
~~~

We could do the Classic Path poison attack and elevate the permissions

~~~bash
level18@f0432d98b859:~$ echo "/bin/sh" > cat
level18@f0432d98b859:~$ chmod +x cat
level18@f0432d98b859:~$ PATH=.:$PATH ./runme 
Attempting to access file:
$ whoami
elev18
$ 
~~~


## Recreating on my Arch System

So take the same code, compile and set the permissions

~~~bash
dang@dang-laptop ~/Coding/Testing$ ls -l
-rwsr-sr-x 1 root dang 16624 Mar 19 19:40 a.out
-rw-r--r-- 1 dang dang   141 Mar 19 19:40 Testcode.c
dang@dang-laptop ~/Coding/Testing$ 
~~~


And try to recreate the exploit

~~~bash
dang@dang-laptop ~/Coding/Testing$ echo "/bin/sh" > cat
dang@dang-laptop ~/Coding/Testing$ chmod +x cat
dang@dang-laptop ~/Coding/Testing$ PATH=.:$PATH ./a.out
Attempting to access file:
sh-5.0$ whoami
dang
sh-5.0$ 
~~~

So it looks like we get different behaviour between the installs, SUID
doesn't elevate permissions on the more modern system.

# Digging a Bit Deeper.

## Setting SUID on the file system
To avoid the confusion with the path poisoning, lets work on a simpler version of the file.

~~~c
#include <stdio.h>
#include <stdlib.h>

void main(void){
  system("/bin/sh");
}

~~~

Set SUID bit in the file systems, and when we run this we still don't
get a root shell??

~~~bash
dang@dang-laptop ~/Coding/Testing$ ls -l
-rwsr-sr-x 1 root dang 16576 Mar 19 19:49 a.out
-rwxr-xr-x 1 dang dang     8 Mar 19 19:41 cat
-rw-r--r-- 1 dang dang    80 Mar 19 19:48 Testcode.c
-rw-r--r-- 1 dang dang   141 Mar 19 19:40 Testcode.c~
dang@dang-laptop ~/Coding/Testing$ ./a.out                                                                          
sh-5.0$ id
uid=1000(dang) gid=1000(dang) groups=1000(dang),977(docker),987(uucp),998(wheel)
sh-5.0$ 
~~~


## Setting SUID in the Binary

However, if we add a call to SUID inside the binary we get slightly
different behaviour

~~~c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

void main(void){
  setuid(0);
  system("/bin/sh");
}
~~~

Compile and Run, which doesn't help us:

~~~bash
dang@dang-laptop ~/Coding/Testing$ ls -l                                                                            
total 32
-rwxr-xr-x 1 dang dang 16624 Mar 19 19:53 a.out
-rwxr-xr-x 1 dang dang     8 Mar 19 19:41 cat
-rw-r--r-- 1 dang dang   136 Mar 19 19:52 Testcode.c
-rw-r--r-- 1 dang dang   141 Mar 19 19:40 Testcode.c~
dang@dang-laptop ~/Coding/Testing$ ./a.out                                                                          
sh-5.0$ id
uid=1000(dang) gid=1000(dang) groups=1000(dang),977(docker),987(uucp),998(wheel)
sh-5.0$ 
~~~

If we also put the file system SUID bit (and owner to root) we get the desired result

~~~bash
dang@dang-laptop ~/Coding/Testing$ ls -l                                                                            
total 36
-rwsr-sr-x 1 root dang 16624 Mar 19 19:53  a.out
-rwxr-xr-x 1 dang dang     8 Mar 19 19:41  cat
-rw-r--r-- 1 dang dang   136 Mar 19 19:54 '#Testcode.c#'
-rw-r--r-- 1 dang dang   136 Mar 19 19:52  Testcode.c
-rw-r--r-- 1 dang dang   141 Mar 19 19:40  Testcode.c~
dang@dang-laptop ~/Coding/Testing$ ./a.out                                                                          
sh-5.0# id
uid=0(root) gid=1000(dang) groups=1000(dang),977(docker),987(uucp),998(wheel)
sh-5.0# 
~~~

So it looks like we need to have SUID set in both the file, and the
file system.


## Dealing with Other users.

Lets add a third "Testuser" to the mix (Note, we need to set
permissions of the ```~/Coding/TEsting``` directory to 777 to allow
the testuser to access the files.

We keep both ```setuserid``` in the source and the file system SUID
bit, and the standard privesc still works as expected.

~~~bash
dang@dang-laptop ~/Coding/Testing$ ./a.out                                  
sh-5.0# whoami
root
sh-5.0# 
~~~

~~~bash
[testuser@dang-laptop Testing]$ ./a.out 
sh-5.0# whoami
root
sh-5.0# exit
exit
[testuser@dang-laptop Testing]$ pwd
/home/dang/Coding/Testing
[testuser@dang-laptop Testing]$ ./a.out 
sh-5.0# whoami
root
sh-5.0# 
~~~

## Setting User ID to another user.

If we modify the userid in the binary (for example to dang).
And then set the file system SUID bit to that user the elevation doesn't work

~~~bash
sh-5.0$ ls -l
total 28
-rwsr-xr-x 1 dang dang 16624 Mar 20 18:01 a.out
-rw-r--r-- 1 root root   139 Mar 20 18:01 Testcode.c
-rw-r--r-- 1 root root   136 Mar 20 17:43 Testcode.c~
sh-5.0$ ./a.out 
sh-5.0$ id
uid=1001(testuser) gid=1001(testuser) groups=1001(testuser)
sh-5.0$ 
~~~

However set the File system owner to root, and the SUID bit and we get the userchange.

~~~bash
sh-5.0$ ls -l
total 28
-rwsr-xr-x 1 root dang 16624 Mar 20 18:01 a.out
-rw-r--r-- 1 root root   139 Mar 20 18:01 Testcode.c
-rw-r--r-- 1 root root   136 Mar 20 17:43 Testcode.c~
sh-5.0$ ./a.out 
sh-5.0$ id
uid=1000(dang) gid=1001(testuser) groups=1001(testuser)
sh-5.0$ 
~~~

This is behaviour is not continued with Group permissions

~~~bash
[testuser@dang-laptop SUIDTest]$ ls -l
total 28
-rwxr-sr-x 1 dang root 16624 Mar 20 18:01 a.out
-rw-r--r-- 1 root root   139 Mar 20 18:01 Testcode.c
-rw-r--r-- 1 root root   136 Mar 20 17:43 Testcode.c~
[testuser@dang-laptop SUIDTest]$ ./a.out 
sh-5.0$ whoami
testuser
~~~

So It looks like we need to have a root user, regardless of the user
we are trying to SUID to.


# Even Weirder.  Doesn't appear to work in TMP.

So we compare the same, setup in user dir and /tmp/

~~~bash
[testuser@dang-laptop Demo]$ pwd
/tmp/Demo
[testuser@dang-laptop Demo]$ ls -l
total 24
-rwsr-sr-x 1 root root 16624 Mar 19 22:36 a.out
-rw-r--r-- 1 dang dang   136 Mar 19 19:52 Testcode.c
[testuser@dang-laptop Demo]$ ./a.out 
sh-5.0$ 
~~~

~~~bash
dang@dang-laptop ~/Coding/Testing$ ls -l                                    
total 32
-rwsr-sr-x 1 root root 16624 Mar 19 22:37 a.out
-rwxr-xr-x 1 dang dang     8 Mar 19 19:41 cat
-rw-r--r-- 1 dang dang   136 Mar 19 22:34 Testcode.c
-rw-r--r-- 1 dang dang   141 Mar 19 19:40 Testcode.c~
dang@dang-laptop ~/Coding/Testing$ ./a.out                                  
sh-5.0# 
~~~

What about other Directories

Stashing the file in OPT appears to work for both users

~~~bash
dang@dang-laptop /opt/SUIDTest$ ls -l                                                                               
total 24
-rwsr-xr-x 1 root root 16624 Mar 20 17:43 a.out
-rw-r--r-- 1 root root   136 Mar 20 17:43 Testcode.c
dang@dang-laptop /opt/SUIDTest$ id                                                                                  
uid=1000(dang) gid=1000(dang) groups=1000(dang),977(docker),987(uucp),998(wheel)
dang@dang-laptop /opt/SUIDTest$ ./a.out                                                                             
sh-5.0# id
uid=0(root) gid=1000(dang) groups=1000(dang),977(docker),987(uucp),998(wheel)
sh-5.0# 
~~~

# Summary

- SUID no longer appears to work on a purely file system basis
- Need to set userid inside the binary also.
- Appears we need the owner of the file to be root, for any kind of
  SUID to happen.

Not sure how I feel about this, while its probably a good idea to force
the coder set the SUID flag in both the binary, and the file system,
other things bother me.  Needing the file to be owned by root
certainly seems a bit strange, and seems likely to lead to cockups.

> NOTE: This is initial thoughts, perhaps doing something other than a
> system call would show more expected behaviour, I will get around to
> testing this soon.


