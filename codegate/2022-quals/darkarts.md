# DarkArts

**CTF:** Codegate 2022 Quals  
**Category:** crypto  
**Original writeup:** <https://mandu-mandu.tistory.com/463> — by **mandu-mandu.tistory.com**. All credit goes to the original author; this is an archived copy.

---

CRYPTO - dark-arts

챌린지 1,2,3,4를 모두 통과해야 플래그를 얻을 수 있다.

CHAL1은 GUESS_MODE()를 64번 통과해야 한다.

GENERATOR1에서 랜덤으로 func_gen 과 func_random 둘 중 하나로 고정되서 리턴한다.

gess_mode에서 x에 값을 직접 넣어서 그 결과를 알 수 있고, 그 값들을 가지고 mode가 0일지 1일지 (func_gen을 통과한 결과인지 func_random을 통괗나 결과인지)를 맞춰야 한다.

func_gen과 func_random 둘다 결과는 무조건 0 또는 1이 나온다.

func_gen은 랜덤 seed와 x의 각 bit를 내적한 값으로 66행의 식을 계산한 값이 결과가 된다.

x가 0 [0]이라면 prod가 0이 되고 66행의 계산 결과는 0이 된다.

x가 1 [1]이라면 prod는 1 또는 0이 되고, 1이라면 66행의 결과는 (1 % 2 + 1 % 3 ) %2 = 0

x가 2 [0,1]이라면 prod는 1 또는 0이 되고, 결과는 0

x가 3 [1,1]이라면 prod는 2 또는 1 또는 0이 되고 2라면 (2 % 2 + 2 % 3 ) % 2 = 0

x가 4, 5, 6, 8, 9, 10, 11 일 때에도 (bit 중 1인 bit의 개수가 3개 미만인 수들) 모두 결과는 0이 된다.

x가 7 [1,1,1] 이라면 prod가 3이 될 수도 있는데 (3 % 2 + 3 % 3) % 2 = 1이다.

따라서 x에 0,1,2,3,4,5,6,8,9,10,11 넣어서 결과가 1 나오면 무조건 mode=1

모두 0이 나오면 mode = 0 으로 판단하도록 코드를 작성하여 돌렸다.

1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 from pwn import * context.log_level = 'debug' p = remote( "13.209.188.120" , 9003 ) p.recvuntil( "Challenge 1

" ) f = 0 for _ in range ( 64 ): print (_) for i in range ( 13 ): if (i = = 7 or i = = 11 ): continue p.sendline( "0" ) p.sendline( str (i)) if ( 1 = = int (p.recv())): p.sendline( "1" ) p.sendline( "1" ) f = 1 break if (f): f = 0 continue p.sendline( "1" ) p.sendline( "0" ) p.interactive() Colored by Color Scripter cs

chal2도 guess_mode()를 64회 통과해야하는데

이제는 리턴 값이 0,1,2,3,4 총 5개 이며

func_gen에서 x를 sha256을 돌려서 사용한다.

일단 x에는 int만 들어가고 x를 0부터 1씩 증가시켜 넣어서 결과를 확인했을 때 규칙을 찾을 수 없었고

sha256(x) 의 결과들 중에서 bit중 1의 개수가 작아봐야 80-90개 사이였기 때문에

chal1처럼 풀 수 없었다.
