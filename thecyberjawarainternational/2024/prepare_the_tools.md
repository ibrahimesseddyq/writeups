# prepare_the_tools

**CTF:** TheCyberJawaraInternational 2024  
**Category:** forensics  
**Original writeup:** <https://gist.github.com/Lydxn/26679096f7675abc3eb30b7fbfc07541> — by **Lydxn**. All credit goes to the original author; this is an archived copy.

---

prepare the tools (46 solves)

lets preparing our tools!

The handout contains a .pcapng file. Opening it in Wireshark and inspecting the TCP packets, we see an enormous amount of packets being sent back and forth, of the form:

flag[1161]aflag[2481] flag[0473]iflag[2590]tflag[3192]iflag[2663]oflag[0802]dflag[0687]oflag[0091]tflag[0051]Sflag[4397]oflag[2480],flag[3788]aflag[3851]nflag[4569]tflag[1432]aflag[3312] flag[3130]cflag[0913]wflag[3352]iflag[1147]dflag[1135]dflag[2369]rflag[0374]uflag[3138]sflag[1230]tflag[4227]hflag[2535]tflag[4682]sflag[1076] flag[3877]rflag[0773]nflag[4893]nflag[1647]

We notice that flag[....] is always immediately followed by an ASCII char. The pattern implies that the .... th character of the flag is simply that ASCII char. Parsing out all of these values would take forever by hand, so I wrote a quick pyshark script to do it automatically:

import pyshark cap = pyshark . FileCapture ( 'preparingtools.pcapng' ) data = [] for packet in cap : if hasattr ( packet . tcp , 'payload' ): b = bytes ( int ( c , 16 ) for c in packet . tcp . payload . split ( ':' )). decode () data . append ( b ) flag = [ '?' ] * ( len ( data ) // 2 ) for i in range ( 0 , len ( data ), 2 ): idx = int ( data [ i ][ 5 : - 1 ]) c = data [ i + 1 ] flag [ idx ] = c print ( '' . join ( flag ))

Program output:

... The next day, Severus concocted a potion that would create colorful smoke, drawing everyone’s attention. He placed it strategically near the entrance of the Great Hall during lunch. As the smoke erupted in a dazzling array of colors, the Marauders rushed to investigate, leaving Lily momentarily alone. Seizing the opportunity, Severus dashed to her side. “Lily! Please, I need to tell you about CJ{warm_up_for_your_scapy/pyshark/tshark}. It’s about how you can enhance your charms study and bypass the Marauders’ tricks!” Lily blinked in surprise, her eyes wide with curiosity. “What? That sounds fascinating! Tell me more!” ...

Flag: CJ{warm_up_for_your_scapy/pyshark/tshark}
