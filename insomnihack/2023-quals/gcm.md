# GCM

**CTF:** insomnihack 2023 Quals  
**Category:** crypto  
**Original writeup:** <https://sylvainpelissier.gitlab.io/posts/2023-01-12-insomnihackteaser23-gcm/> — by **sylvainpelissier.gitlab.io**. All credit goes to the original author; this is an archived copy.

---

This is the solution of the challenge “GCM” given during the Insomni’hack teaser 2023. It was an example of nonce re-use in AES GCM computation.

Description

You came across a custom GCM implementation. You are also given ciphertexts (iv_j, c_j, t_j), where iv is the IV, c the ciphertext, and t the tag. You also know the corresponding messages. Everything is given in base64.

Your goal is to create a valid ciphertext (ivChall, cChall, tChall) of the message “IBr0keGCMD1dntI?At1EastIcanAuth3nT1C413Wh41IWant”. The flag should be “INS{” + MD5(ivChall + cChall + tChall) + “}” A function computing the flag given the ciphertext is given for convenience.
