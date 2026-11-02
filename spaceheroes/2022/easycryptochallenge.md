# EasyCryptoChallenge

**CTF:** SpaceHeroes 2022  
**Category:** crypto  
**Original writeup:** <https://github.com/not1cyyy/CTF-Writeups/wiki/Space-Heroes-CTF-:-Easy-Crypto-Challenge> — by **not1cyyy**. All credit goes to the original author; this is an archived copy.

---

<h1 class="flex-auto min-width-0 mb-2 mb-md-0 mr-0 mr-md-2 gh-header-title">Space Heroes CTF : Easy Crypto Challenge</h1>

<div class="mt-0 mt-lg-1 flex-shrink-0 gh-header-actions">

Jump to bottom

</div>

</div>

<div class="mt-2 mt-md-1 pb-3 gh-header-meta">

Firas Chaib edited this page <relative-time datetime="2022-04-15T01:12:43Z" class="no-wrap">Apr 15, 2022</relative-time>

·



5 revisions



</div>

<div id="wiki-content" class="mt-4">

<div data-view-component="true" class="Layout Layout--flowRow-until-md Layout--sidebarPosition-end Layout--sidebarPosition-flowRow-end">



<div data-view-component="true" class="Layout-sidebar"> <div class="wiki-rightbar">

<div id="wiki-pages-box" class="mb-4 wiki-pages-box js-wiki-pages-box" role="navigation">



<div class="Box Box--condensed color-shadow-small">

<div class="Box-header js-wiki-toggle-collapse" style="cursor: pointer">

<h3 class="Box-title">

<svg aria-hidden="true" height="16" viewBox="0 0 16 16" version="1.1" width="16" data-view-component="true" class="octicon octicon-triangle-down js-wiki-sidebar-toggle-display">

<path d="m4.427 7.427 3.396 3.396a.25.25 0 0 0 .354 0l3.396-3.396A.25.25 0 0 0 11.396 7H4.604a.25.25 0 0 0-.177.427Z"></path>

</svg>

<svg aria-hidden="true" height="16" viewBox="0 0 16 16" version="1.1" width="16" data-view-component="true" class="octicon octicon-triangle-right js-wiki-sidebar-toggle-display d-none">

<path d="m6.427 4.427 3.396 3.396a.25.25 0 0 1 0 .354l-3.396 3.396A.25.25 0 0 1 6 11.396V4.604a.25.25 0 0 1 .427-.177Z"></path>

</svg>

Pages <span>14</span>

</h3>

</div>

<div class=" js-wiki-sidebar-toggle-display">

<div class="filter-bar">

<input type="text" id="wiki-pages-filter" class="form-control input-sm input-block js-filterable-field" placeholder="Find a page…" aria-label="Find a page…">

</div>

</div>

<h5 class="mt-0 mb-2">Clone this wiki locally</h5>

<div class="width-full input-group">

<input

id="wiki-clone-url"

type="text"

data-autoselect

class="form-control input-sm text-small color-fg-muted input-monospace"

aria-label="Clone URL for this wiki"

value="https://github.com/not1cyyy/CTF-Writeups.wiki.git"

readonly>

<span>

<clipboard-copy for="wiki-clone-url" aria-label="Copy to clipboard" data-view-component="true" class="btn btn-sm zeroclipboard-button">

<svg aria-hidden="true" height="16" viewBox="0 0 16 16" version="1.1" width="16" data-view-component="true" class="octicon octicon-copy">

<path d="M0 6.75C0 5.784.784 5 1.75 5h1.5a.75.75 0 0 1 0 1.5h-1.5a.25.25 0 0 0-.25.25v7.5c0 .138.112.25.25.25h7.5a.25.25 0 0 0 .25-.25v-1.5a.75.75 0 0 1 1.5 0v1.5A1.75 1.75 0 0 1 9.25 16h-7.5A1.75 1.75 0 0 1 0 14.25Z"></path><path d="M5 1.75C5 .784 5.784 0 6.75 0h7.5C15.216 0 16 .784 16 1.75v7.5A1.75 1.75 0 0 1 14.25 11h-7.5A1.75 1.75 0 0 1 5 9.25Zm1.75-.25a.25.25 0 0 0-.25.25v7.5c0 .138.112.25.25.25h7.5a.25.25 0 0 0 .25-.25v-7.5a.25.25 0 0 0-.25-.25Z"></path>

</svg>

<svg style="display: none;" aria-hidden="true" height="16" viewBox="0 0 16 16" version="1.1" width="16" data-view-component="true" class="octicon octicon-check color-fg-success">

<path d="M13.78 4.22a.75.75 0 0 1 0 1.06l-7.25 7.25a.75.75 0 0 1-1.06 0L2.22 9.28a.751.751 0 0 1 .018-1.042.751.751 0 0 1 1.042-.018L6 10.94l6.72-6.72a.75.75 0 0 1 1.06 0Z"></path>

</svg>

</clipboard-copy>

</span>

</div>

</div>

</div>

<div data-view-component="true" class="Layout-main"> <div id="wiki-body" class="gollum-markdown-content">

<div class="markdown-body">

<h1><svg class="octicon octicon-link" viewBox="0 0 16 16" version="1.1" width="16" height="16" aria-hidden="true"><path d="m7.775 3.275 1.25-1.25a3.5 3.5 0 1 1 4.95 4.95l-2.5 2.5a3.5 3.5 0 0 1-4.95 0 .751.751 0 0 1 .018-1.042.751.751 0 0 1 1.042-.018 1.998 1.998 0 0 0 2.83 0l2.5-2.5a2.002 2.002 0 0 0-2.83-2.83l-1.25 1.25a.751.751 0 0 1-1.042-.018.751.751 0 0 1-.018-1.042Zm-4.69 9.64a1.998 1.998 0 0 0 2.83 0l1.25-1.25a.751.751 0 0 1 1.042.018.751.751 0 0 1 .018 1.042l-1.25 1.25a3.5 3.5 0 1 1-4.95-4.95l2.5-2.5a3.5 3.5 0 0 1 4.95 0 .751.751 0 0 1-.018 1.042.751.751 0 0 1-1.042.018 1.998 1.998 0 0 0-2.83 0l-2.5 2.5a1.998 1.998 0 0 0 0 2.83Z"></path></svg>Description</h1>



My slingshotter cousin Maneo just discovered a new cryptography scheme and has been raving about it since. I was trying to tell him the importance of setting a large, random private key, but he wouldn't listen. Guess security isn't as important as how many thousands of credits he can win in his next race around the system.

I've recovered a message sent to him detailing the finish line. Can you decrypt the message to find the coordinates so I can beat him there? I'll give you a percentage of the score!

Enter flag as shctf{x_y}, where x & y are the coordinates of the decrypted point.

Hint: You may wish to consult the great Sage of Crypto for help on this challenge.

ecc.txt



"large ... private key" - this means some sort of brute force might be involved :)



opening the file ecc.txt we can quickly see coordinates which indicates that it's an Elliptic Curve Cryptography challenge and that's exactly what the file's name stands for !



Now this challenge is pretty much a straight forward elliptic curve cryptography, using our great friend sagemath we define what we are given above as shown below :

b = 1296618846080155687

p = 11648516937377897327

E = EllipticCurve(GF(p), [a,b])

G = E(4612592634107804164, 6359529245154327104)

PubKey = E(9140537108692473465, 10130615023776320406)">

a = 3820149076078175358 b = 1296618846080155687 p = 11648516937377897327 E = EllipticCurve(GF(p), [a,b]) G = E(4612592634107804164, 6359529245154327104) PubKey = E(9140537108692473465, 10130615023776320406)



As we know the decypted message in the elliptic curve cryptography is just M = Cb-(d*Ca) where d is the private key and Cb,Ca being two points on our elliptic curve, we go ahead and define these latters first:

Cb = E(5414448462522866853, 5822639685215517063)">

Ca = E(7657281011886994152, 10408646581210897023) Cb = E(5414448462522866853, 5822639685215517063)



Now for the fun part ! we need to recover the private key via a discrete_log brute force attack so we set that up :

d = G.discrete_log(PubKey)

Defining our message is just the mathematical expression :

M = Cb-(d*Ca)

Perfect ! now our decrypted message is ready so let's go ahead and print it out :

print(M)

looks like we get some coordinates : (8042846929834025144 : 11238981380437369357 : 1)

and since the flag format is shctf{x_y} we officially found the flag !

shctf{8042846929834025144_11238981380437369357}
