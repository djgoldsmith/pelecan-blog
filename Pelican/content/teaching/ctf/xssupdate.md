title: An update on the XSS Trainer
date: 2019-05-17T21:26:44+01:00
tags: Teaching, CUEH, Hacking, XSS
Summary: >
	An Update to the XSS trainer, Now with added Server side checks

A Quick follow up on the [XSS trainer post]({filename}/teaching/ctf/xssAlerts.md)

So what would Scooby Doo?  Well if its Greg, come up with an
interesting workaround.

Sot the client side detection for Alerts in the console worked OK
(except on Edge).  However, it turns out that we you could still work
around this in debug mode.  Simply open up the console and inject an
new image tag with *on error* triggering an alert.

I suppose the learning point here is to live by my own rules, Client
side verification may seem like it works, bu its **never** a good idea
to rely solely on something the client controls.

Fortunately, the worst that could happen from this is a good student
spent more effort getting some flags they would have got anyway.

Anyway, long story short, added some server side verification,
checking for expected input, this time it should be good, but who
knows.


## Docker Image.

Put together a quick and dirty docker image for people to try, 

<https://cloud.docker.com/u/cueh/repository/docker/cueh/ctf-xss-alert>

you can give it a go with

~~~
$docker run cueh/ctf-xss-alert
~~~




