# Postviewer_v3

**CTF:** GoogleCTF 2024  
**Category:** web  
**Original writeup:** <https://gist.github.com/terjanq/27230afcee73ee75484ac14ac53e78bc> — by **terjanq**. All credit goes to the original author; this is an archived copy.

---

POSTVIEWER V3 [303pt] New year new postviewer. https://postviewer3-web.2024.ctfcompetition.com

Solved by (19):

Friendly Maltese Citizens, DiceGang, BlueWater and more.

Similarly to other Postviewer challenges, a player is welcomed with a simple client-side application where they can store and render some files.

Each file is rendered in an shim iframe hosted on a unique origin that is directly connected to the contents of the file. This ensures that a file A will be protected by Same Origin Policy from a file B.

The goal of the challenge is to find a way to leak admin's file containing the flag.

Shim iframe

Each shim iframe is rendered at a unique URL, shown below.

https://sbx-<hash>.postviewer3-web.2024.ctfcompetition.com/product/shim.html?origin=https://postviewer3-web.2024.ctfcompetition.com

Hash is calculated in the following way

hash = sha256 ( fileBody + product + origin + salt )

Shim iframe receives a file to render ( fileBody ) together with mimeType and salt over postMessage communication. The product and origin are both stored in the URL. The origin 's role is to reject any communication coming from a different origin but also to ensure that a malicious.site can't embed a static file on the same origin as the Postviewer v3 app (since it's effectively also part of shim origin).

After the origin check, the shimIframe calculates the hash from the received fileBody and salt and compares it to the hash stored in the hostname. If it matches it will redirect itself to a blob document created from the fileBody and mimeType .

Salt is used to randomize the origin, it's explained in the next section.

EvaluatorHtml

All files stored in a local database are rendered by the same loader called evaluatorHtml . This is basically another shim iframe which purpose is to evaluate untrusted code.

First, the Postviewer app renders evaluatorHtml with salt set to location.href . The choice of salt is to pin the evaluator's origin to the rendered file, whose sha1(filename) is present in the URL fragment - file-<sha1(filename)> . Then it sends a small JS snippet (together with a file to render) which inserts the file as a blob iframe.

evaluatorHtml :

< html > < head > < meta charset =" utf-8 " > < title > Evaluator </ title > < script > onmessage = e => { if ( e . source !== parent ) { throw / n o t p a r e n t / ; } ; if ( e . data . eval ) { eval ( e . data . eval ) ; } } onload = ( ) => { parent . postMessage ( 'loader ready' , '*' ) ; } </ script > < style > body { padding : 0 px ; margin : 0 px ; } iframe { width : 100 vw ; height : 100 vh ; border : 0 ; } . spinner { background : url (https://storage.googleapis.com/gctf-postviewer/spinner.svg) center no-repeat; } . spinner iframe { opacity : 0.2 } </ style > </ head > < body > < div id =" container " class =" spinner " > </ div > </ body > </ html >

Unsafe hashing

As a careful reader could potentially already spot, the hashing function is unsafe. For two reasons:

It concatantes strings without a delimiter. A dynamic part ( salt ) that can be controlled by an attacker is at the end.

Let's follow a simple example to illustrate the issue in which different files will result in the same hash and hence with the same shim origin.

sha256 ( "fileBody" + "product" + "origin" + "abcdef" ) === sha256 ( "fileBodyproductorigin" + "" + "abcdef" + "" )

The intended solution was to notice that the evaluatorHtml can be split on the https://storage.googleapis.com string. Then, a collision would be possible with the following values:

body == evaluatorHtml . split ( 'https://storage.googleapis.com' ) [ 0 ] product == '' origin == 'https://storage.googleapis.com' salt == evaluatorHtml . split ( 'https://storage.googleapis.com' ) [ 1 ] + 'postviewer' + 'https://postviewer3-web.2024.ctfcompetition.com/' + 'https://postviewer3-web.2024.ctfcompetition.com/#aaaaaaaaaaa'

Pathname must follow the following regex, where the capturing group is the product : /[/]([a-z0-9_-]*)[/]shim.html/ . It's possible to render as an empty product at https://postviewer3-web.2024.ctfcompetition.com/a//shim.html .

Everyone can host their files at storage.googleapis.com by simply uploading some public files to Cloud Storage. It requires adding billing information though which players don't like. Alternative way is to find an XSS there, and that's what I did in a couple of minutes https://storage.googleapis.com/vrview/2.0/index.html?image=<style/onload=alert()>

This was the core idea of the challenge but unfortunately by wanting to introduce a race-condition part and having an unpredictable flag filename, I introduced an easier unintended solution. Players could achieve the collision by forcing the application to set a custom salt (intended), fully controlled (unintended), which can be used smuggle the origin of player exploits quite easily. The idea is very clever too! Don't miss out on other players writeups.

Since the admin's file has an unpredictable name players had to either leak the name somehow or influence it in order to calculate the collistion hash. The former shouldn't be possible, and the latter could be done with some race-condition.

The postviewer application has a support for previewing a file via numeric value after whihc it will replace it with the file hash since the order of files might change. E.g. #0 might become #file-87ebbc317d687eeff47403603cc6dfb9b7d6c817 and only the latter value would be used in salt . Players could dynamically change the hash of the postviewer app so that they will smuggle their known string in the following flow:

setTimeout ( ( ) => { winRef . location = "https://postviewer3-web.2024.ctfcompetition.com/#0" } , 100 ) setTimeout ( ( ) => { winRef . location = "https://postviewer3-web.2024.ctfcompetition.com/#aaaaaaaaaaa" } , 101 )

and with a little bit of luck they will win the race and #aaaaaaaaaaa will be used in salt instead of the unpredictable #file-87ebbc317d687eeff47403603cc6dfb9b7d6c817 .

Exploit

After winning the race and hosting the exploit at storage.googleapis.com the players could access the shimIframe (because it's same-origin), read the inner iframe blob src, fetch it and read the flag.

You can check out the full postviewer-exploit.html , which is also thoroughly commented.

Closing thoughts

There is always a little dissatification when your challenge can be solved in an easier way than intended, but I think it didn't make the challenge that much less interesting. After all, it was based on a real bug which could be exploited in two different ways, I just missed the other attack scenario. Funnily enough, the unintended vulnerability would have a similar impact on our products as the intended one if the bug had not been fixed.
