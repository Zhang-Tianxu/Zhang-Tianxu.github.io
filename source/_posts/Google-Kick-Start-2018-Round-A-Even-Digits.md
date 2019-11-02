---
title: 'Google Kick Start 2018 Round A: Even Digits'
date: 2019-11-01 21:20:21
tags:
    - Python
    - Google Kick Start
categories:
  - Coding
  - Python
---

# Google Kick Start 2018 Round A: Even Digits

## Problem

Supervin has a unique calculator. This calculator only has a display, a plus button, and a minus button. Currently, the integer **N** is displayed on the calculator display.

Pressing the plus button increases the current number displayed on the calculator display by 1. Similarly, pressing the minus button decreases the current number displayed on the calculator display by 1. The calculator does not display any leading zeros. For example, if `100` is displayed on the calculator display, pressing the minus button once will cause the calculator to display `99`.

Supervin does not like odd digits, because he thinks they are "odd". Therefore, he wants to display an integer with only even digits in its decimal representation, using only the calculator buttons. Since the calculator is a bit old and the buttons are hard to press, he wants to use a minimal number of button presses.

Please help Supervin to determine the minimum number of button presses to make the calculator display an integer with no odd digits.



<!--more-->



**input:**

The first line of the input gives the number of test cases, **T**. **T** test cases follow. Each begins with one line containing an integer **N**: the integer initially displayed on Supervin's calculator.

**output:**

For each test case, output one line containing `Case #x: y`, where `x` is the test case number (starting from 1) and `y` is the minimum number of button presses, as described above.

**limits:**

1 ≤ **T** ≤ 100.
Time limit: 20 seconds per test set.
Memory limit: 1GB.

Small dataset (Test set 1 - Visible)

1 ≤ **N** ≤ 105.

Large dataset (Test set 2 - Hidden)

1 ≤ **N** ≤ 1016.

**sample:**

| Input                              | Output                                                       |
| ---------------------------------- | ------------------------------------------------------------ |
| 4<br />42<br />11<br />1<br />2018 | Case #1: 0 <br />Case #2: 3 <br />Case #3: 1 <br />Case #4: 2 |

## solution

1. find 2 numbers, whose all digits are even, closest to **N** on both sides, namely `b` and `s`, `b` is bigger than **N**, `s` is smaller than **N**. Process as follows:
   1. find highest odd digit in N, if N = 4389901, then highest odd digit is 3.
   2. highest odd digit minus 1, and set all digits below to 8, then we get `s`. If N = 4389901, then `s` = 4288888.
   3. highest odd digit plus 1, and set all digits below to 0, in most cases, we get `b`. If N = 4389901, then `b` = 4400000
   4. in 3 I said in most cases, because if highest odd digit is 9, we can't get `b` by 3's method. But fortunately if highest odd digit is 9, `b - N > N -s`. Means we needn't condider `b`.
2. Print `min(b - N, N - s)`

I am not satisfied with this method, because it's not a universal method like knapsack problem.

## Code

```python
def find_highest_odd_bit(num):
    h_odd_bit = 0
    count = 0
    while(num):
        count += 1
        if (num % 10) % 2 == 1 and count > h_odd_bit:
            h_odd_bit = count
        num = int(num/10)
    return h_odd_bit


def num_tail(num,bit):
    return num % 10**bit

def n_8(n):
    result = 0
    while(n):
        result += 8 * 10**(n-1)
        n -= 1
    return result

def even_digits(num,t):
    highest_odd_bit = find_highest_odd_bit(num)
    if(highest_odd_bit == 0):
        print("Case #{}: {}".format(t,0))
        return
    #print(str(num)[-highest_odd_bit])
    n_tail1 = num_tail(num,highest_odd_bit - 1)
    n_tail2 = n_tail1 + 10**(highest_odd_bit - 1)
    #n_tail2 = num_tail(num,highest_odd_bit)
    top = 10**(highest_odd_bit-1)
    bottom = n_8(highest_odd_bit - 1)
    if(int(str(num)[-highest_odd_bit]) == 9):
        print("Case #{}: {}".format(t,n_tail2 - bottom))
        return

    print("Case #{}: {}".format(t,min(top - n_tail1,n_tail2 - bottom)))
    
def main():
    T = int(input())
    for i in range(T):
        n = int(input())
        even_digits(n,i+1)


if __name__ == '__main__':
    main()
```

