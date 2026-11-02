# Mine_The_Gap

**CTF:** GoogleCTF 2023  
**Category:** misc  
**Original writeup:** <https://ctf.o-for.net/mine-the-gap/> — by **ctf.o-for.net**. All credit goes to the original author; this is an archived copy.

---

Mine the gap

You are given a script minesweeper.py , and a text file, gameboard.txt . Invoking the python script requires pygame to be installed.

pip install pygame

It takes several seconds to load. After loading, we get a minesweeper game

Inspect the script and search for CTF / FLAG etc.

We see this part of the code:

if len(violations) == 0: bits = [] for x in range(GRID_WIDTH): bit = 1 if validate_grid[23][x].state in [10, 11] else 0 bits.append(bit) flag = hashlib.sha256(bytes(bits)).hexdigest() print(f'Flag: CTF{{{flag}}}') else: print(violations)

We need to solve it, and then we can reconstruct the flag from the solution.

Inspect gameboard.txt -- it looks like the board is in a simple text format.

The board looks pretty structured. Putting one mine will collapse many other cells, but not all.

❯ wc gameboard.txt 1631 198991 5876831 gameboard.txt

The board is 1600 x 3600 cells. It is huge. It is not possible to solve it by hand.

We need to solve the board with code.

Idea 1 use backtracking and pray to be fast enough.

Idea 2 skip backtracking and use SAT solver (Z3). This is what we did.

With Z3, we can create variables and constraints on the values they can get, then ask for a solution. If there is a solution, Z3 will give us the values for the variables. Z3 will find an answer in a reasonable™️ time.

Check the code to generate the solution. With the answer, we can easily recover the flag using the game's code.
