# simdfmt
Fast integer to string conversion using SIMD instructions

Can SIMD instructions (SSE/AVX) be used to speed up integer to 
string conversion? The basic algorithm for integer to string
conversion is essentially sequential: repeatedly divide by 10
and take the remainder. So the answer seems to be no for a
single integer. However, if there is an array of integers,
maybe speed can be gained by processing several numbers
in parallel.

Here is a method for converting an array of uint16_t
to string using SSE instructions. The numbers are converted
to decimal, with leading zeros removed, separated by comma.

The algorithm is the same as above: divide each number
repeatedly by 10 and take the remainders. The function
processes 8 numbers in parallel. Since SSE does not have
an integer division instruction, the "multiply by reciprocal"
trick is used instead. The remainder of the code is about
shuffling the resulting digits in the correct order. The
problem is similar to transposing a matrix. After the division
part, the digits of number 0 are in element 0 of 5 different
registers, the digits of number 1 are in element 1 of the
same registers, and so on. They must be reordered so that
the digits of each number are sequential in one register.
An additional problem is detecting the number of leading
zeros, which is done using the bsf (__builtin_ctzll)
instruction.

To see how fast this method is, I compare it with several
other methods. The sprintf method simply uses sprintf
to format the numbers. The div10 method is the basic
algorithm, but processing one number at a time using
ordinary non SIMD instructions. The div100 method
is similar, but it divides by 100 in each step. The table
method looks up the string from a pregenerated table
containing all possible uint16_t values.

In the test, an array of 80000000 random uint16_t is converted
to string and the time is measured in seconds.
The results are as follows (lower is better).

| | GCC 7 | GCC 8 | GCC 9 | Clang 6 | Clang 7 | Clang 8 | Clang 9 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| sprintf | 6.85 | 6.81 | 6.52 | 6.81 | 6.55 | 6.62 | 6.53 |
| div10 | 0.91 | 1.24 | 1.16 | 0.82 | 0.79 | 0.87 | 0.83 |
| div100 | 0.86 | 1.10 | 0.95 | 0.79 | 0.74 | 0.77 | 0.83 |
| table | 0.62 | 0.63 | 0.62 | 0.60 | 0.60 | 0.61 | 0.61 |
| sse | 0.46 | 0.45 | 0.44 | 0.46 | 0.46 | 0.45 | 0.45 |

The sprintf method is at least 6 times as slow as anything else.
The speed of the div10 and div100 methods depends quite a lot
on the compiler, with div100 being slightly faster. The table
method is faster still. The sse method is consistently the fastest,
being up to twice as fast as div10 and about 14 times as fast
as sprintf.

The table method would be quite impractical with anything bigger
than uint16_t. While the sse method is the fastest, the simple
div10 and div100 methods have good performance too.

Using AVX would allow processing twice as many numbers in
parallel. However, it's possible that reordering the digits becomes
even more complicated then. It would also be useful to check the
results for converting uint32_t and uint64_t. Since less parallelism
is possible with those types, the speed advantage compared to div10
and div100 might be lost.
