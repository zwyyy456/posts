---
title: "Exponentiating by Squaring"
date: 2022-11-19T11:51:31+08:00
lastmod: 2022-11-19T11:51:31+08:00 #更新时间
author: ["zwyyy456"] #作者
categories: ["tech"]
tags: ["data structure and algorithms"]
description: "" #描述
weight: # 输入 1 可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: false #是否展示评论
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: false #顶部显示当前路径
---

## Definition

Fast exponentiation, also known as binary exponentiation (or exponentiation by squaring), is a technique to compute \( a^n \) in \( \Theta(\log n) \) time, whereas a brute-force calculation would require \( \Theta(n) \) time.

This technique is often used in non-computational scenarios as well because it can be applied to any operation that has an associative property. Notably, it can be applied to modular exponentiation, matrix exponentiation, and other operations, which we will discuss next.

## Explanation

Calculating the \( n \)th power of \( a \) means multiplying \( n \) instances of \( a \) together: \( a^{n} = \underbrace{a \times a \cdots \times a}_{n \text{ times}} \). However, when \( a \) and \( n \) are too large, this method becomes impractical. But we know that: \( a^{b+c} = a^b \cdot a^c \) and \( a^{2b} = a^b \cdot a^b = (a^b)^2 \). The idea behind binary exponentiation is to break the task of exponentiation into smaller tasks according to the binary representation of the exponent.

## Process

First, represent \( n \) in binary. For example:

$$
3^{13} = 3^{(1101)_2} = 3^8 \cdot 3^4 \cdot 3^1
$$

Since \( n \) has \( \lfloor \log_2 n \rfloor + 1 \) binary digits, once we know \( a^1, a^2, a^4, a^8, \dots, a^{2^{\lfloor \log_2 n \rfloor}} \), we only need to perform \( \Theta(\log n) \) multiplications to calculate \( a^n \).

Thus, we only need a fast way to calculate the above sequence of powers of 3. This problem is simple because any element in the sequence (except the first one) is just the square of the previous element. For example:

$$
\begin{align}
3^1 &= 3 \\
3^2 &= \left(3^1\right)^2 = 3^2 = 9 \\
3^4 &= \left(3^2\right)^2 = 9^2 = 81 \\
3^8 &= \left(3^4\right)^2 = 81^2 = 6561
\end{align}
$$

So, to compute \( 3^{13} \), we simply multiply the powers corresponding to the binary digits of 1:

$$
3^{13} = 6561 \cdot 81 \cdot 3 = 1594323
$$

Formalizing the above process, if we write \( n \) in binary as \( (n_tn_{t-1}\cdots n_1n_0)_2 \), then:

$$
n = n_t2^t + n_{t-1}2^{t-1} + n_{t-2}2^{t-2} + \cdots + n_12^1 + n_02^0
$$

where \( n_i \in \{0,1\} \). Then:

$$
\begin{aligned}
a^n & = (a^{n_t 2^t + \cdots + n_0 2^0})\\\\
& = a^{n_0 2^0} \times a^{n_1 2^1} \times \cdots \times a^{n_t2^t}
\end{aligned}
$$

From the above, we see that the original problem is transformed into the product of subproblems of the same form, and we can deduce the \( 2^{i+1} \) term in constant time from the \( 2^i \) term.

This algorithm has a complexity of \( \Theta(\log n) \). We compute \( \Theta(\log n) \) powers of \( 2^k \), and then spend \( \Theta(\log n) \) time multiplying the powers corresponding to the binary digits of 1.

## Implementation

First, we can directly implement this method using recursion:

```cpp
// C++ Version
long long binpow(long long a, long long b) {
  if (b == 0) return 1;
  long long res = binpow(a, b / 2);
  if (b % 2)
    return res * res * a;
  else
    return res * res;
}
```

```python
# Python Version
def binpow(a, b):
    if b == 0:
        return 1
    res = binpow(a, b // 2)
    if (b % 2) == 1:
        return res * res * a
    else:
        return res * res
```

The second method is iterative. During the loop, the power corresponding to the binary digit of 1 is multiplied into the result. Although both have the same theoretical complexity, the second method is faster in practice due to the overhead of recursion.

```cpp
// C++ Version
long long binpow(long long a, long long b) {
  long long res = 1;
  while (b > 0) {
    if (b & 1) res = res * a;
    a = a * a;
    b >>= 1;
  }
  return res;
}
```

```python
# Python Version
def binpow(a, b):
    res = 1
    while b > 0:
        if (b & 1):
            res = res * a
        a = a * a
        b >>= 1
    return res
```

Template: [Luogu P1226](https://www.luogu.com.cn/problem/P1226)

## Applications

### Modular Exponentiation

This is a very common application, such as computing the modular multiplicative inverse.

Since we know that modular operations do not interfere with multiplication, we can simply take the modulus during the calculation.

```cpp
// C++ Version
long long binpow(long long a, long long b, long long m) {
  a %= m;
  long long res = 1;
  while (b > 0) {
    if (b & 1) res = res * a % m;
    a = a * a % m;
    b >>= 1;
  }
  return res;
}
```

```python
# Python Version
def binpow(a, b, m):
    a = a % m
    res = 1
    while b > 0:
        if (b & 1):
            res = res * a % m
        a = a * a % m
        b >>= 1
    return res
```

Note: According to Fermat's Little Theorem, if \( m \) is a prime number, we can compute \( x^{n \mod (m-1)} \) to speed up the algorithm.

### Computing Fibonacci Numbers

Using the recursive formula of the Fibonacci sequence \( F_n = F_{n-1} + F_{n-2} \), we can construct a \( 2 \times 2 \) matrix to represent the transformation from \( F_i, F_{i+1} \) to \( F_{i+1}, F_{i+2} \). Then, when computing the \( n \)th power of this matrix, we can use the idea of fast exponentiation to obtain the result in \( \Theta(\log n) \) time. For more details, see [Fibonacci Sequence](./combinatorics/fibonacci.md).

### Multiple Permutations

Given a sequence of length \( n \) and a permutation, apply this permutation \( k \) times.

Simply compute the \( k \)th power of this permutation and apply it to the sequence. The time complexity is \( O(n \log k) \).

Note: By building a graph for the permutation and computing the \( k \)th power on each cycle (in fact, just performing the operation modulo the cycle length), you can achieve a more efficient algorithm with a complexity of \( O(n) \).

### Accelerating Operations on Point Sets in Geometry

#### Introduction

> In three-dimensional space, given \( n \) points \( p_i \), apply \( m \) operations to these points. The operations include three types:
>
> 1. Shift the position of the points along a vector (Shift).
> 2. Scale the coordinates of the points (Scale).
> 3. Rotate around an axis (Rotate).
>
> There is also a special operation, which is to repeat a sequence of operations \( k \) times (Loop). The sequence may also contain Loop operations (Loop operations can be nested). Now, you need to apply these transformations to the \( n \) points in less than \( O(n \cdot \textit{length}) \) time, where \( \textit{length} \) represents the length of the expanded operation sequence after unfolding all Loop operations.

#### Explanation

Let's observe the effect of these three operations on the coordinates:

1. Shift Operation: Add a constant to each coordinate dimension;
2. Scale Operation: Multiply each coordinate dimension by a constant;
3. Rotate Operation: This is more complex, and we won't go into detail, but we can still represent the new coordinates using a linear combination.

As we can see, each transformation can be represented as a

 linear operation on the coordinates. Therefore, a transformation can be represented by a \( 4 \times 4 \) matrix:

$$
\begin{bmatrix}
a_{11} & a_ {12} & a_ {13} & a_ {14} \\
a_{21} & a_ {22} & a_ {23} & a_ {24} \\
a_{31} & a_ {32} & a_ {33} & a_ {34} \\
a_{41} & a_ {42} & a_ {43} & a_ {44} \\
\end{bmatrix}
$$

Using this matrix, we can transform a coordinate (vector) to obtain the new coordinate (vector):

$$
\begin{bmatrix}
a_{11} & a_ {12} & a_ {13} & a_ {14} \\
a_{21} & a_ {22} & a_ {23} & a_ {24} \\
a_{31} & a_ {32} & a_ {33} & a_ {34} \\
a_{41} & a_ {42} & a_ {43} & a_ {44} \\
\end{bmatrix} \cdot
\begin{bmatrix} x \\ y \\ z \\ 1 \end{bmatrix}
 = \begin{bmatrix} x' \\ y' \\ z' \\ 1 \end{bmatrix}
$$

You might wonder, why does a three-dimensional coordinate have an extra 1? The reason is that without this extra 1, we couldn't use matrix linear transformations to describe the Shift operation.

#### Process

Let's illustrate our idea with some simple examples:

1. Shift Operation: Shift the \( x \) coordinate by 5, the \( y \) coordinate by 7, and the \( z \) coordinate by 9:

    $$
    \begin{bmatrix}
    1 & 0 & 0 & 5 \\
    0 & 1 & 0 & 7 \\
    0 & 0 & 1 & 9 \\
    0 & 0 & 0 & 1 \\
    \end{bmatrix}
    $$

2. Scale Operation: Scale the \( x \) coordinate by 10, and the \( y, z \) coordinates by 5:

    $$
    \begin{bmatrix}
    10 & 0 & 0 & 0 \\
    0 & 5 & 0 & 0 \\
    0 & 0 & 5 & 0 \\
    0 & 0 & 0 & 1 \\
    \end{bmatrix}
    $$

3. Rotate Operation: Rotate around the \( x \) axis by \( \theta \) radians, following the right-hand rule (counterclockwise):

    $$
    \begin{bmatrix}
    1 & 0 & 0 & 0 \\
    0 & \cos \theta & \sin \theta & 0 \\
    0 & -\sin \theta & \cos \theta & 0 \\
    0 & 0 & 0 & 1 \\
    \end{bmatrix}
    $$

Now, each operation is represented as a matrix, and the transformation sequence can be represented by the product of these matrices. A Loop operation is equivalent to taking the \( k \)th power of a matrix. This allows us to compute the final matrix representing the entire transformation sequence in \( O(m \log k) \) time. Finally, applying this matrix to \( n \) points yields a total complexity of \( O(n + m \log k) \).

### Counting Fixed-Length Paths

Given a directed graph (with edge weights of 1), find the number of paths of length \( k \) between any two vertices \( u, v \).

Take the adjacency matrix \( M \) of the graph and compute its \( k \)th power. Then \( M_{i,j} \) represents the number of paths of length \( k \) from \( i \) to \( j \). The complexity of this algorithm is \( O(n^3 \log k) \). For more details, see [Matrices](./linear-algebra/matrix.md).

### Modular Multiplication of Large Integers

> Compute \( a \times b \mod m, \,\, a,b \leq m \leq 10^{18} \).

Similar to binary exponentiation, this time we represent one of the multiplicands as a sum of integer powers of 2. Since performing a multiplication by 2 and taking a modulus can be converted into an addition or subtraction operation to prevent overflow, this can still be solved in \( O(\log_2 m) \) time. The recursive method is as follows:

$$
a \cdot b = \begin{cases}
0 &\text{if }a = 0 \\\\
2 \cdot \frac{a}{2} \cdot b &\text{if }a > 0 \text{ and }a \text{ even} \\\\
2 \cdot \frac{a-1}{2} \cdot b + b &\text{if }a > 0 \text{ and }a \text{ odd}
\end{cases}
$$

#### Fast Multiplication

However, the \( O(\log_2 m) \) "turtle multiplication" is still too slow. In many algorithms that require a high constant speed, such as Miller-Rabin and Pollard-Rho, this is insufficient. Therefore, we introduce a method that handles modulus within the `long long` range, does not require using `__int128` black magic, and has a complexity of \( O(1) \).

We find that:

$$
a \times b \mod m = a \times b - \left\lfloor \dfrac{ab}{m} \right\rfloor \times m
$$

We cleverly use the natural overflow of `unsigned long long`:

$$
a \times b \mod m = a \times b - \left\lfloor \dfrac{ab}{m} \right\rfloor \times m = \left(a \times b - \left\lfloor \dfrac{ab}{m} \right\rfloor \times m \right) \mod 2^{64}
$$

After calculating \( \left\lfloor \dfrac{ab}{m} \right\rfloor \), both the multiplication and subtraction on the left side and the subtraction in the middle can be directly calculated using `unsigned long long`. Now we only need to figure out how to calculate \( \left\lfloor \dfrac{ab}{m} \right\rfloor \).

We consider using `long double` to first calculate \( \dfrac{a}{m} \), then multiply it by \( b \).

Since `long double` is used, there will undoubtedly be precision errors. The extreme case is when the first significant digit (in binary) is one place after the decimal point. On an `x86-64` machine, `long double` is interpreted as an 80-bit extended float (i.e., 1 bit for the sign, 15 bits for the exponent, and 64 bits for the mantissa), so `long double` can represent up to 64 significant digits with precision[^note1]. Therefore, the precision error starts from the 65th digit, with an error range of \( \left(-2^{-64},2^{64}\right) \). Multiplying by the 64-bit integer \( b \), the error range becomes \( (-0.5,0.5) \). After adding 0.5, the error range becomes \( (0,1) \). Taking the integer part, the error range becomes \( \{0,1\} \). Thus, after multiplying by \( -m \), the error range becomes \( \{0,-m\} \), and we need to handle these two cases.

Since \( m \) is within the `long long` range, if the calculated result \( r \) is in the range \( [0,m) \), return \( r \) directly; otherwise, return \( r + m \). Of course, you can also directly return \( (r + m) \mod m \).

The code implementation is as follows:

```cpp
long long binmul(long long a, long long b, long long m) {
  unsigned long long c =
      (unsigned long long)a * b -
      (unsigned long long)((long double)a / m * b + 0.5L) * m;
  if (c < m) return c;
  return c + m;
}
```

### High-Precision Fast Exponentiation

Please first study [High Precision](./bignum.md)

> Example problem [NOIP2003 Modified from Basic Group·Mersenne Number](https://www.luogu.com.cn/problem/P1045):
> Given \( P \) (1000 < P < 3100000), compute the last 100 digits of \( 2^P-1 \) (in decimal high-precision format), with leading zeros if necessary.

The code implementation is as follows:

```cpp
--8<-- "docs/math/code/quick-pow/quick-pow_1.cpp"
```

### Preprocessed Fast Exponentiation with a Common Base and Modulus

In the case of a common base and modulus, we can use block techniques to preprocess in a certain amount of time (usually \( O(\sqrt n)

 \)) and then answer a power query in \( O(1) \) time.

#### Process

1. Choose a number \( s \), preprocess the values from \( a^0 \) to \( a^s \) and \( a^{0 \cdot s} \) to \( a^{\lceil \frac{p}{s} \rceil \cdot s} \), and store them in one (or two) arrays;
2. For each query \( a^b \mod p \), break \( b \) down into \( \left\lfloor \dfrac{b}{s} \right\rfloor \cdot s + b \mod s \), so that \( a^b = a^{\lfloor \frac{b}{s} \rfloor \cdot s} \times a^{b \mod s} \), and the answer can be obtained in \( O(1) \) time.

For the choice of this number \( s \), we generally choose \( \sqrt p \) or an appropriate power of 2 (choosing \( \sqrt p \) makes preprocessing optimal, while choosing a power of 2 can simplify the computation through bitwise operations).

> Reference Code
```cpp
int pow1[65536], pow2[65536];

void preproc(int a, int mod) {
    pow1[0] = pow2[0] = 1;
    for (int i = 1; i < 65536; i++) pow1[i] = 1LL * pow1[i - 1] * a % mod;
    int pow65536 = 1LL * pow1[65535] * a % mod;
    for (int i = 1; i < 65536; i++) pow2[i] = 1LL * pow2[i - 1] * pow65536 % mod;
}

int query(int pows) { return 1LL * pow1[pows & 65535] * pow2[pows >> 16]; }
```

