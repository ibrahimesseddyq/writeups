# math_test

**CTF:** LexingtonInformaticsTournament 2022  
**Category:** rev  
**Original writeup:** <https://ctftime.org/writeup/34656> — by **ctftime.org**. All credit goes to the original author; this is an archived copy.

---

У нас есть программа которая выводит строку `Добро пожаловать на тест по математике. Если вы получите идеальный результат, я распечатаю флаг!`.

# Analyzing IDA PRO

В `main()` задается вопрос и просит ввести результат:

```int __cdecl main(int argc, const char **argv, const char **envp)

{

int i; // [rsp+Ch] [rbp-4h]

puts("Welcome to the math test. If you get a perfect score, I will print the flag!");

puts("All questions will have non-negative integer answers.

");

for ( i = 0; i < NUM_Q; ++i )

{

printf("Question #%d: ", (unsigned int)(i + 1));

puts((&questions)[i]);

__isoc99_scanf("%llu", &submitted[i]);

}

return grade_test();

}

```

Например:

![image](https://user-images.githubusercontent.com/55994705/180990331-af35757f-7d61-4a46-a899-51db647829f1.png)

Посмотрим на функцию `grade_test()`

```int grade_test()

{

int i; // [rsp+8h] [rbp-8h]

unsigned int v2; // [rsp+Ch] [rbp-4h]

v2 = 0;

for ( i = 0; i < NUM_Q; ++i )

{

if ( submitted[i] == answers[i] )

++v2;

}

printf("You got %d out of 10 right!

", v2);

if ( v2 != 10 )

return puts("If you get a 10 out of 10, I will give you the flag!");

puts("Wow! That's a perfect score!");

puts("Here's the flag:");

return generate_flag();

}

```

Ага, есть массив `answers[i]`, с которым сравнивается наш ввод.

![image](https://user-images.githubusercontent.com/55994705/180990827-669303a8-52c3-4748-9994-ff72d1d658da.png)

# Solution

Введем такие же числа и получим

flag is: `LITCTF{y0u_must_b3_gr8_@_m4th_i_th0ught_th4t_t3st_was_imp0ss1bl3!}`
