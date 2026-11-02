# Just_Not_My_Type

**CTF:** KillerQueen 2021  
**Category:** web  
**Original writeup:** <https://github.com/0xRar/KillerQueenCTF-2021/blob/main/JustNotMyType.md> — by **0xRar**. All credit goes to the original author; this is an archived copy.

---

- ## Challenge Information:

| Name | Category | Difficulty | Points | Dev |

|------------------|----------|------------|--------|------------|

| Just Not My Type | Web | Easy | 248 |[ZeroDayTea]|

----

# Description:

I really don't think we're compatible

----

# Solution:

First thing i thought it was an sqli, but then i remembered they already gave us the source code

for the challenge.

### Source-Code:

```php

<h1>I just don't think we're compatible</h1>

<form method="POST">

Password

<input type="password" name="password">

<input type="submit">

</form>

```

The twist of the challenge is first we didn't have any link to the webapp, at first

so the `$FLAG` variable is just a fake flag, so i look and nothing really wrong with

the code but maybe the function `strcasecmp()` has some kind of vulnerability

or not used in secure way, after googling a bit and reading the strcasecmp

php [Documentation](https://www.php.net/manual/en/function.strcasecmp.php)

Turns out that `strcasecmp()` is a single-byte function , after searching what that

means and how to exploit it found that if you don't use it in a secure way it can lead

to **Authentication Bypass** , the idea is to turn the password param into an empty array and the value to %22%22

Example: `http://vulntarget.com/type.php?password[]=%22%22`

and that gave me the flag :)

![Pasted image 20211030045525](https://user-images.githubusercontent.com/33517160/139555131-39686fe2-8548-404a-a845-9aa5e97af02b.png)

## Flag: **`flag{no_way!_i_took_the_flag_out_of_the_source_before_giving_it_to_you_how_is_this_possible}`**
