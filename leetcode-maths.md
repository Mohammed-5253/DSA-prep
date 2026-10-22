# Maths / Number Problems — Two-Week DSA Question Bank

Personal DSA reference for the Mathematics / Number-problems phase.

Despite the filename, this file contains problems from multiple sources. LeetCode is the
primary source; other reputable platforms are used where LeetCode has no suitable equivalent.

- **Scope:** Mathematics and number problems only.
- **Ordering:** Core problems are ordered by learning progression (Stage 1 to Stage 7), not by
  problem number or platform.
- **Status values:** Not Started, Attempted, Solved, Solved With Hint, Needs Revision.
- **Importance:** ⭐⭐⭐⭐⭐ Essential, ⭐⭐⭐⭐ Very useful, ⭐⭐⭐ Useful, ⭐⭐ Additional practice, ⭐ Optional.
- **Difficulty:** taken from the source platform. LeetCode uses Easy/Medium/Hard. GeeksforGeeks
  uses Basic/Easy/Medium/Hard. No difficulty rating here is invented.

---

## Core Problems

Target these during the two-week Maths phase.

### Stage 1 — Basic Number and Digit Manipulation

| # | Problem | Source | Problem # | Difficulty | Maths Concept | Importance | Status |
|---|---------|--------|-----------|------------|---------------|------------|--------|
| 1 | [Count Digits in a Number](https://www.geeksforgeeks.org/problems/count-digits-1606889545/1) | GeeksforGeeks | — | Easy | Digit counting, division loop | ⭐⭐⭐⭐⭐ | Not Started |
```python
class Solution:
    def countDigits(self, n):
        if n<10:
            return 1
        # code here
        return 1 + self.countDigits(n//10)
```
| 2 | [Sum Of Digits](https://www.geeksforgeeks.org/problems/sum-of-digits1742/1) | GeeksforGeeks | — | Easy | Digit extraction, sum of digits | ⭐⭐⭐⭐⭐ | Not Started |
```python
class Solution:
    def sumOfDigits(self, n):
        count =0
        # code here
        while n>0:
            count += n % 10
            n //= 10
        return count;
```
| 3 | [Reverse Digits](https://www.geeksforgeeks.org/problems/reverse-digit0316/1) | GeeksforGeeks | — | Basic | Reversing a number | ⭐⭐⭐⭐⭐ | Not Started |
```python
	def reverseDigits(self, n):
		# Code here
		num = n
		rev_num = 0
		while num >0:
		    digit =0
		    digit = num %10
		    rev_num = rev_num*10 +digit
		    num//=10
		return rev_num
```
| 4 | [Reverse Integer](https://leetcode.com/problems/reverse-integer/) | LeetCode | 7 | Medium | Reversing a number, integer overflow | ⭐⭐⭐⭐⭐ | Not Started |
```java
class Solution {
    public int reverse(int x) {
        long rnum = 0;
        while (x != 0) {
        rnum = rnum * 10 + (x%10);
        x /= 10;
        }
        if(rnum > Integer.MAX_VALUE || rnum < Integer.MIN_VALUE) {
            return 0;
        }
        return (int)rnum;
    }
}
```
| 5 | [Palindrome Number](https://leetcode.com/problems/palindrome-number/) | LeetCode | 9 | Easy | Palindrome, digit manipulation | ⭐⭐⭐⭐⭐ | Not Started |
```java
class Solution {
  public boolean isPalindrome(int x) {
    if (x<0 || (x!=0 && x%10 == 0)) {
      return false;
    }
    int revhalf = 0;
    while(x>0 && revhalf <x) {
      revhalf = revhalf*10 + x%10;
      x/=10;
    }
    return (x==revhalf || x == revhalf/10);

  }
}
```
| 6 | [Subtract the Product and Sum of Digits of an Integer](https://leetcode.com/problems/subtract-the-product-and-sum-of-digits-of-an-integer/) | LeetCode | 1281 | Easy | Product and sum of digits | ⭐⭐⭐⭐ | Not Started |
```java
class Solution {
    public int subtractProductAndSum(int n) {
        int p = 1;
        int t = 0;
        while (n>0) {
            int d = n%10;
            p = d*p;
            t = d+t;
            n /= 10;
        }
        return p-t;
    }
}
```
| 7 | [Add Digits](https://leetcode.com/problems/add-digits/) | LeetCode | 258 | Easy | Repeated digit sum, digital root | ⭐⭐⭐⭐ | Not Started |
```java
class Solution {
    public int addDigits(int num) {
     return num == 0 ? 0 : 1 + (num - 1) % 9;
    }
}
```
| 8 | [Count the Digits That Divide a Number](https://leetcode.com/problems/count-the-digits-that-divide-a-number/) | LeetCode | 2520 | Easy | Digit extraction, divisibility | ⭐⭐⭐⭐ | Not Started |
```java
class Solution {
    public int countDigits(int num) {
        int n = num;
        int count = 0;
        while(num>0) {
            int digit = num%10;
            if(n%digit==0){
                count++;
            }
            num = num/10;
        }
        return count;
    }
}
```
| 9 | [Find Numbers with Even Number of Digits](https://leetcode.com/problems/find-numbers-with-even-number-of-digits/) | LeetCode | 1295 | Easy | Digit counting | ⭐⭐⭐ | Not Started |
```java
class Solution {
    public int findNumbers(int[] nums) {
        int count = 0;
        for(int num : nums) {
            if(num>=10 && num<100 || num>=1000 && num<10000 || num==100000) {
                count++;
            }
        }
        return count;
    }
}
```
| 10 | [Fizz Buzz](https://leetcode.com/problems/fizz-buzz/) | LeetCode | 412 | Easy | Modulo, divisibility | ⭐⭐⭐⭐ | Not Started |
```java
class Solution {
    public List<String> fizzBuzz(int n) {
        List<String> ans = new ArrayList<>(n);
        for (int i = 1, fizz = 0, buzz = 0; i <= n; i++) {
            fizz++; buzz++;
            if (fizz == 3 && buzz == 5) { ans.add("FizzBuzz"); fizz = 0; buzz = 0; }
            else if (fizz == 3)         { ans.add("Fizz");     fizz = 0; }
            else if (buzz == 5)         { ans.add("Buzz");     buzz = 0; }
            else                        ans.add(String.valueOf(i));
        }
        return ans;
    }
}
```
| 11 | [Divisible and Non-divisible Sums Difference](https://leetcode.com/problems/divisible-and-non-divisible-sums-difference/) | LeetCode | 2894 | Easy | Divisibility, summation | ⭐⭐⭐ | Not Started |
```java
class Solution {
    public int differenceOfSums(int n, int m) {
        int total = n*(n+1)/2;
        int k = n / m;
        int num2 = m*k*(k+1)/2;
        return total - 2 * num2;
    }
}
```

### Stage 2 — Number Properties

| # | Problem | Source | Problem # | Difficulty | Maths Concept | Importance | Status |
|---|---------|--------|-----------|------------|---------------|------------|--------|
| 12 | [Prime Number](https://www.geeksforgeeks.org/problems/prime-number2314/1) | GeeksforGeeks | — | Easy | Primality test up to sqrt(n) | ⭐⭐⭐⭐⭐ | Not Started |
| 13 | [Count Primes](https://leetcode.com/problems/count-primes/) | LeetCode | 204 | Medium | Primes in a range, sieve | ⭐⭐⭐⭐⭐ | Not Started |
| 14 | [All Divisors of a Number](https://www.geeksforgeeks.org/problems/all-divisors-of-a-number/1) | GeeksforGeeks | — | Easy | Factors, divisor pairing | ⭐⭐⭐⭐⭐ | Not Started |
| 15 | [Three Divisors](https://leetcode.com/problems/three-divisors/) | LeetCode | 1952 | Easy | Counting divisors | ⭐⭐⭐ | Not Started |
| 16 | [Armstrong Numbers](https://www.geeksforgeeks.org/problems/armstrong-numbers2727/1) | GeeksforGeeks | — | Easy | Number property, digit powers | ⭐⭐⭐⭐⭐ | Not Started |
| 17 | [Perfect Number](https://leetcode.com/problems/perfect-number/) | LeetCode | 507 | Easy | Sum of proper divisors | ⭐⭐⭐⭐ | Not Started |
| 18 | [Strong Numbers](https://www.geeksforgeeks.org/problems/strong-numbers3315/1) | GeeksforGeeks | — | Basic | Factorial of digits | ⭐⭐⭐⭐ | Not Started |
| 19 | [Happy Number](https://leetcode.com/problems/happy-number/) | LeetCode | 202 | Easy | Digit squares, cycle detection | ⭐⭐⭐⭐⭐ | Not Started |
| 20 | [Harshad Number](https://leetcode.com/problems/harshad-number/) | LeetCode | 3099 | Easy | Divisibility by digit sum | ⭐⭐⭐ | Not Started |
| 21 | [Ugly Number](https://leetcode.com/problems/ugly-number/) | LeetCode | 263 | Easy | Prime factor reduction | ⭐⭐⭐ | Not Started |

### Stage 3 — Mathematical Algorithms (GCD, LCM, Factorization)

| # | Problem | Source | Problem # | Difficulty | Maths Concept | Importance | Status |
|---|---------|--------|-----------|------------|---------------|------------|--------|
| 22 | [GCD of two numbers](https://www.geeksforgeeks.org/problems/gcd-of-two-numbers3459/1) | GeeksforGeeks | — | Basic | GCD, Euclidean algorithm | ⭐⭐⭐⭐⭐ | Not Started |
| 23 | [LCM And GCD](https://www.geeksforgeeks.org/problems/lcm-and-gcd4516/1) | GeeksforGeeks | — | Basic | LCM via GCD relation | ⭐⭐⭐⭐⭐ | Not Started |
| 24 | [Find Greatest Common Divisor of Array](https://leetcode.com/problems/find-greatest-common-divisor-of-array/) | LeetCode | 1979 | Easy | GCD of multiple numbers | ⭐⭐⭐⭐ | Not Started |
| 25 | [Number of Common Factors](https://leetcode.com/problems/number-of-common-factors/) | LeetCode | 2427 | Easy | Common factors, GCD reasoning | ⭐⭐⭐⭐ | Not Started |
| 26 | [Smallest Even Multiple](https://leetcode.com/problems/smallest-even-multiple/) | LeetCode | 2413 | Easy | LCM reasoning | ⭐⭐⭐ | Not Started |
| 27 | [All Prime Factors in Any Order](https://www.geeksforgeeks.org/problems/prime-factorization-using-sieve/1) | GeeksforGeeks | — | Medium | Prime factorization using sieve | ⭐⭐⭐⭐⭐ | Not Started |
| 28 | [Sieve of Eratosthenes](https://www.geeksforgeeks.org/problems/sieve-of-eratosthenes5242/1) | GeeksforGeeks | — | Medium | Sieve, primes in a range | ⭐⭐⭐⭐ | Not Started |
| 29 | [Factorial](https://www.geeksforgeeks.org/problems/factorial5739/1) | GeeksforGeeks | — | Basic | Factorial, iterative product | ⭐⭐⭐⭐ | Not Started |
| 30 | [Factorial Trailing Zeroes](https://leetcode.com/problems/factorial-trailing-zeroes/) | LeetCode | 172 | Medium | Factorial, factors of 5 | ⭐⭐⭐⭐ | Not Started |

### Stage 4 — Sequences

| # | Problem | Source | Problem # | Difficulty | Maths Concept | Importance | Status |
|---|---------|--------|-----------|------------|---------------|------------|--------|
| 31 | [Fibonacci Number](https://leetcode.com/problems/fibonacci-number/) | LeetCode | 509 | Easy | Fibonacci sequence | ⭐⭐⭐⭐⭐ | Not Started |
| 32 | [Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) | LeetCode | 70 | Easy | Fibonacci recurrence | ⭐⭐⭐⭐ | Not Started |
| 33 | [AP Term](https://www.geeksforgeeks.org/problems/ap-term/1) | GeeksforGeeks | — | Easy | Arithmetic progression nth term | ⭐⭐⭐ | Not Started |
| 34 | [Arranging Coins](https://leetcode.com/problems/arranging-coins/) | LeetCode | 441 | Easy | Triangular numbers, series sum | ⭐⭐⭐ | Not Started |
| 35 | [Missing Number](https://leetcode.com/problems/missing-number/) | LeetCode | 268 | Easy | Gauss sum formula | ⭐⭐⭐⭐ | Not Started |

### Stage 5 — Powers and Roots

| # | Problem | Source | Problem # | Difficulty | Maths Concept | Importance | Status |
|---|---------|--------|-----------|------------|---------------|------------|--------|
| 36 | [Power of Two](https://leetcode.com/problems/power-of-two/) | LeetCode | 231 | Easy | Powers of two | ⭐⭐⭐⭐⭐ | Not Started |
| 37 | [Power of Three](https://leetcode.com/problems/power-of-three/) | LeetCode | 326 | Easy | Powers, repeated division | ⭐⭐⭐ | Not Started |
| 38 | [Sqrt(x)](https://leetcode.com/problems/sqrtx/) | LeetCode | 69 | Easy | Integer square root | ⭐⭐⭐⭐⭐ | Not Started |
| 39 | [Valid Perfect Square](https://leetcode.com/problems/valid-perfect-square/) | LeetCode | 367 | Easy | Perfect squares | ⭐⭐⭐⭐ | Not Started |
| 40 | [Pow(x, n)](https://leetcode.com/problems/powx-n/) | LeetCode | 50 | Medium | Fast exponentiation | ⭐⭐⭐⭐⭐ | Not Started |

### Stage 6 — Binary and Basic Bit Mathematics

| # | Problem | Source | Problem # | Difficulty | Maths Concept | Importance | Status |
|---|---------|--------|-----------|------------|---------------|------------|--------|
| 41 | [Decimal to binary](https://www.geeksforgeeks.org/problems/decimal-to-binary-1587115620/1) | GeeksforGeeks | — | Easy | Base conversion, decimal to binary | ⭐⭐⭐⭐ | Not Started |
| 42 | [Binary to Decimal](https://www.geeksforgeeks.org/problems/binary-number-to-decimal-number3525/1) | GeeksforGeeks | — | Basic | Base conversion, positional value | ⭐⭐⭐⭐ | Not Started |
| 43 | [Number of 1 Bits](https://leetcode.com/problems/number-of-1-bits/) | LeetCode | 191 | Easy | Set-bit counting | ⭐⭐⭐⭐⭐ | Not Started |
| 44 | [Add Binary](https://leetcode.com/problems/add-binary/) | LeetCode | 67 | Easy | Binary addition, carry handling | ⭐⭐⭐⭐ | Not Started |
| 45 | [Number Complement](https://leetcode.com/problems/number-complement/) | LeetCode | 476 | Easy | Bit complement | ⭐⭐⭐ | Not Started |

### Stage 7 — Basic Mathematical Simulation

| # | Problem | Source | Problem # | Difficulty | Maths Concept | Importance | Status |
|---|---------|--------|-----------|------------|---------------|------------|--------|
| 46 | [Number of Steps to Reduce a Number to Zero](https://leetcode.com/problems/number-of-steps-to-reduce-a-number-to-zero/) | LeetCode | 1342 | Easy | Repeated operations, simulation | ⭐⭐⭐⭐ | Not Started |
| 47 | [Excel Sheet Column Number](https://leetcode.com/problems/excel-sheet-column-number/) | LeetCode | 171 | Easy | Base-26 conversion | ⭐⭐⭐⭐ | Not Started |
| 48 | [Find Digits](https://www.hackerrank.com/challenges/find-digits/problem) | HackerRank | — | Easy | Digit divisibility counting | ⭐⭐⭐ | Not Started |

---

## Additional Problems

Optional practice once the Core list is finished.

| # | Problem | Source | Problem # | Difficulty | Maths Concept | Importance | Status |
|---|---------|--------|-----------|------------|---------------|------------|--------|
| 1 | [Self Dividing Numbers](https://leetcode.com/problems/self-dividing-numbers/) | LeetCode | 728 | Easy | Digit divisibility over a range | ⭐⭐⭐ | Not Started |
| 2 | [Automorphic Number](https://www.geeksforgeeks.org/problems/automorphic-number4721/1) | GeeksforGeeks | — | Basic | Number property, trailing digits | ⭐⭐ | Not Started |
| 3 | [Common Divisors](https://www.geeksforgeeks.org/problems/common-divisors4712/1) | GeeksforGeeks | — | Basic | Common divisors, GCD | ⭐⭐⭐ | Not Started |
| 4 | [Sign of the Product of an Array](https://leetcode.com/problems/sign-of-the-product-of-an-array/) | LeetCode | 1822 | Easy | Sign reasoning, positive/negative | ⭐⭐⭐ | Not Started |
| 5 | [Maximum 69 Number](https://leetcode.com/problems/maximum-69-number/) | LeetCode | 1323 | Easy | Digit replacement, place value | ⭐⭐ | Not Started |
| 6 | [A Number After a Double Reversal](https://leetcode.com/problems/a-number-after-a-double-reversal/) | LeetCode | 2119 | Easy | Reversal, trailing zeros | ⭐⭐ | Not Started |
| 7 | [Minimum Sum of Four Digit Number After Splitting Digits](https://leetcode.com/problems/minimum-sum-of-four-digit-number-after-splitting-digits/) | LeetCode | 2160 | Easy | Digit rearrangement | ⭐⭐ | Not Started |
| 8 | [Count Symmetric Integers](https://leetcode.com/problems/count-symmetric-integers/) | LeetCode | 2843 | Easy | Digit-half sums | ⭐⭐⭐ | Not Started |
| 9 | [Sum of Digits in Base K](https://leetcode.com/problems/sum-of-digits-in-base-k/) | LeetCode | 1837 | Easy | Base conversion, digit sum | ⭐⭐⭐ | Not Started |
| 10 | [Base 7](https://leetcode.com/problems/base-7/) | LeetCode | 504 | Easy | Base conversion | ⭐⭐⭐ | Not Started |
| 11 | [Convert a Number to Hexadecimal](https://leetcode.com/problems/convert-a-number-to-hexadecimal/) | LeetCode | 405 | Easy | Base-16 conversion, two's complement | ⭐⭐⭐ | Not Started |
| 12 | [Counting Bits](https://leetcode.com/problems/counting-bits/) | LeetCode | 338 | Easy | Set-bit counting patterns | ⭐⭐⭐ | Not Started |
| 13 | [Reverse Bits](https://leetcode.com/problems/reverse-bits/) | LeetCode | 190 | Easy | Bit reversal | ⭐⭐⭐ | Not Started |
| 14 | [Hamming Distance](https://leetcode.com/problems/hamming-distance/) | LeetCode | 461 | Easy | XOR, bit counting | ⭐⭐⭐ | Not Started |
| 15 | [Number of Even and Odd Bits](https://leetcode.com/problems/number-of-even-and-odd-bits/) | LeetCode | 2595 | Easy | Bit positions | ⭐⭐ | Not Started |
| 16 | [Power of Four](https://leetcode.com/problems/power-of-four/) | LeetCode | 342 | Easy | Powers, bit reasoning | ⭐⭐⭐ | Not Started |
| 17 | [Perfect Squares](https://leetcode.com/problems/perfect-squares/) | LeetCode | 279 | Medium | Squares decomposition | ⭐⭐⭐ | Not Started |
| 18 | [Sum of Square Numbers](https://leetcode.com/problems/sum-of-square-numbers/) | LeetCode | 633 | Medium | Squares, two-sum on squares | ⭐⭐⭐ | Not Started |
| 19 | [Sherlock and Squares](https://www.hackerrank.com/challenges/sherlock-and-squares/problem) | HackerRank | — | Easy | Counting squares in a range | ⭐⭐⭐ | Not Started |
| 20 | [Modular Exponentiation](https://www.geeksforgeeks.org/problems/modular-exponentiation-for-large-numbers5537/1) | GeeksforGeeks | — | Medium | Modular arithmetic, binary exponentiation | ⭐⭐⭐ | Not Started |
| 21 | [Roman to Integer](https://leetcode.com/problems/roman-to-integer/) | LeetCode | 13 | Easy | Numeral systems | ⭐⭐⭐⭐ | Not Started |
| 22 | [Integer to Roman](https://leetcode.com/problems/integer-to-roman/) | LeetCode | 12 | Medium | Numeral systems, greedy value mapping | ⭐⭐⭐ | Not Started |
| 23 | [Excel Sheet Column Title](https://leetcode.com/problems/excel-sheet-column-title/) | LeetCode | 168 | Easy | Base-26 conversion, reverse direction | ⭐⭐⭐ | Not Started |
| 24 | [Can Make Arithmetic Progression From Sequence](https://leetcode.com/problems/can-make-arithmetic-progression-from-sequence/) | LeetCode | 1502 | Easy | Arithmetic progression check | ⭐⭐⭐ | Not Started |
| 25 | [Ugly Number II](https://leetcode.com/problems/ugly-number-ii/) | LeetCode | 264 | Medium | Number generation by prime factors | ⭐⭐⭐ | Not Started |
| 26 | [Nim Game](https://leetcode.com/problems/nim-game/) | LeetCode | 292 | Easy | Modulo pattern reasoning | ⭐⭐⭐ | Not Started |
| 27 | [Construct the Rectangle](https://leetcode.com/problems/construct-the-rectangle/) | LeetCode | 492 | Easy | Factor pairs, sqrt bound | ⭐⭐ | Not Started |
| 28 | [Count of Matches in Tournament](https://leetcode.com/problems/count-of-matches-in-tournament/) | LeetCode | 1688 | Easy | Counting simulation | ⭐⭐ | Not Started |
| 29 | [Nth Digit](https://leetcode.com/problems/nth-digit/) | LeetCode | 400 | Medium | Digit counting across ranges | ⭐⭐ | Not Started |

---

## Verification Notes

- LeetCode numbers, official titles and difficulties were checked against LeetCode's own
  public problem index (`leetcode.com/api/problems/algorithms/`) rather than from memory.
- GeeksforGeeks titles and difficulty labels were read from each problem page; every URL in
  this file returned HTTP 200.
- HackerRank difficulty for "Find Digits" and "Sherlock and Squares" was read from the
  problem pages (both Easy).
- LeetCode 1134 "Armstrong Number" exists but is **Premium-only**, so the GeeksforGeeks
  version is used instead.
- No LeetCode or judge-backed problem was found for a few very basic topics: even/odd check,
  swapping two numbers, absolute value, and average/max/min of two numbers. They are excluded
  rather than linked to an unverified page. They are trivial enough to practise without a judge.
- "Nth Fibonacci Number" (GeeksforGeeks) was dropped as a duplicate of LeetCode 509.
- Nothing in this file is marked *Verification Required*.
