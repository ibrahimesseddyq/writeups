# Change_VM

**CTF:** justCTF 2019  
**Category:** rev  
**Original writeup:** <https://github.com/Pusty/writeups/tree/master/justCTF2019#change_vm> — by **Pusty**. All credit goes to the original author; this is an archived copy.

---

The challenge consists out of change_my_mind, a 64bit Linux binary which requires a password which it confirms or denies after running calculations within a custom VM.

## Solution

At first I analysed the binary and tried to run angr on it with more specific constraints, as it turns out a very generic angr approach works as well:

```python

import angr

import claripy

p = angr.Project("change_my_mind")

symsize = claripy.BVS('inputLength', 64) # Make length of input symbolic as it has to be 41 characters

simfile = angr.SimFile('/tmp/stdin',size=symsize)

state = p.factory.entry_state(stdin=simfile)

simgr = p.factory.simulation_manager(state)

simgr.explore(find=lambda s: b"Good" in s.posix.dumps(1)) # Search the path that leads to "Good password"

f = simgr.found[0]

print(f)

print((b"STDOUT: "+f.posix.dumps(1)).decode())

print((b"FLAG: "+f.posix.dumps(0)).decode()) # print input for the correct path, which is the flag

```

Running the script returns the correct input:

```bash

STDOUT: Hello!

Please provide your credentials!

>>Lets'check...

Good password. Congratulations!

FLAG: justCTF{1_ch4ng3d_my_m1nd_by_cl34r1n6_7h3_r00m}
