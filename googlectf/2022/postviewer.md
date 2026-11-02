# Postviewer

**CTF:** GoogleCTF 2022  
**Category:** web  
**Original writeup:** <https://gist.github.com/terjanq/7c1a71b83db5e02253c218765f96a710> — by **terjanq**. All credit goes to the original author; this is an archived copy.

---

The rumor tells that adm1n stores their secret split into multiple documents. Can you catch 'em all? https://postviewer-web.2022.ctfcompetition.com

The challenge consisted of an all client-side simple page, i.e. no backend code was involved. A user can upload any file which will be then locally stored in indexedDB. They can preview their files by either clicking on the title or by visiting file's URL, for example https://postviewer-web.2022.ctfcompetition.com/#file-01d6039e3e157ebcbbf6b2f7cb2dc678f3b9214d. The preview of the file is rendered inside a blob created from data: URL. The rendering occurs by sending file's contents to the iframe via postMessage({ body, mimeType }, '*')

Additionally, there is a /bot endpoint which lets players send URLs to an xss-bot imitating another user. The goal is to steal their documents.

The idea

There were only two intentional vulnerabilities left in the code:

User controlled CSS selector passed to querySelector via location.hash ( const fileDiv = document.querySelector(location.hash); ) Unsafe data transfer to an iframe ( iframe.contentWindow?.postMessage({ body, mimeType }, '*'); )

The first vulnerability allows another site to display any file via #a,.list-group-item:nth-child(${n}) and another to intercept the message and steal admin's files.

The idea for the challenge comes from an almost-real bug discovered internally.

Intercepting a message

File preview

Every file goes through safe-frame.js script which has the following flow:

Create an iframe with URL data:text/html,<something> . The iframe redirects itself to a blob URL that registers onmessage event. After the iframe loads, the parent site sends the file's contents to it, but only once. When the iframe receives an onmessage event it creates a new blob URL from the message data and redirects itself to that URL.

In step 2. the iframe is put into a separate process because blob documents created from null origins are isolated for security benefits. This information will be helpful in a later stage of the solution.

The practice of using isolated documents is intended to protect against SPECTRE-alike attacks as all documents can attack other documents when in the same process.

Race condition

It's relatively easy to make the website render an attacker-controlled file inside an iframe from another page, let's call it evil.com . All the malicious website, evil.com , needs to do is to send win[0].postMessage({body:exploit, mimeType:'text/html'}, '*') multiple times ( win is a reference pointing to the challange's website).

The issue is that to intercept the message, evil.com needs to render a malicious site before the flag is sent. Even if they manage to do it, the flag is sent almost instantly which is problematic.

Slowing down the parent

As we mentioned earlier, the iframe is process-isolated from its parent. The implication of this is that if the parent process gets busy and becomes unresponsive the iframe will work fine because it's not affected as it is executed by a different process. With that fact, evil.com wants to somehow make the challenge website busy but in the meantime render malicious documents inside the iframe.

It probably can be done in many ways but the technique I used is to send a big chunk of data to the website and make it convert it to string which will take time. I left a simple gadget in the challenge that will do it:

if ( e . data == 'blob loaded' ) { $ ( "#previewModal" ) . modal ( ) ; }

Because of loose comparison, if e.data is not a string it will be converted to it. In the snippet below, I did exactly that and transferred the whole object in the most efficient way possible with a transferable object.

const buffer = new Uint8Array ( 1e7 ) ; //... win ?. postMessage ( buffer , '*' , [ buffer . buffer ] ) ;

Winning the race

After the iframe is inserted to the challenge page it takes around ~4ms for the document to be sent. That is because the onload will only trigger after the document finished loading all subresources and parsed the whole document. It's not a lot of time but enough to solve the challenge.

Open a new window ( win ) pointing to the challenge website. In an infinite loop, check if win.length === 1 and if so run the attack and stop the loop. Slow down the win by sending a huge message. Wait 500ms (this might need tweaking depending on PC performance) and send the exploit. Intercept the message.

The complete solution

By combining the two mentioned vulnerabilities, players were expected to steal 3 admin's files in one run and each file looked like the following:

Congratulations on fetching admin's file! The flag needs to be deciphered with a password that has been split into three random files. Because the password is random with each run, you will have to collect all three files. When you do so, just visit: https://postviewer-web.2022.ctfcompetition.com/dec1pher File info: Cipher: ${flag_cipher} Password part [${i}/3]: ${password} The challenge is easily solvable under 5 seconds, but as a token of appreciation I set up a secret endpoint for you that have a limit of 20 seconds: https://postviewer-web.2022.ctfcompetition.com/bot?s=s333cret_b00t_3ndop1nt

The requirement of stealing all three files was to prevent solutions that were able to only leak one file with multiple attempts. I included my exploit at the bottom of the writeup.

Closing thoughts

Process isolation is a security enhancement but can introduce race-condition bugs as showcased in this challenge. Developers must be careful more than ever when writing unsafe JavaScript code.
