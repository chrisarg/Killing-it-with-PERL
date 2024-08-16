---
title: "The Day Perl Stood Still: Unveiling A Hidden Power Over C"
date: 2024-08-15
---
Sometimes the unexpected happens and must be shared with the world ... this one is such a case.

Recently, I've started experimenting with Perl for workflow management and high-level supervision of low level code for data science applications. A role I'd reserve for Perl in this context is that of lifecycle management of memory buffers, using the Perl application to "allocate" memory buffers and shuttle it between computing components written in C, Assembly, Fortran and the best hidden gem of the Perl world, the [Perl Data Language](https://metacpan.org/pod/PDL). 
There at least 3 ways that Perl can be used to allocate memory buffers:

1. Generate a list of bytes and use the _pack_ function to convert them to a string. 
2. Use the repetition operator (_x_) to generate a string of length equal to the size of the buffer one wants **minus one** (strings are terminated via null bytes in Perl, thus the null byte compensates for the **mine one**).
3. Access an external memory allocator library either through [Inline](https://metacpan.org/dist/Inline/view/lib/Inline.pod) or [FFI::Platypus](https://metacpan.org/pod/FFI::Platypus) to allocate the buffer.

The following Perl code implements these three methods (**pack** , **string** and malloc in **C**)  and allows one to experiment with different buffer sizes, initial values and precision of the results (by averaging over many iterations of the allocation routines)

```perl
#!/home/chrisarg/perl5/perlbrew/perls/current/bin/perl
use v5.38;
use Inline (
    C         => 'DATA',
    cc        => 'g++',
    ld        => 'g++',
    inc       => q{},      # replace q{} with anything else you need
    ccflagsex => q{},      # replace q{} with anything else you need
    lddlflags => join(
        q{ },
        $Config::Config{lddlflags},
        q{ },              # replace q{ } with anything else you need
    ),
    libs => join(
        q{ },
        $Config::Config{libs},
        q{ },              # replace q{ } with anything else you need
    ),
    myextlib => ''
);

use Benchmark qw(cmpthese);
use Getopt::Long;
my ($buffer_size, $init_value, $iterations);
GetOptions(
    'buffer_size=i' => \$buffer_size,
    'init_value=s'  => \$init_value,
    'iterations=i'  => \$iterations,
) or die "Usage: $0 --buffer_size <size> --init_value <value> --iterations <count>\n";
my $init_value_byte = ord($init_value);
my %code_snippets   = (
    'string' => sub {
        $init_value x ( $buffer_size - 1 );
    },
    'pack' => sub {
        pack "C*", ( ($init_value_byte) x $buffer_size );
    },
    'C' => sub {
        allocate_and_initialize_array( $buffer_size, $init_value_byte );
    },
);

cmpthese( $iterations, \%code_snippets );

__DATA__

__C__

#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>

SV* allocate_and_initialize_array(size_t length, short initial_value) {
    // Allocate memory for the array
    char* array = (char*)malloc(length * sizeof(char));
    char initial_value_byte = (char)initial_value;
    if (array == NULL) {
        fprintf(stderr, "Memory allocation failed\n");
        exit(1);
    }

    // Initialize each element with the initial_value
    memset(array, initial_value_byte, length);
    return newSVuv(PTR2UV(array));
}

```
calling the script as:
```bash
./time_mem_alloc.pl -buffer_size=1000000 -init_value=A -iterations=20000
```
yielded the surprising result : 
```text
          Rate   pack      C string
pack     322/s     --   -92%   -99%
C       4008/s  1144%     --   -92%
string 50000/s 15417%  1147%     --
```
with the **Perl _string_ method outperforming C by 10 fold**. 

Not believing the massive performance gain, and thinking I am dealing with a bug in _Inline::C_,  I recoded the allocation in pure C (adding the usual embellishments for commandline processing/timing etc) :
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

char* allocate_and_initialize_array(size_t length, char initial_value) {
    // Allocate memory for the array
    char* array = (char*)malloc(length * sizeof(char));
    if (array == NULL) {
        fprintf(stderr, "Memory allocation failed\n");
        exit(1);
    }

    // Initialize each element with the initial_value
    memset(array, initial_value, length);

    return array;
}

double time_allocation_and_initialization(size_t length, char initial_value) {
    clock_t start, end;
    double cpu_time_used;

    start = clock();
    char* array = allocate_and_initialize_array(length, initial_value);
    end = clock();

    cpu_time_used = ((double) (end - start)) / CLOCKS_PER_SEC;
    /* This rudimentary loop prevents the compiler from optimizing out the 
     * allocation/initialization with the de-allocation
    */
    for(size_t i = 1; i < length; i++) {
       array[i]++;
       if(i % 100000 == 0) {
           printf("array[%zu] = %c\n", i, array[i]);
       }
    }
    free(array); // Free the allocated memory

    return cpu_time_used;
}

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(stderr, "Usage: %s <length> <initial_value>\n", argv[0]);
        return 1;
    }

    size_t length = strtoull(argv[1], NULL, 10);
    char initial_value = argv[2][0];

    double time_taken = time_allocation_and_initialization(length, initial_value);
    printf("Time taken to allocate and initialize array: %f seconds\n", time_taken);
    printf("Initializes per second: %f\n", 1/time_taken);

    return 0;
}

/*
Compilation command:
gcc -O2 -o time_array_allocation time_array_allocation.c -std=c99

Example invocation:
./time_array_allocation 10000000 A
*/
```
Invoking the C program as the comment in the C code says to do, 
I obtained the following result:
```text
Time taken to allocate and initialize array: 0.000203 seconds
Initializes per second: 4926.108374
```
which is practically executes in the same order of magnitude as the equivalent allocation as the _Inline::C_ malloc/C approach. 
After researching the issue further, I discovered that the malloc I grew to admire trades speed in memory allocation for generality, and there is a plethora of faster memory allocators out there. It seems that Perl is using one such allocator for its strings, and kicks C's butt in this task of allocating buffers. 
