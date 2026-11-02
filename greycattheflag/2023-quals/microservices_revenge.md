# Microservices_Revenge

**CTF:** GreyCatTheFlag 2023 Quals  
**Category:** web  
**Original writeup:** <https://jh123x.com/blog/2023/greyctf23-microservices-revenge/> — by **jh123x.com**. All credit goes to the original author; this is an archived copy.

---

This is the author’s writeup for the challenges Microservices Revenge in the web category.

This describes the intended solution when I made the challenge.

Challenge description #

I’ve upgraded the security of this website and added a new feature. Can you still break it?

Junhua

Similar to Microservices, a dist.zip was also provided for this challenge with the server set up.s

My Notes #

Was expecting the challenge to be solve after a few hours, but it was solved around the first hour.

There were unexpected solutions due to the nature of the setup.

This was meant to be more of a Medium difficulty web challenge for CTF Veterans.

This challenge tests on the following: Server Side Template Injection (SSTI) with black listed characters Sending a request through a SSTI using jinja2 functions
