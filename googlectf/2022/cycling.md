# Cycling

**CTF:** GoogleCTF 2022  
**Category:** crypto  
**Original writeup:** <https://ur4ndom.dev/posts/2022-07-04-gctf-cycling/> — by **ur4ndom.dev**. All credit goes to the original author; this is an archived copy.

---

[GCTF 2022] Cycling

Robin Jadoul

Points: 201 (50 solves)

It is well known that any RSA encryption can be undone by just encrypting the ciphertext over and over again. If the RSA modulus has been chosen badly then the number of encryptions necessary to undo an encryption is small. However, if the modulus is well chosen then a cycle attack can take much longer. This property can be used for a timed release of a message. We have confirmed that it takes a whopping 2^1025-3 encryptions to decrypt the flag. Pack out your quantum computer and perform 2^1025-3 encryptions to solve this challenge. Good luck doing this in 48h.

Exploring the challenge

Let’s have a brief look at the source code we’re provided with:

e = 65537 n = ... # snip ct = ... # snip # Decryption via cycling: pt = ct for _ in range ( 2 ** 1025 - 3 ): pt = pow ( pt , e , n ) # Assert decryption worked: assert ct == pow ( pt , e , n ) # Print flag: print ( pt . to_bytes (( pt . bit_length () + 7 ) // 8 , 'big' ) . decode ())

In short, we’re faced with an RSA encryption of the flag, and one additional “fact” that’s supposed to help us in some way. The fact is that when we repeat the encrypting exponentiation R = 2 1025 − 3 R = 2^{1025} - 3 times, we achieve the same as decrypting the ciphertext. Working through the math, this tells us that x e R ≡ x x^{e^{R}} \equiv x holds, at the very least when x x represents the flag.

A well-known fact when dealing with RSA, and modular exponentiation in general, is that the order of the multiplicative group mod n n is equal to the number of integers < n < n that are coprime to n n . This quantity is known as Euler’s totient φ ( n ) \varphi(n) .

Furthermore, from Euler’s theorem (or Lagrange’s theorem if we frame it in a group-theoretic way), we know that any number x x taken to the φ ( n ) \varphi(n) th power (mod n n ), results in the identity 1 1 . This property is in fact vital for the correctness of RSA, as we rely on the fact that x φ ( n ) ≡ 1 ( mod n ) x^{\varphi(n)} \equiv 1 \pmod n when we say that x e d ≡ ( x φ ( n ) ) k x ≡ x x^{ed} \equiv \left(x^{\varphi(n)}\right)^k x \equiv x .

From all this known theory underlying the RSA cryptosystem, we can now finally make a first deduction: e R + 1 ≡ 1 ( mod φ ( n ) ) e^{R + 1} \equiv 1 \pmod{\varphi(n)} .

Background: Carmichael’s λ \lambda

Actually, that previous statement is what you would say with a basic understanding of the principles underlying RSA, but it’s in fact not entirely correct. It could very well be the case that x ℓ ≡ 1 ( mod n ) x^\ell \equiv 1 \pmod n for ℓ < φ ( n ) \ell < \varphi(n) . Moreover, this will for the case for every x x when n = p q n = pq is an RSA modulus. One thing that Lagrange’s theorem will still give us, even when ℓ ≠ φ ( n ) \ell

e \varphi(n) , is that ℓ ∣ φ ( n ) \ell \mid \varphi(n) .

The smallest exponent such that x ℓ ≡ 1 ( mod n ) x^\ell \equiv 1 \pmod n for all x x is known as the Carmichael function λ ( n ) \lambda(n) . We know that λ ( n ) ∣ φ ( n ) \lambda(n) \mid \varphi(n) , but we can even write down a nicer formula:

λ ( p 1 r 1 p 2 r 2 … p m r m ) = l c m ( p 1 r 1 − 1 ( p 1 − 1 ) , … , p m r m − 1 ( p m − 1 ) ) \lambda(p_1^{r_1}p_2^{r_2}\ldots p_m^{r_m}) = \mathrm{lcm}(p_1^{r_1 - 1}(p_1 - 1), \ldots, p_m^{r_m - 1}(p_m - 1))

which, when we apply this to our RSA modulus n n , becomes

λ ( p q ) = l c m ( p − 1 , q − 1 ) \lambda(pq) = \mathrm{lcm}(p - 1, q - 1)

Returning to our wrong statement from before, we now know that e R + 1 ≡ 1 ( mod ℓ ) e^{R + 1} \equiv 1 \pmod \ell where ℓ ∣ λ ( n ) \ell \mid \lambda(n) , and furthermore, since e e doesn’t look too suspicious, nor can a readable flag be influenced all that much, we can in fact hope that e R + 1 ≡ 1 ( mod λ ( n ) ) e^{R + 1} \equiv 1 \pmod{\lambda(n)} .

Factors of factors of factors; and some subtractions

Now that we understand the nuances of the formula x ℓ ≡ 1 x^\ell \equiv 1 a bit better, we can think further towards solving this challenge. Remember what we wrote down earlier?

e R + 1 ≡ 1 ( mod λ ( n ) ) e^{R + 1} \equiv 1 \pmod{\lambda(n)}

This tells us something more, since we have exactly the form of statement that lead us to introducing λ ( n ) \lambda(n) in the first place. We could now say that R + 1 ∣ λ ( λ ( n ) ) R + 1 \mid \lambda(\lambda(n)) , which gives us a somewhat nice relation between the value R R we’d been given, and our RSA modulus n n .

Let’s not worry about the possibility that R + 1 R + 1 is only a divisor, and instead assume that it holds with equality R + 1 = λ ( λ ( n ) ) R + 1 = \lambda(\lambda(n)) . Then we can try to write down what we expect R + 1 R + 1 to be:

R + 1 = λ ( λ ( n ) ) = λ ( l c m ( p − 1 , q − 1 ) ) = λ ( 2 s 1 s 2 … s m ) = l c m ( s 1 − 1 , … , s m − 1 ) \begin{aligned} R + 1 = \lambda(\lambda(n)) &= \lambda(\mathrm{lcm}(p - 1, q - 1)) \\ &= \lambda(2s_1s_2\ldots s_m) \\ &= \mathrm{lcm}(s_1 - 1, \ldots, s_m - 1) \end{aligned}

We now would like to relate these values s i − 1 s_i - 1 to R R somehow. By the above, it should be clear that any s i − 1 ∣ R + 1 s_i - 1 \mid R + 1 , so when we list all divisors of R + 1 R + 1 in turn, and add 1 1 to them, we should end up with a set of candidates 𝒞 \mathcal{C} , such that { s i } i ⊆ 𝒞 \{s_i\}_i \subseteq \mathcal{C} . The value R + 1 R + 1 itself is not particularly easy to factor in a short amount of time, but luckily it’s not an esoteric, unknown value, but a nicely structured one. And as it often happens to nicely structured values, they show up on factordb.

Who cares if it’s not the private key? It works

With the set 𝒞 \mathcal{C} , what could we do? One option to consider is applying the same trick again, but using 𝒞 \mathcal{C} rather than the factorization of R + 1 R + 1 , to recover p p and q q . Annoyingly, 𝒞 \mathcal{C} is a rather large set, and enumerating all subsets of it takes exponential time, so we’ll have to throw that idea out.

Instead, let’s look back at our initial, more naive, understanding of the RSA cryptosystem, where we used φ ( n ) \varphi(n) rather than λ ( n ) \lambda(n) to compute the decryption exponent d = e − 1 ( mod φ ( n ) ) d = e^{-1} \pmod{\varphi(n)} . Even though we didn’t have the smallest modulus possible, we still had full correctness, since — as we know by now — λ ( n ) ∣ φ ( n ) \lambda(n) \mid \varphi(n) . We can take that to the extreme: we know/assume that all factors of λ ( n ) \lambda(n) are among our values s i s_i , so if we simply take Ξ = ∏ i s i \Xi = \prod_i s_i , it should hold that λ ( n ) ∣ Ξ \lambda(n) \mid \Xi , and as such, we could use Ξ \Xi as a “replacement” for λ ( n ) \lambda(n) or φ ( n ) \varphi(n) when it comes to computing a decryption exponent.

With this, we have enough ideas and information to finally solve this challenge .

CTF{Recycling_Is_Great}

All your factorbase are belong to us

The official intended solution relies on similar observations, but instead of finding some number k λ ( n ) k\lambda(n) , it uses those potential factors of λ ( n ) \lambda(n) to directly factor the modulus n n . To understand how this factorization works, we look back at a well-known, simple special-purpose factoring algorithm: Pollard’s p - 1 algorithm.

Pollard’s algorithm allows finding a factor p p (say for n = p q n = pq ) under the assumption that p − 1 p - 1 is B B -powersmooth. That is, all prime power divisors s r ∣ p − 1 s^r \mid p - 1 are bounded by s r < B s^r < B . This property enables us to find some product M M of prime powers less than B B , such that p − 1 ∣ M p - 1 \mid M but q − 1 ∤ M q - 1

mid M . In turn, looking at Fermat’s little theorem, we can see that for all a a , it holds that a M ≡ 1 ( mod p ) a^M \equiv 1 \pmod p , but it will often be the case that a M ≢ 1 ( mod q ) a^M

ot\equiv 1 \pmod q , and so looking for gcd ( a M − 1 , n ) \gcd(a^M - 1, n) should allow us to recover p p . Traditionally, Pollard’s method computes a M ( mod n ) a^M \pmod n by repeatedly taking s s th powers of an accumulator and testing whether the gcd \gcd results in a factorization. The advantage of this is twofold: one doesn’t need to fully compute M M in its entirety and as long as the largest factor of p − 1 p - 1 differs from the largest factor of q − 1 q - 1 , the method will still work, rather than yielding n n as the gcd \gcd .

To abstract the bound B B away, we simply notice that all it gives us is some superset 𝒞 \mathcal{C} of prime (power) factors of p − 1 p - 1 . This is exactly what we already found by enumerating primes s i s_i such that s i − 1 ∣ R + 1 s_i - 1 \mid R + 1 ! If we call such a set 𝒞 \mathcal{C} a factor base, and slightly generalize Pollard’s algorithm, we can apply it to our situation as well. And this is exactly the approach we see in the official solution script: we take some base a a (there called m m ), repeatedly exponentiate it by potential prime factors, and check if gcd ( a ′ − 1 , n ) ∉ { 1 , n } \gcd(a' - 1, n)

otin \{1, n\} .

Once a factorization of n n is found, it is of course only a matter of performing regular RSA decryption to obtain the flag.

Does this always work?

Until now, we’ve made several assumptions that turned out to be correct, in order to solve this challenge. Even the official solution turns out to rely on those assumptions, so we’d like to have a look at how much trouble we’d be in when the assumptions would be invalidated. To restate our assumptions more explicitly:

R + 1 = λ ( λ ( n ) ) R + 1 = \lambda(\lambda(n)) The inner λ \lambda corresponds to the multiplicative order of the flag, | m | ( mod n ) |m| \pmod n

corresponds to the multiplicative order of the flag, The outer λ \lambda corresponds to the multiplicative order of e e , modulo | m | |m| p − 1 p - 1 and q − 1 q - 1 are square-free, that is, none of their prime factors occur with multiplicity > 1 > 1 . All s i − 1 s_i - 1 are square-free, where s i ∣ λ ( n ) s_i \mid \lambda(n) , but this has been confirmed by the factorization of R + 1 R + 1

And let’s also introduce some counterexamples for all of these, where our assumption is invalidated, and our solution becomes broken:

We explore the possibility for both λ \lambda s: Let n = 77 n = 77 , then λ ( n ) = l c m ( 6 , 10 ) = 30 \lambda(n) = \mathrm{lcm}(6, 10) = 30 , but for the message m = 15 m = 15 , it’s already true that m 5 ≡ 1 ( mod n ) m^5 \equiv 1 \pmod n , rather than the expected m 30 m^{30} . This in turn implies that we only need e R + 1 ≡ 1 ( mod 5 ) e^{R + 1} \equiv 1 \pmod 5 to complete decrypt this message by cycling.

, then , but for the message , it’s already true that , rather than the expected . This in turn implies that we only need to complete decrypt this message by cycling. We now use n = 989 = 23 × 43 n = 989 = 23\times43 , so λ ( n ) = 2 × 3 × 7 × 11 = 462 \lambda(n) = 2\times3\times7\times11 = 462 and λ ( λ ( n ) ) = l c m ( 2 , 6 , 10 ) = 30 \lambda(\lambda(n)) = \mathrm{lcm}(2, 6, 10) = 30 , but for instance e = 379 e = 379 only has multiplicative order 5 5 modulo λ ( n ) \lambda(n) . Consider n = 163 × 67 = 10921 n = 163\times67 = 10921 , then λ ( n ) = l c m ( 2 × 3 4 , 2 × 3 × 11 ) = 2 × 3 4 × 11 = 1782 \lambda(n) = \mathrm{lcm}(2\times3^4, 2\times3\times11) = 2\times3^4\times11 = 1782 . λ ( λ ( n ) ) \lambda(\lambda(n)) would then be 2 × 3 3 × 5 2\times3^3\times5 , and we’d never pick up 3 4 3^4 as a potential factor out of any subset. Similar issues to the earlier point occur, except they are introduced slightly later in the process of computing λ ( λ ( n ) ) \lambda(\lambda(n)) .

Other than those assumptions, there’s some more things that can go wrong. We’ve relied on enumerating all subsets of factors of R + 1 R + 1 , but — as we remarked with an initial failed idea of repeating such an enumeration on the result of that — that takes exponential time in the number of factors. If we end up with a large amount of these factors, we might in fact already get in trouble trying to enumerate all subsets, and spend a long time waiting for that. On the other side of that medallion, to obtain that initial list of factors, we need to be able to factor this value R + 1 R + 1 . Now, if we consider for instance the worst case, where p p and q q are what we could call doubly-safe primes, i.e. p = 2 ( 2 p ′ + 1 ) + 1 p = 2(2p' + 1) + 1 , p p and p − 1 2 \frac{p - 1}{2} are both safe primes, there’s only a minor difference in number of bits between n n and λ ( λ ( n ) ) \lambda(\lambda(n)) and the latter is even a new RSA modulus. By assumed security of RSA (unless you get extra information like in this challenge), factoring that would not be feasible.

Can we fix it?

Yes we can!

Sort of, at least. Unless my very vague notion from the footnote in the previous section pans out, I don’t expect we could get around the exponential time enumeration, or the factoring problems. We can however try to fix up some of the problems our assumptions brought along, and get a slightly more generic solution.

Let’s first look at the case where | m | < λ ( n ) |m| < \lambda(n) . Since for our original approach, we only care about m m itself, and not about factoring, we only need to find a multiple of | m | |m| , which we still get from our powerset enumeration (under the assumption, for now, that we get λ ( | m | ) \lambda(|m|) ). This means we can still compute an effective decryption exponent that works for m m itself, and any element with an order dividing | m | |m| . Moreover, we can still use this to fully factor n n too. Once we decrypt m m , we know that m k | m | ≡ 1 ( mod n ) m^{k|m|} \equiv 1 \pmod n , which means we can again apply our p - 1 approach with a factor base to find an exponent M M such that m M ≡ 1 ( mod p ) m^M \equiv 1 \pmod p . Note that we require the use of m m as basis here, rather than an arbitrary number a a . The only condition under which this will still necessarily fail is when | m | |m| divides gcd ( p − 1 , q − 1 ) \gcd(p - 1, q - 1) , since then any exponent such that m M ≡ 1 ( mod p ) m^M \equiv 1 \pmod p also satisfies m M ≡ 1 ( mod q ) m^M \equiv 1 \pmod q .

Next, we investigate the case where R + 1 ≠ λ ( | m | ) R + 1

e \lambda(|m|) . Unfortunately, the solution here isn’t quite as clean. When the order of e e is too small, we simply lack the information to recover enough primes. When the order of e e is only slightly too small, we should be able to salvage it with only a constant cost to our computation time. Pick a fixed-size (multi)set of small primes 𝒫 \mathcal{P} , and let 𝒞 ′ = 𝒞 ∪ 𝒫 \mathcal{C}' = \mathcal{C} \cup \mathcal{P} . This increases the computation time with a factor 2 | 𝒫 | 2^{|\mathcal{P}|} , but as long as the “lost” factor of λ ( | m | ) \lambda(|m|) is factorable over 𝒫 \mathcal{P} , our algorithm works again. Optionally, if we would like to deal with more complex situations, we could also construct more complex ways to add extra factors. For example, an approach comparable to the “two-stage” variant of the p - 1 algorithm is possible, where we take e.g. at most 4 “small” primes and 1 “medium” sized extra prime.

Finally, how can we deal with the annoyance of prime powers? To deal with prime power divisors of λ ( n ) \lambda(n) , it’s possible to apply a strategy similar to the p - 1 factoring algorithm, where every prime factor is simply included multiple times. Since exponents larger than 2 2 can be detected in the factorization of R + 1 R + 1 , we recommend including each factor twice, unless evidence to the contrary is present from the factorization of R + 1 R + 1 . The more conservative approach would be to include each prime s s a number of times proportional to log ( n ) log ( s ) \frac{\log(n)}{\log(s)} , which comes at an obvious computational cost. For the prime power divisors of λ ( λ ( n ) ) \lambda(\lambda(n)) , we’ll need to do just a bit more work. If we see a prime power s r s^r when factoring R + 1 R + 1 , it should be clear from how λ \lambda works that we expect to also see the factors of s − 1 s - 1 appear. In that case, it’s obvious that s r + 1 s^{r + 1} should be in 𝒞 \mathcal{C} . When however s 2 ∣ s i s^2 \mid s_i , we only see s s and the factors of s − 1 s - 1 appear when factoring R + 1 R + 1 . Hence, when we add a prime to 𝒞 \mathcal{C} that still divides R + 1 R + 1 , we should add it with a higher multiplicity.

Talk is cheap, show me the code

We present here the code as we implemented it during the CTF. That is, making maximal assumptions such that it still gets the flag. The code for factorization based on Pollard’s p - 1 algorithm can be found in the official solution. Interested readers are encouraged to understand, implement and share the suggested improvements of this article :)
