# round-the-bases

**CTF:** redpwn 2021  
**Category:** crypto  
**Original writeup:** <https://blog.pipeframe.xyz/post/redpwn2021> — by **blog.pipeframe.xyz**. All credit goes to the original author; this is an archived copy.

---

crypto/round-the-bases

This crypto challenge uses a text file with some hidden information. If you open up the file in a text editor, and adjust your window width, you’ll eventually see the repeating pattern line up. This makes it very easy to see what part of the pattern is actually changing:

----------------------xxxx---- [9km7D9mTfc:..Zt9mTZ_:K0o09mTN [9km7D9mTfc:..Zt9mTZ_:K0o09mTN [9km7D9mTfc:..Zt9mTZ_:IIcu9mTN [9km7D9mTfc:..Zt9mTZ_:IIcu9mTN [9km7D9mTfc:..Zt9mTZ_:K0o09mTN [9km7D9mTfc:..Zt9mTZ_:K0o09mTN [9km7D9mTfc:..Zt9mTZ_:IIcu9mTN [9km7D9mTfc:..Zt9mTZ_:IIcu9mTN [9km7D9mTfc:..Zt9mTZ_:K0o09mTN [9km7D9mTfc:..Zt9mTZ_:K0o09mTN [9km7D9mTfc:..Zt9mTZ_:IIcu9mTN [9km7D9mTfc:..Zt9mTZ_:K0o09mTN [9km7D9mTfc:..Zt9mTZ_:K0o09mTN [9km7D9mTfc:..Zt9mTZ_:IIcu9mTN [9km7D9mTfc:..Zt9mTZ_:IIcu9mTN

I wrote a simple python script to parse this into binary data, and it worked on the first try:

# read the file into a string file = open ( " ./round-the-bases " ) content = file . read () file . close () # split on every 30th character into a list n = 30 arr = [ content [ i : i + n ] for i in range ( 0 , len ( content ), n ) ] bin = [] for line in arr : sub = line [ 16 : 20 ] # the part that changes if sub == ' IIcu ' : # IIcu -> 0x0 bin . append ( ' 0 ' ) else : # K0o0 -> 0x1 bin . append ( ' 1 ' ) bin = '' . join ( bin ) # join all the list indices together into a string # decode the binary string into ascii characters for i in range ( 0 , len ( bin ), 8 ): print ( chr ( int ( bin [ i : i + 8 ], 2 )), end = '' ) # newline for good measure print ( "

" , end = '' )
