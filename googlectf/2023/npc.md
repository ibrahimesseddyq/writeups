# NPC

**CTF:** GoogleCTF 2023  
**Category:** misc  
**Original writeup:** <https://bronson113.github.io/2023/06/26/googlectf-2023-writeup.html> — by **bronson113**. All credit goes to the original author; this is an archived copy.

---

npc

1 2 3 4 A friend handed me this map and told me that it will lead me to the flag. It is confusing me and I don't know how to read it, can you help me out? solves: 102

Show encrypt.py 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63 64 65 66 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95 96 97 98 99 100 101 102 103 104 105 106 107 108 109 110 111 # Copyright 2023 Google LLC # # Licensed under the Apache License, Version 2.0 (the "License"); # you may not use this file except in compliance with the License. # You may obtain a copy of the License at # # http://www.apache.org/licenses/LICENSE-2.0 # # Unless required by applicable law or agreed to in writing, software # distributed under the License is distributed on an "AS IS" BASIS, # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. # See the License for the specific language governing permissions and # limitations under the License. """This file encrypts a given file using a generated, easy to remember password. Additionally, it generates a hint in case you forgot the password. """ import dataclasses import re import secrets import sys from pyrage import passphrase def get_word_list (): with open ( 'USACONST.TXT' , encoding = 'ISO8859' ) as f : text = f . read () return list ( set ( re . sub ( '[^a-z]' , ' ' , text . lower ()). split ())) def generate_password ( num_words ): word_list = get_word_list () return '' . join ( secrets . choice ( word_list ) for _ in range ( num_words )) @ dataclasses . dataclass class Node : letter : str id : int @ dataclasses . dataclass class Edge : a : Node b : Node @ dataclasses . dataclass class Graph : nodes : list [ Node ] edges : list [ Edge ] class IdGen : def __init__ ( self ): self . ids = set () def generate_id ( self ): while True : new_id = secrets . randbelow ( 1024 ** 3 ) if new_id not in self . ids : self . ids . add ( new_id ) return new_id def generate_hint ( password ): random = secrets . SystemRandom () id_gen = IdGen () graph = Graph ([],[]) for letter in password : graph . nodes . append ( Node ( letter , id_gen . generate_id ())) for a , b in zip ( graph . nodes , graph . nodes [ 1 :]): graph . edges . append ( Edge ( a , b )) for _ in range ( int ( len ( password ) ** 1.3 )): a , b = random . sample ( graph . nodes , 2 ) graph . edges . append ( Edge ( a , b )) random . shuffle ( graph . nodes ) random . shuffle ( graph . edges ) for edge in graph . edges : if random . random () % 2 : edge . a , edge . b = edge . b , edge . a return graph def write_hint ( graph , out_file ): out_file . write ( 'graph {

' ) for node in graph . nodes : out_file . write ( f ' { node . id } [label= { node . letter } ];

' ) for edge in graph . edges : out_file . write ( f ' { edge . a . id } -- { edge . b . id } ;

' ) out_file . write ( '}

' ) def encrypt ( num_words , secret ): password = generate_password ( num_words ) hint = generate_hint ( password ) with open ( 'hint.dot' , 'w' ) as hint_file : write_hint ( hint , hint_file ) filename = 'secret.age' with open ( filename , 'wb' ) as f : f . write ( passphrase . encrypt ( secret , password )) print ( f 'Your secret is now inside password-protected file { filename } .' ) print ( f 'Use the password { password } to access it.' ) print ( 'In case you forgot the password, maybe hint.dot will help your memory.' ) if __name__ == '__main__' : encrypt ( num_words = int ( sys . argv [ 1 ]), secret = sys . argv [ 2 ]. encode ( 'utf-8' ))

In this challenge, the flag file is encrypted using a passphrase that generated from a wordlist. A hint.dot file is also provided. THe most import part is to identiy how the hint is generate. In the encrypt.py file we can see that each letter is assigned a node, and each consecutive letters are link with a directly edge. Afterward, random edges are added, the node orders are shuffled, and the edge order are randomly flipped. Since there are a lot of randomness in the hint, all we can extract from the hint are what letters are in the passphrase, and what letters can be connected. I parse the hint file and get the list of letters presented. I then treat all edges as bi-directional, and filter the words based on the following criteria.

The letters in the word can be found in the passphrase There are edges between any two consecutive letter pairs

This filters the word list down to 86 words, I then randomly generates passphrase from that wordlist that have the same word sets as the passphrase. I then permutes that word sets and test all permutations until the correct passphrase is found. See solve.py for more detail.

I think that the word set phase can potentially be change to a more efficient approach, as that problem is similar to a subset sum, not sure if there are any potential in a more efficient algorithm.

CTF{S3vEn_bR1dg35_0f_K0eN1g5BeRg}
