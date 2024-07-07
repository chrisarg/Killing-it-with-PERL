---
title: " The Quest for Performance Part I : Inline C, OpenMP and PDL "
date: 2024-07-06
---

Sometimes, one's code must simply perform and principles, such as aesthetics, "cleverness" or commitment to a single language solution simply go out of the window. 
At the TPRC I gave a talk (here are the [slides](https://www.slideshare.net/slideshow/enhancing-non-perl-bioinformatic-applications-with-perl/269925371)) about how this 
can be done for bioinformatics applications, but I feel that a simpler example is warranted to illustrate the potential venues to maximize performance that a Perl 
programmer has at their disposal when working in data intensive applications. 

So here is a **toy problem** to illustrate these options. Given a very large array of double precision floats _transform_ them in place with the following function : _cos(sin(sqrt(x)))_. 
The function has 3 nested floating point operations. This is an expensive function to evaluate, especially if one has to calculate for a large number of values. We can generate reasonably 
quickly the array values in Perl (and some copies for the solutions we will be examining) using the following code:
```perl
my $num_of_elements = 50_000_000;
my @array0 = map { rand } 1 .. $num_of_elements;    ## generate random numbers
my @array1 = @array0;                               ## copy the array
my @array2 = @array0;                               ## another copy
my @array3 = @array0;                               ## yet another copy
my @rray4  = @array0;                               ## the last? copy
my $array_in_PDL      = pdl(@array0);    ## convert the array to a PDL ndarray
my $array_in_PDL_copy = $array_in_PDL->copy;    ## copy the PDL ndarray
```
The posssible solutions include the following: 

**Inplace modification using a for-loop in Perl.**
```perl
for my $elem (@array0) {
    $elem = cos( sin( sqrt($elem) ) );
}
```

**Using Inline C code to walk the array and transform in place in C.** . Effectively one does a inplace map using C. Accessing elements of Perl arrays (AV* in C) in C is particularly
performant if one is using perl 5.36 and above because of an optimized fetch function introduced in that version of Perl. 

```c
void map_in_C(AV *array) {
  int len = av_len(array) + 1;
  for (int i = 0; i < len; i++) {
    SV **elem = av_fetch_simple(array, i, 0); // perl 5.36 and above
    if (elem != NULL) {
      double value = SvNV(*elem);
      value = cos(sin(sqrt(value))); // Modify the value
      sv_setnv(*elem, value);
    }
  }
}
```

**Using Inline C code to transform the array, but break the transformation in 3 sequential C for-loops.** This is an experiment really about tradeoffs: modern x86 processors have a specialized, 
vectorized square root instruction, so perhaps the compiler can figure how to use it to speed up at least one part of the calculation. On the other hand, we will be reducing the arithmetic intensity
of each loop and accessing the same data value twice, so there will likely be a price to pay for these repeated data accesses. 
```c
void map_in_C_sequential(AV *array) {
  int len = av_len(array) + 1;
  for (int i = 0; i < len; i++) {
    SV **elem = av_fetch_simple(array, i, 0); // perl 5.36 and above
    if (elem != NULL) {
      double value = SvNV(*elem);
      value = sqrt(value); // Modify the value
      sv_setnv(*elem, value);
    }
  }
  for (int i = 0; i < len; i++) {
    SV **elem = av_fetch_simple(array, i, 0); // perl 5.36 and above
    double value = SvNV(*elem);
    value = sin(value); // Modify the value
    sv_setnv(*elem, value);
  }
  for (int i = 0; i < len; i++) {
    SV **elem = av_fetch_simple(array, i, 0); // perl 5.36 and above
    double value = SvNV(*elem);
    value = cos(value); // Modify the value
    sv_setnv(*elem, value);
  }
}
```

**Parallelize the C function loop using OpenMP.** In a [previous entry](https://chrisarg.github.io/Killing-It-with-PERL/2024/07/01/Rudimentary-control-of-OpenMP-from-Perl.html) we discussed how to control the OpenMP environment from within Perl and compile OpenMP aware Inline::C code for 
use by Perl, so let's put this knowledge into action! On the Perl side of the program we will do this:
```perl
use v5.38;
use Alien::OpenMP;
use OpenMP::Environment;
use Inline (
    C    => 'DATA',
    with => qw/Alien::OpenMP/,
);
my $env = OpenMP::Environment->new();
my $threads_or_workers = 8; ## or any other value
## modify number of threads and make C aware of the change
$env->omp_num_threads($threads_or_workers);
_set_num_threads_from_env();

## modify runtime schedule and make C aware of the change
$env->omp_schedule("guided,1");    ## modify runtime schedule
_set_openmp_schedule_from_env();
```
On the C part of the program, we will then do this (the helper functions for the OpenMP environment have been discussed
previously, and thus not repeated here).
```c
#include <omp.h>
void map_in_C_using_OMP(AV *array) {
  int len = av_len(array) + 1;
#pragma omp parallel
  {
#pragma omp for schedule(runtime) nowait
    for (int i = 0; i < len; i++) {
      SV **elem = av_fetch_simple(array, i, 0); // perl 5.36 and above
      if (elem != NULL) {
        double value = SvNV(*elem);
        value = cos(sin(sqrt(value))); // Modify the value
        sv_setnv(*elem, value);
      }
    }
  }
}
```
**Perl Data Language (PDL) to the rescue.** The PDL set of modules is yet another way to speed up operations and can save the programmer from C. It also autoparallelizes given the right directives so why not use it?
```perl
use PDL;
## set the minimum size problem for autothreading in PDL
set_autopthread_size(0);
my $threads_or_workers = 8; ## or any other value

## PDL
## use PDL to modify the array - multi threaded
set_autopthread_targ($threads_or_workers);
$array_in_PDL->inplace->sqrt;
$array_in_PDL->inplace->sin;
$array_in_PDL->inplace->cos;


## use PDL to modify the array - single thread
set_autopthread_targ(0);

$array_in_PDL_copy->inplace->sqrt;
$array_in_PDL_copy->inplace->sin;
$array_in_PDL_copy->inplace->cos;

```
Using **8 threads** we get something like this
```text
Inplace benchmarks
Inplace  in         Perl took 2.85 seconds
Inplace  in Perl/mapCseq took 1.62 seconds
Inplace  in    Perl/mapC took 1.54 seconds
Inplace  in   Perl/C/OMP took 0.24 seconds

PDL benchmarks
Inplace  in     PDL - ST took 0.94 seconds
Inplace  in     PDL - MT took 0.17 seconds
```
Using **16 threads** we get this!
```text
Starting the benchmark for 50000000 elements using 16 threads/workers

Inplace benchmarks
Inplace  in         Perl took 3.00 seconds
Inplace  in Perl/mapCseq took 1.72 seconds
Inplace  in    Perl/mapC took 1.62 seconds
Inplace  in   Perl/C/OMP took 0.13 seconds

PDL benchmarks
Inplace  in     PDL - ST took 0.99 seconds
Inplace  in     PDL - MT took 0.10 seconds
```
A few observations:
* The OpenMP and the multi-threaded (MT) of the PDL are responsive to the number of workers, while the solutions are not. Hence, the timings of the pure Perl and the inline non-OpenMP solution timings in these benchmarks give
an idea of the natural variability in performance
* Writing the map version of the code in C improved performance by about 180% (contrast Perl and Perl/mapC).
* Using PDL **in a single thread** improved performance by 285-300% (contrast PDL - ST and Perl timings).
* There was a price to pay for repeated memory access (contrast Perl/mapC to Perl/mapCseq)
* OpenMP and multi-threaded PDL operations gave similar performance (though PDL appeared faster in these examples). The code run 23-30 times faster.

In summary, there are both native (PDL modules) and foreign (C/OpenMP) solutions to speed up data intensive operations in Perl, so why not use them widely and wisely to make Perl programs performant?
