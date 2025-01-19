---
title: "Timing Peak DRAM Use in R With Perl"
date: 2025-01-18
---

Another year, another opportunity for Perl to excel as a system's language. Today I decided to take Perl for a (?)wild ride and use it to monitor peak physical DRAM use in a R script.
Due to the multi-language nature of the post, there will be **a lot** of R code in the body; however, the code is self-explanatory, and should not be difficult to understand (the same applies to the Perl code for those coming from a R background).
First, a little bit of background about R's memory management and the tools that one can use _within_ R to monitor how the process is managing memory. R similar to Perl (?)frees the programmer from having to manage memory manually by providing dynamically allocated containers. R features a garbage collector, which similar to Perl's uses a reference counting mechanism to return memory back to the operating system. 
Managing memory in R is as critical as managing memory in Perl, and there are tools available that are built-in the language (the [Names and Values](https://adv-r.hadley.nz/names-values.html) in the 2nd edition of the book "Advanced R" is a valuable introduction to memory management, while the Chapter [Memory](http://adv-r.had.co.nz/memory.html) in the first edition of that book is also a useful read).
The basic tool used to profile R code is the builtin function [Rprof](https://www.rdocumentation.org/packages/utils/versions/3.6.2/topics/Rprof) that samples the call stack and **writes it out** to a log-file that can be subsequently parsed. 
In the documentation of Rprof, one finds the following disclaimer:
```text
     Note that the (timing) interval cannot be too small.  With
     ‘"cpu"’, the time spent in each profiling step is currently added
     to the interval.  With all profiling events, the computation in
     each profiling step causes perturbation to the observed system and
     biases the results.  What is feasible is machine-dependent.  On
     Linux, R requires the interval to be at least 10ms, on all other
     platforms at least 1ms.  Shorter intervals will be rounded up with
     a warning.
```
The relative slow sampling frequencing implies that one must somehow slow the code down to capture peak memory usage. One solution for those of us in Linux systems is to take the hint and release the [valgrid](https://valgrind.org/) Kraken as detailed in [Profiling R code for memory use](https://github.com/chrisarg/Killing-It-with-PERL/new/main/_posts);
this will **really slow the code down** and then we can capture stuff (note that taking the hint, also applies to Perl, i.e. see [Test::Valgrind](https://metacpan.org/pod/Test::Valgrind) in MetaCPAN).
But ultimately, we would like not to slow the code that much, especially if we are also trying to obtain performance information at the same time as we profile the memory.

Assuming that one does not want to use an interactive tool to profile the code (the excellent [profvis](https://profvis.r-lib.org/) comes to mind), then one is stuck with the low-level option provided by Rprof logging and summaryRprof parsing. 
This leads us to the next question: what is the overhead by these tools? Overhead is paid in both hard disk space and execution time. Let'talk about space first: for a high resolution logging (e.g. sampling every 10msec), the log file will grow by ~ 10kb per second of calculation.
This may not seem much, but profiling an involved calculation quickly adds up: **to profile an expression that takes 10shr to execute will consume 10 x 3600 x 10 = 360000 KB ~ 360MB** (incidentally ~10hr is the [longest calculation I have ever done in R](https://pmc.ncbi.nlm.nih.gov/articles/PMC8310602/)).To give a sense of measure, the total size of the files in my /var/log is ~1.1GB. And yes, while hard-disks are getting bigger, this is no excuse to fill them with log files!
To give an idea of the time overhead, we will need to provide an alternative to hard-disk logging (hint: this will be a Perl program!), but first let's provide a straightforward R implementation of a logging function.

```R
bench_time_mem<-function(x) {  ## x is the R expression to profile
  gc(reset=FALSE,verbose=FALSE)  ## force a gc here
  profing_fname <- tempfile() ## create a temporary file
  Rprof(filename=profing_fname, memory.profiling=TRUE,interval=0.01)
  time<-system.time(x,gcFirst = FALSE)
  Rprof(NULL) ## stop profiling and logging
  memprof<-summaryRprof(profing_fname, memory="tseries",lines="hide",diff=TRUE)
  mem_alloc<-sum(memprof[-1,"vsize.large"]) ## get memory allocated via malloc
  time_prof<-summaryRprof(profing_fname, memory="none",lines="hide",diff=TRUE)
  times<-time_prof$by.total[2,-2]
  retval<-unlist(c(times[1,1],mem_alloc,file.info(profing_fname)$size))
  unlink(profing_fname) ## delete the logfiles
  names(retval)<-c("total_time","R_gc_alloc","logfile_size")
  retval
}
```
The code should be self-explanatory even for non R users : it first triggers the garbage collector, and then in sequence creates a temporary file, starts logging to that file, execute the R expression (R code within brackets, similar to how one would provide an anonymous code reference in Perl), then parses the file to get memory usage and timing), and returns the total time, the size of the memory allocated by R in the stack, when running the expression and the size of the log file.

Here is a minimally working example:

```R
## function for busy waiting for a fixed number of seconds
busy_wait <- function(seconds) {
  start_time <- Sys.time()
  stop_time <- Sys.time()
  while (difftime(stop_time, start_time, units = "secs") < seconds) {
    stop_time <- Sys.time()
  }
}

N<-100000
work_time <- 1 # second(s)

val<-bench_time_mem(
	{  	
		busy_wait(work_time);  ## work without allocation
		cat("\nAllocating\n")
		q<-rnorm(N);            ## Gaussian random variables
		busy_wait(work_time);   ## work without allocation
		rm(q);gc(reset=F)       ## free memory/trigger the gc
		q2<-rnorm(N);           ## another allocation
		busy_wait(work_time);   ## busy working!
		rm(q2);gc(reset=F)      ## free the memory yet again

	}
)
```
In this example, we first provide an implementation of a function `busy_wait` that simulates a work load without further allocation. 
The actual code to be profiled, alternates periods of busy waiting with allocation and de-allocation. With the values of `N` and `work_time` in the code snippet, I obtain the following profile result:
```text
> val
  total_time   R_gc_alloc logfile_size 
         3.4    1600000.0      32446.0 
```
Let's talk about these figures: first note that parsing the Rprof, does not allow one direct access to the peak memory used during the calculation 
(only the **total amount of memory** allocated in the stack. 
**While this information is crucial for performance, i.e. one pays a price for every byte moved around, it is not very informative about whether t
he program will even run in the machine, as the datasets scale: R processes data in memory, and when the _physical_ memry is exhausted, 
the operating system will put the R process out of its misery.**
In the example above, the peak memory usage while the expression executes is N*8 (since rnorm returns a double precision floating number in C parlance), 
but since two allocations were done, the profiler cn only report their sum. In fact, the output is virtually indistinguishable from the following code which allocates
an array of 2 x N doubles in a single go.

```R
valcp<-bench_time_mem(
	{  
		busy_wait(work_time);
		cat("\nAllocating\n")
		q<-rnorm(N*2);
		rm(q);gc(reset=F)
		busy_wait(work_time*2);
	}
)

```

and 

```text
valcp
  total_time   R_gc_alloc logfile_size 
         3.2    1600000.0      30773.0 
```
Yet while both codes allocated the same total amount of memory, the first code would work if one allocated the entirety of the free memory in a given machine, 
while the second would croak when roughly half the free memory was requested.

The difference of ~0.2sec execution time, is the price one has to pay for triggering the garbage collector. The following shows that while allocating 
an array of 10^5 doubles and filling it with random numbers takes 6msec in my machine, triggering the gaarbage collector is 32x slower.
```text
> system.time(rnorm(N))
   user  system elapsed 
  0.006   0.000   0.006


> system.time({q<-rnorm(N);rm(q);gc(reset=FALSE)})
   user  system elapsed 
  0.193   0.000   0.194 
```

_So how can one get the peak DRAM usage without logging anything to the hard disk_? 
The answer, which will be provided in Part 2, blends together R and Perl and is conceptually 
very simple, design-wise:
* write a Perl script that probes the `ps` command line utility for resident set size (RSS) i.e. the footprint of the R process in DRAM.
* put the probing of the RSS in a monitoring loop, so that awakens periodically to sample the size of the RSS in _realtime_ and compares
* it to that at the beginning of the monitoring phase
* start the Perl monitoring script as sub-process of the R process, provide it with the Process ID (PID) of the R process and put it in the background
* when the R expression concludes, _kill the Perl process_ and report the maximum difference of the RSS from the baseline; this gives the peak (over the baseline) footprint of the program in DRAM.

(to be continued...)
