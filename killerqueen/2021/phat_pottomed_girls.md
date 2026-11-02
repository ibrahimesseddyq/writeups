# PHat_Pottomed_Girls

**CTF:** KillerQueen 2021  
**Category:** web  
**Original writeup:** <https://github.com/kimmypracha/CTF-Writeups/blob/main/KillerQueen2021/PhatPottomedGirls/writeup.md> — by **kimmypracha**. All credit goes to the original author; this is an archived copy.

---

This challenge gives us a website of note memo that you can create a note as much as you want and it'll store on the website.

The challenge also gives us the backend source code as the following.

## Source code

```php

", "", $notetoadd);

$notetoadd = str_replace("

```

## Analysis

From looking at the source code, the input will be sanitized from a group of reserved words for three times. Which means, if there is an input that need to be sanitized for four times, this sanitization wouldn't work. So we'll exploit from here.

For example :

`'<<<>>

```

which will be sanitized to

```php

```

which will display the current directory of the website, that is

```

/var/www/html/

```

To go to root directory `/` we have to go to parent directory 3 times (parent of html and www and var respectively.) Therefore, if there is any a.txt in the root path, we can access them in the following path

```

../../../a.txt

```

Since I'm not sure about the name and type of the flag file, so I decide to scan in that directory first, by injecting the following input.

```php

<<<'; print_r(scandir('../../../')); echo '';>>>>

```

Note : `scandir($path)` will return the list of the file or directory in `$path`.

So we find the `flag.php` in that directory, then, here is the final input we will use to display the `flag.php` file.

```php

<<<>>>

```

and we finally got the flag.

flag :

```

flag{wait_but_i_fixed_it_after_my_last_two_blunders_i_even_filtered_three_times_:(((}
