---
title: " The Quest for Performance Part IV : May the SIMD Force be with you "
date: 2024-07-09
---
At this point one may wonder how numba, the Python compiler around numpy Python code, delivers a performance premium over numpy. To do so, let's inspect timings individually for all trigonometric functions (and yes, the exponential and the logarithm are trigonometric functions if you recall your complex analysis lessons from high school!). But the test is not relevant only for those who want to do high school trigonometry: physics engines e.g. in games will use these function, and machine learning and statistical calculations heavily use log and exp. So getting the trigonometric functions right is one small, but important step, towards implementing a variety of applications. The table below shows the timings for 50M in place transformations:
|Function|Library|Execution Time    |
|:------:|:-----:|:----------------:|
|   Sqrt | numpy |  1.02e-01 seconds|
|    Log | numpy |  2.82e-01 seconds|
|    Exp | numpy |  3.00e-01 seconds|
|    Cos | numpy |  3.55e-01 seconds|
|    Sin | numpy |  4.83e-01 seconds|
|    Sin | numba |  1.05e-01 seconds|
|   Sqrt | numba |  1.05e-01 seconds|
|    Exp | numba |  1.27e-01 seconds|
|    Log | numba |  1.47e-01 seconds|
|    Cos | numba |  1.82e-01 seconds|

The table holds the first hold to the performance benefits: the square root, a function that has a dedicated **SIMD** instruction for vectorization takes exactly the same time to execute in numba and numpy, while all the other functions are speed up by 2-2.5 time, indicating either that the code auto-vectorizes using SIMD or auto-threads. A second clue is provided by examining the difference in results between the numba and numpy, using the [ULP](https://en.wikipedia.org/wiki/Unit_in_the_last_place) (Unit in the Last Place). ULP is a measure of accuracy in numerical calculations and can easily be computed for numpy arrays using the following Python function:

```python
def compute_ulp_error(array1, array2):
## maxulp set up to a very high number to avoid throwing an exception in the code
    return np.testing.assert_array_max_ulp(array1, array2, maxulp=100000)
```
These _numerical_ benchmarks indicate that the square root function utilizes pretty much equivalent code in numpy and numba, while for all the other trigonometric functions the mean, median, 99.9th percentile and maximum ULP value over all 50M numbers differ. This is a subtle hint, that SIMD is at play: vectorization changes slightly the semantics of floating point code to make use of associative math, and floating point numerical operations are not associative. 
|Function|Mean ULP   | Median ULP | 99.9th ULP |Max ULP |
|:------:|:----------------:|:----------------:|:----------------:|:----------------:|
|Sqrt| 0.00e+00|0.00e+00|0.00e+00|0.00e+00
|Sin| 1.56e-03|0.00e+00|1.00e+00|1.00e+00
|Cos| 1.43e-03|0.00e+00|1.00e+00|1.00e+00
|Exp| 5.47e-03|0.00e+00|1.00e+00|2.00e+00
|Log| 1.09e-02|0.00e+00|2.00e+00|3.00e+00

Finally, we can inspect the code of the numba generated functions for vectorized assembly instructions as detailed [here](https://tbetcke.github.io/hpc_lecture_notes/simd.html), using the code below:
```python
@njit(nogil=True, fastmath=False, cache=True)
def compute_sqrt_with_numba(array):
    np.sqrt(array, array)


@njit(nogil=True, fastmath=False, cache=True)
def compute_sin_with_numba(array):
    np.sin(array, array)


@njit(nogil=True, fastmath=False, cache=True)
def compute_cos_with_numba(array):
    np.cos(array, array)


@njit(nogil=True, fastmath=False, cache=True)
def compute_exp_with_numba(array):
    np.exp(array, array)


@njit(nogil=True, fastmath=False, cache=True)
def compute_log_with_numba(array):
    np.log(array, array)

## check for vectorization
## code lifted from https://tbetcke.github.io/hpc_lecture_notes/simd.html
def find_instr(func, keyword, sig, limit=5):
    count = 0
    for l in func.inspect_asm(func.signatures[sig]).split("\n"):
        if keyword in l:
            count += 1
            print(l)
            if count >= limit:
                break
    if count == 0:
        print("No instructions found")

# Compile the function to avoid the overhead of the first call
compute_sqrt_with_numba(np.array([np.random.rand() for _ in range(1, 6)]))
compute_sin_with_numba(np.array([np.random.rand() for _ in range(1, 6)]))
compute_exp_with_numba(np.array([np.random.rand() for _ in range(1, 6)]))
compute_cos_with_numba(np.array([np.random.rand() for _ in range(1, 6)]))
compute_exp_with_numba(np.array([np.random.rand() for _ in range(1, 6)]))
```
And this is how we probe for the presence f the vmovups instruction indicating that the YMM AVX2 registers are being used in calculations
```python
print("sqrt")
find_instr(compute_sqrt_with_numba, keyword="vsqrtsd", sig=0)
print("\n\n")
print("sin")
find_instr(compute_sin_with_numba, keyword="vmovups", sig=0)
```
As we can see in the output below the square root function uses the x86 SIMD vectorized square root instruction vsqrtsd , and the sin (but also all trigonometric functions) use SIMD instructions to access memory.

```text
sqrt
        vsqrtsd %xmm0, %xmm0, %xmm0
        vsqrtsd %xmm0, %xmm0, %xmm0


sin
        vmovups (%r12,%rsi,8), %ymm0
        vmovups 32(%r12,%rsi,8), %ymm8
        vmovups 64(%r12,%rsi,8), %ymm9
        vmovups 96(%r12,%rsi,8), %ymm10
        vmovups %ymm11, (%r12,%rsi,8)
```

Let's shift attention to Perl and C now (after all this is a Perl blog!). In [Part I](https://chrisarg.github.io/Killing-It-with-PERL/2024/07/06/The-Quest-For-Performance-Part-I-InlineC-OpenMP-PDL.html) 
we saw that PDL and C gave similar performance when evaluating the nested function cos(sin(sqrt(x))), and in [Part II](https://chrisarg.github.io/Killing-It-with-PERL/2024/07/07/The-Quest-For-Performance-Part-II-PerlVsPython.md.html) that the single-threaded Perl code was as fast as numba. But what about the individual trigonometric functions in PDL and C without the Inline module? The C code block that will be evaluated in this case is:
```c
#pragma omp for simd
  for (int i = 0; i < array_size; i++) {
    double x = array[i];
    array[i] = foo(x);
  }
```
where foo is one of sqrt, sin, cos, log, exp. We will use the simd omp pragma in conjunction with the following compilation flags and the gcc compiler to see if we can get the code to pick up the hint and auto-vectorize the machine code generated using the SIMD instructions. 

```make
CC = gcc
CFLAGS = -O3 -ftree-vectorize  -march=native -mtune=native -Wall -std=gnu11 -fopenmp -fstrict-aliasing -fopt-info-vec-optimized -fopt-info-vec-missed
LDFLAGS = -fPIE -fopenmp 
LIBS =  -lm
```
During compilation, gcc informs us about all the wonderfull missed opportunities to optimize the loop. The performance table below also demonstrates this; note that the standard C implementation and PDL are equivalent and equal in performance to numby. Perl through PDL can deliver performance in our data science world. 
|Function|Library|Execution Time    |
|:------:|:-----:|:----------------:|
|     Sqrt | PDL | 1.11e-01 seconds|
|     Log  | PDL | 2.73e-01 seconds|
|     Exp  | PDL | 3.10e-01 seconds|
|     Cos  | PDL | 3.54e-01 seconds|
|     Sin  | PDL | 4.75e-01 seconds|
|Sqrt | C | 1.23e-01 seconds|
|Log | C | 2.87e-01 seconds|
|Exp | C | 3.19e-01 seconds|
|Cos | C | 3.57e-01 seconds|
|Sin | C | 4.96e-01 seconds|

To get the compiler to use SIMD, we replace the flag -O3 by -Ofast and all these wonderful opportunities for performance are no longer missed, and the C code now delivers (but with the usual [caveats](https://simonbyrne.github.io/notes/fastmath/) that apply to the -Ofast flag). 

|Function|Library|Execution Time    |
|:------:|:-----:|:----------------:|
|Sqrt | C - Ofast| 1.00e-01 seconds|
|Sin | C - Ofast| 9.89e-02 seconds|
|Cos | C - Ofast| 1.05e-01 seconds|
|Exp | C - Ofast| 8.40e-02 seconds|
|Log | C - Ofast| 1.04e-01 seconds|

With these benchmarks, let's return to our initial [Perl benchmarks](https://chrisarg.github.io/Killing-It-with-PERL/2024/07/06/The-Quest-For-Performance-Part-I-InlineC-OpenMP-PDL.html) and contrast the timings obtained with the non-SIMD aware invokation of the Inline C code:

```perl
use Inline (
    C    => 'DATA',
    build_noisy => 1,
    with => qw/Alien::OpenMP/,
    optimize => '-O3 -march=native -mtune=native',
    libs => '-lm'
);
```
and the SIMD aware one (in the code below, one has to incluce the vectorized version of the mathematics library for the code to compile):
```perl
use Inline (
    C    => 'DATA',
    build_noisy => 1,
    with => qw/Alien::OpenMP/,
    optimize => '-Ofast -march=native -mtune=native',
    libs => '-lmvec'
);
```
The non-vectorized version of the code yield the following table
```text
Inplace  in  base Python took 11.9 seconds
Inplace  in PythonJoblib took 4.42 seconds
Inplace  in         Perl took 2.88 seconds
Inplace  in Perl/mapCseq took 1.60 seconds
Inplace  in    Perl/mapC took 1.50 seconds
C array  in            C took 1.42 seconds
Vector   in       Base R took 1.30 seconds
C array  in   Perl/C/seq took 1.17 seconds
Inplace  in     PDL - ST took 0.94 seconds
Inplace in  Python Numpy took 0.93 seconds
Inplace  in Python Numba took 0.49 seconds
Inplace  in   Perl/C/OMP took 0.24 seconds
C array  in   C with OMP took 0.22 seconds
C array  in    C/OMP/seq took 0.18 seconds
Inplace  in     PDL - MT took 0.16 seconds
```

while the vectorized ones this one: 

```text
Inplace  in  base Python took 11.9 seconds
Inplace  in PythonJoblib took 4.42 seconds
Inplace  in         Perl took 2.94 seconds
Inplace  in Perl/mapCseq took 1.59 seconds
Inplace  in    Perl/mapC took 1.48 seconds
Vector   in       Base R took 1.30 seconds
Inplace  in     PDL - ST took 0.96 seconds
Inplace in  Python Numpy took 0.93 seconds
Inplace  in Python Numba took 0.49 seconds
C array  in   Perl/C/seq took 0.30 seconds
C array  in            C took 0.26 seconds
Inplace  in   Perl/C/OMP took 0.24 seconds
C array  in   C with OMP took 0.23 seconds
C array  in    C/OMP/seq took 0.19 seconds
Inplace  in     PDL - MT took 0.17 seconds
```
To facilitate comparisons against the various flavors of Python and R, we inserted the results we [presented previously](https://chrisarg.github.io/Killing-It-with-PERL/2024/07/07/The-Quest-For-Performance-Part-II-PerlVsPython.md.html) in these 2 tables. 

The take home points (some of which may be somewhat surprising are):
1. Performance of base Python was horrendous - nearly 4 times slower than base Perl. Even under parallel processing (Joblib) the 8 threaded Python code was slower than Perl
2. R performed the best out of the 3 dynamically typed languages considered : nearly 9 times faster than base Python and 2.3 times faster than base Perl.
3. Inplace modification of Perl arrays by accessing the array containers in C (Perl/mapC) narrowed the gap between Perl and R
4. Considering variability in timings, the three codes, base R, Perl/mapC and an equivalent inplace modification of a C array are roughly equivalent
5. The uni-threaded PDL code delivered the same performance as numpy
6. SIMD aware code, e.g. through numba or (for the case of native C code compiler pragmas, e.g. cntract C array in C timings between the vectorized and the non-vectorized versions of the table) delivered the best single thread performance.
7. OpenMP pragmas can really speed up operations (such as map) of Perl containers.
8. Adding threads (OMP code) or the multi-threaded versions of the PDL delivered the best performance.

These observations generate the following big picture questions:
* Observations 1 and 2 make it quite surprising that base Python earned a reputation for a data science language. In fact, with these performance characteristics of a rather simple numerical code, one should approach the replacement of R (or Perl for those of us who use it) by base Python in data analysis codes. 
* Since hybrid implementations that involve C and Perl can deliver real performance benefits using SIMD aware code, even if a single thread is used, should we be upgrading our Inline::C codes to use SIMD friendly compiler flags? Or (for those who are are afraid of _-Ofast_, perhaps a carefully prepared mixology of intrinsics, and even assembly?
* Should be looking into upgrading various C/XS modules for Perl, so that they use OpenMP? 
* Why are not more people use the jewel of software that is PDL? The auto-threading property alone should make people think about using it for demanding, data intensive tasks.
* Is it possible to create hybrid applications that rely on both PDL and OpenMP/Inline::C for different computations?
* Should we look into special builds for PDL that leverage the SIMD properties and compiler pragmas to allow users to have their cake (SIMD vectorization) and eat it (autothreading) too?
