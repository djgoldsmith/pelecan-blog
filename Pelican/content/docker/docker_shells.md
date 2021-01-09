title: On Demand, per user Docker shells
date: 2019-06-26T08:15:04+01:00
tags: Teaching, CUEH, Hacking, Docker
Summary: >
	Experiments with Spawning Docker Containers on demand over an SSH connection.

A common use case when we are teaching on the hacking modules is for
the student to gain access to a server and do some task.

Previously we have used VM's for this, setting up the challenge in a
"Stripped down" VM and having the student download it to play with in
the Lab. While this works, its not without its disadvantages:

   - The average VM weighs in at a couple of GB, its not a lot, but it
     takes time to download. (its certainly prohibitive on my majestic
     1MB connection at home)
   - Ensuring the student gets a "Fresh" copy of the VM free of anyone
     elses exploit.  Personally, I think there is nothing more
     frustrating, than finding a copy of the root flag, or someone
     elses POC, as it ruins the challenge.
   - It needs virtualisation software, while this is installed by default
     in the Hacking Lab, its not on the standard Uni Image, and
     depending on the students own hardware it may limit the
     opportunity to practice outside of the lab

We could set up a "Cloud" version of the target, allowing the student
to connect remotely (in much the same way as CTF like
[Hack the Box](http://hackthebox.eu) do.)  This works well, for access outside of
the lab but we get major issues with ensuring the student gets the
"Fresh" copy.  Additionally, I have around 50 challenges (and counting,
and that's just one module), so resource management on the server we have
starts to become interesting.

## The Concept

I have been experimenting with using Docker for this type of
teaching. (See [Docker for Ethical Hacking]({filename}/docker/using_docker_to_teach.md) 
for why I think docker is way forward).

It turns out it is possible to connect the docker containers to an SSH
session. So each time a user connects over SSH, a new docker container
is fired up with the image used based on the username they are
connecting with.  This would allow containers to be started "On
Demand".


> NOTE: I was inspired by the excellent Over the Wire Advent
> challenges. which used a similar approach.

### Requirements

 - Accessible outside of the hacking lab
 - Allow students access challenges "On Demand"
 - *Sandbox* each student onto their own instance, to stop
   "accidental" P0wning through someone elses hard work.

## Implementation


### First Cut
The key part of our implementation is that we can create (and / or)
attach to a running container, This is pretty standard docker stuff.
For example, lets imagine we have the container
*7024cem/ctf-lsudo-demo*, and want to connect as the unprivileged
*cueh* user.

On the command line we could use:

~~~.term
dang@dang-laptop ~$ docker run -it --user cueh --workdir /home/cueh --rm 7024cem/ctf-lsudo-demo /bin/bash                                                                                               
cueh@9e63b8eadc4c:~$ pwd
/home/cueh
cueh@9e63b8eadc4c:~$ ls
README.txt
cueh@9e63b8eadc4c:~$ 
~~~

As we can set the shell that is spawned when the user logs in, its
easy enough to create a script that runs this command and set it as
the users default shell.

~~~.term
[root@zaphod ~]# cat /home/testuser/runshell 
#!/bin/bash
docker run -it --user cueh --workdir /home/cueh --rm 7024cem/ctf-lsudo-demo /bin/bash
~~~

If we add it the */etc/shells*

~~~.term
[root@zaphod testuser]# cat /etc/shells 
# Pathnames of valid login shells.
# See shells(5) for details.

/bin/sh
/bin/bash
/home/testuser/runshell
~~~

And set it as the users default shell:

~~~.term
[root@zaphod ~]# chsh -s /home/testuser/runshell testuser
Changing shell for testuser.
~~~

Each time they login a docker container is launched, and the user is dropped into it.

## Implementation.  Second Cut.

While this is pretty cool, we have some issues.  First off is the 50+
Containers.  While I could create a separate spawning script for each,
"I ain't got time for that stuff". Also I want to make it nice and easy
to add new elements.

So I cobbled together a quick and dirty python script that checks in a
database, (yes I appreciate that is probably overkill, but it also
works nicely with the flag server), and uses the python docker
bindings to spawn the relevant container.

The stripped down version of the code is below (removed various error check etc, fill version is in the git repo)

~~~.python
import docker
import dockerpty

def startDocker(user, ipadd = None):
	"""Start a docker container
	
	@param user: User who as initiated the SSH connection
	@param ipadd: Ip address conneciton was initiated from
	"""
	
	#Create docker Client
	client = docker.from_env()
	
	#Get the relevant container from DB driver
	thecontainer = driver.getDetails(user)

	# ...SNIP... Setting Various options
	
	#Create a Container
	container = client.containers.create(
        image = thecontainer.image,
        user = thecontainer.user,
        working_dir = thecontainer.workdir,
        command=thecommand,
        auto_remove=autoremove,
        tty=True,
        stdin_open=True
        )

	#And Connect it to a pseudoterminal
	dockerpty.start(client.api, container.id)

~~~

We can then call the script with command line arguments to fill in the relevant parts.

~~~.python
if __name__ == "__main__":
    logging.basicConfig(level=logging.DEBUG)
    log.debug("Testing")

    #Get the thing to accept command line args
    if len(sys.argv) < 3:
        log.warning("Called without user")
        sys.exit(-1)
        
    args = sys.argv
    log.debug("Arguments are {0}".format(args))

    theuser = sys.argv[1]
    log.debug("--> Starting process with user {0}".format(theuser))

    #And get the IP address
    theIp = sys.argv[2]
    log.debug("--> Ip address of connecting client {0}".format(theIp))
    
    dockershell.dispatcher.startDocker(theuser, theIp)
~~~


Now if we call the script using the $USER and $SSH_CLIENT system
variables, we get a new session for each user who connects.
I created a helper script for this.

~~~.term
$cat dockershell.sh
#!/bin/sh
python dockershell.py $USER $SSH_CLIENT
~~~


### Wiring it up as a users shell

Now, rather than have a custom shell for each user we can just assign
the dockershell to each of them, and let the system variables set by
SSH on login deal with everything.

Add the dockershell.sh script to /etc/shells

Then just create a user for each container, with the dispatcher as their shell.

~~~
useradd -M -G docker -s /opt/dockershells/dispatcher.sh <whoever>
passwd <whoever>
~~~

## Persistent Images

One issue with the approach above, is that there is a 1:1 mapping
between SSH sessions and containers.  Therefore if we start two SSH
sessions from the same machine, each ends up in its own "fresh"
instance.  This can make some approaches difficult, as you may want a
second SSH channel to monitor processes while you make changes, or you
save changes that take effect on login.

To work around this I have a "persistant" flag for each image in the
database. If this is set we store the IP address of the connecting
machine, and the docker generated name of the container that is
created.  (For example 127.0.0.1 , "amazing_hopper") We then start the
container without the *auto_remove* flag set.  When the user connects
we can check if the container exists, and reconnect to it.

~~~.python
if thecontainer.persistant:
	log.debug("Persistant Container")
	autoremove=False
	
	#Fetch from database#
	details = driver.getPersist(thecontainer.image,  ipadd)
	if details:
		#Update the Last Access Time
		driver.storePersist(thecontainer.image, ipadd, details.containerName)

	    #And Connect
		dockerpty.start(client.api, details.containerName)
	    return #Exit the function

#Otherwise Create the container in the way we did above.
container = client.containers.create(
        image = thecontainer.image,
        user = thecontainer.user,
        working_dir = thecontainer.workdir,
        command=thecommand,
        auto_remove=autoremove, #Override the setting
        tty=True,
        stdin_open=True
        )
		
#Store in the database
if thecontainer.persistant:
	driver.storePersist(thecontainer.image, ipadd, container.name)
    
    
log.debug("Spawning Termnal")
dockerpty.start(client.api, container.id)
~~~

# Evaluation

So far I have tested the concept, and everything appears to work nicely.

The next stage is to see how well it copes under load.
Currently have 100 Sessions connected to the server in the hacking lab, trying to simulate student activity, with a range of tasks:

  - Idling:  (Proper simulation of a student)
  - Top:  (Gives me some network traffic)
  - *finding* SUID files, every 5 seconds.

System load is around 10%, will see if it holds together over the weekend.

## Share and Enjoy!

You can find the repository at.  https://github.coventry.ac.uk/aa9863/dockershells/



Comments and feedback welcome.


