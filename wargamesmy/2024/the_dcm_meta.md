# The_DCM_Meta

**CTF:** Wargames.MY 2024  
**Category:** misc  
**Original writeup:** <https://medium.com/@T0pN0xch/wargamesmy-ctf-2024-writeup-7960f7de58a3> — by **T0pN0xch**. All credit goes to the original author; this is an archived copy.

---

The DCM Meta — 50pts (57 solves) Easy

“[25, 10, 0, 3, 17, 19, 23, 27, 4, 13, 20, 8, 24, 21, 31, 15, 7, 29, 6, 1, 9, 30, 22, 5, 28, 18, 26, 11, 2, 14, 16, 12]” Hint : The element is the number in the list, combine for the flag. Wrap in wgmy{}

We were provided with a file named challenge.dcm , which contained a series of random numbers resembling the format of a flag,

WGMY f 6 3 a c d 3 b 7 8

1

2 7

c 1 d 7 d 3 e 7 0 0 b 5 5 6 6 5 3 5 4

write a python code to decode it

# Given list of indices

indices = [25, 10, 0, 3, 17, 19, 23, 27, 4, 13, 20, 8, 24, 21, 31, 15, 7, 29, 6, 1, 9, 30, 22, 5, 28, 18, 26, 11, 2, 14, 16, 12]



# Given DCM file content as a list of characters

dcm_content = ['f', '6', '3', 'a', 'c', 'd', '3', 'b', '7', '8', '1', '2', '7', 'c', '1','d', '7', 'd', '3', 'e', '7', '0', '0', 'b', '5', '5', '6', '6', '5', '3', '5', '4']



# Function to generate the flag

def generate_flag(indices, dcm_content):

# Check if indices are valid and within the range of dcm_content

flag_characters = []

for i in indices:

if i < len(dcm_content): # Ensure the index is within range

flag_characters.append(dcm_content[i])

else:

print(f"Warning: Index {i} is out of range for dcm_content.")



# Join the characters and wrap in 'wgmy{}'

flag = "wgmy{" + ''.join(flag_characters) + "}"



return flag



# Generate the flag

flag = generate_flag(indices, dcm_content)

print("The flag is:", flag)

This code extracts specific characters from the dcm_content list using the indices list and combines them to generate a flag

Function generate_flag :

Iterates over each index in indices .

. Checks if the index is valid (within the range of dcm_content ).

). Extracts the corresponding character from dcm_content and adds it to a list ( flag_characters ).

Flag

wgmy{51fadeb6cc77504db336850d53623177}
