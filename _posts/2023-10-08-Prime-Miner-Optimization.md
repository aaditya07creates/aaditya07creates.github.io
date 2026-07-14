---
layout: post
title: Optimized prime miner
subtitle: My Take on Optimization
cover-img: /assets/img/prime1.png
thumbnail-img: /assets/img/prime2.png
share-img: /assets/img/path.jpg
tags: [DSA, optimization]
author: Aaditya Bhave
---
<br />

## Optimized prime miner ##

I used the Square Root Theorem for Primality Testing as the base program. It works by taking the new number's square root and checking if prime numbers less than the root can divide it. If they cannot, it is prime. The program's goal is to compute as many primes as it can.

In programming, there are many types of optimizations, but most of them work towards saving resources and time. I implemented such optimizations and mathematical logic to optimize my program.
<br />

## Iterating by 2
We know no prime numbers other than two are even, so we can skip the even numbers by iterating by 2. This optimization halves the loops already.


~~~
number += 2
~~~
<br />

## Skipping multiples of three
We just threw away all the even numbers. Can we throw away more before we even check them? Turns out we can. Every prime bigger than 3 fits the pattern 6k ± 1, which is a fancy way of saying it sits right next to a multiple of six. Everything else is divisible by 2 or 3, so it's not worth our time.

Instead of always adding 2, we bounce our step between +2 and +4. Starting from 7 that gives us 7, 11, 13, 17, 19, 23... skipping every multiple of 2 AND 3 for free.

~~~
gap = 4
while True:
    # ...check the number...
    number += gap
    gap = 6 - gap   # flips between 4 and 2 forever
~~~

This alone drops the numbers we even look at to about a third of the range. A few composites like 25 still sneak through (it's 6×4+1), but our primality check catches those, so no harm done.
<br />

## Memoization
Memoization is an optimization where the results of expensive function calls are stored for reuse. This helps in recursive programs like ours. The primes are calculated and stored in a list to compare with the new number. So we don't have to recalculate each prime every loop.

~~~
while True:
    if is_prime(number, known_primes):
        largest_prime = number
        prime_count += 1
        known_primes.append(number)
~~~

Where do we store these primes? Well, we store them on the RAM! Numbers take up very little digital space so we can store hundreds of thousands of numbers on our RAM with ease. We can monitor the RAM usage with a library called psutil.

~~~
import psutil
process = psutil.Process()
ram_usage_gb = process.memory_info().rss / (1024 ** 3)
print(f"Program RAM Usage: {ram_usage_gb:.2f} GB")
~~~
<br />

{: .box-note}
**Note:** The data structure for storing these vast numbers is a simple Python list. We could have used other data structures like numpy arrays or dictionaries, but on testing, they give a higher time complexity. Numpy arrays are great, but not for our use case. They are excellent for vectorization and operations on massive datasets, but we only need to store and retrieve data.
<br />

## Five divisibility
If a number ends with 0 or 5, it is divisible by 5. We can use this logic to skip many checks and reduce our complexity even further.

~~~
if n % 10 in (0, 5):
    return False
~~~

<br />

## Integer square roots
Here's a sneaky one. In my check I was calling `math.sqrt(n)` to get the square root and comparing my primes against it. But `math.sqrt` hands back a float, and once our numbers get really big, floats start losing precision and rounding in ways we don't want. On top of that, computing a square root every single call adds up.

The trick is to never take the square root at all. Instead of asking "is this prime bigger than √n?", we ask "is this prime squared bigger than n?". Same question, but now it's pure integer multiplication with no float and no `math.sqrt`.

~~~
for prime in known_primes:
    if prime * prime > n:   # instead of prime > math.sqrt(n)
        break
    if n % prime == 0:
        return False
return True
~~~

Small change, but it removes a whole function call from the hottest loop in the program.
<br />

## Redundant calculations
Instead of iterating through known primes and checking each time if the prime's square is greater than the new number we can reverse it. Initially, on receiving the new number we can calculate its square root and simply check if the prime in our list is greater than the square root. As the numbers get bigger this optimization makes a huge difference.

<br />

## A different approach: the Sieve
Everything above makes trial division faster, but there's a completely different way to hunt primes called the Sieve of Eratosthenes. Instead of testing one number at a time, you write down every number up to a limit and start crossing out multiples. Cross out every multiple of 2, then every multiple of 3, then 5, and so on. Whatever survives is prime.

~~~
def sieve(limit):
    is_prime = [True] * (limit + 1)
    is_prime[0] = is_prime[1] = False
    for i in range(2, int(limit ** 0.5) + 1):
        if is_prime[i]:
            # start at i*i, everything smaller is already crossed out
            for multiple in range(i * i, limit + 1, i):
                is_prime[multiple] = False
    return [i for i, p in enumerate(is_prime) if p]
~~~

The sieve is *ridiculously* fast for finding every prime up to some limit. The catch is right there in the word "limit". My miner is supposed to run forever and keep climbing, but the sieve needs to know how big the box is before it starts, and that box has to fit in RAM. Test 10 million? Easy. Test up to a trillion? Now you need a trillion booleans and your computer taps out.

The grown-up fix is a **segmented sieve**: instead of one giant box, you sieve one chunk at a time, throw it away, and move to the next chunk. That keeps memory flat while still marching upward. I've got the idea but I'm still wrapping my head around the implementation, so that's a project for another day.

So which is better? For "all primes below N" the sieve wins hands down. For "keep mining bigger and bigger primes with no ceiling" my trial-division miner is the more honest fit. Different tools for different jobs.
<br />

## Multithreading
This is where things get complicated, my knowledge on thread locking is limited and I came across many errors while trying to optimize the code using this method. Multithreading means to utilize more of your cpu's resources when usually python scripts only run on a single thread, here we mine multiple primes at once across many threads. The errors I came across are called "Race errors" that occur in recursive functions like mine or where shared resources are being accessed. The error is that the prime numbers are being checked require information that the other threads are still working on and have not contributed yet, so one function is trying to access missing data and fails to complete. We can fix this by using proper thread locking and bypassing memoization on coming across a race error.

After digging around a bit more, I found out there's an even bigger reason threads didn't speed things up: Python has something called the **GIL** (Global Interpreter Lock). It only lets one thread actually run Python code at a time, so for a heavy number-crunching job like mine, spinning up ten threads doesn't get me ten times the speed, it just takes turns. Threads are really meant for waiting on things like files or the network, not for raw math.

The tool that *does* run on all your cores is **multiprocessing**, which launches separate Python processes instead of threads. The trade-off is that each process has its own memory, so sharing the `known_primes` list between them is the hard part, and my race condition basically comes back wearing a different hat. This is exactly where the segmented sieve shines, since each chunk can be handed to its own process and worked on completely independently. I will try my best to learn and apply this knowledge when I finish researching more about this topic.

<br />

## Benchmarking the miner
None of this means much if I can't measure it, so I time the run and print how long it's been going alongside the RAM. Watching the "primes per second" hold steady (or not) as the numbers climb is honestly the most satisfying part.

~~~
elapsed = time.time() - start_time
print(f"Time Elapsed: {elapsed:.1f}s")
print(f"Primes/sec: {prime_count / elapsed:.0f}")
~~~
<br />

## Code
Here's the miner with the new bits folded in, the 6k ± 1 stepping and the integer square-root check, plus the timing readout.

{% highlight python linenos %}

# importing libraries
import os
import time
import psutil


# function for clearing the screen
def clear_screen():
    os.system('cls')


# function to check if the number is prime
def is_prime(n, known_primes):

    if n % 10 in (0, 5):
        return False

    for prime in known_primes:
        # no square root needed, pure integer math
        if prime * prime > n:
            break
        if n % prime == 0:
            return False
    return True


# function to record and iterate numbers
def find_primes():
    largest_prime = 0
    prime_count = 0
    checked_count = 0
    number = 7
    gap = 4                      # step flips between 4 and 2 (the 6k +- 1 wheel)
    known_primes = [2, 3, 5]
    process = psutil.Process()
    start_time = time.time()

    while True:
        if is_prime(number, known_primes):
            largest_prime = number
            prime_count += 1
            known_primes.append(number)

        checked_count += 1

        # print statements
        if checked_count % 100000 == 0:
            elapsed = time.time() - start_time
            ram_usage_gb = process.memory_info().rss / (1024 ** 3)
            clear_screen()
            print(f"Highest Prime Found: {largest_prime}")
            print(f"Checked Numbers: {checked_count}")
            print(f"Prime Count: {prime_count}")
            print(f"Program RAM Usage: {ram_usage_gb:.2f} GB")
            print(f"Time Elapsed: {elapsed:.1f}s")
            print(f"Primes/sec: {prime_count / elapsed:.0f}")

        number += gap
        gap = 6 - gap            # 4, 2, 4, 2, ... forever


find_primes()
{% endhighlight %}
