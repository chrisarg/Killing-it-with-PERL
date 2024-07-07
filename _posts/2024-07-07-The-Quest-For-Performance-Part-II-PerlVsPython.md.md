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
And here are our contestants: 
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

def compute_inplace_with_joblib(chunk):
    return np.cos(np.sin(np.sqrt(chunk))) #parallel function for joblib

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
The numba is surprisingly slower!? Could it be due to the overhead of compilation as pointed out by [mohawk2](https://github.com/mohawk2) in an IRC exchange about this [issue](https://towardsdatascience.com/why-numba-sometime-way-slower-than-numpy-15d077390287)?
To test this, we should call compute_inplace_with_numba once **before** we execute the benchmark. Doing so, shows that Numba is now faster than Numpy.

```text
In place in (  base Python): 11.89 seconds
In place in (Python Joblib): 4.42 seconds
In place in ( Python Numpy): 0.93 seconds
In place in ( Python Numba): 0.49 seconds
```
Finally, I decided to take base R for ride in the same example:
```r
n<-50000000
x<-runif(n)
start_time <- Sys.time()
result <- cos(sin(sqrt(x)))
end_time <- Sys.time()

# Calculate the time taken
time_taken <- end_time - start_time

# Print the time taken
print(sprintf("Time in base R: %.2f seconds", time_taken))
```
which yielded the following timing result:

```text
Time in base R: 1.30 seconds
```
Compared to the [Perl results](https://chrisarg.github.io/Killing-It-with-PERL/2024/07/06/The-Quest-For-Performance-Part-I-InlineC-OpenMP-PDL.html) we note the following about this example:
* Inplace operations in base Python were ~ 3.5 **slower** than Perl
* Single threaded PDL and numpy gave nearly identical results, followed closely by base R
* Failure to account for the compilation overhead of Numba yields the **false** impression that it is slower than Numpy. When accounting for the compilation overhead, Numba is x2 faster than Numpy
* Parallelization with Joblib did improve upon base Python, but was still inferior to the single thread Perl implementation
* Multi-threaded PDL (and OpenMP) crashed every other implementation in all languages

I hope this post and the [Perl one](https://chrisarg.github.io/Killing-It-with-PERL/2024/07/06/The-Quest-For-Performance-Part-I-InlineC-OpenMP-PDL.html), provide some food for thought about
the language to use for your next data/compute intensive operation. 
The next part in this series will look into the same example using arrays in C. This final installment will (hopefully) provide some insights about the impact of memory locality and the overhead incurred by using dynamically typed languages. 
