---
layout: post
category: T
title: Passwords and bcrypt (repost)
---

_This is a repost of a blog post I made in February 2013 on another blog I am writing for (on?) with some minor corrections.  The original can be viewed [here](http://50thoughts1dollar.blogspot.com/2013/02/passwords-and-bcrypt.html)._

With the recent news of [Twitter losing up to a quarter of a million passwords](http://techcrunch.com/2013/02/01/twitter-hacked-250k-affected-just-go-change-your-password-now-though/) and [The New York Times' computer systems being compromised](http://www.nytimes.com/2013/01/31/technology/chinese-hackers-infiltrate-new-york-times-computers.html?pagewanted=all&_r=1&) now sounds like a good time to discuss some computer security and why you might not have to panic about all those accounts for which you've used the same password (thanks to bcrypt!).  This post focuses on what happens once someone steals your password file and not on how to protect that file, which is a whole different story.

Firstly, what does it mean when someone breaks into a computer system and steals your password?  In the Twitter case, it means that someone was somehow able to grab a salted and hashed password.  I'll explain what that means in a bit, but what the attacker gets is a list of passwords that look roughly like this:

`$2a$10$vI8aWBnW3fID.ZQ4/zo1G.q1lRps.9cGLcZEiGDMVr5yUP1KUOYTa`

Although you would never know it from looking at it, this is what your password actually looks like in a website's databases.  Now, that looks nothing like your password that you type into login pages, and that is because they are salted and hashed.  Hashing is a fancy way of saying that a password has been transformed in such a way that it is basically impossible to know the original text from the ciphertext (result of the transformation). For instance, using the md5 hashing algorithm, we can turn the input `coolcat` into `f7f70fefd4bed17191df6ec8bc24c63d` (try it out with this [online md5 generator](http://www.adamek.biz/md5-generator.php)).  Notice that the hash for `coolcat2` is `0054cc598c64e0780ddcc5ed7798c1c6`, which is radically different, even though they're off by just one character.  This is what hash algorithms are like: give me something and I'll give you random gibberish.  Note: I use hashing function and hashing algorithm interchangeably.

Cryptographic hashing functions, which are just hashing functions that are used to deal with stuff that needs to stay secret need to satisfy some basic rules (ripped from [Wikipedia](http://en.wikipedia.org/wiki/Cryptographic_hash)):

1. it is easy to compute the hash value for any given message
2. it is infeasible to generate a message that has a given hash
3. it is infeasible to modify a message without changing the hash
4. it is infeasible to find two different messages with the same hash

When taken in the context of passwords, these four rules make a lot of sense.  The first means that it should be easy to take a password like `coolcat` and get the hashed result, because that's how your password is checked.  When you type `coolcat` into that password box to log in to Twitter a hashing algorithm takes it and makes it into the hashed result, and if that result matches what they've stored when you signed up, then they know the password you typed is correct, because of rules #3 and #4. These two rules basically mean that you can't come up with a different password that somehow also hashes to the same thing as your password, because that would be whats called a collision (if this did not hold, it could mean that `coolcat` and 'dumbcat' could both be used to grant access to your account, allowing random racist tweets!).  Number 2 is a little more subtle, but it means that given a hash you can't go from that hash to the original password (it is nearly impossible to find the inverse of the hash function).

_The tl;dr is that when you hash a password, there is no easy way to figure out what the original password was._  Going back to our Twitter example, this means that the hashed passwords the attackers got are useless until they crack them.

To figure out the original passwords, what the attackers now need to do is to try different combinations of letters, numbers and symbols until they find a hash that matches one they stole - when that happens, they know your password.  Do this enough and eventually you'll have all 250k passwords, but how long does that take?  Given md5, assuming that all passwords are alphanumeric and 8 characters in length (2,821,109,907,456 - I had to write it out, for effect) then it takes...5.64 seconds.  This is based on slightly out of date estimates by [Coda Hale on his blog](http://codahale.com/how-to-safely-store-a-password/) (whose post was an inspiration for this one) that for $300/hr you can rent a CPU to do 500 billion passwords per second.

So what's the point of hashing (and this article) if it only takes 5.64 seconds to crack 2.8 billion potential passwords?  I know I've been rambling a bit, but stick with me because shit is going to get real.

Passwords were originally hashed using algorithms like md5, or SHA-256, and that was fine, because computers were really slow, so it would take a thousand years to crack any meaningful number of passwords.  As we've been able to cram more power onto tiny silicon chips, we've also made our hashing algorithms obsolete.  This is where bcrypt comes in.

Whereas md5 or other cryptographic hash functions are designed to consume as much data as quickly as possible (md5 is often used to verify quickly the integrity of large files, such as videos), brcrypt was designed to do this very slowly!  It was designed to be intentionally slow and intentionally difficult to parallelize to compensate for ever increasing hardware speeds, and was designed to protect your password even when the hashed password was stolen.

To learn more about this, lets look at the hashed example I provided earlier, which is actually what a bcrypt hashed password looks like:

`$2a$10$vI8aWBnW3fID.ZQ4/zo1G.q1lRps.9cGLcZEiGDMVr5yUP1KUOYTa`

Here's an explanation of the password from [Stack Overflow](http://stackoverflow.com/questions/6832445/how-can-bcrypt-have-built-in-salts#answer-6833165):

* 2a identifies the bcrypt algorithm version that was used.
* 10 is the cost factor; 210 iterations of the key derivation function are used (which is not enough, by the way. I'd recommend a cost of 12 or more.)
* `vI8aWBnW3fID.ZQ4/zo1G.q1lRps.9cGLcZEiGDMVr5yUP1KUOYTa` is the salt and the cipher text, concatenated and encoded in a modified Base-64. The first 22 characters decode to a 16-byte value for the salt. The remaining characters are cipher text to be compared for authentication.
* $ are used as delimiters for the header section of the hash.

Lets forget about the version and the salt for now, though I've been putting off the salt for a while.  The real thing we need to focus on here is the cost factor.  bcrypt takes two inputs to generate a hashed password: the password itself, and a cost factor.  The cost factor, as stated above, determines how expensive it is to compute the bcrypt hash.  By adding one to the cost factor, you increase the cost of bcrypt, by a factor of two.  This is huge, because by adding something like 4, you've now made it cost 16 times more to calculate a password.  Of course, if you've looking to log in to a website where you know a password, waiting 1/2 a second for your password to be hashed is ok, but if you're computing trillions of passwords, getting all trillion at 2 passwords per second is...impossible.

So bcrypt is intentionally slow, but there's other really cool stuff about it.  Even though its slow, it only uses operations (xor and other bit operations) that computers are already really good at.  This makes it really difficult to design a specific circuit that is even better at cracking bcrypt passwords than modern cpus (this attack has been done for other hashing algorithms, notably DES which used bitslicing, a relatively expensive operation for cpus).  Another really interesting thing about is that it's very hard to parallelize.  Every step of the hashing involves the inputs to be hashed by the other inputs, so each input to the next step is dependent on the previous step.  In simpler terms, its impossible to parallelize something where the input to A depends on the result of B, which in turn depends on the result of C (and so forth, 2^10 times).  This is important because much of the speedup in processing power and computation in the past few years has been because of advances in hardware and algorithms that allow us to better parallelize computation.

So, now we know that bcrypt is really cool, and hopefully if Twitter used bcrypt with a high enough cost factor, your password may be safe (if it was a good password with enough letters, numbers symbols, blah blah).  The original paper is really cool, and if you're technically inclined, you should really check it out [here](http://static.usenix.org/event/usenix99/provos/provos.pdf).

One last thing (extra credit I suppose) are the salts.  I introduced them in the beginning and they came up in the explanation of bcrypt.  A salt is something that solves the problem of people not picking unique passwords, and that given enough time, you can pre-compute a vast number of passwords.  Think about it this way: `coolcat` will always hash to the same value when given the same inputs, so once `coolcat` has been cracked as a password, you know that password pretty much forever.  However, if we add some random element to your password `coolcat`, such as the number 1203 and then store that in plain text (not hashed), then we've added another level of difficulty to cracking your password (note: the random salt is added in by the program, not by the user).  The attacker now needs to compute your password with the addition of '1203', but where it really hurts is when every user has a different random number attached to their password.  Cracking `coolcat1203` is completely separate from cracking `coolcat285`.  Now we can no longer pre-compute passwords (given a large enough random salt) because there are simply too many possible salts to use with your password.  When someone gets your hashed, salted password, they need to crack that `coolcat` password just for you.  bcrypt uses a 128 bit salt, which means that the salt can be any number between 0 and 2^128.
