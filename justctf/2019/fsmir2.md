# FSMIR2

**CTF:** justCTF 2019  
**Category:** rev  
**Original writeup:** <https://github.com/Pusty/writeups/tree/master/justCTF2019> — by **Pusty**. All credit goes to the original author; this is an archived copy.

---

The challenge consists out of a SystemVerilog file which depending on the current state and the input `di` advance to the next state and outputs a signal if it reached the solution state.

## Solution

By iterating over the file and filtering out the conditions for state transfers I created a map from the new states to the previous states and required `di`'s (which construct the flag).

```python

f = open("fsmir2.sv")

lines = f.read().split("

")

entries = {}

for line in lines:

line = line.strip()

if "case(di)" in line: # filter for the require previous state

last_cond = int(line.split(" : ")[0].split("'b")[1], 2) # extract the previous state as a number

elif "default: c <= 9'b0;" in line: pass # ignore the default case as it's just a fallback

elif ": c <= " in line: # filter for the conditional state changes

check_di = int(line.split(": c <=")[0].split("'b")[1], 2) # extract the di value required for the state transfer

new_value = int(line.split("9'b")[1].split(";")[0], 2) # extract the new state value as a number

if not new_value in entries:

entries[new_value] = []

entries[new_value].append((last_cond, check_di)) # add the state change to the "new state" -> (previous state, di) map

```

Starting from the solved state `assign solved = c == 9'b101001101;` I traversed the map backwards and reconstructed the `di` values:

```python

last_value = int("101001101", 2) # the solution state

before = last_value

flag = []

while before != 0: # if we reached the 0 state we are at a dead end

l = entries[before] # extract the list for a given new state

before, flag_part = l[0] # extract the previous state and di/flag part value, set the previous state to the new next state

flag.append(chr(flag_part)) # append the character to the flag

flag.reverse() # reverse the flag as we traversed the map backwards

```

This reveals the flag:

```python

>>> print(''.join(flag))

justCTF{I_h0p3_y0u_us3d_v3r1L4t0r_0r_sth...}
