# Needle_in_the_Wifi_Stack

**CTF:** BuckeyeCTF 2023  
**Category:** misc  
**Original writeup:** <https://meashiri.github.io/ctf-writeups/posts/202309-buckeyectf/> — by **meashiri**. All credit goes to the original author; this is an archived copy.

---

Needle in the Wifi Stack¶

We are given a network capture file. The protocol statistics show that it contains about 100 thousand frames of WLAN SSID broadcasts.

% tshark -r frames.pcap -z io,phs =================================================================== Protocol Hierarchy Statistics Filter: radiotap frames:101000 bytes:11316000 wlan_radio frames:101000 bytes:11316000 wlan frames:101000 bytes:11316000 wlan.mgt frames:101000 bytes:11316000 ===================================================================

The SSID field of frame seems to contain a base64 string. So, the conjecture is that the flag is hidden in one of the base64 SSID strings. The following bash pipeline automates searching for this needle in the haystack.

% tshark -r frames.pcap -T fields -e wlan.ssid | xxd -p -r | base64 -d | grep bctf bctf { tw0_po1nt_4_g33_c0ng3s7i0n } bctf { tw0_po1nt_4_g33_c0ng3s7i0n } ...
