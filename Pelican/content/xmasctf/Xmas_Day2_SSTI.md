title: Xmas CTF Day 2 Dev Notes
date: 2021-01-11 18:35:14
tags: Hacking, CUEH, CTF, SSTI, Comsec
Summary: >
	An introduction to SSTI and design notes on template injection for the Comsec Xmas CTF
	
	
# Introduction

This is the first post in part of a series of posts on developing
challenges for the Comsec 12 Days of Christmas CTF.

As well as the walkthrough of the challenges, I will have a bit of
analysis / dev notes on the process, and the fun behind the scenes
stuff I did to get things going.

# Dev Notes

As always the first thing here is to come up with a topic.  Recently,
been playing around with template injection (I blame the web
challenges on HTB) so thought a series of challenges around this would
be interesting for the students to play around with.

As we had students from all 3 years playing, my plans for the CTF were
to have a few levels, the a nice and easy introduction to the topic.
Then build on this upping the difficulty for the next levels.  This
should mean that our first years could get through the first level
with some basic googling, and then (hopefully), catch the bug and try
harder for the next ones.

For these challenges we would also avoid the usual "boot to root"
approach, and just have the students grab one user flag per level.
While this takes away some of the fun of elevating from standard user
to root, it keeps the skills for each level self contained.

## Challenge Design

I used docker (surprise) to build and host the challenges, as well as
fitting well with my workflow, it also meant they would work with the
VM I put together to host all my challenges.

Two of the images were built with the excellent
[Flask](https://flask.palletsprojects.com/en/1.1.x/) framework.  While
I still prefer [Pyramid](https://trypyramid.com/) for larger projects,
I didn't need a database of complex routing and its a bit overkill for
something that is essentially a template based WSGI server.

Templates was through Jinja, mostly because its what I would use normally.

One of the machines was built using Node.js, its the first time I have
really used it in anger and was pleasantly surprised over how easy it
is.  Looks like by bad experiences with NPM have tainted my view of
the concept.  Still not sold on using JavaScript for server side
stuff, when there are much better languages available, but I see its place.

# SSTI

Server Side Template Includes, are in interesting class of vulnerability.

## Template Engines

Having a template engine to generate your web page is a brilliant
idea, it allows you to use the power of the underlying programming
language to help build your pages, without resorting to a either
generating chunks of rendered code by hand, or a fully "dynamic"
JavaScript / XHR style solution.

They also help us to be "Good Programmers(TM)" and do all that MVC
style separation of concerns stuff. This is good, as being a
coder doesn't necessarily mean you are good at the design elements[^thisblog]
(and vice-versa), so you can let the design mooks build what the site
looks like (and get excited about mobile first and UX), and leave
placeholders for the backend bods to do their magic with the data. 

For example, suppose we wanted a alert box that displayed the users
name, our template could look something like this.

```
<div class="alert">
	Hello {{ user }} 
</div>
```

We then provide the template with provide it with the value for user
(for example ```user = dang```) which is substituted when the template
is rendered.

## The Problem with flexibility

While this works well for single, well specified variables, what
happens when we have more complex data?  How do we handle things like
lists?  What happens for variables that may only exist in a given
situation.

We could move the formatting into the backend code, supplying a fully
formed list to the template engine.  But this negates all of the
advantages of templates. We worked hard to remove some of the
presentation logic from the coders, but still trust them with
things like lists.

The more sensible alternative is to give the template engine itself
some limited processing power.  It makes sense that we can keep the
presentation logic separate by allowing simple calculations, selection
and iteration in the template itself.
 
For example, we could build and display list of users preferences like this:

```
<ul>
{% for item in preferences %}
  <li>{{ item }}</li>
{% endfor %}
</ul>
```

While this is an sensible way to give the templates the functionality
needed to separate program logic and presentation. It leaves us with a
whole new issue.  How much functionality do we want to push to the
template, and how do we filter potentially insecure input.

Security is often a trade-off.  We have lock down functionality to
increase security, but this restricts what we can do.  In the case of
templates, completely stopping the template from executing code would
increase security, but at the expense of usefulness.  We could stop
things like iteration, but this then pushes the view logic back to the
controller breaking our MVC pattern.

So we have to trust our developers to manage some risk in return for
convenience. Giving the template engine some of the powers of the
underlying interpreter, makes the tool more useful. However, the devs
need to accept that it also comes with potential security drawbacks.

The template engine itself may also have some protection against this
type of exploit.  This is usually though filtering input though
*allow* or *deny* lists, or running code through some form of sandbox
that restricts the interpreters system access.  However, as with any
filtering or sandbox based approach, this is only as successful as the
assumptions made when developing the filter. We will look at some of
the methods to work around these features later.





On the whole the template based system works well, the vast majority
of template based systems wont have a security flaw.  However, if the
dev tries to do something unusual, or passes un-sanitised user input
to the engine, problems can occur (as usual, trusting the user is a
bad idea).


# Executing code within a Flask / Jinja template.

Having looked at the trade-off between functionality and security in
template engines, lets see how we can use this to get unexpected
behaviour.

Imagine we have an application that takes user input from a form. 
For example:

```
<form>
	<input name="payload">
	<button class="submit">Go</button>
</form>
```

## A "Safe" version

Now lets imagine our flask code is simply takes the variable and
passes it to a template Something along the lines of:

```
theTemplate = """
<html>
<form>
  <input name="payload">
  <button class="submit">Go</button>
</form>
  
 <p>User Input is {{ userinput }}</p>
</html>
"""

@app.route('/')
def main():

    payload = flask.request.args.get("payload")
    
    return flask.render_template_string(theTemplate, userinput = payload)
```

NOTE: To keep the examples similar I am building the template from a string, its the
same if you use a file)

Now this *should*[^should] be safe and behave in the way we want, Our
user can specify an input, and the same input is escaped, and passed
to the template engine.  There is even some protection against XSS[^should], as
tags like ```<``` are escaped by the template engine.

## An Unsafe version

To see an unsafe version we can make a small modification to the template.
This time we are going to use pythons string formatting syntax to add the user input.

```
unsafeTemplate = """
<html>
<form>
  <input name="payload">
  <button class="submit">Go</button>
</form>
  
 <p>Unsafe Input is {}</p>
</html>
""".format(payload)
```

Again, I like format strings so am using them, we get the same effect
with f-strings, or string replace functions.

  - If we use a payload of ```1+1``` we get our expected output of ```Unsafe Input is 1+1```.
  - If we use the an input of ```{{ 1+1 }}```  which gives us an output of ```Unsafe Input is 2```.

So whats going on here?  It turns out we are being bitten by the flexibility of the template engine.
When we specify an input of ```{{ 1 + 1}}}``` the  template that gets created and passed to the engine becomes

```
<html>
  ... snip ...
 <p>Unsafe Input is {{ 1 + 1 }}</p>
</html>
```

Now the ```{{ 1+1 }}}``` is a valid template substitution string,
Jinja has no idea (and doesn't really care as it trusts you to do the
right thing) if its user input or not.  Thus it sees the string,
checks it knows what to do with it (yes it knows how to add two
numbers together), so evaluates it and returns the output.

So our small logical error, using the wrong form of string
replacement, has given us a potential security issue. (its also given
us an XSS problem, but that's another story)

If you are wondering is this would work with our safe example, the
answer is no.  I haven't actually dug around in the internals here, but
it looks like the string escape on user input keeps us safe.  So if we
enter ```1+1``` in the form for the safe version, what gets passed to
the template is ```"1+1"```


## Taking it further

So now we have some basic code execution,  quite how far can we take this?

This depends somewhat on the temple engine, and framework used. Most
of them also pass some meta data (things like the request details, and
internal state) to the template before rendering. Again from a design
point this is a sensible choice, being able to grab session variables
(such as the username), is a common use case when presenting web data.

For example Flask has the following global variables accessible to all templates

	- config:  The current flask configuration
	- session: The current sesion (if it exists)
	- request: The current request object
	- g: The variables currently mapped to flasks global.

We may also be able to start using the built in methods of the
underlying language.  Building input that can give us Remote code
execution on the system, leading to further exploit.

I will cover these in the writeup for the CTF challenges themselves.


## Summary

In this article we have a brief introduction to SSTI, looked at what
the problem is, and the types of development mistakes that can cause
it.

While SSTI is certainly an security issue, but in my view its one of
the ones where the blame falls squarely on the shoulders of the
developer.  Template engines are a fantastic idea, but rely on this
flexibility to execute code for their usefulness.

Much in the same way as we don't blame SQL, or stop using it because
SQL injection happens (another example of where the blame rests almost
entirely with developers),  its only when the developer misuses (or
misunderstands) the functionality that it becomes a problem.


[^thisblog]: You are looking at an example of this.
[^should]: For a given value of should, It seems secure against some most prodding, but things are only secure until they aren't.
