title: CLI basics wargame
date: 2019-02-14T17:20:41Z
tags: Teaching, Linux, CUEH, Docker
Summary: >
	Learning through doing.  A "wargame" to build basic Linux Skills

One of the more "fun" parts of my job is working out how to get the
students engaged with the material.  A common way of doing this in the
Cyber Sec community is the "Wargame" or "CTF" (Capture the flag).

Inspired by things like Bandit at [overthewire.org](http://overthewire.org) I have put
together a Linux trainer for my classes.

The docker version is available on [docker hub @cueh/nixtrainer](https://hub.docker.com/r/cueh/nixtrainer)
have a look an tell me what you think.

# The Problem

For the Ethical hacking module there is a certain subset of command
line-fu that we need.  Previously, I had done a limited "Chalk and
Talk" approach, talking about the key items, before unleashing the
students on a VM with several challenges to test their skills.

While this is a world away from the "Chalk and Talk" approach, it
still wasn't working as well as I would like.  Students tended to zone
out during the talk on the commands (and to be honest i cant blame
them, a set of slides on less isn't the most inspiring thing), so the
practical could be difficult without them revisiting the lecture.

Watching the students, it was clear that they would flick back and
forth through the material to find the bit of information they needed
to solve the problem, rather than actually applying the concepts
taught to solve the problem.

# A solution?

Given that war games and CTF are popular ways of honing skills, and
inspired by the excellent Bandit wargame from <overthewire.org> I have
put together a basic Linux trainer that teaches some of the core
skills that the students will need to complete the course.

Its the standard wargame concept, there are various levels, you start
at level0 and as you complete each challenge you get the password for
the next level.   The game covers concepts including:

 - Where am I and how do I get elsewhere (ls, cd, and paths)
 - Access control and Permissions
 - Finding things on the system (Find, locate)
 - Finding things in files (grep)


## Driven by Manpages

One issue I have tried to solve is the "Can I google that for
you?" problem.  While it is my job to introduce people to the tools,
and discuss the technical topics..  Being able to think around a
problem is a core skill.  I hope none of my students end up just
pressing *the nessus button(tm)*, then pressing win on metasploit.
Being able to think about the problems you face is a key skill, and
being able to RTFM is part of the solution.

To address this, all the documentation for the challenges is in man
pages, hopefully the processes of using the built in help files will
stick.

## Docker to the rescue!

Another, issue that comes up with creating material is it tends to be
*heavyweight*, putting together a full VM for this is pretty hardcore,
around 3GB for < 20M of data.  The other issue is reverting any of the
tasks when the students inevitably ```rm -rf``` something.

Docker turns out to be a great solution for this kind of standalone
thing, as we end up with an "image" of a couple of hundred meg, that
can be reset with a simple ```docker rm <whatever>``` Even better, if
we reuse the base image we end up only downloading the diff between
the versions, so I can put together various web based challenges for
the same infrastructure for the $\sqrt(fa)$ disk space.

I plan on discussing docker for teaching in a future post.


# Reflections

 -  Overall the students seem to have enjoyed it, but its too early
    to tell if it has made a difference.
 -  Not sure on the balance of leading people through it, VS discover
    for yourself.  It seems that there is still a bit of a learning
    curve some struggle to get over. To be honest, its trivial stuff,
    so perhaps its more a problem with motivation, rather than the
    content.

