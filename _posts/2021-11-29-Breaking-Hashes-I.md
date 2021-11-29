---
title: Breaking Hashes, Part I. What is a broken hash?
author: Simeon Widdis
date: 2021-11-29 15:20:00 -0700
categories: [Blogging, Explorations]
tags: [cryptography, math, arrays, number theory, computer science, hash functions]
math: true
---

I've long been interested in hash functions. It's hard to point to a specific reason why, but there's something interesting about the fact that you can compute a deterministic value for a given input, and that you can't easily find what the original input was. It's not hard to imagine how something like this could exist, but I've long been interested in the specifics. What specifically makes a given hash function cryptographic or non-cryptographic? What tools exist for breaking them? What do existing cryptographic hash functions do well?

In particular, one family of functions that grabs my attention are the Secure Hashing Algorithms (SHA), and specifically the SHA-2 family of hash functions. This is the family behind the often-used algorithms SHA256 and SHA512. If you google "Why is SHA-2 secure," the answer is unfortunately quite bare: it's designed to be so. Nobody seems to answer in digestible terms the mechanisms that make these particular functions secure. Why *can't* we break it? What types of attacks are known, and why do they fail? For the past while I've sought to answer these, looking around but failing to find anything that satisfied my curiosity.

In a multi-part series of posts, I plan to explore hash functions and methods for breaking them, ultimately getting to why currently used cryptographic hash functions haven't been broken. In this post I'm going to start us on that somewhat lengthy path by going into what hash functions are and what types of attacks cryptographers care about.

## An Introduction to Hash Functions
Let's begin with a story that closely parallels the origins of hashing. It's the early 1950s, and numbers are starting to become a lot more important in daily life. Social security numbers, credit cards, barcodes, there are a lot of cases which rely on having a number (which is unwieldy to memorize). One issue that you consistently face is fakes: people like to tamper with credit cards, social security, and barcodes, for an assortment of nefarious purposes. You want to be able to check if something is valid. This is the point where you might want pause and ponder if you've not been exposed to the idea before.

Some thought might lead you to the following idea. If you ultimately want to know if a number is valid or not, you want to assign some semantic information to a number. That information might be added to the end of the number, and look like `123-45-6782 VALID` or `123-45-6783 INVALID`. Since validity has two states, it can be a little more wieldy to just write `0` for valid, and anything else (like `1`) for invalid (this might seem weird, but having one value for valid and many for invalid turns out to make things easier). However, if we try to implement this directly, maybe by having a big list of numbers and if they're `0` or `1`, we'll quickly run into storage issues. Instead, we want to be able to look at a number and figure out if it's valid or invalid entirely on its own merit.

When Hans Peter Luhn approached this problem in the early and mid 1950s, he invented the following method:
- Take your number `123-45-6782`
- Double every other digit in-place, meaning we don't care about overflows: `143-85-127162`
- Add the digits, then multiply by 9: `40`, `360`
- Check if the last digit is valid, that is, 0. We see that our number was valid.

When we run the same process on the earlier `123-45-6783`, we get a `9`, indicating that the number is invalid. One further leap in logic, and we might try using the last digit of the number as our validity check: instead of our calculation on all values, we can find what happens if we just do the function on `123-45-678`. As it turns out, we get `2`, indicating that the last digit of the number should be `2` for a valid result. Now we can check validity by seeing if the last digit is 2.

This, as it turns out, is a simple version of what is called a *hash function*. We use the data we have to compute a new result (called a hash or checksum), and we can use that result to authenticate our data. Now trivially tweaking the data won't work: if we hastily try to spoof a new social security number, for example with `123-45-6582`, we'll find that our hash doesn't check out.

In many applications, hash functions are used for some type of authentication. We want to know if the data we received is valid, without having to store the data somewhere to look it up later. This is what happens to your password when you log in to any service worth its salt, or use an authenticated encryption protocol. Only, instead of a digit from 0 to 9, we're looking at a number from 0 to 115792089237316195423570985008687907853269984665640564039457584007913129639935 ($2^{256}-1$), or some other absurdly large power-of-2-based number.

For many applications, we want a *cryptographic* hash function. For a hash function to be cryptographic, it should be resistant to both pre-image attacks and collision attacks. I'll cover both of these in the next section, and show that Luhn's method isn't cryptographically secure because it's susceptible to both of these.

## When is a Hash Function Broken?
One issue that you may have noticed with the earlier scheme is that we've barely stopped people tampering with our numbers. Someone could easily just guess random numbers until it works. However, this relies on the small outspace (from 0 to 9 here). However, for practical purposes the output will be way larger, and we'll need to be more clever. Unfortunately, that doesn't save our friend Luhn. To demonstrate this, let's start by writing some python code to compute hashes per his method:
```py
def luhn_hash(number):
	# Retrieve all digits from our string
	digits = [int(i) for i in number if i.isdigit()]
	# Double every other digit
	doubled = [d*2 if i%2>0 else d for i, d in enumerate(digits)]
	# Add up all the total digits
	digit_sum = sum(map(
		lambda x: sum(map(int, str(x))), # Nested sum for 2-digit results
		doubled
	))
	# Multiply by 9 and extract the last digit
	return (digit_sum * 9) % 10
```

And now let's compute some hashes:
```py
>>> luhn_hash('123-45-678')
2
>>> luhn_hash('223-45-678')
1
>>> luhn_hash('223-46-678')
0
>>> luhn_hash('224-45-678')
9
>>> luhn_hash('225-45-678')
8
```

You might notice a pattern: if we increase an even-indexed digit in the input by 1, the value decreases by 1 (wrapping around at 0). Using this rule, it turns out we can easily generate an input that corresponds to any output we want: take any even-indexed digit, and increment it until we get out desired output. A simple algorithm might look something like:
- Compute the hash.
- Find the offset of our current hash from our target hash.
- Increment the first digit by that much (wrapping if necessary).

After implementing it and verifying it works, we've accomplished the goal: for a given hash, we can quickly generate an input that has that hash. This is called a *pre-image attack*. This also means that if the receiver is expecting a specific hash for the data, we can tweak the data arbitrarily to ensure that it looks valid.

Another way we might want to break a hash is to find two values that look the same, without necessarily targeting any specific hash. In essence, find two values whose hashes *collide*, in what's called a *collision attack*. This is a bit less versatile than a pre-image attack, but it still has its uses (such as in digital signatures). It turns out a collision attack is often easier than a pre-image attack: for example, md5, a widely broken algorithm, has widespread collision attacks, but is still considered pre-image resistant.

In this case, a collision attack does turn out to be easier than a pre-image attack: by using our earlier rule of an increment in an even index decrementing the hash, we can quickly generate collisions by taking any fixed input, `000-00-000`, and incrementing two different indices, yielding `100-00-000` and `001-00-000`.

So now we have two types of attacks that we can use to "break" a hash function. When we get to dissecting SHA-2, we'll be looking at both of these, and seeing why exactly the function is so difficult to crack. Before that, however, in the next part I plan to look at breaking more non-cryptographic hash functions, to survey the scene.

## Unimportant Digressions
1. The term pre-image attack was confusing to me for a while, until one day it suddenly clicked. Mathematically, an image of a function on a set is the outputs you get when applying that function on every element of the set. For example, the image of $f(x)=x^2$ on $\\{2, 3, 4\\}$ is $\\{4, 9, 16\\}$. The pre-image of a function is then the inputs that produce those outputs, so the pre-image of $f$ on $\\{4, 9, 16\\}$ is $\\{2, -2, 3, -3, 4, -4\\}$. The term pre-image attack then makes sense: given a cryptographic hash function, we want the pre-image of the function on a given output hash.
2. An often-stated requirement for a cryptographic hash function is the avalanche effect: for a small change in input, the output should change significantly and sufficiently randomly. This is a big topic, and proving the avalanche-ness of a function turns out to be quite difficult (see: strict avalanche criterion, bit independence criterion). I've brushed over it since it is closely related to pre-image resistance, and isn't too important for where we're going.
