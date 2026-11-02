# cookie clicker

**CTF:** UTCTF 2024  
**Category:** web  
**Original writeup:** <https://ctftime.org/writeup/38982> — by **ctftime.org**. All credit goes to the original author; this is an archived copy.

---

We are given a page where when we click the cookie the counter goes up

![Webpage]

If we take a look at the source code the event listener code for the `DOMContentLoaded` event is what we are looking for.

```html

<script>

document.addEventListener('DOMContentLoaded', function() {

var count = parseInt(localStorage.getItem('count')) || 0;

var cookieImage = document.getElementById('cookieImage');

var display = document.getElementById('clickCount');

display.textContent = count;

cookieImage.addEventListener('click', function() {

count++;

display.textContent = count;

localStorage.setItem('count', count);

if (count >= 10000000) {

fetch('/click', {

method: 'POST',

headers: {

'Content-Type': 'application/x-www-form-urlencoded'

},

body: 'count=' + count

})

.then(response => response.json())

.then(data => {

alert(data.flag);

});

}

});

});

</script>

```

There a couple different ways to solve this. The simplest way would be to just make a POST request to `http://betta.utctf.live:8138/click` with the body of `count=10000001` and read the flag in the response.

```bash

┌──(kali㉿kali)-[~]

└─$ curl -d "count=10000001" http://betta.utctf.live:8138/click

{"flag":"Wow, you beat me. Congrats! utflag{y0u_cl1ck_pr3tty_f4st}"}

```

FLAG: `utflag{y0u_cl1ck_pr3tty_f4st}`
