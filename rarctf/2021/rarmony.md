# rarmony

**CTF:** RaRCTF 2021  
**Category:** pwn  
**Original writeup:** <https://yysecurity.tistory.com/439?category=924819> — by **yysecurity.tistory.com**. All credit goes to the original author; this is an archived copy.

---

🧡 RaRMony (500 points)

이 문제는 함수가 너무 많아서 취약점이 발생하는 부분만 설명하도록 하겠다. 일단, init 함수를 통해 힙이 생성된다. 힙의 구조는 다음과 같다. 이게 왜 500점?

This problem has so many functions that I will try to explain only the areas where vulnerabilities occur. First, the init function produces a heap. The structure of the heap is as follows.

취약점은 update_username 함수에서 발생한다. username의 size는 최대 32칸 까지만 입력해야 하지만, 40칸까지 입력이 가능해 update_username을 가리키는 포인터를 덮어쓸 수 있다. 해당 부분을 set_role 함수로 바꿔 권한을 상승시키고, 2번 채널을 읽으면 flag를 획득할 수 있다.

The vulnerability exists in the function update_username. The size of username should be up to 32 spaces, but it can be inputted up to 40 spaces to overwrite the pointer pointing to update_username. Changing the part to the set_role function to raise the authority, and read channel 2 to get a flag.
