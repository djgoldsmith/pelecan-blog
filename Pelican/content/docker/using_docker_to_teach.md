title: Docker for teaching EH
date: 2019-06-25T13:27:19+01:00
tags: Teaching, CUEH, Hacking, Docker
Summary: >
	Thoughts on using docker for teaching PenTesting, and an introduction to a Linux Trainer


Those who know me know I like docker, its simplified teaching a lot of
the hacking examples to the students.  Yes it has some flaws (networking) or
requires some gnarly workarounds for things, but its a major feature
of my material now.


## Example:  The Linux trainer

"Inspired" by the work of Under The Wire's bandit.  I built a docker
based Linux trainer.  This is intended to give the students a way to
practice the basic commands (they should already know). 


The [Linux trainer]({filename}/docker/nixtrainer.md) Its relatively standard wargame stuff. SSH into each level, complete a
task and get the creds for the next. 

Nice thing here is it weighs in at a mighty 190MB.  Compare that to
the overhead of setting up a VM to host it.

Feedback on the whole has been positive, few gumble about having to
use docker itself. (IE typing ```docker run cueh/nixtrainer``` can be
hard) but this was mostly around getting the tools setup.


# The Good, the Bad and the Ugly.
## Benefits

The main benefits from my point of view are:

  - Image size, most of my stuff is built from the same "Base"
    containers, this means that after the first image download, the
    base image remains unchanged and all that the students need to
    download is the content.  For Example: My Base Debian CTF
    container is around 120MB, after downloading that, its only 5-10MB
    of data that needs to be grabbed for each challenge.  Compare that
    to around 1GB for a basic Debian VM.
		
  - Persistence: (Or the lack of it), containers store whatever
    changes happen in a thin layer on-top of the image.  This is
    really cool, as when the students invariably screw something up,
    they just kill and restart the container.  Everything is reset
    back to "factory defaults".  Compare this to a VM where we have to
    either have "snapshots", or some other way of resetting.
	
  - Speed. On the whole starting containers is **FAST**, a few seconds
    for something simple using SSH.  Yes things like databases take
    some time to initialise, but that's the Database itself, rather
    than docker.
	
  - Getting **Root**. We all know there is nothing like the feeling of
    getting root. This is not so much a benefit of docker itself, but
    I am a bit leaary of giving students root on a VM, especially when
    there are a set of challenges.  With containers, I tend to break
    each challenge into a container, and let them get root, meh Its
    only another 10 Meg for the next challenge.
	
  - "Platform Independence", the containers can be run on various OS,
    docker for windows does a pretty good job (although I find it
    tends to break my exposed ports), Meaning regardless of the base OS,
    the students still get the benefit.
	
## So-So

Things that can be both good and bad depending on the situation.

 - Simplicity: This is a bit of a double edged sword, but default the
   images I use come stripped down.  It means that there are few
   distractions for the proof of concept. For example, if we are doing
   anything with cron, cron will be present, otherwise it wont exist.
   This is nice as it lets students focus on the task, but lacks
   realism.  It also makes it a PITA to make more complex systems.

 - The One Process per container Philosophy.  I understand why this is
   a decision, and for the whole agree with it.  However, at times,
   its a pain to have to plumb stuff together just to allows an Apache
   Server with SSH.  Especially the hoops I have to jump through with
   volumes and sharing data.

## The Bad

 - Networking, Want to simulate a network of machines?  Welcome to a
   world of pain.  Problem here is its out of context for what docker
   is designed to do, but it would be lovely.
 
 - Doing anything complex requires magic.  Overflows, GDB anything
   that requires complex privileges, can be a right pain to setup.
   Much easier to have a VM for this.

 - Container Security.  Forget breakouts, there is a more fundamental
   problem here, If the student can run the container at home its as
   simple as using ```docker exec <words>``` to get a root
   shell. Perhaps I am nieve, I trust people not to do this, but.....




