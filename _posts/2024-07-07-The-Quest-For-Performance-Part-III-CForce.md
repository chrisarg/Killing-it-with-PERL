---
title: " The Quest for Performance Part III : C Force "
date: 2024-07-07
---
In the two prior installments of this series, we considered the performance of floating operations  in [Perl](https://chrisarg.github.io/Killing-It-with-PERL/2024/07/06/The-Quest-For-Performance-Part-I-InlineC-OpenMP-PDL.html), 
[Python and R](https://chrisarg.github.io/Killing-It-with-PERL/2024/07/07/The-Quest-For-Performance-Part-II-PerlVsPython.md.html) in a toy example that computed the function _cos(sin(sqrt(x)))_, where x was a **very large** array of 50M double precision floating numbers. 
Hybrid implementations that delegated the arithmetic intensive part to C were among the most performant implementations. In this installment, we will digress slightly and look at the performance of a pure C code implementation of the toy example. 
The C code will provide further insights about the importance of memory locality for performance (by default elements in a C array are stored in sequential addresses in memory, and numerical APIs such as PDL or numpy interface with such containers) vis-a-vis containers, 
e.g. Perl arrays which do not store their values in sequential addresses in memory. Last, but certainly not least, the C code implementations will allow us to assess whether flags related to floating point operations for the low level compiler (in this case gcc) can affect performance. 
This point is worth emphasizing: common mortals are entirely dependent on the choice of compiler flags when "piping" their "install" or building their Inline file. If one does not touch these flags, then one will be blissfully unaware of what they may missing, or pitfalls they may be avoiding.
The humble C file makefile allows one to make such performance evaluations explicitly. 

The C code for our toy example is listed in its entirety below. The code is rather self-explanatory, so will not spend time explaining other than pointing out that it contains four functions for
* Non-sequential calculation of the expensive function : all three floating pointing operations take place inside a single loop using one thread
* Sequential calculations of the expensive function : each of the 3 floating point function evaluations takes inside a separate loop using one thread
* Non-sequential OpenMP code : threaded version of the non-sequenctial code
* Sequential OpenMP code: threaded of the sequential code

In this case, one may hope that the compiler is smart enough to recognize that the square root maps to packed (vectorized) floating pointing operations in assembly, so that one function can be vectorized using the appropriate SIMD instructions (note we did not use the simd program for the OpenMP codes). 
Perhaps the speedup from the vectorization may offest the loss of performance from repeatedly accessing the same memory locations (or not).  
```c

#include <stdlib.h>
#include <string.h>
#include <math.h>
#include <stdio.h>
#include <omp.h>

// simulates a large array of random numbers
double*  simulate_array(int num_of_elements,int seed);
// OMP environment functions
void _set_openmp_schedule_from_env();
void _set_num_threads_from_env();



// functions to modify C arrays 
void map_c_array(double* array, int len);
void map_c_array_sequential(double* array, int len);
void map_C_array_using_OMP(double* array, int len);
void map_C_array_sequential_using_OMP(double* array, int len);

int main(int argc, char *argv[]) {
    if (argc != 2) {
        printf("Usage: %s <array_size>\n", argv[0]);
        return 1;
    }

    int array_size = atoi(argv[1]);
    // printf the array size
    printf("Array size: %d\n", array_size);
    double *array = simulate_array(array_size, 1234);

    // Set OMP environment
    _set_openmp_schedule_from_env();
    _set_num_threads_from_env();

    // Perform calculations and collect timing data
    double start_time, end_time, elapsed_time;
    // Non-Sequential calculation
    start_time = omp_get_wtime();
    map_c_array(array, array_size);
    end_time = omp_get_wtime();
    elapsed_time = end_time - start_time;
    printf("Non-sequential calculation time: %f seconds\n", elapsed_time);
    free(array);

    // Sequential calculation
    array = simulate_array(array_size, 1234);
    start_time = omp_get_wtime();
    map_c_array_sequential(array, array_size);
    end_time = omp_get_wtime();
    elapsed_time = end_time - start_time;
    printf("Sequential calculation time: %f seconds\n", elapsed_time);
    free(array);

    array = simulate_array(array_size, 1234);
    // Parallel calculation using OMP
    start_time = omp_get_wtime();
    map_C_array_using_OMP(array, array_size);
    end_time = omp_get_wtime();
    elapsed_time = end_time - start_time;
    printf("Parallel calculation using OMP time: %f seconds\n", elapsed_time);
    free(array);

    // Sequential calculation using OMP
    array = simulate_array(array_size, 1234);
    start_time = omp_get_wtime();
    map_C_array_sequential_using_OMP(array, array_size);
    end_time = omp_get_wtime();
    elapsed_time = end_time - start_time;
    printf("Sequential calculation using OMP time: %f seconds\n", elapsed_time);

    free(array);
    return 0;
}



/*
*******************************************************************************
* OMP environment functions
*******************************************************************************
*/
void _set_openmp_schedule_from_env() {
  char *schedule_env = getenv("OMP_SCHEDULE");
  printf("Schedule from env %s\n", getenv("OMP_SCHEDULE"));
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

void _set_num_threads_from_env() {
  char *num;
  num = getenv("OMP_NUM_THREADS");
  printf("Number of threads = %s from within C\n", num);
  omp_set_num_threads(atoi(num));
}
/*
*******************************************************************************
* Functions that modify C arrays whose address is passed from Perl in C
*******************************************************************************
*/

double*  simulate_array(int num_of_elements, int seed) {
  srand(seed); // Seed the random number generator
  double *array = (double *)malloc(num_of_elements * sizeof(double));
  for (int i = 0; i < num_of_elements; i++) {
    array[i] =
        (double)rand() / RAND_MAX; // Generate a random double between 0 and 1
  }
  return array;
}

void map_c_array(double *array, int len) {
  for (int i = 0; i < len; i++) {
    array[i] = cos(sin(sqrt(array[i])));
  }
}

void map_c_array_sequential(double* array, int len) {
  for (int i = 0; i < len; i++) {
    array[i] = sqrt(array[i]);
  }
  for (int i = 0; i < len; i++) {
    array[i] = sin(array[i]);
  }
  for (int i = 0; i < len; i++) {
    array[i] = cos(array[i]);
  }
}

void map_C_array_using_OMP(double* array, int len) {
#pragma omp parallel
  {
#pragma omp for schedule(runtime) nowait
    for (int i = 0; i < len; i++) {
      array[i] = cos(sin(sqrt(array[i])));
    }
  }
}

void map_C_array_sequential_using_OMP(double* array, int len) {
#pragma omp parallel
  {
#pragma omp for schedule(runtime) nowait
    for (int i = 0; i < len; i++) {
      array[i] = sqrt(array[i]);
    }
#pragma omp for schedule(runtime) nowait
    for (int i = 0; i < len; i++) {
      array[i] = sin(array[i]);
    }
#pragma omp for schedule(runtime) nowait
    for (int i = 0; i < len; i++) {
      array[i] = cos(array[i]);
    }
  }
}

```
A critical question is whether the use of fast floating compiler flags, [a trick that trades speed for accuracy of the code](https://simonbyrne.github.io/notes/fastmath/), can affect performance. 
Here is the makefile withut this compiler flag
```make
CC = gcc
CFLAGS = -O3 -ftree-vectorize  -march=native  -Wall -std=gnu11 -fopenmp -fstrict-aliasing 
LDFLAGS = -fPIE -fopenmp
LIBS =  -lm

SOURCES = inplace_array_mod_with_OpenMP.c
OBJECTS = $(SOURCES:.c=_noffmath_gcc.o)
EXECUTABLE = inplace_array_mod_with_OpenMP_noffmath_gcc

all: $(SOURCES) $(EXECUTABLE)

clean:
	rm -f $(OBJECTS) $(EXECUTABLE)

$(EXECUTABLE): $(OBJECTS)
	$(CC) $(LDFLAGS) $(OBJECTS) $(LIBS) -o $@

%_noffmath_gcc.o : %.c 
	$(CC) $(CFLAGS) -c $< -o $@
```
and here is the one with this flag:

```make
CC = gcc
CFLAGS = -O3 -ftree-vectorize  -march=native -Wall -std=gnu11 -fopenmp -fstrict-aliasing -ffast-math
LDFLAGS = -fPIE -fopenmp
LIBS =  -lm

SOURCES = inplace_array_mod_with_OpenMP.c
OBJECTS = $(SOURCES:.c=_gcc.o)
EXECUTABLE = inplace_array_mod_with_OpenMP_gcc

all: $(SOURCES) $(EXECUTABLE)

clean:
	rm -f $(OBJECTS) $(EXECUTABLE)

$(EXECUTABLE): $(OBJECTS)
	$(CC) $(LDFLAGS) $(OBJECTS) $(LIBS) -o $@

%_gcc.o : %.c 
	$(CC) $(CFLAGS) -c $< -o $@
```

And here are the results of running these two programs
* **Without -ffast-math**
```text
OMP_SCHEDULE=guided,1 OMP_NUM_THREADS=8 ./inplace_array_mod_with_OpenMP_noffmath_gcc 50000000
Array size: 50000000
Schedule from env guided,1
Number of threads = 8 from within C
Non-sequential calculation time: 1.12 seconds
Sequential calculation time: 0.95 seconds
Parallel calculation using OMP time: 0.17 seconds
Sequential calculation using OMP time: 0.15 seconds
```
* **With -ffast-math**
```text
OMP_SCHEDULE=guided,1 OMP_NUM_THREADS=8 ./inplace_array_mod_with_OpenMP_gcc 50000000
Array size: 50000000
Schedule from env guided,1
Number of threads = 8 from within C
Non-sequential calculation time: 0.27 seconds
Sequential calculation time: 0.28 seconds
Parallel calculation using OMP time: 0.05 seconds
Sequential calculation using OMP time: 0.06 seconds
```
Note that one can use the fastmath in Numba code as follows (the default is fastmath=False):
```python
@njit(nogil=True,fastmath=True)
def compute_inplace_with_numba(array):
    np.sqrt(array,array)
    np.sin(array,array)
    np.cos(array,array)
```
A few points that are worth noting:
* The -ffast-math gives major boost in performance (about 300% for both the single threaded and the multi-threaded code), but it can generate erroneous results
* Fastmath also works in Numba, but should be avoided for the same reasons it should be avoided in any application that strives for accuracy
* The sequential C single threaded code gives performance similar to the single threaded PDL and Numpy
* Somewhat surprisingly, the sequential code is about 20% faster than the non-sequential code when the correct (non-fast) math is used. 
* Unsurprisingly, multi-threaded code is faster than single threaded code :)
* I still cannot explain how numbas delivers a 50% performance premium over the C code of this rather simple function. 
