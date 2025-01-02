---
layout: post
title: Optimized prime miner
subtitle: My Take on Optimization
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [DSA, optimization]
author: Aaditya Bhave
---
<br />

## Optimized prime miner ##

I used the Square Root Theorem for Primality Testing as the base program. It works by taking the new number's square root and checking if prime numbers less than the root can divide it. If they cannot, it is prime. The program's goal is to compute as many primes as it can.

In programming, there are many types of optimizations, but most of them work towards saving resources and time. I implemented such optimizations and mathematical logic to optimize my program.
Iterating by 2
We know no prime numbers other than two are even, so we can skip the even numbers by iterating by 2. This optimization halves the loops already.
<br />

~~~
number += 2
~~~

Memoization
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

### Notification

{: .box-note}
**Note:** The data structure for storing these vast numbers is a simple Python list. We could have used other data structures like numpy arrays or dictionaries, but on testing, they give a higher time complexity. Numpy arrays are great, but not for our use case. They are excellent for vectorization and operations on massive datasets, but we only need to store and retrieve data.

Five divisibility
If a number ends with 0 or 5, it is divisible by 5. We can use this logic to skip many checks and reduce our complexity even further.

~~~
if n % 10 in (0, 5):
    return False
    return False
~~~

Redundant calculations
Instead of iterating through known primes and checking each time if the prime's square is greater than the new number we can reverse it. Initially, on receiving the new number we can calculate its square root and simply check if the prime in our list is greater than the square root. As the numbers get bigger this optimization makes a huge difference.

## Code

{% highlight javascript linenos %}

# importing libraries
import math
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

    sqrtn = math.sqrt(n)

    for prime in known_primes:
        if prime > sqrtn:
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
    known_primes = [2, 3, 5]
    process = psutil.Process()

    while True:
        if is_prime(number, known_primes):
            largest_prime = number
            prime_count += 1
            known_primes.append(number)


        checked_count += 1

        # print statements
        if checked_count % 100000 == 0:
            ram_usage_gb = process.memory_info().rss / (1024 ** 3)
            clear_screen()
            print(f"Highest Prime Found: {largest_prime}")
            print(f"Checked Numbers: {checked_count}")
            print(f"Prime Count: {prime_count}")
            print(f"Program RAM Usage: {ram_usage_gb:.2f} GB")

        number += 2



start_time = time.time()
find_primes()
{% endhighlight %}