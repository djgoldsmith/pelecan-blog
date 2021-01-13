Title: Moving from master to main in Git
Date: 2020-12-18 12:00
Category: Notes and Cheatsheets
Tags: git, cli
Summary: >
	Making the default branch **main** instead of **master**


For a while now been using **Main** rather than **Master** when it comes to my
Git repos. 

Its a trivial thing to do, and while its still a bit weird (10+ years
of git muscle memory) to use, if it stops one person feeling
uncomfortable then its worth it[^muscle-excuse].


## Introduction 

There has been lots of discussion around some of the terminology in
Computer Science.  With some of the words we have used to describe
things may or may not be suitable for the modern world.

While (personally) I think some of it is an over-reaction. Just having
the word Black in something doesn't automatically make it racist.  I
am 99% sure that there was never any offence intended when first
created. 

However, If it might cause offence, and its easy for me to change a
small bit of behaviour, why not.  Being reasonable, and dealing with
others points of view is what makes civilisation work[^not-an-excuse].

Language is constantly evolving, so kicking up a fuss about you not
being able to call something master "Because we always have", when it
has no real impact on life, and is trivial to implement seems unreasonable.

The [NCSC](https://www.ncsc.gov.uk/blog-post/terminology-its-not-black-and-white) 
have a great blog post on a the whole "whitelist", "blacklist" issue.


# Commands for changing Repo preferences

## Using Main by Default on the Git Command line

A trivial change we can make via the global git configuration

```
$ git config --global init.defaultBranch main
```

## Modifying existing repo's to use main

If we have older repositories they may use the old master by default.
To change this to main by *moving* the branch.  This should retain all the history

```
$ git branch -m master main
```

Then push with 

```.bash
$ git push -u origin main
```


## GitHub and GitHub enterprise

Our last change is to the central repository.

By default full fat GitHub now uses main by default for all new repositories.

However, our Enterprise doesn't yet and defaults to master. We may
also want to update older repo's, to use the new naming scheme.

We can do this under 

  - Settings
    - Branches
	- Change Default Branch

![Git Options]({static}/images/GitMain.png)




[^muscle-excuse]:  Muscle memory isn't an excuse either.  If your old enough to be really attached to master, your probably old enough to have converted through the CVS / Subversion / Git chain,  you managed to cope then, so I am sure you can deal with it now.

[^not-an-excuse]:  However, this is not an justification for a general "It offends me, so you cant use it" approach.  This seems to come up a counter argument against this sort of stuff (usually by the sort of people who take great delight in moaning about this sort of thing, while calling everyone else snowflake).  Yes calling something a **main** rather than **master** might offend you, but why?  Does it beat, "I feel uncomfortable using words that could be linked to some pretty nasty parts of history".
