# lsass_crackdown

**CTF:** SpookyCTF 2024  
**Category:** forensics  
**Original writeup:** <https://k3sero.github.io/posts/Lsass-Crackdown-Forense-SpookyCTF2024/> — by **k3sero**. All credit goes to the original author; this is an archived copy.

---

Lsass Crackdown - SpookyCTF2024

Autor del reto: Trent

Dificultad: Fácil

Enunciado

Anna Circoh has intercepted a highly sensitive memory dump, but The Consortium has fortified it with advanced encryption, hiding their deepest secrets within. Participants must analyze the data and navigate through layers of defenses to find a key piece of information we are thinking its a leaked password. Dr. Tom Lei has rigged the memory with decoys and traps, so tread carefully—one wrong step could lead you down a path of misdirection.

Some AV’s may detect the attachment as malicious. This is a false positive and can be ignored

Archivos

En este reto nos dan el siguiente archivo.

dump.DMP : Archivo que contiene el dumpeo de un proceso en Windows.

Archivos utilizados aquí.

Analizando el Código

En este reto básicamente tenemos un dumpeo de memoria proveniente de un equipo en Windows, concretamente se trata de un proceso LSASS (Local Security Authority) el cual se encarga de gestionar la autenticación de los usuarios, creación de tokens de acceso y su almacenamiento en caché. La clave para obtener la flag es encontrar la contraseña de DR.Tom. Para ello, tendremos que navegar por dicho dumpeo de memoria y analizar el contenido.

Solución

En este caso voy a utilizar la herramienta ypykatz la cual se encarga de descomponer dicho proceso LSASS en secciones acorde a la información que encuentra.

Para instalar la herramienta simplemente utilicé los siguientes comandos.

1 2 3 sudo apt install python3-pypykatz python3 -m venv venv pip3 install minidump minikerberos aiowinreg msldap winacl

Una vez tenemos la herramienta instalada, utilizando el siguiente comando nos arroja un informe detallado de toda la información de dicho proceso (Dumpeo completo en archivos)

1 2 ┌──(kesero㉿kali)-[~] └─$ pypykatz lsa minidump dump.DMP

Llegados a este punto, la clave para resolver el reto es filtrar de forma correcta por el hash NTLM correcto para obtener la contraseña de DR.Tom. Como tenemos un usuario que se llama igual que la corporación Consortium vamos a extraer el hash NTLM de dicho usuario.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 FILE: ======== dump.DMP ======= == LogonSession == authentication_id 6471305 (62be89) session_id 2 username Consortium domainname DESKTOP-UBFFHS2 logon_server DESKTOP-UBFFHS2 logon_time 2024-10-18T01:40:41.720992+00:00 sid S-1-5-21-996221637-1914836208-3740248221-1011 luid 6471305 == MSV == Username: Consortium Domain: DESKTOP-UBFFHS2 LM: NA NT: f6c479f4b9904f884fede1b2d4328d98 SHA1: 8e0cf85ff4c266ff4ef626580cce1ff025118c6f DPAPI: 8e0cf85ff4c266ff4ef626580cce1ff025118c6f == WDIGEST [62be89]== username Consortium domainname DESKTOP-UBFFHS2 password None password (hex) == Kerberos == Username: Consortium Domain: DESKTOP-UBFFHS2 == WDIGEST [62be89]== username Consortium domainname DESKTOP-UBFFHS2 password None password (hex) == DPAPI [62be89]== luid 6471305 key_guid cd06bac7-841e-4615-afdf-735caa9878b6 masterkey 9447b5b0b2e2ebbec0bd1298e3e411612413f39cfbb8a6019a0bd2e14be8e45d602fdd4f96d22f977161988f0d4b6090bf992bea9abb4dc64873bfedff8ed10f sha1_masterkey 1256160a5105caf7146b49f397a95efee53e4376

Una vez tenemos el hash NTLM de dicho usuario f6c479f4b9904f884fede1b2d4328d98 podemos crackearlo de forma manual o directamente usar crackstation y podemos observar que dicho hash coincide con 1987evilovekoen

Flag
