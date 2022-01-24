---
title: "Nif Validation"
date: 2022-01-24T14:42:34-05:00
draft: false
toc: false
images:
tags: 
  - Portugal
---

Número De Identificação Fiscal or NIF, is the nine digit number which is used to identify a tax paying entity in Portugal. Any NIF can be validated using the [mod 11 algorithm](http://www.pgrocer.net/Cis51/mod11.html) with one twist. I will describe what that twist is and how it affects validating a NIF.

When you buy anything in Portugal, in person or online, you give your NIF. This includes everything from the 100€ you spend on your mall haul, to the 2€ you spend on an espresso and a pastel de nata.

This number is relayed often so mistakes are very possible. Possible causes are bit flips during transmission when buying online — or mispronouncing seis as sete. To address this, an algorithm exists to determine if any give series of nine digits is a valid NIF. It can be defined as:

$$ \text{let } n\_i \text{be the }i\text{th digit of the NIF as numbered from right to left. Then a NIF is valid if:}$$

$$ \sum\_{i = 1}^{8} i\cdot n\_i \equiv 0 \pmod{11}$$

Said differently, enumerate the digits from right to left, and sum each number multiplied by it's index. The result modulo 11 should be 0.

I implemented this in haskell because its a neat language I almost never use
```haskell
import Data.List
-- Turn a number into a list of digits
bd_impl a = if a == 0 
  then Nothing
  else Just ((mod a 10), (div a 10)))
bd num = unfoldr bd_impl num
-- Sum all numbers times their position
ps_mult tup = ((fst tup) * (snd tup))
ps num = foldl (+) 0 (map ps_mult (zip (bd num) [2..]))
validate nif = if ((mod (ps nif) 11) == 0)
  then putStrLn "valid"
  else putStrLn "not valid"
```

Testing our implementation against some NIFs


```
$ docker run --rm -it -v `pwd`:/app -w /app haskell ghci
GHCi, version 9.2.1: https://www.haskell.org/ghc/  :? for help
ghci> :load validate.hs
[1 of 1] Compiling Main             ( validate.hs, interpreted )
Ok, one module loaded.
ghci> validate 507306244
valid
ghci> validate 111111111
not valid
ghci> validate 510486100
not valid
ghci> validate 111111170
not valid
```

We have a bug. Those last two should both be valid. The first is a recently allocated NIF from [nif.pt](https://nif.pt). The other is a contrived example to demonstrate the problem with our implementation.

It is because of a quirk of the mod 11 algorithm. The last digit is actually a control digit. It exists to make sure that the sum of the digits 2-8 multiplied by their index mod 11 is zero. But when the sum of digits 2-8 modulo 11 is 1, the check digit needs to be 10.

The [ISBN-10](https://en.wikipedia.org/wiki/International_Standard_Book_Number#ISBN-10_check_digits) implementation of mod 11 uses an "x" in this case. Portugal uses a 0. I believe this would actually affect the effectiveness of the mod 11 algorithm, but I do not have the math chops to check. I have not found an official reason why. The only explanation I have found is that the government did not want 10% of the tax paying population to feel othered by having an "X" in their NIF.

What this means for our validation algorithm is that we need to actually compare the calculated check digit to the one provided. Adjusting our code from earlier we get something like the below.

```haskell
ps_pt num = foldl (+) 0 (map ps_mult (zip (bd num) [2..]))
-- maps [10, 11] -> 0
fx num = -num * ((div num 10) - 1) 
-- compute the check digit
cd num = fx (11 - (mod (ps_pt num) 11))
-- compare calculated vs given check digits
validate_pt nif = if (cd (div nif 10)) == (mod nif 10)
  then putStrLn "valid"
  else putStrLn "not valid"
```

And now we are ready to validate any NIF.

```
ghci> validate_pt 507306244
valid
ghci> validate_pt 111111111
not valid
ghci> validate_pt 510486100
valid
ghci> validate_pt 111111170
valid
```

##  Conclusion
This is an interesting example of a government body dealing with a quirk a fundamental math problem — cramming two digits where only one will fit. This was also a fun excuse to practice some functional programming with haskell. 

## More neat NIF / Portugal stuff
I found a [webservice](https://nif.pt) to validate a NIF and retrieve some other information encoded into the digits. This service also has an API you can use to validate a NIF with a web request rather than writing your own validator.  There is a limit on the number of free requests you can submit.

To submit more requests you load money on to your account using a Multibanco ATM. Or your banking app but that is less fun. I find it fascinating that you can pay for access to an API service by going to a physical ATM.
