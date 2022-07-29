---
layout: post
title: Understanding Reed-Solomon Error Correction Basics with Integer Math Only
date:  2022-07-19 00:00:00 -1000
categories:
---

<script type="text/x-mathjax-config">
  MathJax.Hub.Config({
    jax: ["input/TeX", "output/HTML-CSS"],
    tex2jax: {
      inlineMath: [ ['$', '$'], ["\\(", "\\)"] ],
      displayMath: [ ['$$', '$$'], ["\\[", "\\]"] ],
      processEscapes: true,
      skipTags: ['script', 'noscript', 'style', 'textarea', 'pre', 'code']
    }
    //,
    //displayAlign: "left",
    //displayIndent: "2em"
  });
</script>
<script src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS_HTML" type="text/javascript"></script>

* TOC
{:toc}

# Introduction

I've always been intimidated by coding techniques: encryption and decryption, hashing operations,
error correction codes, and even compression and decompression techniques. It's not that I
didn't know what they do, and there have been cases where I've used with them professionally, 
but I felt that I never quite understood the basics, let alone had an intuitive understanding of 
how they worked.

One coding technique that I've found intriguing was Reed-Solomon forward
error correction. Until the discovery of even better coding techniques (Turbo codes and
low-density parity codes), it was one of the most powerful ways to make data storage or
data transmission resilient against corruption: the Voyager spacecrafts used Reed-Solomon
coding to transmit images of planets past Saturn, and CDs can recover from scratches that 
corrupt up to 4000 bits thanks to some clever use of not one but two Reed-Solomon codes. 

The subject is covered in many college-level courses on coding and signal processing techniques, 
but a lot of the material online assumes a certain theoretical base and builds on that: start with 
Galois Fields and polynomials, throw a bunch of mathematical formulas, but in the end that 
intuitive understanding is still not there. At least not for me...

That changed when stumbled into this 
[Introduction to Reed-Solomon](https://innovation.vivint.com/introduction-to-reed-solomon-bc264d0794f8)
article. It explains how polynomials and polynomial evaluation at certain points are a way to
create a code with redundancy, and how to recover the original message back from it. The article
is excellent, and it makes some of that I'm covering below unnecessary or redundant (ha!), 
because some parts of what follows will be a recreation of that material. However it's dumbed down 
even more, and it tries to cover a larger variety of Reed-Solomon codes. 

One of the best parts of that article is the focus on integer math. Academic papers or books about 
coding theory almost always start with the theory of finite fields, aka Galois fields, and then 
build on that when explaining coding algorithms. I found this one of the bigger obstacles in 
understanding the fundamentals of Reed-Solomon coding (and BCH coding, a close relative): instead of 
getting to know one new topic, you have to tackle two at the same time. I'll be doing the same in 
this blog post: everything is integer math only. Because of that the resulting algorithms won't be 
practical for the real world, but it will result in a better feel about why finite fields are necessary.

Many books are written about coding theory and applications, and a single blog post won't even
scratch the surface of the topic. I'm also still only beginning to understand some of the
finer points. So my usual disclaimer applies: I'm writing these blog posts primarily for myself, 
as a way to solidify what I've learned after reading stuff on the web. Major errors are 
possible, inaccuracies should be expected.

You'll find some mathematical formulas below, but I always try to switch to real examples as soon
as possible.

# A Quick Recap on Polynomials 

It's impossible to discuss anything that's related to coding without touching the subject of polynomials.
I'm assume that you've learned about integer based polynomials during some algebra class in high school
or in college. A polynomial $$f(x)$$ of degree $$n$$ is a function that looks like this:

$$f(x) = c_0 + c_1 x + c_2 x^2 + ... c_n x^n$$

$$n+1$$ fixed coefficients $$c_i$$ are multiplied by function variable $$x$$ to the power of $$i$$. 
The polynomial can evaluated by replacing variable $$x$$ by some number for which you want to know the 
value of the function. You can also add, subtract, multiply or divide polynomials with each other.

Let's illustarte this with some examples, and define $$f(x)$$ and $$g(x)$$ as follows:

$$\begin{aligned}
f(x) & = 3 + 2 x + 5 x^2 - 4 x^3 \\
g(x) & = 7 x - x^2 \\
\end{aligned}$$

You add polynomials by adding together the coefficients that belong to the same $$x^i$$:

$$\begin{aligned}
f(x)+g(x) & = (3+0) + (2+7) x + (5-1) x^2 + (4+0) x^3 \\
          & = 3 + 9 x + 4 x^2 + 4 x^3 \\
\end{aligned}$$

You can multiply polynomials:

$$\begin{aligned}
f(x) g(x) {} & = (3 + 2 x + 5 x^2 - 4 x^3)(7 x - x^2) \\
 & = (3 + 2 x + 5 x^2 - 4 x^3)7 x + (3 + 2 x + 5 x^2 - 4 x^3) (-x^2) \\
 & = 3 \cdot 7x + 2 x \cdot 7x + 5 x^2 \cdot 7x - 4 x^3 \cdot 7x + 3 \cdot -x^2  + 2 x \cdot -x^2 + 5 x^2 \cdot -x^2 - 4 x^3 \cdot -x^2  \\
 & = 21 x + 14 x^2 + 35 x^3 - 28 x^4  - 3 x^2  - 2 x^3 - 5 x^4 + 4 x^5   \\
 & = 21 x + 11 x^2 + 33 x^3 - 23 x^4 + 4 x^5   \\
\end{aligned}$$

And you can divide them, using long division

$$
\begin{aligned}
\frac{f(x)}{g(x)} & =  \frac{-4 x^3 + 5 x^2 +2x + 3}{-x^2+7x} \\
    & = \begin{align*}
        &\text{ }\text{ }\text{ }\bf{4x+23}\\
        -x^2 + 7x &\overline{\big)-4 x^3 + 5 x^2 +2x + 3}\\
        &\underline{\text{ }-4 x^3 + 28 x^2}\\
        &\text{ }\text{ }\text{ }-23x^2 +2x +3\\
        &\text{ }\text{ }\underline{\text{ }-23x^2 + 161 x}\\
        &\text{ }\text{ }\text{ }\text{ }\text{ }\text{ }\bf{-159x+3}
    \end{align*} \\
    & = 4x+23 + \frac{-159x+3}{-x^2+7x} \\
\end{aligned}
$$

In the examples above, the coefficients $$c_i$$, and function variable $$x$$ are regular integers, 
but polynomials can be used for any mathematical system that has the concept of addition and
multiplication. But that's for another day.

# Minimum Information to Define a Polynomial

Here's one the most important characteristics of polynomials:

**Any polynomial function of degree $$n-1$$ is uniquely defined by any $$n$$ points that 
lay on this function.**

Being uniquely defined means that it goes both ways: 

* I can give you the $$n$$ coefficients $$c_0$$ to $$c_{n-1}$$ of $$f(x)$$, and you can evaluate the function value by filling in 
any $$x$$ coordinate you want, and thus also the function values for $$n$$ specific $$x$$ input values.

* Or, the other way around, I can give you the value $$f(x)$$ of any $$n$$ values of $$x$$ and 
you can derive the $$n$$ coefficients $$c_0$$ to $$c_{n-1}$$ of $$f(x)$$. 

Here's an example of polynomial of degree 3, $$f(x)=2 + 3x -5x^2 + x^3$$, evaluated for $$x$$ values of $$-1,0,1,2$$:

<img src="/assets/reed_solomon/desmos-graph-1.png" alt="f(x) graph, annotated with points -1,0,1,2" width="80%"/>

*I used the excellent [desmos graphing calculator](https://www.desmos.com/calculator) to create this
function plot.*

When given the points $$(-1, 7), (0,2), (1, 1), (2, -4)$$, we can set up an equation for each of those
4 points, which results in a set of linear equations with 4 unknowns:

$$
c_0 + c_1 (-1)^1 + c_2 (-1)^2 + c_3 (-1)^3 = -7 \\
c_0 + c_1 (0)^1 + c_2 (0)^2 + c_3 (0)^3 = 2 \\
c_0 + c_1 (1)^1 + c_2 (1)^2 + c_3 (1)^3 = 1 \\
c_0 + c_1 (2)^1 + c_2 (2)^2 + c_3 (2)^3 = -4 \\
$$

Such a set of equations can be solved with the well-known 
[Gaussian elimination algorithm](https://en.wikipedia.org/wiki/Gaussian_elimination). 
In our case, the values are such that we can solve it using a less systematic method:

The second row immediately reduces to $$\bf{c_0 = 2}$$. When we fill that back in, the remaining
rows become:

$$
-c_1 + c_2 - c_3 = -9 \\
c_1 + c_2 + c_3 = -1 \\
2 c_1  + 4 c_2  + 8 c_3  = -6 \\
$$

If we add the first and second row, we get: $$2c_2 = -10$$, or $$\bf{c_2=-5}$$, which simplifies
the 3 rows to:

$$
-c_1 - c_3 = -4 \\
c_1 + c_3 = 4 \\
2 c_1  + 8 c_3  = 14 \\
$$

After substituting the second equation in the third one, we get:

$$2 c_1 + 8 (4 - c_1) = 14$$

which reduces to $$-6 c_1 = -18$$ or $$\bf{c_1 = 3}$$.

And if we fill that into $$c_1 + c_3 = 4$$, we get $$\bf{c_3 = -1}$$.

Conclusion: $$(c_0, c_1, c_2, c_3) = (2, 3, -5, 1)$$. These coefficients match those of function $$f(x)$$ that
we started with!

<img src="/assets/reed_solomon/desmos-graph-2.png" alt="f(x) graph, annotated with points 1,2,3,4" width="80%"/>

We can do the same exercise with points $$(1, 1), (2, -4), (3, -7), (4, -2)$$, and we'll 
end up with the same result: $$(c_0, c_1, c_2, c_3) = (2, 3, -5, 1)$$. This is left as an exercise
for the reader.


When given a set of points of a polynomial, we can generate the coefficient by constructing
the [Lagrange interpolating polynomial](https://en.wikipedia.org/wiki/Systematic_code),
or in short, but using Lagrange interpolation.

A Lagrange polynomial looks like this:

$$
L(x) = y_0 \cdot \frac{x-x_1}{x_0-x_1} \frac{x-x_2}{x_0-x_2} ... \frac{x-x_n}{x_0-x_n} + \\
       y_1 \cdot \frac{x-x_0}{x_1-x_0} \frac{x-x_2}{x_1-x_2} ... \frac{x-x_n}{x_1-x_n} + \\
       ... \\
       y_n \cdot \frac{x-x_0}{x_n-x_0} \frac{x-x_1}{x_n-x_2} ... \frac{x-x_{n-1}}{x_n-x_{n-1}} + \\
$$


# Some Often Used Terminology

There is no rigid standard about which kind of letters or symbols to use when discussing coding
theory, but there's a least some commonality between different texts. 

* a message 

    A message is the original piece of information that need to be encoded for storage or
    transmission. A message is a sequence of symbols.

    For example, a message could be "Hello world ".

* a message word 

    A message word is a fixed length section of the overall message. Reed-Solomon 
    coding is a block coding algorithm. This means that a message is split up into multiple 
    message words, and the encoding algorithm is performed on each message word without any
    dependency on previously received message words.

    Message words are usually indicated with the letter $$m$$, and $$k$$ is often used to
    indicate the number of symbols in the message word. In other words, $$k$$ is the size 
    or the length of the message word.

    In vector notation: $$m = (m_0, m_1, ... m_{k-1})$$.

    If a message word is defined as having a size 4, the message would be split into the 
    following message words: ('H', 'e', 'l', 'l'), ('o', ' ', 'w', 'o'), ('r', 'l', 'd', ' ').

* an alphabet

    An alphabet is the set of values that can be used for each symbol of a message word. A
    message worth of length $$k$$ is a sequence of $$k$$ elements, where each element is part of the 
    alphabet.

    $$q$$ is often used as the number of elements in an alphabet.

    If we had a system where messages only consist of lower case letters and a space, then our
    alphabet would be ('A', ..., 'Z', 'a', 'b', 'c', ... , 'z', ' '), and the size of the alphabet 
    would be 53. Almost all coding algorithms algorithms require the ability to perform mathematical 
    operations such as addition, subtraction, multiplication and division on symbols of a word, so don't
    expect to see this kind of alphabet in the real world!

    In practice, the alphabet of most coding algorithms is a sequence of binary digits. 8 bits
    is common, but it doesn't have to be. I hesitate to call it an 8-bit number, because
    that would imply that regular integer math would be used on it, and that's almost never
    the case. Instead, the operations of such an alphabet use the rules of a Galois field.

* a code word

    A code word is what you get after you run a message word through an encoder. In the case of
    a Reed-Solomon encoder, a code word has a length $$n$$ symbols, where $$n>k$$ and the symbols
    are from the same alphabet as the message word.

    Code words are often indicated with a vector $$s = (s_0, s_1, ... , s_{n-1})$$.

    When our message of three 4-symbol messages words is converted into three 6-symbol code words, 
    it looks like this:

    ('H', 'e', 'l', 'l', 'a', 'b'), ('o', ' ', 'w', 'o', 'g', 't'), ('r', 'l', 'd', ' ', 'm', 'j')

    Notice how the first 4 symbols of each code word are the same as their corresponding message word, 
    but that 2 symbols are added for error detection and correction. This makes it a 
    [systematic code](https://en.wikipedia.org/wiki/Systematic_code). As we'll soon see, not all coding 
    schemes are systematic in nature, but it's a nice property to have.

    For Reed-Solomon code, the length $$n$$ of a code word must be smaller or equal than the number of
    elements of the alphabet that is uses.

In all the examples below, the message words will have a length of 4, and code words will have a length 6. 
We will be using an alphabet that consists of regular, unlimited range integer

In other words: $$k=4$$, $$n=6$$, and the size of the alphabet will be unlimited.

Using integers as symbols is a terrible choice: there are no real-world, practical encoding algorithms
that use them. But my primary goal is to first understand how Reed-Solomon codes work fundamentally,
and that's easier with integers. Later on, we can improve the algorithm to something practical, by using an
alphabet that's a Galois field.

If we'd want to send the "Hello world" message using an alphabet of integers, one possibility is
to just convert each letter to their corresponding integer. For example, "Hello world " would be
converted to $$(7, 30, 37, 37, 40, ...)$$, because 'A' is assigned a value of 0, 'B' a value of
1, and so forth.

# Reed Solomon Encoding through Polynomial Evaluation

Reed-Solomon codes were introduced to the world by Irving S. Reed and Gustave Solomon with a
paper with the very unassuming title "Polynomial Codes over Certain Finite Fields." The
4-page paper can be [purchased](https://doi.org/10.1137/0108018) for the low price of $36.75.
You should definitely not use Google to find one of the many copies for free.

Once you understand how Reed-Solomon coding and Galois fields work, the paper is surprisingly
readable... by today's standards at least, and light on math too! But you don't even need to 
know Galois fields to understand the core idea: regular integers do the job just fine.

So let's finally get to business!

In an earlier section, we learned that we can set up a third degree polynomial with 4 numbers: either
the 4 coefficients $$(c_0, c_1, c_2, c_3)$$, or 4 points $$f(x)$$ on the polynomial for some 
given $$x$$ values.

Once we know the coefficients, we can evaluate the polynomial at more than 4 values of $$x$$. 
Those additional points are not required to specify the polynomial, they are 
redundant, which is *exactly* what we're looking for: redundant information that allows us find the original
values in case a value gets corrupted during transmission!

And that's what the original way of Reed-Solomon encoding is all about:

**Creating redundant information by evaluating a polynomial for more $$x$$ values than is strictly necessary**.

Let's go through a concrete example where I want to setup a communication protocol to transmit information that
consists of a sequence of numbers, but where I also want to add redundancy so that the message still can be 
recovered after a corruption.

Here's one way to go about it:

**Protocol Specification**

The encoding and decoding parties must come up with some fixed parameters of the coding protocol
are not message dependent: you set the parameters once and that's it:

* Agree on an alphabet.

* Agree on the length of the message word. 

* Agree on the length of the code word.

    The longer the code word, the more redundancy, the more corrupted symbols
    can be error corrected.

* Agree on how some polynomial $$p(x)$$ should be constructed out of the symbols of the message word.

    Let's just follow the paper and use the message word symbols as coefficients of the polynomial.
    (But that's not always the best choice. See later sections...)

* Agree on values of $$x$$ for which we'll evaluate the polynomial $$p(x)$$.

* Agree that the code word is formed by $$p(x)$$ for the numbers of $$x$$ that were agreed on 
  in the previous step.

That's really it!

Let's apply this to an example with the following protocol settings:

* The alphabet are integers.
* The size $$k$$ of the message word $$m$$ is 4.
* We want to 2 redundant symbols, so the code word has length $$n$$ of 6.
* The polynomial $$p(x)$$ is always evaluated for the following values of $$x$$: $$(-1, 0, 1, 2, 3, 4)$$.

Here's what happens of a message word with a value of $$(2, 3, -5, 1)$$:

**Encoder**

* Create polynomial $$p(x)=2 + 3x -5x^2 + x^3$$.

    *This is obviously the same polynomial as the one that I used for the example in a
     previous section!*

* Evaluate the polynomial $$p(x)$$ at the 6 locations of $$x$$: $$(-1, 7), (0,2), (1, 1), (2, -4), (3, -7), (4, -2)$$.
* The code word is: $$(7,2,1,-4,-7,-2)$$.

**Decoder**

The decoder, on the other hand, does the following:

* Get the 6 symbols of the code word. If there was no corruption, that's still $$(7,2,1,-4,-7,-2)$$.
* Take any 4 of the 6 received symbols, and link them to their corresponding $$x$$ value. 

    If we take the first 4 code word symbols, we get $$(-1, 7), (0,2), (1, 1), (2, -4)$$.

* Use these 4 points to derive the coefficients of the polynomial $$p(x)$$ that was used by the 
  transmitter, using Lagrange interpolation.

    In the section on Lagrange interpolation, we already saw how the points $$(-1, 7), (0,2), (1, 1), (2, -4)$$
    result in coefficients $$(2, 3, -5, 1)$$.

    These coefficients are the recovered the original message words!

The decoder picked 4 of the 6 code word symbols to recover the coefficients, and ignored the 2 others, 
so what was the point of sending those 2 extra symbols? A real decoder will obviously be smarter and use
those 2 additional symbols to either check the integrity of the received message, or even to correct
corrupted values.

Let's talk about that next...

# A Simple Error Correcting Reed Solomon Decoder

Reed-Solomon encoding is simple. Reed-Solomon decoding is harder. In this blog post, I'll 
only cover the decoding algorithm that was proposed in the original Reed-Solomon paper.

Let's first talk about how many redundant numbers are needed to correct an error:
to correct errors:

**To correct up to $$s$$ symbol errors, you need at least $$2s$$ redundant symbols.**

In the example above, 2 additional symbols were added, which allows us to correct 1 corrupt symbol.

The original Reed-Solomon paper proposes the following algorithm: 

* From $$n$$ the received symbols of the code word, go through all combinations of $$k$$ out 
  of $n$$ symbols.
* For each such combination, calculate the coefficients of the polynomial.
* Count how many times each unique set of coefficients occurs.
* If there were $$s$$ corruptions or less, the set of coefficients with the highest count will 
  be coefficients of the polynomial that was used by the encoder.

Let's apply this algorithm to our example...

* Let's say the decoder received the following code word: $$(7, 2, 6, -4, -7, -2)$$.

    Notice how the third symbol isn't 1 but 6. There was a corruption!

* Associate each symbol with its corresponding $$x$$ value:
  $$(-1, 7), (0,2), (1, 6), (2, -4), (3, -7), (4, -2)$$.

* From these 6 coordinates, we need to draw all combinations of 4 elements:

  $$(-1, 7),(0, 2),(1, 6),(2, -4)$$

  $$(-1, 7),(0, 2),(1, 6),(3, -7)$$

  $$(-1, 7),(0, 2),(1, 6),(4, -2)$$

  $$(-1, 7),(0, 2),(2, -4),(3, -7)$$

  $$(-1, 7),(0, 2),(2, -4),(4, -2)$$

  $$(-1, 7),(0, 2),(3, -7),(4, -2)$$

  $$(-1, 7),(1, 6),(2, -4),(3, -7)$$

  $$(-1, 7),(1, 6),(2, -4),(4, -2)$$

  $$(-1, 7),(1, 6),(3, -7),(4, -2)$$

  $$(-1, 7),(2, -4),(3, -7),(4, -2)$$

  $$(0, 2),(1, 6),(2, -4),(3, -7)$$

  $$(0, 2),(1, 6),(2, -4),(4, -2)$$

  $$(0, 2),(1, 6),(3, -7),(4, -2)$$

  $$(0, 2),(2, -4),(3, -7),(4, -2)$$

  $$(1, 6),(2, -4),(3, -7),(4, -2)$$

  There are 15 combinations.

* For each of these 15 combinations, determine the 4 polynomial coefficients through
  Lagrange interpolation:

  $$(-1, 7),(0, 2),(1, 6),(2, -4) \rightarrow (2, 8, -5/2, -3/2)$$

  $$(-1, 7),(0, 2),(1, 6),(3, -7) \rightarrow (2, 27/4, -5/2, -1/4)$$

  $$(-1, 7),(0, 2),(1, 6),(4, -2) \rightarrow (2, 19/3, -5/2, 1/6)$$

  $$(-1, 7),(0, 2),(2, -4),(3, -7) \rightarrow (2, 3, -5, 1)$$

  $$(-1, 7),(0, 2),(2, -4),(4, -2) \rightarrow (2, 3, -5, 1)$$

  $$(-1, 7),(0, 2),(3, -7),(4, -2) \rightarrow (2, 3, -5, 1)$$

  $$(-1, 7),(1, 6),(2, -4),(3, -7) \rightarrow (19/2, 17/4, -10, 9/4)$$

  $$(-1, 7),(1, 6),(2, -4),(4, -2) \rightarrow (26/3, 14/3, -55/6, 11/6)$$

  $$(-1, 7),(1, 6),(3, -7),(4, -2) \rightarrow (7, 61/12, -15/2, 17/12)$$

  $$(-1, 7),(2, -4),(3, -7),(4, -2) \rightarrow (2, 3, -5, 1)$$

  $$(0, 2),(1, 6),(2, -4),(3, -7) \rightarrow (2, 18, -35/2, 7/2)$$

  $$(0, 2),(1, 6),(2, -4),(4, -2) \rightarrow (2, 49/3, -15, 8/3)$$

  $$(0, 2),(1, 6),(3, -7),(4, -2) \rightarrow (2, 13, -65/6, 11/6)$$

  $$(0, 2),(2, -4),(3, -7),(4, -2) \rightarrow (2, 3, -5, 1)$$

  $$(1, 6),(2, -4),(3, -7),(4, -2) \rightarrow (2, 3, -5, 1)$$

* The coefficients $$(2,3,-5,1)$$ come up 6 times. All other solutions are different
  from each other, so $$(2,3,-5,1)$$ is clearly the winner, and the correct solution!

Conclusion: we have a very straightforward error correcting algorithm. However, it's not
a practical algorithm: even for this toy example with a message word of size 4 and
only 2 additional redudant symbols, we need to perform Lagrange interpolation 15 times.

In the real world, a very popular choice is to have a message word of size 223, and a
code word of size 255.

The [formula to calculate the number of combinations](https://en.wikipedia.org/wiki/Combination) is: 

$$\frac{n!}{r!(n-r)!}$$

With $$r=223$$ and $$n=255$$, this gives a total of 50,964,019,775,576,912,153,703,782,274,307,996,667,625 combinations!

That's just not very practical...

It's clear that a much better algorithm is required to make Reed-Solomon decoding useful. Luckily,
these algorithms exists, and some of them aren't even that complicated, but that's worthy of a 
separate blog post.

# A Systematic Reed-Solomon Code

Let's recap how the previous encoder worked:

* The symbols of the message word are used as *coefficients* of a polynomial $$p(x)$$.
* The polynomial $$p(x)$$ is evaluated for $$n$$ values of $$x$$.
* The code word is the $$n$$ values of $$p(x)$$.

This encoding system works fine, but note how all symbols of the code word are different
from the symbols of the message word. In our example, message word $$(2,3,-5,1)$$ was 
encoded as $$(7, 2, 6, -4, -7, -2)$$.

Wouldn't it be nice if we could encode our message word so that the original symbols are part
of the code word, with some addition symbols tacked on for redunancy? In other words,
encode $$(2,3,-5,1)$$ into something like $$(2,3,-5,1,r_4, r_5)$$. We already
saw earlier that such a code is called a systematic code.

One of the benefits of a systematic code is that you don't need to run a decoder at all
if there is some way to show early on that the code word was received without any error.

It's easy to modify the original encoding scheme into one that is systematic 
**by treating the message word symbols as the result of evaluating a $$p(x)$$ polynomial**.

**Encoder**

* The message word symbols $$(m_0, m_1, ..., m_{k-1})$$ are the value of the polynomial 
  $$p(x)$$ at certain agreed upon values of $$x$$.
* Construct the coefficients of this polynomial $$p(x)$$ from the $$(x_i, m_i)$$ pairs.

    Once again, Lagrange interpolation is used.

* Evaluate this polynomial $$p(x)$$ for an additional $$(n-k)$$ values of $$x$$ to create
  redundant symbols.
* The code word consists of the message word followed by the redundant symbols.

Let's try this out with the same number sequence as before: 

* The message word is still $$(2,3,-5,1)$$. These are now considered the result of evaluating
  polynomial $$p(x)$$ for the corresponding values $$(-1, 0, 1, 2)$$ of $$x$$.
* Construct the polynomial $$p(x)$$ out of these coordinate pairs: $$((-1, 2), (0, 3), (1, 1), (2, 1))$$.

    I found [this website](https://www.dcode.fr/lagrange-interpolating-polynomial) to do this for me.

    The result is: $$p(x) = \frac{23}{6}x^3 + \frac{9}{2}x^2 + \frac{22}{3}x + 3$$

    *Note how some of the coefficients of $$p(x)$$ are rational numbers instead of integers. This
    is one of the reasons why, in practice, integers shouldn't be used for Reed-Solomon coding!*

* Evaluating $$p(x)$$ for $$x$$ values of 3 and 4 gives 44 and 147.
* The code word is $$(2,3,-5,1,44,147).$$

The decoder is *exactly* the same as before! 

# A Code Word as a Sequence of Polynomial Coefficients

In the two encoding methods above, the code word consists of evaluated values of some polynomial $$p(x)$$. 
There is yet another Reed-Solomon coding variant where the code word consists of the coefficients 
of a polynomial. And since this variant is the most popular, we have to cover it too. To understand
how it works, there's a bit more math involved, but an example later should make it all clear.

A message word has $$k$$ symbols, and we need $$n$$ symbols to form a code word. If a code word
consists of coefficients, we need to construct some polynomial $$s(x)$$ of degree $$(n-1)$$.

Here are some desirable properties for $$s(x)$$:

1. $$k$$ of its coefficients should be the same as the symbols of the message word to create
  a systematic code.
2. When $$s(x)$$ is evaluated at $$(n-k)$$ values of $$x$$, the result should be 0.

    We'll soon see why that's useful.

Here's a way to create a polynomial with these properties:

* Create a polynomial $$p(x)$$ with symbols $$(m_0, m_1, ..., m_{k-1})$$ as coefficients, just like
  before.
* Create a so-called generator polynomial $$g(x) = (x-x_0)(x-x_1)...(x-x_{n-k-1})$$.

    As before, the values $$(x_0, x_1, ..., x_{n-k-1})$$ are fixed parameters of the protocol and
    agreed upon between encoder and decoder up front. However, this time there are only are as $$(n-k)$$ 
    values of $$x$$, as many as there are redundant symbols.

    Polynomial $$g(x)$$ has a degree of $$(n-k)$$.

* Perform a polynomial division of $$\frac{p(x)x^{n-k}}{g(x)}$$, such that 
  $$p(x)x^{n-k} = q(x)g(x) + r(x)$$.

    In other words, $$q(x)$$ is the quotient of the divison, and $$r(x)$$ is the remainder.

* Define polynomial $$s(x)$$ as $$p(x)x^{n-k} - r(x)$$.

Let's see what this gets us:

**Desirable property 1: $$k$$ coefficients of $$s(x)$$ are $$(m_0, m_1, ..., m_{k-1})$$**

* $$p(x)$$ has degree $$(k-1)$$. When multiplied by $$x^{n-k}$$, that results in 
  a polynomial $$m_{0}x^{n-k} + m_{1}x^{n-k+1} + ... + m_{k-1}x^{n-1}$$. 
  The degree of $$p(x)x^{n-k}$$ is $$(n-1)$$, but its first non-zero coefficient is the one for
  $$x^{n-k}$$.
* $$r(x)$$ is the remainder of the division of polynomial $$p(x)x^{n-k}$$, degree $$(n-1)$$,
  and $$g(x)$$, degree $$(n-k)$$. The remainder of a polynomial division has a degree
  that is at most the degree of the divident minus 1: $$(n-k-1)$$. And thus, 
  $$r(x) = r_0 + r_1 + ... + r_{n-k-1}$$.
* There is no overlap in coefficients for $$x^i$$ between $$p(x)x^{n-k}$$ and $$r(x)$$: $$r(x)$$
  goes from 0 to $$(n-k-1)$$ and $$p(x)x^{n-k}$$ goes from $$(n-k)$$ to $$(n-1)$$.
  So if we subtract $$p(x)x^{n-k}$$ from $$r(x)$$, the coefficients of the 2 terms don't
  interact. As a result, the coefficients of $$s(x)$$ for terms $$x^{n-k}$$ to $$x^{n-1}$$
  are the same as coefficients $$m_0..m_{k-1}$$ of the message word. 
  This satisfies the first desirable property.

**Desirable property 2:$$s(x_i) = 0$$**

* By definition, $$s(x) = p(x)x^{n-k} - r(x)$$
* By definition, $$p(x)x^{n-k} = q(x)g(x) + r(x)$$, so after substitution:
  $$s(x) =  q(x)g(x) + r(x) - r(x)$$ and $$s(x) =  q(x)g(x)$$.
* By definition, $$g(x)= (x-x_0)(x-x_1)...(x-x_{n-k-1})$$. Another substitution gives:
  $$s(x) =  q(x)(x-x_0)(x-x_1)...(x-x_{n-k-1})$$.
* Fill in any value $$x_i$$ and $$s(x_i)$$ evaluates to 0!

What can we do with these properties? For that, we need to look at the decoder.

The decoder will use the code word symbols as coefficients of a polynomial $$s'(x)$$. 
If there was no corruption, $$s'(x)$$ will be the same as $$s(x)$$. Due to property
2, the decoder can verify this by filling in $$x_i$$ into $$s'(x)$$ and check that the result is 0.

The vector $$(s'(x_0), s'(x_1), ..., s'(x_{n-k-1}))$$ is called the *syndrome* of the
received code word.

If the syndrome is not equal to 0, we know that the received polynomial $$s'(x)$$ is not
the same as the transmitted polynomial, and we can define an error polynomial $$e(x)$$ so
that $$s'(x) = s(x) + e(x)$$. Polynomial $$e(x)$$ has the same degree $$(n-1)$$ as $$s(x)$$.

Since $$s(x_i) = 0$$, it follows that $$e(x_i) = s'(x_i)$$. There are $$(n-k)$$ such
equations.

If we can find a way to derive the $$n$$ coefficients of $$e(x)$$ out of the $$(n-k)$$
equations, we can derive $$s(x)$$ as $$s'(x)-e(x)$$. This is, of course, not 
generally possible, but it *is* possible when half or less of the coefficients of 
$$s'(x)$$ are correct, just like it was for the other coding variant.

And the simply way to figure this out is again by going through all combinations and solving
a set of equations.





# References

* [Introduction to Reed-Solomon](https://innovation.vivint.com/introduction-to-reed-solomon-bc264d0794f8)

    Very good explanation on polynomial based interpolation and error correction.

    * [Joseph Louis Lagrange and the Polynomials](https://medium.com/@jtolds/joseph-louis-lagrange-and-the-polynomials-499cf0742b39)

	Side story about Lagrange interpolation.

    * [infectious - RS implementation](https://pkg.go.dev/github.com/vivint/infectious)

        Code that goes with the RS article.

* [NASA - Tutorial on Reed-Solomon error correction coding](https://ntrs.nasa.gov/citations/19900019023)

    * [Actual PDF file](https://ntrs.nasa.gov/api/citations/19900019023/downloads/19900019023.pdf)

* [Reed-Solomon on Wikipedia](https://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction)

* [Original paper: Polynomial Codes over Certain Finite Fields](https://faculty.math.illinois.edu/~duursma/CT/RS-1960.pdf)

    Only 4 pages!
