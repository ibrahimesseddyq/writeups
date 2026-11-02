# rsa

**CTF:** BuckeyeCTF 2024  
**Category:** crypto  
**Original writeup:** <https://medium.com/@n0r37urn/soluci%C3%B3n-de-algunos-retos-buckeyectf-2024-99003f6d3a68> — by **n0r37urn**. All credit goes to the original author; this is an archived copy.

---

crypto / rsa — 50 pnt

Solución:

Este tipo de ejercicio es común en muchos CTFs: un reto de criptografía basado en RSA. En este caso, la dificultad fue baja, ya que el valor de N se podía factorizar fácilmente utilizando la herramienta favorita para este tipo de tareas, FactorDB. Una vez factorizado N, solo fue necesario aplicar las fórmulas correspondientes y resolver el reto utilizando un código desarrollado en Python.

import Crypto.Util.number as cun



# Valores dados

p = 213055785127022839309619937270901673863

q = 310165339100312907369816767764432814137

e = 65537

n = p * q

c = 19146395818313260878394498164948015155839880044374872805448779372117637653026



# Paso 1: Calcular φ(n)

phi = (p - 1) * (q - 1)



# Paso 2: Calcular la clave privada d

d = pow(e, -1, phi)



# Paso 3: Descifrar el mensaje

m = pow(c, d, n)



# Paso 4: Convertir el mensaje descifrado a bytes

decrypted_message = m.to_bytes((m.bit_length() + 7) // 8, "big")



# Imprimir el mensaje descifrado

print(f"Decrypted message: {decrypted_message.decode()}")

bctf{f4c70r1z3_b3773r_4d3b35e4}
