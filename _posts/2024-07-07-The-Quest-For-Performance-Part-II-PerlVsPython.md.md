---
title: " The Quest for Performance Part II : Perl vs Python "
date: 2024-07-07
---

Having run this toy performance example, we will now digress somewhat and contrast the performance against 
a few Python implementations. First let's set up the stage for the calculations, and provide commandline 
capabilities to the Python script. 
```python
import argparse
import time
import math
import numpy as np
import os
from numba import njit
from joblib import Parallel, delayed

parser = argparse.ArgumentParser()
parser.add_argument("--workers", type=int, default=8)
parser.add_argument("--arraysize", type=int, default=100_000_000)
args = parser.parse_args()
# Set the number of threads to 1 for different libraries
print("=" * 80)
print(
    f"\nStarting the benchmark for {args.arraysize} elements "
    f"using {args.workers} threads/workers\n"
)

# Generate the data structures for the benchmark
array0 = [np.random.rand() for _ in range(args.arraysize)]
array1 = array0.copy()
array2 = array0.copy()
array_in_np = np.array(array1)
array_in_np_copy = array_in_np.copy()
```
And here are our contenstants: 
* Base Python
  ```python
  for i in range(len(array0)):
    array0[i] = math.cos(math.sin(math.sqrt(array0[i])))
  ```
* Numpy (Single threaded)
```python
np.sqrt(array_in_np, out=array_in_np)
np.sin(array_in_np, out=array_in_np)
np.cos(array_in_np, out=array_in_np)
```
* Joblib (note that this example is not a true in-place one, but I have not been able to make it run using the out arguments)
```python
#parallel function for joblib
def compute_inplace_with_joblib(chunk):
    return np.cos(np.sin(np.sqrt(chunk)))

chunks = np.array_split(array1, args.workers)  # Split the array into chunks
numresults = Parallel(n_jobs=args.workers)(
        delayed(compute_inplace_with_joblib)(chunk) for chunk in chunks
    )# Process each chunk in a separate thread
array1 = np.concatenate(numresults)  # Concatenate the results
```
* Numba
```python
@njit
def compute_inplace_with_numba(array):
    np.sqrt(array,array)
    np.sin(array,array)
    np.cos(array,array)
    ## njit will compile this function to machine code
compute_inplace_with_numba(array_in_np_copy)
```

And here are the timing results:
```text
In place in (  base Python): 11.42 seconds
In place in (Python Joblib): 4.59 seconds
In place in ( Python Numba): 2.62 seconds
In place in ( Python Numpy): 0.92 seconds
```

Compared to the [Perl results](https://chrisarg.github.io/Killing-It-with-PERL/2024/07/06/The-Quest-For-Performance-Part-I-InlineC-OpenMP-PDL.html) we note the following about this example:
* Inplace operations in base Python were ~ 3.5 **slower** than Perl
* Single threaded PDL and numpy gave nearly identical results
* Compilation using Numba under-delivered relative to Numpy (still scratching my head about this one)
* Parallelization with Joblib did improve upon base Python, but was still inferior to the single thread Perl implementation
* Multi-threaded PDL (and OpenMP) crashed every other implementation in either language

I hope this post and the [Perl one](https://chrisarg.github.io/Killing-It-with-PERL/2024/07/06/The-Quest-For-Performance-Part-I-InlineC-OpenMP-PDL.html), provide some food for thought about
the language to use for your next data/compute intensive operation. 
