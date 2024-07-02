---
title: " How can we get and set the OpenMP environment from Perl? "
date: 2024-07-01
---

Those who attended the TPRC2024 conference last week, had the opportunity to listen to Brett Estrade talking about intermediate uses of OpenMP in Perl. 
(If you had not had the chance to go, well here is his [excellent talk](https://www.youtube.com/watch?v=_pzG5DerDT0)).

While OpenMP is (probably) the most popular and pain-free way to add parallelization to one's C/C++/Fortran code, it can also be of great value 
when programming in dynamic languages such as Perl (indeed I used [OpenMP](https://www.openmp.org/) to add parallelization to 
some bioinformatics codes as I am discussing in this [preprint](https://arxiv.org/pdf/2406.10271)). OpenMP's runtime environment provides a way to control many aspects of 
parallelization from the command-line. The OpenMP Perl application programmer can use Brett's excellent [OpenMP::Environment](https://metacpan.org/pod/OpenMP::Environment) 
module to set many of these environmental variables. However, these changes must make it to the C(C++/Fortran) side to affect the OpenMP code ; in the absence of an explicit 
readout of the environment from the C code, one's parallel Perl/C code will only use the environment as it existed when the Perl application was fired, not the environment as
modified **by** the Perl application. 

This blog post will show a simple OpenMP Perl/C program that is responsive to changes of the environment (scheduling of threads, number of threads) from **within** the Perl part
of the application. It makes use of the [Alien::OpenMP](https://metacpan.org/pod/Alien::OpenMP) and [OpenMP::Environment](https://metacpan.org/pod/OpenMP::Environment) to set up 
the compilation flags and the environment respectively. The code itself is very simple:
1. It sets up the OpenMP environment from within Perl
2. It jumps to C universe to set and print these variables and return their values back to Perl for printing and verification
3. It runs a simple parallel loop to show that the numbers of threads were run correctly.

So here is the **Perl** part:
```perl
use v5.38;
use Alien::OpenMP;
use OpenMP::Environment;
use Inline (
    C    => 'DATA',
    with => qw/Alien::OpenMP/,
);

my $env = OpenMP::Environment->new();
$env->omp_num_threads(5);

say $env->omp_schedule("static,1");
my $num_of_OMP_threads = get_thread_num();
printf "Number of threads = %d from within Perl\n",$num_of_OMP_threads;
my $schedule_in_C = get_schedule_in_C();
say "Schedule from Perl: $schedule_in_C";

test_omp();
__DATA__
__C__
```

And here is the **C** part (which would ordinarly follow the \_\_C\_\_ token (but we parsed it out to highlight the C syntax).

```C
#include <omp.h>
#include <stdlib.h>
#include <string.h>

void set_openmp_schedule_from_env() {
    char *schedule_env = getenv("OMP_SCHEDULE");
    if (schedule_env != NULL) {
        char *kind_str = strtok(schedule_env, ",");
        char *chunk_size_str = strtok(NULL, ",");

        omp_sched_t kind;
        if (strcmp(kind_str, "static") == 0) {
            kind = omp_sched_static;
        } else if (strcmp(kind_str, "dynamic") == 0) {
            kind = omp_sched_dynamic;
        } else if (strcmp(kind_str, "guided") == 0) {
            kind = omp_sched_guided;
        } else {
            kind = omp_sched_auto;
        }
        int chunk_size = atoi(chunk_size_str);
        omp_set_schedule(kind, chunk_size);
    }
}

char * get_schedule_in_C() {
    int chunk_size;
    omp_sched_t kind;

    set_openmp_schedule_from_env();
    omp_get_schedule(&kind, &chunk_size);
    printf("Schedule in C: %x\n",kind);
    printf("Schedule from env %s\n",getenv("OMP_SCHEDULE"));
    return getenv("OMP_SCHEDULE");

}

int get_thread_num() {
    int num_threads = atoi(getenv("OMP_NUM_THREADS"));
    omp_set_num_threads(num_threads);
    printf("Number of threads = %d from within C\n",num_threads);
    return num_threads;
}

int test_omp() {
    int n = 0;
    #pragma omp parallel for schedule(runtime)
    for (int i = 0; i < omp_get_num_threads(); i++) {
        int chunk_size;
        omp_sched_t kind;
        omp_get_schedule(&kind, &chunk_size);
        printf("Schedule in C: %x as seen from thread %d\n",kind,omp_get_thread_num());
    }
    return n;
}
```
The Perl and C parts use functions from OpenMP::Environment and omp.h to get/set the environment. The names are rather self explanatory, so will save everyone space and not repeat them. 
The general idea is that one **sets** those variables in Perl and then heads over to C, to **get** the variable names and **set them again** in C. To make a parallel loop responsive to 
these changes, one would also like to set the schedule to **runtime**, rather than the default value, using the appropriate clause as in:
```c

    #pragma omp parallel for schedule(runtime)

```

The output of the code is : 
```
static,1
Number of threads = 5 from within C
Number of threads = 5 from within Perl
Schedule in C: 1
Schedule from env static
Schedule from Perl: static
Schedule in C: 1 as seen from thread 0
Schedule in C: 1 as seen from thread 3
Schedule in C: 1 as seen from thread 1
Schedule in C: 1 as seen from thread 4
```
confirming that we used 5 threads and static scheduling (the numerical code for the static scheduling is 1, but in any case we also read the value as text in C and sent it back to Perl).
Since one of the main reasons to use Perl with OpenMP is to benchmark different runtime combinations for speed, I use the clause **schedule(runtime)** in all my Perl/OpenMP codes!
Hopefully these snippets can be of use to those experimenting with OpenMP applications in Perl!
